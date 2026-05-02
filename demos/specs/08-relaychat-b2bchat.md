# RelayChat — B2B Chat for Teams That Ship

> A Slack-clone for SMBs, built in Phoenix. Realtime channels, DMs, file shares, slash-command integrations, presence — and a backend that hides every authz gap an Elixir shop ever shipped to staging.

---

## Premise & Vibe

RelayChat is the kind of product a 12-person engineering team ships in a quarter when they want to "own the chat layer" instead of paying Slack $14/user/month. It looks legitimate: workspaces, channels, threaded messages, DMs, presence dots, file uploads, `/giphy`-style slash commands, and a small marketplace of webhook integrations. The marketing site says "End-to-end audited" (it isn't). The Helm chart says `replicas: 3` (one is mis-tagged). The CTO believes Phoenix Channels are "secure by default because BEAM."

The vibe is competent-but-rushed. The code is mostly idiomatic Phoenix 1.7. The bugs are the kind that ship when a team treats `socket.assigns.current_user` as a load-bearing structural beam and forgets that channel topics are strings clients can type. This is a deliberately serious demo: no toy SQLi, no `eval(req.body)`. The vulnerabilities mirror real disclosures in Mattermost, Rocket.Chat, Zulip, and Slack itself.

The demo's secondary purpose is to humble the assumption that "novel stack = safe stack." Phoenix is rare in security demos. Apex has to crawl an authenticated WebSocket, fuzz channel topics, reason about LiveView event handlers, and tell a planted herring (looks-bad-isn't) apart from a real bug.

---

## Why This Stack

Phoenix is chosen for three reasons:

1. **Credibility through novelty.** Most pentest demos are Express, Rails, or Django. A working Phoenix target signals that Apex can read AST it has never seen at scale — `handle_event/3`, `handle_info/2`, `Ecto.Changeset.cast/4`, `Phoenix.Channel.intercept/1`. This is the same muscle Apex needs for Go, Rust, and Crystal codebases.
2. **Realtime is where authz dies.** Slack-clone vulnerabilities cluster around channel join, message broadcast, and presence diff events. Phoenix Channels expose `join/3`, `handle_in/3`, `handle_out/3` as separate authz boundaries — perfect for showing how the judge agent reasons about *which* boundary is the actual gate.
3. **LiveView is a new attack surface.** `phx-click="delete_message"` looks like a button. It's actually an authenticated RPC. The handler is an Elixir function pattern-matching on `params`. Apex's symbolic crawler has to learn that `phx-value-id` is user-controlled input, not a CSS attribute.

PostgreSQL + Ecto is the obvious DB choice. Tigris (S3-compatible object storage out of Fly.io) is used for file uploads to demonstrate cross-cloud presigned-URL bugs without the demo requiring AWS credentials. Guardian provides JWT for the mobile/desktop client and webhooks; the LiveView side uses signed-session cookies.

---

## Stack Details

| Layer | Choice | Version | Notes |
|---|---|---|---|
| Language | Elixir | 1.17.2 | OTP 27 |
| Web framework | Phoenix | 1.7.14 | LiveView 1.0 |
| ORM | Ecto | 3.12 | Postgres adapter |
| Realtime | Phoenix.Channels + Phoenix.PubSub | 1.7 | PG2 adapter |
| Presence | Phoenix.Presence | 1.7 | CRDT-based |
| DB | PostgreSQL | 16 | `pgcrypto`, `citext` ext |
| Object store | Tigris (S3-compatible) | — | `ex_aws_s3` 2.5 |
| Auth (JWT) | Guardian | 2.3 | HS256, see vuln V6 |
| Auth (browser) | `Plug.Session` cookie | — | signing salt static |
| Background jobs | Oban | 2.18 | for webhooks, file scans |
| Frontend | LiveView + Tailwind + AlpineJS | — | minimal JS |
| Search | `pg_trgm` + tsvector | — | no Elastic |
| Slash command sandbox | `:erl_eval` (V4 herring), `Code.eval_string` (V4 real) | — | see vuln section |
| Webhook signing | `:crypto.mac/4` HMAC-SHA256 | — | constant-time compared (V11 herring) |
| Container | `hexpm/elixir:1.17.2-erlang-27.0-alpine-3.20` | — | distillery release |
| Egress for integrations | `Finch` HTTP client | 0.18 | SSRF in V12 |

The repo is a single-app umbrella (no umbrella project — just `mix phx.new --live`). One thing the engineer should resist: do not break this into umbrella apps. The real Apex value comes from cross-context analysis (a `MessagesController` calling into `Workspaces` calling into `Storage`). Umbrella'ing hides those edges.

---

## Architecture

```
                   +------------------+
   Browser <--->   | Phoenix Endpoint |
   (LiveView,      |  - LiveSocket    |
    UserSocket)    |  - UserSocket    |
                   +---------+--------+
                             |
              +--------------+--------------+
              |              |              |
        +-----v----+   +-----v-----+  +-----v-----+
        | Channels |   | LiveViews |  | REST API  |
        | rooms,   |   | message   |  | (Guardian |
        | presence,|   | composer, |  |  JWT)     |
        | DMs      |   | settings  |  +-----+-----+
        +-----+----+   +-----+-----+        |
              |              |              |
              +------+-------+------+-------+
                     |              |
              +------v------+ +-----v------+
              |  Contexts   | |  Oban jobs |
              | Accounts    | | Webhooks   |
              | Workspaces  | | FileScan   |
              | Messaging   | | Integration|
              | Storage     | +-----+------+
              | Slash       |       |
              +------+------+       |
                     |              |
              +------v------+ +-----v------+
              | PostgreSQL  | |  Tigris/S3 |
              +-------------+ +------------+
```

Three protocol surfaces, all with their own authz model that doesn't quite match the others:

- **REST API (`/api/v1/*`)** — Guardian JWT, used by mobile + integrations. Pipe-through `:api_authed`.
- **LiveView (`/app/*`)** — session cookie, `mount/3` puts `current_user` in assigns. `handle_event/3` re-checks (sometimes).
- **Phoenix Channels (`socket "/socket", RelayChatWeb.UserSocket`)** — token in connect param. `join/3` checks topic ownership (sometimes). `handle_in/3` mostly trusts.

The mismatch between these three is where most of the bugs live, and it's where Apex's authenticated-WebSocket crawler earns its keep: REST scanners never see channel topics.

---

## Data Model

```elixir
# Accounts
users:           id, email (citext, unique), hashed_password, totp_secret, role (:owner/:admin/:member/:guest), inserted_at
sessions:        id, user_id, token, ip, ua, last_seen_at
api_tokens:      id, user_id, name, hashed_token, scopes (array), expires_at

# Workspaces
workspaces:      id, slug (unique), name, owner_id, plan, settings (jsonb)
memberships:     id, workspace_id, user_id, role (:owner/:admin/:member/:guest), joined_at
                 unique(workspace_id, user_id)
invitations:     id, workspace_id, email, token, invited_by, expires_at, role

# Messaging
channels:        id, workspace_id, name, kind (:public/:private/:dm), topic, created_by
channel_members: id, channel_id, user_id, last_read_at, notify_level
                 unique(channel_id, user_id)
messages:        id, channel_id, user_id, body (text), body_html (text),
                 thread_root_id (nullable), edited_at, deleted_at, attachments (jsonb)
reactions:       id, message_id, user_id, emoji
                 unique(message_id, user_id, emoji)

# Files
files:           id, workspace_id, uploader_id, channel_id (nullable),
                 key (s3 key), filename, content_type, size, sha256,
                 scan_status (:pending/:clean/:infected), inserted_at

# Integrations
integrations:    id, workspace_id, kind (:incoming_webhook/:outgoing/:slash_command),
                 name, config (jsonb), secret, created_by
slash_commands:  id, workspace_id, command (text), url, method, secret, escape_html (bool)

# Admin / ETL (vuln V4)
etl_scripts:     id, workspace_id, name, source (text), created_by, last_run_at
audit_logs:      id, actor_id, workspace_id, action, target_type, target_id, meta (jsonb)
```

Notable design choices that matter for the bug-planting:

- `messages.body_html` is rendered with `Phoenix.HTML.raw/1` on the assumption that the renderer (`Earmark` markdown + `HtmlSanitizeEx`) sanitized it. Slash command output bypasses this on purpose (V8).
- `files.key` is `"#{workspace_id}/#{uuid}/#{filename}"` and `filename` is user-supplied in the presign call (V3).
- `channels.kind = :dm` is a real `Channel`, not a separate table. Topic = `"workspace:#{ws_id}:dm:#{channel_id}"`. Authz lives on `channel_members`, but the channel-join code only checks the workspace (V2).
- `etl_scripts.source` is Elixir source. There are *two* ways to run it (V4 real + V4 herring).

---

## Key Routes / Surfaces

### HTTP Routes

| Method | Path | Controller / LiveView | Auth | Notes |
|---|---|---|---|---|
| GET | `/` | `PageController.home` | none | marketing |
| POST | `/auth/register` | `AuthController.register` | none | rate-limited |
| POST | `/auth/login` | `AuthController.login` | none | issues session + JWT |
| POST | `/auth/totp` | `AuthController.totp` | partial | step-up |
| GET | `/app/:workspace_slug` | `WorkspaceLive.Index` | session | mounts presence |
| GET | `/app/:workspace_slug/c/:channel_id` | `ChannelLive.Show` | session | LiveView, see V1 |
| GET | `/app/:workspace_slug/dm/:user_id` | `DmLive.Show` | session | resolves channel |
| GET | `/app/:workspace_slug/admin` | `AdminLive.Index` | session+admin | hosts ETL panel (V4) |
| GET | `/app/:workspace_slug/files/:file_id` | `FileController.show` | session | redirects to presign (V3) |
| POST | `/api/v1/files/presign` | `Api.FileController.presign` | JWT | returns PUT url (V3) |
| POST | `/api/v1/messages` | `Api.MessageController.create` | JWT | mass-assignment (V10) |
| POST | `/api/v1/workspaces/:id/members` | `Api.MemberController.create` | JWT | role tampering (V10) |
| POST | `/api/v1/integrations/webhook/:token` | `Api.WebhookController.in` | token | HMAC verify (V12 SSRF target) |
| POST | `/api/v1/slash/:command` | `Api.SlashController.run` | JWT | invokes integration |
| GET | `/uploads/*path` | `Plug.Static` | none | mis-scoped (V5) |
| GET | `/.well-known/*path` | `Plug.Static` | none | accidentally serves `priv/` (V5) |
| GET | `/healthz` | `HealthController.show` | none | leaks build sha + env name |

### LiveView Events

| LiveView | Event | Params | Authz Check | Notes |
|---|---|---|---|---|
| `ChannelLive.Show` | `send_message` | `body, channel_id` | yes (member) | ok |
| `ChannelLive.Show` | `edit_message` | `id, body` | yes (author) | ok |
| `ChannelLive.Show` | `delete_message` | `id` | **no** | V1 — no ownership check |
| `ChannelLive.Show` | `react` | `id, emoji` | partial | atom leak via `String.to_atom/1` (V9) |
| `ChannelLive.Show` | `pin_message` | `id` | role check | ok |
| `ChannelLive.Show` | `invite_user` | `email, role` | admin | role from params (V10) |
| `AdminLive.Index` | `run_etl_safe` | `script_id, args` | admin + AST allowlist | **herring** (V4-H) |
| `AdminLive.Index` | `run_etl` | `source, args` | admin only | V4 — `Code.eval_string` |
| `AdminLive.Index` | `set_workspace_settings` | `settings` | admin | jsonb-merge clobbers role caps |
| `FileLive.Uploader` | `presign_upload` | `filename, content_type` | session | V3 — filename in S3 key |
| `ProfileLive.Edit` | `save` | `params` | self | `cast(:all)` (V10 sibling) |
| `IntegrationLive.New` | `test_webhook` | `url` | admin | SSRF (V12) |

### Phoenix Channel Topics

| Topic pattern | Module | `join/3` check | Events | Notes |
|---|---|---|---|---|
| `workspace:<ws_id>` | `WorkspaceChannel` | membership | `presence_diff`, `notification` | ok |
| `workspace:<ws_id>:room:<channel_id>` | `RoomChannel` | membership of workspace only | `new_msg`, `typing`, `read` | V2 — does not check channel membership for private rooms |
| `workspace:<ws_id>:dm:<channel_id>` | `DmChannel` | **none beyond authn** | `new_msg`, `typing` | V2 — topic spoofing |
| `presence:<ws_id>` | `PresenceChannel` | membership | `track`, `untrack` | V7 — `GenServer.cast` write |
| `user:<user_id>` | `UserChannel` | self only | `notification`, `mention` | ok |
| `system:broadcast` | `SystemChannel` | none | `announcement` | intentional, public |

---

## Auth Model

Three principals, three transports, four roles:

- **Roles:** `:owner` (workspace creator, billing), `:admin` (manage members, integrations, ETL), `:member` (post, react, upload), `:guest` (read-only in shared channels).
- **Transports:**
  - Browser: `Plug.Session` cookie + CSRF token. `mount/3` rebuilds `current_user` from session.
  - Mobile / API: Guardian JWT, `Authorization: Bearer …`, HS256 (V6).
  - Channels: token via connect-param `token`, verified with `Phoenix.Token.verify/4` (1 day TTL). The token contains only `user_id`. **Workspace membership is re-checked per-topic on `join/3` — except where it isn't (V2).**
- **Step-up:** TOTP required for `:admin` actions (settings, ETL, integrations). Implemented via `assigns.totp_verified_until` (timestamp). Not enforced on the `run_etl` LiveView event because the LiveView mount checks role but the handler doesn't recheck the timer (chained-vuln pairing with V4).

The intended invariant — *"every state transition is authorized at the boundary closest to the data"* — is violated in five distinct places. That's the point.

---

## Intentional Vulnerabilities

Twelve real bugs plus two planted herrings. CWE + bounty parallels in the Real-World Parallels section.

| ID | Severity | Class | Surface | Anchor |
|---|---|---|---|---|
| V1 | High | Broken Access Control | LiveView | `delete_message` no owner check |
| V2 | Critical | Broken Access Control | Channels | DM topic spoofing |
| V3 | High | IDOR / SSRF-adjacent | REST | S3 presign with user filename |
| V4 | Critical | RCE | LiveView | `Code.eval_string` in admin ETL |
| V4-H | (herring) | — | LiveView | safe `:erl_eval` w/ AST allowlist |
| V5 | High | Info Disclosure | HTTP | `Plug.Static` serves `priv/` |
| V6 | Critical | Broken Crypto / Auth | JWT | dev secret reused in prod |
| V7 | High | Logic / Race | GenServer | `cast` instead of `call` |
| V8 | Medium | Stored XSS | LiveView render | slash-command HTML injection |
| V9 | Medium | DoS | Channels | `String.to_atom/1` on user input |
| V10 | High | Mass Assignment | REST + LiveView | `cast/3` with `__schema__(:fields)` |
| V11-H | (herring) | — | Webhooks | HMAC compared with `==` but constant-time wrapper applied |
| V12 | High | SSRF | LiveView | `test_webhook` follows redirects |
| V13 | Low | Info Disclosure | HTTP | `/healthz` leaks build/env |

### V1 — LiveView event handler authz gap (High, CWE-284)

```elixir
# lib/relaychat_web/live/channel_live/show.ex
def handle_event("delete_message", %{"id" => id}, socket) do
  msg = Messaging.get_message!(id)
  Messaging.delete_message(msg)
  {:noreply, stream_delete(socket, :messages, msg)}
end
```

No check that `msg.user_id == socket.assigns.current_user.id` or that current user is a channel admin. Any member can delete any message by clicking-then-replaying with a tampered `phx-value-id`. Apex's LiveView crawler should diff handlers vs. data ownership.

### V2 — Channel topic spoofing (Critical, CWE-285)

```elixir
# lib/relaychat_web/channels/dm_channel.ex
def join("workspace:" <> rest, _params, socket) do
  [_ws_id, "dm", _channel_id] = String.split(rest, ":")
  {:ok, socket}  # workspace membership "implied" by socket auth
end
```

A user authenticated to workspace 7 can `socket.channel("workspace:42:dm:99").join()` and silently subscribe to a DM in workspace 42 between two strangers. They will receive every `new_msg` push. The fix is to load the DM channel and verify membership; the bug is that the join handler trusts the topic string. This is the marquee finding for the demo.

### V3 — File-share IDOR via presigned URL not bound to user (High, CWE-639)

```elixir
def presign(conn, %{"filename" => filename, "content_type" => ct}) do
  key = "#{current_workspace(conn).id}/#{Ecto.UUID.generate()}/#{filename}"
  url = ExAws.S3.presigned_url(:put, bucket(), key, expires_in: 3600)
  json(conn, %{url: url, key: key})
end
```

Two issues, surfaced by Apex's HTTP fuzzer + S3 plugin:

1. `filename` is user-supplied and concatenated into the key. A user can pass `../../shared/secrets.txt` to write outside their UUID prefix on lax bucket policy.
2. The download path `/app/:ws/files/:file_id` redirects to a `:get` presign that does not check that the requesting user is a member of `files.workspace_id`. Anyone with a file UUID can fetch.

### V4 — `Code.eval_string` in admin ETL panel (Critical, CWE-94)

```elixir
def handle_event("run_etl", %{"source" => source, "args" => args}, socket) do
  if socket.assigns.current_user.role in [:owner, :admin] do
    {result, _binding} = Code.eval_string(source, args: args)
    {:noreply, assign(socket, :etl_result, inspect(result))}
  else
    {:noreply, socket}
  end
end
```

Admin-only, but admin is enough for full RCE on the BEAM node. No TOTP recheck (Auth Model note).

### V4-H — The planted herring (the "looks-bad-isn't")

Right next to V4, in the same module:

```elixir
def handle_event("run_etl_safe", %{"script_id" => id, "args" => args}, socket) do
  with :ok <- Authz.require_admin(socket),
       :ok <- Authz.require_totp_recent(socket),
       %EtlScript{source: src} <- Workspaces.get_script(socket, id),
       {:ok, ast} <- Code.string_to_quoted(src),
       :ok <- AstAllowlist.validate(ast, @safe_calls),
       {result, _} <- :erl_eval.exprs(quoted_to_erl(ast), bindings(args)) do
    {:noreply, assign(socket, :etl_result, inspect(result))}
  end
end
```

This *looks* like RCE — it evaluates user code — but the AST is parsed, validated against an allowlist (`Enum.map/2`, `Map.get/2`, arithmetic, `Kernel.{+,-,*,/}`, comparison), TOTP is required, and `:erl_eval.exprs/2` operates on the validated tree only. A naive scanner flags it; the **judge agent** is supposed to read the allowlist, prove the AST is bounded, and downgrade the finding. This is the showpiece for planted-herring detection.

### V5 — `Plug.Static` misconfig (High, CWE-538)

```elixir
# lib/relaychat_web/endpoint.ex
plug Plug.Static,
  at: "/",
  from: :relaychat,
  gzip: false,
  only: ~w(assets fonts images favicon.ico robots.txt uploads .well-known)
```

`uploads` is local in dev; in prod the bucket is Tigris but the plug is left enabled and serves `priv/static/uploads/*` which contains old seeds. Worse, `.well-known` is included to support ACME but the `from: :relaychat` resolves to the app root, exposing `priv/repo/seeds.exs`, `priv/cert/dev_ca.key`, and a stale `.env` placed in `priv/` "for the demo."

### V6 — Weak Guardian JWT (Critical, CWE-321 / CWE-798)

```elixir
# config/dev.exs
config :relaychat, RelayChat.Guardian,
  issuer: "relaychat",
  secret_key: "DEV-NOT-FOR-PROD-secret-9f3a"

# config/runtime.exs
config :relaychat, RelayChat.Guardian,
  secret_key: System.get_env("GUARDIAN_SECRET") || "DEV-NOT-FOR-PROD-secret-9f3a"
```

The fallback in `runtime.exs` reuses the dev secret. The string also appears in `.git`, the Helm `values.yaml`, and `priv/` (compounding V5). Apex's secrets module should grep, then forge a token, then walk the JWT-protected REST surface end-to-end.

### V7 — `GenServer.cast` instead of `call` (High, CWE-362)

```elixir
# lib/relaychat/presence/tracker.ex
def set_status(user_id, status), do: GenServer.cast(__MODULE__, {:set, user_id, status})

def handle_cast({:set, user_id, status}, state) do
  # no auth, no validation — caller is "the BEAM"
  {:noreply, Map.put(state, user_id, status)}
end
```

Called from a channel handler:

```elixir
def handle_in("set_status", %{"user_id" => uid, "status" => s}, socket) do
  Tracker.set_status(uid, s)  # cast — fire-and-forget, no return, no authz
  {:reply, :ok, socket}
end
```

Anyone authenticated can mutate anyone's presence. The `cast` shape hides the auth gap because there's no return value to inspect; a `call` would have forced the developer to think about the response. Apex's BEAM-aware analyzer should flag the asymmetry between caller and handler.

### V8 — Stored XSS via slash command output (Medium, CWE-79)

```elixir
def render_message(%{kind: :slash_output, body_html: html}) do
  if message.slash_command.escape_html do
    Phoenix.HTML.html_escape(html)
  else
    Phoenix.HTML.raw(html)
  end
end
```

`slash_commands.escape_html` defaults to `false` for "rich integrations." A workspace admin can register a slash command pointing at attacker-controlled URL, and the response body is rendered raw into every viewer's LiveView. Persistent across reloads because it lives in `messages.body_html`. Cookies are HttpOnly, but the LiveView socket isn't — `document.querySelector('meta[name=csrf-token]')` is fair game.

### V9 — Atom DoS (Medium, CWE-400)

```elixir
def handle_in("react", %{"emoji" => name}, socket) do
  emoji_atom = String.to_atom(name)  # unbounded
  Reactions.add(socket.assigns.message_id, socket.assigns.user.id, emoji_atom)
  {:noreply, socket}
end
```

Each unique `name` allocates a new atom. The atom table is fixed-size (default 1,048,576). A loop of `:rand.bytes(8) |> Base.encode16()` exhausts it and crashes the node. This is an Erlang/Elixir-native bug class Apex's CWE module should know about — not transferable from JS demos.

### V10 — Mass assignment via `cast/3` over `__schema__(:fields)` (High, CWE-915)

```elixir
def changeset(member, attrs) do
  member
  |> cast(attrs, __MODULE__.__schema__(:fields))  # equivalent to "cast :all"
  |> validate_required([:user_id, :workspace_id])
end
```

A `POST /api/v1/workspaces/:id/members` with `{"role": "owner"}` succeeds. Same pattern in `ProfileLive.Edit#save`, where a member can set `role: :admin` on themselves.

### V11-H — HMAC herring on webhooks

```elixir
def verify_signature(body, sig, secret) do
  expected = :crypto.mac(:hmac, :sha256, secret, body) |> Base.encode16(case: :lower)
  Plug.Crypto.secure_compare(expected, sig)
end
```

Looks suspicious because it reads like a hand-rolled HMAC. It's actually correct: SHA-256, constant-time compare via `Plug.Crypto.secure_compare/2`. The judge agent should suppress, citing the constant-time call.

### V12 — SSRF in webhook tester (High, CWE-918)

```elixir
def handle_event("test_webhook", %{"url" => url}, socket) do
  Finch.build(:post, url, [], "ping")
  |> Finch.request(RelayChat.Finch, [follow_redirects: true])
  ...
end
```

No allowlist, no IP filter, follows redirects, runs as the cluster pod with metadata service reachable. Standard cloud-IMDS pivot.

### V13 — `/healthz` info disclosure (Low, CWE-200)

Returns `{"build": "git-sha", "env": "prod", "deps": [...]}`, useful for fingerprinting. Pairs with V6 (helps confirm prod uses dev secret) and V5 (confirms `priv/` paths).

---

## Real-World Parallels

- **V1, V2** — Mattermost CVE-2023-6458 (private channel access via WebSocket), CVE-2022-1252 (DM authorization bypass). Slack HackerOne #573104 (channel topic info leak via realtime subscribe).
- **V3** — Slack file presigned-URL leak, public disclosure 2019; Zoom recording IDOR (CVE-2020-11500-adjacent); GitHub bounty #1207223 (S3 key path traversal).
- **V4** — GitLab CVE-2022-2884 (RCE via GitHub import), Rocket.Chat CVE-2021-22886 (NoSQL → RCE via integration scripts). Custom-script ETL panels are a recurring pattern in B2B tools.
- **V4-H** — Rails sandboxed-AST evaluators (e.g., Liquid templates, Mathn), Discourse `markdown-it` plugin sandbox: real allowlists exist; scanners often misclassify.
- **V5** — Rails CVE-2018-3760 (Sprockets path traversal), Express `serve-static` issues. Phoenix's own CVE-2024-32030 (`Plug.Static` chunked encoding). `.env` exposure is a classic — TruffleHog reports thousands per year.
- **V6** — Discourse CVE-2019-11479 (hardcoded secret), countless dev-secret-in-prod incidents. Guardian-specific: HackerOne #1088872.
- **V7** — BEAM-specific class. `cast` vs `call` confusion documented in Saša Jurić's "Elixir in Action," 2nd ed. Practical exploitation in Bleacher Report 2018 incident.
- **V8** — Mattermost CVE-2022-1295 (XSS via slash command), Slack #284770 (RTM message HTML injection via integration).
- **V9** — Erlang/OTP guidance in "Designing for Scalability with Erlang/OTP" (Cesarini & Vinoski). Practical exploitation: Phoenix bug 2017, RabbitMQ management UI.
- **V10** — Rails CVE-2012-2054 (mass assignment on GitHub itself). Ecto's documented "antipattern" in `Ecto.Changeset` docs.
- **V12** — Capital One 2019 (SSRF → IMDS), GitLab CVE-2021-22214. Webhook testers are SSRF magnets.
- **V13** — Always-on `/healthz` leaks: AWS Elastic Beanstalk default exposure, Spring Boot `/actuator/info`.

---

## Apex Features Showcased

1. **Authenticated WebSocket / channel crawling.** Apex connects via `UserSocket`, enumerates channel topics from JS bundles + LiveView source + observed traffic, and fuzzes `join/3` with cross-workspace topic strings. V2 is unreachable from REST scanners.
2. **Planted-herring detection / judge agent.** V4 vs V4-H sit twenty lines apart in the same file. The judge agent must read the AST allowlist, simulate inputs, and confirm that V4-H is bounded. Same module: V11-H constant-time compare. Demo highlights the suppress-with-rationale flow.
3. **Novel-stack credibility.** Phoenix + LiveView + Channels are uncommon in published security tooling. Apex's Elixir AST module (built on `Code.string_to_quoted/2`) reasons about pattern-matching `handle_event`, `handle_in`, `handle_cast` arms, distinguishing them as authz boundaries.
4. **CVSS scoring + findings registry.** Twelve real bugs with chained vectors (V6 → V10 → V4: dev-secret JWT, forge admin token, escalate role, RCE). The registry deduplicates the V4 / V4-H pair.
5. **Patching agent.** Demos generated PRs for V1 (add owner check), V2 (load and verify channel), V10 (explicit field list). V8 patch swaps `raw/1` for `Phoenix.HTML.html_escape/1` plus opt-in trusted-renderer pipeline.
6. **Memory.** Across runs, Apex remembers the workspace token and the forged JWT, jumping straight to the chained exploit on re-run.
7. **Threat modeling.** Apex emits a STRIDE-style summary highlighting the three-transport authz mismatch (REST / LiveView / Channels) as the dominant theme.
8. **Attack-surface + JS endpoint extraction.** LiveView `phx-*` attributes and the compiled `app.js` reveal the channel topic templates — this is how Apex finds `workspace:<ws>:dm:<id>` to fuzz V2.
9. **Kali container + Playwright.** Headless browser logs into the LiveView app, captures the `csrf-token`, drives `phx-click` events to reach LiveView handlers like V1.

---

## Demo Storyline

Eight-minute screencast, three acts.

**Act I — Recon (0:00–2:00).**

Operator runs `apex /pentest https://relaychat.demo --auth=creds.json`. Apex hits `/healthz` (V13), notes `env: prod, build: 7ab3...`. Attack-surface module crawls REST + LiveView routes, parses `app.js` for channel topic templates, and notices `workspace:<int>:dm:<int>` as a parametric topic. Static-file fuzzer finds `/.well-known/seeds.exs` (V5) — leaking the Guardian secret string referenced in dev config. Findings registry: 2 medium, 1 critical (secret).

**Act II — Pivot (2:00–5:30).**

Apex forges a Guardian JWT (V6) using the leaked secret, scopes it `workspace_id: 1, role: :member`. Mass-assignment probe on `POST /api/v1/workspaces/1/members` with `role: "owner"` lands (V10). Now an owner of workspace 1. Operator opens `/app/relay-demo/admin`. Apex's LiveView reasoner spots two ETL handlers and emits a finding for each. The judge agent reads `AstAllowlist.validate/2` for `run_etl_safe`, simulates inputs, and **suppresses V4-H with rationale**: "AST is parsed and validated against `@safe_calls` allowlist; `:erl_eval.exprs/2` cannot escape." Highlight on screen. Then it confirms V4 (`Code.eval_string`) and proves RCE by reading `/etc/hostname`.

**Act III — The marquee bug (5:30–7:30).**

Channel crawler. Apex joins `workspace:1` normally, then attempts `workspace:7:dm:42` using its existing socket. Subscription succeeds (V2). Apex dumps three `new_msg` payloads from a DM in a workspace it never joined. The judge agent confirms: "Topic does not require membership of channel `42`." This is the demo's emotional peak — the kind of bug you cannot find with a HAR-replay scanner.

**Outro (7:30–8:00).**

Patching agent opens four PRs: V1, V2, V10, V8. Findings panel shows 12 confirmed, 2 suppressed (with the planted-herring suppression rationale highlighted). CVSS rollup: 2 critical, 5 high, 4 medium, 1 low.

---

## Build Notes

Three-to-five day build for one engineer comfortable with Phoenix.

**Day 1 — Skeleton.**

- `mix phx.new relaychat --live`. Add Guardian, Oban, ExAws, HtmlSanitizeEx, Earmark, Finch.
- Schemas + migrations for all tables in Data Model. `mix ecto.gen.migration` per context.
- Seed data: 3 workspaces, ~20 users across them, ~50 messages per channel, 3 DMs (one cross-workspace for the V2 demo).
- Auth flows: register / login / TOTP. Skip the email side; use Bamboo local adapter.

**Day 2 — Realtime + LiveView.**

- `UserSocket`, `WorkspaceChannel`, `RoomChannel`, `DmChannel`, `PresenceChannel`, `UserChannel`, `SystemChannel`.
- `ChannelLive.Show` with stream-based message rendering, composer, slash commands, file uploads.
- Wire V1 (delete handler with no owner check), V2 (DM channel join with no membership check), V7 (cast handler, status mutator), V9 (atom-on-emoji).

**Day 3 — Storage + Integrations + Admin.**

- Tigris/S3 presign; LiveView uploader. Wire V3 (filename in key, no auth on download).
- Webhook + slash-command system. Wire V8 (XSS via slash output) and V12 (SSRF in webhook tester). Wire V11-H correctly (constant-time compare).
- Admin LiveView with both `run_etl` (V4) and `run_etl_safe` (V4-H). Implement `AstAllowlist.validate/2` carefully — it has to actually be safe for the demo to work. Cover: `Enum.map/2`, `Map.get/2/3`, arithmetic, comparison, `Kernel.{abs,max,min,length,hd,tl}`. Reject everything else.

**Day 4 — Auth glue + the static plug + JWT bug.**

- Wire V6: hardcoded fallback secret. Place a copy of the secret in `priv/repo/seeds.exs` for the V5+V6 chain.
- Wire V5: include `.well-known` and `uploads` in `Plug.Static`'s `:only` list, ensure `priv/` paths resolve to seeds.
- Wire V10 mass assignment in `MemberController`, `MessageController`, `ProfileLive.Edit`. Include `role`, `workspace_id`, `is_admin` as cast targets.
- Wire V13 `/healthz`.

**Day 5 — Polish + demo data + recording.**

- Seed the cross-workspace DM (workspace 7, channel 42, two strangers) used by V2.
- Seed an `etl_scripts` row used by `run_etl_safe`.
- Tailwind pass: presence dots, channel sidebar, message threading. Make it *look* like Slack so operators feel the parallel.
- Smoke-test each vuln by hand, write a `lib/relaychat_demo/exploit_smoke.exs` script with one PoC per vuln (used to validate Apex didn't regress).
- Add a `Makefile`: `make seed`, `make reset`, `make demo`.
- Write a one-page `RUNBOOK.md` for the demo operator.

**Things to avoid:**

- Do not over-engineer multi-tenancy. One database, `workspace_id` columns, Ecto query helpers — that's the realistic SMB shape.
- Do not gate V4-H behind clever obfuscation. The judge agent should be able to *understand* the safety argument; if it's hidden, the demo is unfair to the agent.
- Do not put `IO.inspect` in production paths; some operators screen-share.

**Compose / deploy:**

`docker-compose.yml` with `postgres:16`, the Phoenix app, and a `minio` container masquerading as Tigris (signed URLs work locally). Single command boot: `docker compose up`.

---

## Recording Notes

- Record at 1440x900, 24fps, mono mic. Two-window layout: terminal left, browser right.
- Pre-stage: workspace `relay-demo` logged in as `alex@relay.demo` (member). Apex's auth token already in `creds.json`.
- Slow the channel-crawler segment — V2 is the climax. When the cross-workspace DM messages render in the Apex TUI, hold the frame for two seconds.
- For the planted herring, split-screen the judge agent's reasoning trace next to the source of `AstAllowlist.validate/2`. The viewer needs to see *why* the agent suppressed.
- Cut the patch-agent PR generation tight: show the diff for V2 (load channel + verify membership) full-screen, then a one-line caption.
- End on the findings panel sorted by severity, CVSS rollup visible, "2 suppressed (planted-herring)" highlighted. Hold three seconds. Fade.
- Do not show real cloud credentials, even revoked ones. The demo's Tigris bucket is scoped to a throwaway account; the IMDS pivot in V12 should be against a local mock metadata server (`169.254.169.254` routed via container network, returning fake creds).
- Caption every CVE/bounty parallel as it appears on screen. Operators learn the mapping faster when it's repeated visually.
