# Apex → Agent Skills Migration Plan

**Status:** Draft
**Branch:** `claude/apex-to-agent-skills-22fZE`
**Authors:** Claude Code (synthesis from a multi-agent codebase audit)

---

## 1. Premise

Apex today is a security-specific harness. A lot of valuable pentesting *knowledge* lives in TypeScript: hardcoded system prompts, a fixed `discovery → swarm → report` pipeline, eleven specialized sub-agents wired into a workflow orchestrator, an approval gate, a task system, a plan store, a findings registry, etc.

That harness was the right call when models were weaker — determinism made up for reasoning gaps. As models get better at agent primitives (memory, planning, sub-agent spawning, tool use) the harness starts *limiting* the model rather than helping it.

The goal of this migration is to **invert the architecture**: instead of Apex driving the model through a fixed pipeline, the model drives itself, and Apex contributes only what it has that the base model doesn't:

1. **Procedural knowledge** — pentest methodology written *for* an LLM, in markdown.
2. **Reference data** — CVSS lookup tables, CWE catalog, vulnerability taxonomy, wordlists.
3. **Helper scripts** — small, optional utilities the agent can call deterministically when it wants speed or reproducibility (e.g. `score-cvss <vector>`).

Everything else — the pipeline, the gates, the swarm orchestrator, the sub-agent classes, the message-passing scaffold — is harness mechanism that should be deleted in favor of a generic coding-agent harness (Claude Code or equivalent) reading the skills.

The bet: a strong general agent + Apex's distilled domain skills > Apex's bespoke harness with embedded prompts.

---

## 2. Current state at a glance

```
src/core/
├── agents/                           ← 11 sub-agents, mostly methodology + harness glue
│   ├── offSecAgent/                  ← Base class wrapping AI SDK + tool dispatch
│   └── specialized/
│       ├── attackSurface/            ← Blackbox recon (HIGH-VALUE prompt)
│       ├── whiteboxAttackSurface/    ← Source-code recon (HIGH-VALUE prompt)
│       ├── authenticationAgent/      ← Login/session (HIGH-VALUE prompt)
│       ├── pentest/                  ← Targeted pentest (THE high-value prompt)
│       ├── codeAgent/                ← Generic code nav (low value)
│       ├── patching/                 ← Fix development (HIGH-VALUE prompt)
│       ├── environment/              ← Docker/dev env bootstrap (medium)
│       ├── findingJudge/             ← Validates POCs (small but useful)
│       ├── cvssScorer/               ← CVSS 4.0 (data + small prompt)
│       └── benchmark/                ← CI benchmark runner (low value)
├── workflows/
│   ├── pentest.ts                    ← runPentestWorkflow() — THE prescriptive pipeline
│   ├── whiteboxAttackSurface.ts
│   └── threatModel.ts
├── operator/                         ← Approval gate, permission tiers, stage manager
├── tasks/                            ← Per-agent task system (file-backed)
├── plan/                             ← Plan-file I/O
├── memory/                           ← ~/.pensar/memories/{category}/{id}.json
├── toolset/                          ← 30+ tool definitions, presets, on/off state
├── findings/                         ← Finding schema + dedup + registry
├── skills/                           ← Already exists! 2 builtins (pentest, threat-model)
├── api/                              ← Public entry points (runPentest, runAttackSurface…)
├── session/, messages/, services/    ← Session lifecycle, rate limiter, etc.
└── lib/cvss, lib/cwe, lib/evidence   ← Pure domain libs

assets/wordlists/                     ← common.txt, large.txt, tiny.txt
scripts/                              ← attack-surface.ts, auth.ts, blackbox-pentest.ts,
                                        gmail-oauth.ts, generate-models.ts, watch.ts, …
container/                            ← Kali Linux + tooling (Dockerfile + compose)
decisions/                            ← PDR-001 … PDR-007
```

Approximate volume:
- ~3,100 lines of agent system prompts in TS template literals
- ~2,400 lines more in user-prompt builders, judge prompts, scoring prompts
- ~1,500 lines of workflow orchestration
- ~500 lines of operator gate/permission machinery

The skills system is already in place (skills.sh-compatible: SKILL.md with YAML frontmatter + markdown body + optional `scripts/` subdir). Two builtins exist: `pentest` and `threat-model`. The `pentest` builtin is currently a thin wrapper that asks the agent to invoke `run_pentest_workflow` — i.e. it just hands control back to the harness pipeline. That's the first thing this migration replaces.

---

## 3. Target architecture

```
A general coding agent (e.g. Claude Code)
   │
   ├── reads skill manifests from .claude/skills/, .skills/, ~/.agents/skills/
   │
   ├── Skill: pentest                       ← procedure, not pipeline
   │   ├── SKILL.md                         (orient → choose mode → delegate → aggregate)
   │   └── scripts/render-report.ts
   │
   ├── Skill: attack-surface-mapping
   │   ├── SKILL.md                         (blackbox + whitebox methodology, both modes)
   │   └── scripts/extract-routes.ts        (optional whitebox helper)
   │
   ├── Skill: authenticated-testing
   │   ├── SKILL.md
   │   └── scripts/persist-session.ts
   │
   ├── Skill: vulnerability-testing         ← the big targeted-pentest prompt
   │   ├── SKILL.md
   │   └── scripts/poc-scaffold.sh
   │
   ├── Skill: threat-model                  (already exists; refine)
   ├── Skill: vulnerability-patching
   ├── Skill: environment-setup
   ├── Skill: pentest-reporting
   │
   ├── Skill: cvss-scoring                  ← reference data + calculator script
   │   ├── SKILL.md
   │   ├── assets/macrovector-scores.json
   │   ├── assets/cvss4-metrics.yaml
   │   └── scripts/score-cvss.ts
   │
   ├── Skill: cwe-classification            ← reference data + classifier
   │   ├── SKILL.md
   │   ├── assets/cwe-catalog.json
   │   └── scripts/classify-finding.ts
   │
   ├── Skill: vulnerability-taxonomy        ← finding schema + class regex patterns
   │   ├── SKILL.md
   │   ├── assets/vuln-classes.yaml
   │   └── assets/finding-schema.json
   │
   ├── Skill: pentest-wordlists             ← bundled wordlists + usage notes
   │   ├── SKILL.md
   │   └── assets/{common,large,tiny}.txt
   │
   └── Skill: finding-validation            ← optional self-check methodology
       └── SKILL.md
```

The general agent reads SKILL.md when relevant, optionally invokes bundled scripts when deterministic helpers are valuable (e.g. CVSS scoring), and otherwise improvises. There's no enforced state machine, no pipeline tool, no per-phase agent class.

What survives outside of skills:
- The CLI entry points that *trigger* skills (`pensar pentest`, `pensar threat-model`) — they just inject runtime context and hand off to the agent.
- Pensar Console integration (`pensar login`, `pensar projects`, `pensar issues`, …) — pure API client, unrelated to agent behavior.
- The TUI shell — but pruned of pipeline-aware widgets.
- Container/Docker setup for Kali Linux runtime.
- Build/dev infra (`generate:models`, `watch`, etc.).

---

## 4. Skill catalog

This is the centerpiece. Each row maps a current code surface to a target skill.

### 4.1 Operational skills (procedural — describe how to do work)

| Slug | Purpose | Source material today | Bundled scripts | Notes |
|---|---|---|---|---|
| `pentest` | Top-level: orient, pick mode, dispatch, aggregate. Replaces `run_pentest_workflow`. | `src/core/skills/builtins/pentest.ts`, `src/core/workflows/pentest.ts:342-584`, `src/core/agents/offSecAgent/prompt.ts:115-215` | `render-report.ts` (markdown + JSON), optional `aggregate-findings.ts` | Replaces the pipeline tool with a procedure that explains *when* to do discovery vs. when to just test, *when* to spawn parallel sub-agents, etc. |
| `attack-surface-mapping` | Discover endpoints / apps / assets. Both modes documented; agent picks. | `agents/specialized/attackSurface/prompts.ts:1-349` (blackbox), `agents/specialized/whiteboxAttackSurface/prompts.ts:1-139` (whitebox) | `extract-routes.ts` (whitebox), `subdomain-enum.sh` (blackbox) | Merging blackbox + whitebox into one skill avoids duplication; ~70 % of the methodology overlaps. |
| `authenticated-testing` | Detect auth scheme, log in, persist session, reuse across tests. | `agents/specialized/authenticationAgent/prompts.ts` (139 + 291 + 107 + 50 lines, four blocks) | `persist-session.ts`, `extract-cookies-from-har.ts` | The four prompt blocks in that file collapse into one coherent procedure once duplication is removed. |
| `vulnerability-testing` | Targeted exploit methodology (the big PentestAgent prompt). | `agents/specialized/pentest/agent.ts:365-522` (4 variants × ~160 lines methodology each) | `poc-scaffold.sh` (POSIX-portable PoC template) | The richest single prompt in the codebase. Subsumes guidance on PoC portability, browser interaction, security headers, CORS, IDOR detection. |
| `threat-model` | App-centric threat modeling from source. | `src/core/skills/builtins/threatModel.ts` | none | Already a skill; minimal change beyond removing `create_file` tool reference (use generic write). |
| `vulnerability-patching` | Apply security fixes, verify, prepare PR. | `agents/specialized/patching/prompts.ts:7-188` + `194-350` | `verify-patch.sh` (lint → tsc → tests → PoC-fail check) | 8-step methodology + per-CWE remediation guidance. |
| `environment-setup` | Bootstrap Docker Compose / dev env, validate auth flows. | `agents/specialized/environment/prompts.ts:7-145` | `compose-health-check.sh`, `wait-for-service.sh` | |
| `pentest-reporting` | Aggregate findings → PentestReport → markdown/JSON. | `src/core/report/`, `src/core/findings/registry.ts` (root-cause grouping) | `render-markdown.ts`, `render-json.ts`, `dedupe-findings.ts` | Pure utility skill — used by `pentest` skill as a sub-procedure. |

### 4.2 Reference / asset skills (data-heavy — describe what is)

| Slug | Purpose | Source material today | Bundled assets/scripts | Notes |
|---|---|---|---|---|
| `cvss-scoring` | CVSS 4.0 reference + scoring procedure. | `src/lib/cvss/{types,calculator,macrovector-scores}.ts`, `agents/specialized/cvssScorer/index.ts:147-327` | `assets/cvss4-metrics.yaml`, `assets/macrovector-scores.json`, `assets/eq-thresholds.json`, `scripts/score-cvss.ts` | Calculator stays as a script — agents can optionally call it for deterministic scoring; otherwise they reason from the YAML reference. |
| `cwe-classification` | CWE catalog + selection heuristics. | `src/lib/cwe/{cwe-catalog,types,validate}.ts` | `assets/cwe-catalog.json` (~150 entries), `scripts/lookup-cwe.ts` | Validation script ensures CWE IDs resolve to canonical MITRE names. |
| `vulnerability-taxonomy` | Vuln-class regex patterns + Finding schema. | `src/core/findings/registry.ts`, `src/core/findings/schemas.ts`, `src/lib/evidence/types.ts` | `assets/vuln-classes.yaml` (30+ regex→class tuples), `assets/finding-schema.json`, `assets/evidence-schema.json` | Canonical vocabulary for what counts as a finding and how to classify it. |
| `pentest-wordlists` | Bundled wordlists with usage guidance. | `assets/wordlists/{common,large,tiny}.txt` | `assets/wordlists/*` (verbatim) | SKILL.md describes when to pick which list. |

### 4.3 Optional cross-cutting skills

| Slug | Purpose | Source | Notes |
|---|---|---|---|
| `finding-validation` | Pre-report self-check: hallucination detection, hardcoded-evidence rejection, severity alignment. | `agents/specialized/findingJudge/index.ts:105-188` | Useful as an *optional* sub-procedure the agent invokes on itself before recording a finding. Could collapse into `vulnerability-testing` if we want fewer skills. |
| `pentest-planning` | Task decomposition for complex multi-objective engagements. | `agents/specialized/pentest/planPrompt.ts:9-32` | Could collapse into `pentest`. Keeping separate iff we want it reusable for non-pentest planning. |

**Recommendation:** start with the 11 in 4.1+4.2, fold 4.3 in only if usage warrants. Ship the small set first.

---

## 5. What gets deleted

These are pure harness mechanism — they constrain the agent without contributing knowledge. Once the agent reads skills directly, none of it is needed.

| Module | Reason for removal |
|---|---|
| `src/core/workflows/pentest.ts` (`runPentestWorkflow`, `runPentestSwarm`, phased event-bus emission, manifest-resume bookkeeping) | Replaced by the agent's own reasoning + the `pentest` skill. |
| `src/core/operator/{approvalGate,permissionPolicy,toolClassifier,stageManager}.ts` | Approval/permission concerns become harness-level, not Apex-specific; Claude Code's permission model handles this. |
| `src/core/agents/offSecAgent/offensiveSecurityAgent.ts` (the base class), and all of `src/core/agents/specialized/*/agent.ts` | Each specialized agent class is just a system prompt + tool list + result schema. The prompt becomes a SKILL.md. The class disappears. |
| `src/core/tasks/index.ts` | Task lifecycle is enforced *because* the harness wanted to gate completion. A general agent can use a TODO-list tool from its host harness; we don't need our own. |
| `src/core/plan/index.ts` | Plan files were coupled to the prescriptive plan-then-execute mode. Drop. |
| `src/core/api/blackboxPentest.ts`, `targetedPentest.ts`, `attackSurface.ts`, `threatModel.ts` (the agent-runner exports) | The `pensar pentest` CLI just spawns the host agent with the right skill; no bespoke API needed. |
| `src/core/toolset/index.ts` (toolset *state* + presets + on/off toggling) | Tool selection is the host agent's responsibility; we don't ship a toolset registry. The *tool definitions* themselves get re-homed (see §6). |
| `src/core/services/rateLimiter/` (as a global service) | Rate limiting becomes a script in the relevant skill, not a global harness service. |
| `src/core/messages/`, `src/core/session/persistence.ts` (subagent manifest, message dedup) | Session/message persistence is the host harness's job. |
| Hardcoded prompts in `src/core/agents/**/prompts.ts` and inline in `agent.ts` files | All migrated into SKILL.md. Delete the source-of-truth in TypeScript so prompts can't drift. |
| `src/core/agents/specialized/benchmark/` | The benchmark runner is harness orchestration — keep it as a top-level `scripts/run-benchmarks.ts` that just invokes the host agent on each fixture. |
| TUI dashboards tightly coupled to the swarm pipeline (`src/tui/components/swarm-dashboard/`, `src/tui/components/pentest/`) | If the swarm/pipeline goes, these are dead. The operator-dashboard remains for general agent observability. |

Approximate code deletion: **~6–8k lines** of TypeScript.

---

## 6. What gets kept (re-homed, not deleted)

Some pieces have value as standalone *tools* for the host agent — they just stop being agents themselves.

| What | Where it goes | Why |
|---|---|---|
| Tool implementations (`http_request`, `execute_command`, `browser_*`, `web_search`, `add_memory`/`list_memories`, `create_file`, `update_file`) | Become host-harness tool plugins (e.g. an MCP server `pensar-pentest-tools` or a local tool bundle the host agent can load). Or rely on the host agent's existing tools. | These are domain-useful (a browser tool that drives Playwright, a `cve_lookup` tool, a `smart_enumerate` tool). They're not Apex-specific orchestration. |
| `src/lib/cvss/calculator.ts` | Becomes `scripts/score-cvss.ts` inside the `cvss-scoring` skill. | Useful as a deterministic helper; the agent can fall back to reasoning. |
| `src/lib/cwe/validate.ts` + the CWE catalog | Becomes `scripts/lookup-cwe.ts` + `assets/cwe-catalog.json` inside `cwe-classification`. | |
| `src/core/findings/registry.ts` classification helpers | Becomes `scripts/classify-finding.ts` inside `vulnerability-taxonomy`. The dedup/root-cause-grouping logic becomes `scripts/dedupe-findings.ts` inside `pentest-reporting`. | |
| `src/core/report/renderers/{markdown,json}.ts` | Becomes `scripts/render-{markdown,json}.ts` inside `pentest-reporting`. | |
| `src/core/obfuscation/engine.ts` | Stays in the TUI (it's a privacy layer for screenshot-safe output). Not a skill. | |
| `src/core/credentials/` | Stays as a CLI-side credential vault (`pensar` config). Skills reference *types* but don't touch secrets directly — the host agent's tool layer does. | |
| `src/core/auth/` (Pensar Console device flow, tokens, HMAC signing) | Stays. Pure CLI integration, unrelated to agent behavior. | |
| `src/cli/{login,projects,pentests,issues,fixes,logs,uninstall}.ts` | Stays. Pensar Console API client. | |
| `assets/wordlists/*` | Re-homed into `pentest-wordlists` skill. | |
| `decisions/PDR-001 … PDR-007` | Stays. Add a new PDR-008 documenting this migration. | |
| `container/` (Kali Linux Dockerfile + compose) | Stays. The general agent runs *inside* the Kali container so it has nmap/sqlmap/etc. on PATH. | |
| Build/dev scripts (`generate:models`, `generate:ascii`, `watch`, lint/test/format) | Stays. Not user-facing. | |

---

## 7. Per-skill migration sheets

Short notes on the non-obvious work for each operational skill. Reference data/skills (§4.2) are mechanical extractions and are not detailed here.

### 7.1 `pentest`

**Replaces:** the existing `pentest` builtin (which is a single `run_pentest_workflow` tool call) and the `runPentestWorkflow` pipeline.

**Methodology to document:**
1. Orient — read target context (URL, codebase, auth, scope).
2. Choose mode — blackbox / whitebox / hybrid. Decision rubric inline.
3. Map attack surface — invoke `attack-surface-mapping` skill.
4. Authenticate (if creds present) — invoke `authenticated-testing` skill, persist session.
5. Test — for each target, apply `vulnerability-testing` skill. **Don't** mandate parallel swarm; the host agent decides whether to spawn sub-agents based on attack-surface size.
6. Validate — apply `finding-validation` (or fold inline) before recording.
7. Aggregate & report — invoke `pentest-reporting` skill.

**Key pivot:** the existing builtin's instruction "Call `run_pentest_workflow` and don't manually orchestrate" inverts entirely. The new instruction is "*you are the orchestrator*; here are the sub-skills you can invoke in any order".

**Inputs (frontmatter):** target, cwd?, auth-url?, auth-user?, auth-pass?, hosts?, ports?, strict?, prompt?, threat-model?

**Outputs:** findings.json, report.md (paths configurable).

### 7.2 `attack-surface-mapping`

**Merges** blackbox (`attackSurface/prompts.ts`) and whitebox (`whiteboxAttackSurface/prompts.ts`). Both have a Phase-1 (auth context), Phase-2 (enumerate), Phase-3 (coverage double-check) — they should share that scaffold.

**Methodology:**
- Mode-A (blackbox): subdomain enum → JS extraction → browser exploration → asset documentation.
- Mode-B (whitebox): repo identification → per-app code analysis (delegate via host agent's sub-agent capability) → mandatory coverage double-check (workspaces, framework configs, IaC, Dockerfiles).

**Apex-specific tool refs to drop:** `document_app`, `document_endpoint`, `submit_results`. Replace with "write your structured asset list to a JSON file at `${session}/attack-surface.json`".

**Bundled scripts:**
- `scripts/extract-routes.ts` — parses Express/Fastify/Next.js/FastAPI/etc. route files. Optional helper for whitebox mode.
- `scripts/subdomain-enum.sh` — wraps subfinder/amass with sane defaults. Optional.

### 7.3 `authenticated-testing`

**Source:** `authenticationAgent/prompts.ts` has four overlapping blocks (`AUTH_SUBAGENT_SYSTEM_PROMPT`, `buildAuthUserPrompt`, `AuthDiscoverySystemPrompt`, `BrowserFlowGuidance`). Consolidate into one SKILL.md with three sections: scheme detection, login execution (form/JSON/Basic/OAuth/MFA), token/cookie persistence.

**Bundled scripts:**
- `scripts/persist-session.ts` — writes `auth-data.json` with cookies + headers in a stable format the host agent can re-load on subsequent tool calls.
- `scripts/extract-cookies-from-har.ts` — convenience for browser-flow exports.

### 7.4 `vulnerability-testing`

**The crown jewel.** The PentestAgent prompt has four variants (base, exfil, task-driven, task-driven-exfil) with ~160 lines of shared methodology each. The variants become *sections* in a single SKILL.md:

- Default mode (the `base` variant)
- Section: "When pivoting / exfil is in scope" (the `exfil` variant)
- Section: "When the engagement is task-driven" (drops naturally if the host agent doesn't have a task tool)

**Reference content to keep verbatim** (proven valuable):
- Source-code prohibition rule (don't peek at source during blackbox)
- PoC portability rules (POSIX, no GNU-isms, no `bc`)
- Browser-interaction tips (xpath patterns, screenshot library)
- Security-headers analysis (don't false-flag deprecated `X-XSS-Protection`)
- CORS analysis (wildcard *without* credentials is low-risk)
- Rate-limit handling (when to back off vs. when to flag as a finding)

**Bundled scripts:**
- `scripts/poc-scaffold.sh` — POSIX-portable PoC template the agent can fill in.

### 7.5 `threat-model`

Already a skill. Two changes:
1. Replace `create_file` reference with the host agent's generic file write.
2. Verify the 8-phase methodology survives independently of the runtime context wrapper (`buildThreatModelPrompt`).

### 7.6 `vulnerability-patching`

**Source:** `patching/prompts.ts` has both system prompt (8-step workflow + per-CWE best practices) and user-prompt builder (vuln details + dataflow). The system prompt is the SKILL.md; the user-prompt builder dissolves into runtime context the calling skill (or CLI) injects.

**Bundled scripts:**
- `scripts/verify-patch.sh` — runs project's lint → tsc → tests, then re-runs the PoC and asserts it now fails.

### 7.7 `environment-setup`

**Source:** `environment/prompts.ts:7-145`. Six steps: assess compose → create if needed → start → troubleshoot → auth probe → return EnvironmentResult.

**Bundled scripts:**
- `scripts/compose-health-check.sh` — waits for services + probes well-known endpoints.
- `scripts/wait-for-service.sh` — generic TCP/HTTP wait helper.

### 7.8 `pentest-reporting`

**Source:** `src/core/report/renderers/`, `src/core/findings/registry.ts` (dedup + root-cause grouping).

This is mostly a utility skill. SKILL.md is short; the value is in the bundled scripts:
- `scripts/render-markdown.ts`, `scripts/render-json.ts` — straight ports of the existing renderers.
- `scripts/dedupe-findings.ts` — ports `findingsRegistry.groupByRootCause()`.

The `pentest` skill invokes these; nothing else has to.

---

## 8. Migration phases

A sequenced plan that keeps `main` functional throughout. Each phase ends in a green CI build.

### Phase 0 — Conventions & scaffolding (1 PR, ~half day)

- [ ] Add `assets/` subdir to the skill-entry layout (alongside `scripts/`); update `scanner.ts` to discover it.
- [ ] Pick a project-level skills location: `.skills/` (already supported). Document it in `AGENTS.md`.
- [ ] Add a `decisions/PDR-008-agent-skills-migration.md` with the high-level architectural decision.
- [ ] Add a smoke test: `scanSkillRoots()` finds a fixture skill with assets + scripts.

### Phase 1 — Reference skills first (lowest risk, highest reuse)

These are pure data/utility extractions; no behavior changes elsewhere.

- [ ] `cvss-scoring` — port `src/lib/cvss/*` to the skill. Keep `src/lib/cvss/` for now; have it import the bundled JSON (so we get DRY without breaking callers).
- [ ] `cwe-classification` — port `src/lib/cwe/*` similarly.
- [ ] `vulnerability-taxonomy` — extract VULN_CLASS_PATTERNS to YAML; ship Finding/Evidence schemas as JSON.
- [ ] `pentest-wordlists` — copy `assets/wordlists/*` into the skill.
- [ ] `pentest-reporting` — port `src/core/report/renderers/*`.

Acceptance: each reference skill is loadable, its scripts run standalone, existing `src/lib/*` and `src/core/report/*` callers still work because they delegate to the skill assets.

### Phase 2 — Methodology extraction (no behavior change yet)

For each operational skill in §4.1:
- [ ] Write SKILL.md by copy-edit from the corresponding hardcoded prompt.
- [ ] Strip Apex-specific tool references; replace with generic capability descriptions ("write a JSON file at…", "run a shell command", "spawn a sub-agent if useful").
- [ ] Bundle the scripts listed in §7.
- [ ] In the existing TS prompt files, replace the inline string with a load-from-skill call (using the existing `parser.ts`). This keeps current Apex behavior identical but reads prompts from disk.

Acceptance: Apex behaves identically to today, but every prompt is sourced from a skill file. CI green.

### Phase 3 — Replace `run_pentest_workflow` with skill-driven flow

This is the architectural pivot.

- [ ] Rewrite the `pentest` builtin so its instructions describe the *procedure* (orient → map → auth → test → report) instead of mandating a single tool call.
- [ ] Stop registering `run_pentest_workflow` as a tool — let the agent invoke sub-skills directly.
- [ ] Remove the operator approval gate as a hard requirement (host harness handles permissioning).
- [ ] Verify `pensar pentest --target ...` still produces a finding report (now via agent reasoning + skills, not the pipeline).

Risk: this is the first phase where outputs may diverge from today. Add a regression test in `benchmarks/argus` (the existing benchmark suite) and require parity (within tolerance) before merging.

### Phase 4 — Drop the agent classes

- [ ] Delete `src/core/agents/offSecAgent/offensiveSecurityAgent.ts` and all `specialized/*/agent.ts` (the wrapper classes, not the prompts — those are gone in Phase 2).
- [ ] Delete `src/core/workflows/pentest.ts`, `whiteboxAttackSurface.ts`, `threatModel.ts`.
- [ ] Delete `src/core/operator/`, `src/core/tasks/`, `src/core/plan/`, `src/core/messages/`.
- [ ] Delete `src/core/api/blackboxPentest.ts`, `targetedPentest.ts`, `attackSurface.ts`, `threatModel.ts`.
- [ ] Delete `src/core/toolset/` registry; lift the *tool implementations* into a `tools/` package the host harness loads.

Acceptance: `bun run tsc`, `bun run test`, and the headless `pensar pentest` smoke test all pass. Code base is ~6–8k lines lighter.

### Phase 5 — Polish

- [ ] Update `README.md` and `AGENTS.md` to describe the new architecture.
- [ ] Add `decisions/PDR-008` finalizing the architecture.
- [ ] Mark or supersede PDR-002 (pentest vs. operator), PDR-003 (specialized agents), PDR-004 (rigid 7-step) — they describe an architecture we're leaving behind. Keep the originals (don't rewrite history) and add an addendum block linking to PDR-008.
- [ ] Publish the skill bundle to npm or a separate repo so non-Apex consumers (Claude Code users) can install it.
- [ ] Add an example `claude.md` showing a Claude Code session that does a pentest using only the skills.

---

## 9. Risks, trade-offs, open questions

### Risks

1. **Behavior regression on `pensar pentest`.** The current pipeline produces predictable outputs by construction. A skill-driven agent may discover *more* (good) or *miss* something the pipeline catches (bad). Mitigation: hold Phase 3 behind a benchmark-parity gate (Argus / xben suites under `benchmarks/`).

2. **Loss of the operator approval gate.** PDR-002 made a deliberate distinction between automated `/pentest` and interactive `/operator`. If approval gating is important for your customer base, the host harness needs it. Claude Code does have a permission model but the policies are different (per-tool, not per-stage). Open question: do we re-implement stage-aware permissions as a tiny harness shim, or rely on the host agent's permission system?

3. **Skill discovery surface.** Skills are content. Without good defaults, a vanilla Claude Code session won't know to pull in `pentest`. Two viable distribution paths: (a) bundle them as Claude Code "plugin skills" installable via a single command; (b) keep `pensar` as a thin CLI that wraps Claude Code with the skills pre-loaded. Recommendation: do (b) first (lowest user-friction), enable (a) for advanced users.

4. **Benchmark drift.** The benchmark agent and comparison script depend on a specific output schema. If we change the report shape, benchmarks break. Mitigation: keep `pentest-reporting` outputs schema-stable; treat the JSON shape as a public API.

5. **PDR-004 force majeure.** PDR-004 explicitly argued that the rigid 7-step methodology was load-bearing — without it, agents drift, hallucinate, or fail to document. The wager of this migration is that *better models* + *the same methodology in markdown* is sufficient. If that bet is wrong, we lose ground vs. today's `/pentest`.

### Open questions for the user

1. **Distribution model for skills.** Bundle them inside `@pensar/apex` only, or publish them as a separate `@pensar/pentest-skills` package that Claude Code / Cursor / etc. can install standalone? I lean toward standalone — it's the whole point of skills — but it doubles the release surface.

2. **TUI fate.** The TUI's swarm-dashboard and pentest-dashboard are tightly coupled to today's pipeline. Keep TUI as a "thin observer" that watches the host agent's stream, or retire TUI entirely and lean on Claude Code's own UI? PDR-001 made TUI primary; this migration weakens that decision.

3. **Pensar Console integration.** The `pensar pentests dispatch` command tells the Console to run a scan in the cloud. Does the Console run the same skills (so behavior is identical local vs. cloud), or does the Console keep the old pipeline indefinitely?

4. **Operator-mode prompt.** I haven't proposed migrating `/operator` mode (the freeform interactive mode). It's basically already the pattern we're moving toward — a coding agent with security tools loaded. Should `/operator` simply *become* the default and `/pentest` become a skill-invoked variant? That's actually the cleanest end-state.

5. **Are there any methodology bits the user knows are weak today** that we should *re-write* during migration rather than copy verbatim? (e.g. the 7-step methodology in PDR-004 was tuned for a specific model generation.) Migrating verbatim is safe but conserves possibly-stale guidance.

---

## 10. Quick-reference: file → skill mapping

For implementers — every chunk of valuable source mapped to its destination skill.

| Source file (range) | Destination |
|---|---|
| `src/core/agents/specialized/attackSurface/prompts.ts:1-349` | `attack-surface-mapping/SKILL.md` (blackbox section) |
| `src/core/agents/specialized/whiteboxAttackSurface/prompts.ts:1-139` | `attack-surface-mapping/SKILL.md` (whitebox section) |
| `src/core/agents/specialized/authenticationAgent/prompts.ts:11-659` (4 blocks) | `authenticated-testing/SKILL.md` |
| `src/core/agents/specialized/pentest/agent.ts:365-522` (4 variants) | `vulnerability-testing/SKILL.md` |
| `src/core/agents/specialized/pentest/planPrompt.ts:9-32` | `pentest/SKILL.md` (planning section) — or its own skill |
| `src/core/agents/offSecAgent/prompt.ts:115-215` | `pentest/SKILL.md` (top-level orchestration) |
| `src/core/agents/specialized/patching/prompts.ts:7-188` | `vulnerability-patching/SKILL.md` |
| `src/core/agents/specialized/environment/prompts.ts:7-145` | `environment-setup/SKILL.md` |
| `src/core/agents/specialized/findingJudge/index.ts:105-188` | `finding-validation/SKILL.md` (or section in `vulnerability-testing`) |
| `src/core/agents/specialized/cvssScorer/index.ts:147-327` | `cvss-scoring/SKILL.md` |
| `src/core/skills/builtins/threatModel.ts` | `threat-model/SKILL.md` (already exists) |
| `src/lib/cvss/{types,calculator,macrovector-scores}.ts` | `cvss-scoring/{assets/*.json, scripts/score-cvss.ts}` |
| `src/lib/cwe/{cwe-catalog,types,validate}.ts` | `cwe-classification/{assets/cwe-catalog.json, scripts/lookup-cwe.ts}` |
| `src/core/findings/{registry,schemas}.ts`, `src/lib/evidence/types.ts` | `vulnerability-taxonomy/assets/*` + `scripts/classify-finding.ts` |
| `src/core/report/renderers/{markdown,json}.ts` | `pentest-reporting/scripts/render-{markdown,json}.ts` |
| `assets/wordlists/*` | `pentest-wordlists/assets/*` |

Roughly 5,500 lines of prompt text and ~1,500 lines of report/CVSS/CWE utility code move out of TS source and into skill bundles.

---

## 11. Summary

- **What we're moving:** ~5,500 lines of methodology prompts + ~1,500 lines of domain reference code → 11 skills (8 procedural, 3 reference, 1 utility).
- **What we're deleting:** ~6–8k lines of harness orchestration (agent base classes, workflow pipeline, operator gates, task system, plan store, message glue, toolset registry).
- **What we're keeping:** tool implementations (re-homed as host-agent plugins), credential vault, Pensar Console CLI, container, build infra, decision records, TUI shell (pruned).
- **Architectural pivot:** from "Apex drives the model through a fixed pipeline" to "the model drives itself, reading Apex's distilled domain skills."
- **Phasing:** five phases over (estimated) 4–6 weeks, with each phase ending in a green CI build and Phase 3 gated on benchmark parity.

The end state is a thin Apex CLI that pre-loads skills, plus a portable skill bundle that any general coding agent can use to perform pentests, threat models, and patching — no Apex-specific harness required.
