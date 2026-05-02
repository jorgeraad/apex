# BenefitBridge

*A state-level benefits-application portal where the eligibility engine trusts the wrong field, the caseworker queue leaks SSNs, and a Celery worker happily unpickles whatever you hand it.*

## Premise & Vibe

BenefitBridge is a fictional state portal — call it "Commonwealth of New Albion, Department of Human Services" — that consolidates intake for SNAP (food assistance), LIHEAP (energy aid), and CCDF (child-care subsidies) into a single application. Residents authenticate through a Login.gov-style federated identity provider, complete a long-form application, upload supporting documents (pay stubs, lease, utility bills, birth certificates), and track status. Caseworkers triage incoming cases in a queue UI, request additional documentation, and either deny, approve, or escalate to a supervisor. Supervisors handle high-dollar approvals and appeals.

The visual language is deliberately reverent: USWDS (U.S. Web Design System) typography, the Public Sans typeface, generous whitespace, plain language ("We need a few more documents"), Section 508 accessibility annotations, a banner pattern at the top ("An official website of the Commonwealth of New Albion"), and a footer with NIST 800-53 control IDs and an OMB control number. There are no animations, no marketing language, no dark patterns. The application explicitly states "It is a federal crime to knowingly provide false information" on the certification page. Caseworker dashboards look like government internal tooling — dense tables, no chrome, keyboard shortcuts, audit-log toasts.

The bugs are not gimmicky. They are the bugs that have actually shipped to production in real benefits portals over the past decade: state Medicaid systems leaking PII in admin tools, SNAP portals with broken eligibility state machines, federal contractors leaving DEBUG=True. The premise is that BenefitBridge is the kind of application where a finding genuinely matters — vulnerable people, regulated PII, federal funds. Apex's tone in the demo should match: terse, professional, no swagger.

The narrative the demo carries: a CISO at a state agency, three weeks before an annual FISMA assessment, wants a sanity check on the contractor-built portal before the federal auditors arrive. She has a threat model on her laptop and an engagement letter scoped to one hostname. She runs Apex. Everything that follows is what a competent operator could do over a long weekend with a strong sub-agent swarm and a good registry. Nothing in the demo is implausible; everything is grounded in published advisories or HackerOne reports.

## Why This Stack

Python + Django is the de facto stack for civic tech. 18F's `cg-django-uaa`, USDS's Medicare.gov tooling, the Centers for Medicare & Medicaid Services' QPP, and dozens of state benefits systems are built on Django. The stack also surfaces the exact bug classes we want to demo:

- Django's ORM has well-documented escape-hatches (`QuerySet.extra`, `RawSQL`, `raw()`) that civic-tech contractors reach for when joins get hairy — and these are the historical source of injection bugs in Django apps.
- `permission_required` is the canonical Django access-control decorator and the canonical place where developers forget that "user can view this view" is not the same as "user can view this row."
- `django-allauth` with an OIDC backend mirrors the Login.gov integration that any production .gov benefits portal uses, giving us a realistic federated-auth surface and a realistic place to plant a session-fixation bug.
- HTMX is increasingly mainstream in government tooling (the HTMX team has explicitly courted civic tech) and gives us partial-page rendering with caching pitfalls — perfect for the cross-user CSRF-token reuse bug.
- Celery + Redis is the standard async stack and the standard place where someone passes a model instance through `pickle` without thinking.
- Cloud.gov-style 12-factor deployment with environment-variable config is the standard configuration model and the standard place where a `DEBUG=True` from a staging buildpack survives the promotion to prod.

The stack is also Apex-friendly: Django gives us predictable URL patterns, HTMX gives us crawlable hypermedia surfaces, OIDC gives Apex a real federated-auth flow to reason about, and the threat-model artifact we feed Apex (a real public-sector pattern) maps cleanly onto Django's MTV layout.

## Stack Details

| Layer | Choice | Notes |
|---|---|---|
| Language | Python 3.12 | Type hints throughout; `mypy --strict` on `core/` |
| Web framework | Django 5.0 LTS | Class-based and function-based views mixed (realistic legacy) |
| Auth | django-allauth 0.61 + OIDC | Login.gov-style IdP stub; `acr_values=urn:gov:gsa:ac:classes:sp:PasswordProtectedTransport:id/AAL2` |
| Frontend | HTMX 1.9 + USWDS 3.7 | No SPA; `hx-boost` on nav; partials in `templates/_partials/` |
| DB | PostgreSQL 16 | `pg_trgm` for case search; row-level security NOT enabled (intentional) |
| Cache | Redis 7 | Sessions in DB (intentional choice for the fixation bug); cache for HTML fragments |
| Queue | Celery 5.3 | Default `pickle` serializer (intentional); two workers — `default` and `documents` |
| Storage | S3-compatible (MinIO in dev) | Bucket: `nadhs-applicant-documents`; `BUCKET_PRIVATE = True` but presigned URLs are predictable |
| Edge | nginx 1.25 | TLS termination; `client_max_body_size 25M`; `X-Forwarded-For` trusted (intentional) |
| Runtime | cloud.gov-style buildpacks | `manifest.yml`, `runtime.txt`, `Procfile`; staging and prod share a buildpack |
| CI | GitHub Actions | `bandit`, `pip-audit`, `django-upgrade --check`; intentionally skipped on `hotfix/*` branches |
| Observability | structlog + OpenTelemetry | Logs to stdout; PII filter middleware exists but is bypassed in error pages |

Repository layout:

```
benefitbridge/
  apps/
    accounts/        # allauth glue, OIDC adapter, session handling
    applications/    # the application form, state machine, eligibility engine
    documents/       # uploads, virus-scan task, retrieval
    casework/        # caseworker queue, review, decisioning
    admin_portal/    # supervisor + admin tooling
    audit/           # immutable-ish audit log
  core/
    middleware.py
    settings/
      base.py
      staging.py
      production.py
  manifest.yml
  Procfile
  requirements.txt
```

## Architecture

```
                    +-------------------------+
   Applicant ---->  |  nginx (TLS, 80/443)    | <---- Caseworker / Supervisor
                    +-----------+-------------+
                                |
                    +-----------v-------------+
                    |  Gunicorn (Django app)  |
                    |  4 workers, 2 threads   |
                    +--+----------+-----------+
                       |          |
        +--------------+          +-----------------+
        |                                           |
+-------v-------+                       +-----------v-----------+
|  PostgreSQL   |                       |  Redis (cache + broker)|
|  rds-style    |                       +-----------+-----------+
+-------+-------+                                   |
        |                                  +--------v---------+
        |                                  |  Celery workers  |
        |                                  |  default | docs  |
        |                                  +--------+---------+
        |                                           |
        |                                  +--------v---------+
        +--------------------------------> |  S3 (MinIO)      |
                                           +------------------+

  External:  IdP (Login.gov-style OIDC)  ·  ClamAV REST shim  ·  SES (email)
```

A request lifecycle for a typical applicant action:

1. Browser hits nginx, which forwards to Gunicorn with `X-Forwarded-For` and `X-Forwarded-Proto`.
2. `SecurityMiddleware` -> `SessionMiddleware` -> `AuthenticationMiddleware` -> `PIIRedactionMiddleware` (custom).
3. View renders an HTMX partial; partial is cached via `@cache_page(60)` on a few list views (this is the CSRF-token reuse vector).
4. Document upload posts multipart to `/applications/<id>/documents/` which writes to S3 and enqueues `documents.tasks.scan_and_extract` with the file metadata pickled.
5. Eligibility recompute is a Celery task triggered on state transitions; it pulls policy rules from a JSON fixture and writes results to `EligibilityDetermination`.

## Data Model

Core tables (ER summary; PK `id` UUID v4 unless noted):

| Table | Key fields | Notes |
|---|---|---|
| `accounts_user` | `email`, `subject_id` (OIDC `sub`), `is_caseworker`, `is_supervisor`, `office_id` | Extends `AbstractUser`; `subject_id` unique |
| `accounts_office` | `name`, `region_code`, `tenant_id` | Caseworkers belong to one office; queue scoping is supposed to use this |
| `applications_application` | `applicant_id`, `program` (SNAP/LIHEAP/CCDF), `status`, `submitted_at`, `household_size`, `monthly_income_cents`, `ssn_encrypted`, `ssn_last4` | `status` is a CharField, not an FSM (intentional) |
| `applications_householdmember` | `application_id`, `name`, `dob`, `relationship`, `ssn_encrypted` | Children's SSNs stored same as applicant |
| `applications_eligibilitydetermination` | `application_id`, `program`, `eligible`, `monthly_benefit_cents`, `rationale_json`, `computed_at` | One per program per recompute |
| `documents_document` | `application_id`, `kind`, `s3_key`, `sha256`, `uploaded_by_id`, `scan_status` | `s3_key` pattern: `uploads/{user_id}/{seq}.{ext}` |
| `casework_assignment` | `application_id`, `caseworker_id`, `assigned_at`, `released_at` | Queue assignment |
| `casework_note` | `application_id`, `author_id`, `body`, `visibility` (`internal`/`applicant_visible`) | Markdown, sanitized via `bleach` |
| `casework_decision` | `application_id`, `decided_by_id`, `decision`, `reason_code`, `decided_at` | Decisions are append-only |
| `audit_event` | `actor_id`, `action`, `target_type`, `target_id`, `ip`, `user_agent`, `payload_json`, `created_at` | Append-only via DB trigger |
| `applications_statetransition` | `application_id`, `from_status`, `to_status`, `actor_id`, `at` | Logged but not enforced (intentional) |

Application `status` allowed values (string-typed, no DB-level constraint):
`draft`, `submitted`, `in_review`, `info_requested`, `pending_supervisor`, `approved`, `denied`, `withdrawn`, `appeal_open`, `appeal_resolved`.

The forward-only state machine is *documented* in `apps/applications/state.py` as a dict-of-allowed-transitions but is *not* enforced on save — the actual transition gate is a single function `can_transition(old, new)` that callers must remember to invoke. The `submit()` view forgets.

## Key Routes / Surfaces

Public + applicant routes:

| Method | Path | View | Purpose |
|---|---|---|---|
| GET | `/` | `home` | Marketing landing, USWDS hero |
| GET | `/accessibility/` | `accessibility_statement` | Section 508 statement |
| GET | `/accounts/login/` | allauth | Initiates OIDC flow |
| GET | `/accounts/oidc/callback/` | allauth callback | Token exchange, session establish |
| GET, POST | `/apply/start/` | `start_application` | Creates Application in `draft` |
| GET, POST | `/apply/<uuid>/step/<n>/` | `application_step` | Multi-step wizard (1..7) |
| POST | `/apply/<uuid>/submit/` | `submit_application` | Transitions `draft -> submitted` |
| POST | `/apply/<uuid>/withdraw/` | `withdraw_application` | Transitions to `withdrawn` |
| POST | `/apply/<uuid>/resubmit/` | `resubmit_application` | The state-machine bypass surface |
| GET, POST | `/apply/<uuid>/documents/` | `documents_list` | Upload form + list |
| GET | `/apply/<uuid>/documents/<doc_uuid>/` | `document_download` | Issues presigned URL |
| GET | `/dashboard/` | `applicant_dashboard` | Status of all my applications |

Caseworker + supervisor routes (gated by `permission_required`):

| Method | Path | View | Permission required |
|---|---|---|---|
| GET | `/casework/queue/` | `queue` | `casework.view_assignment` |
| GET | `/casework/case/<uuid>/` | `case_detail` | `casework.view_application` |
| POST | `/casework/case/<uuid>/claim/` | `claim_case` | `casework.assign_self` |
| POST | `/casework/case/<uuid>/note/` | `add_note` | `casework.add_note` |
| POST | `/casework/case/<uuid>/request-info/` | `request_info` | `casework.request_info` |
| POST | `/casework/case/<uuid>/decide/` | `decide_case` | `casework.decide` |
| GET | `/casework/search/` | `case_search` | `casework.search` (raw-SQL surface) |
| GET | `/admin-portal/` | `admin_home` | `admin_portal.access` |
| GET | `/admin-portal/applicants/` | `applicant_list` | `admin_portal.view_applicants` |
| GET | `/admin-portal/applicants/<uuid>/` | `applicant_detail` | full SSN visible here |
| GET | `/admin-portal/exports/<uuid>/` | `export_download` | exports are pickled |

Internal / ops:

| Method | Path | Notes |
|---|---|---|
| GET | `/healthz` | Liveness; returns app version |
| GET | `/readyz` | Readiness; touches DB + Redis |
| GET | `/static/...` | served via WhiteNoise in stage, nginx in prod |
| GET | `/__debug__/` | django-debug-toolbar; gated by `DEBUG` (which is the bug) |

Adjacent .gov subdomains that exist on the demo network (Apex's scope-guard target):

- `benefits.newalbion.gov` — the production target, in scope
- `auth.newalbion.gov` — the IdP, OUT OF SCOPE per engagement letter
- `legacy-snap.newalbion.gov` — a different system, OUT OF SCOPE
- `cms.newalbion.gov` — content site, OUT OF SCOPE

## Auth Model

Identity:
- All human users authenticate via OIDC against `auth.newalbion.gov` (a stub IdP we ship in docker-compose, modeled on Login.gov: PKCE, `acr_values` AAL2, `ial=2` claim).
- `subject_id` (the OIDC `sub`) is the stable join key. `email` is mutable.
- Local password auth is *disabled* for applicants but *enabled* for a single `break-glass` admin account whose creds live in the `manage.py` fixture (intentional finding waiting to happen).

Sessions:
- Server-side sessions in PostgreSQL (`django.contrib.sessions.backends.db`).
- Cookie: `__Host-bb_sessionid`, `Secure`, `HttpOnly`, `SameSite=Lax`.
- The login signal handler does *not* call `request.session.cycle_key()` after IdP callback (the fixation bug).
- Idle timeout: 20 minutes (NIST 800-63B AAL2 nominal). Absolute: 12 hours.

Authorization:
- Two coarse role flags on `User`: `is_caseworker`, `is_supervisor`. Plus Django groups: `Caseworker`, `Supervisor`, `OfficeAdmin`, `SystemAdmin`.
- Per-view gating is via `@permission_required('casework.view_application')`. The permission is granted to the `Caseworker` group at migration time.
- Per-row gating is *supposed* to come from `Application.objects.for_user(user)`, which scopes by `office_id`. Several views forget to call it.
- Document downloads are gated by ownership check on `Application`, not `Document` (so cross-application docs within the same applicant are fine, but...).

CSRF:
- Standard Django CSRF middleware. Token rendered into HTMX partials.
- A few list views are wrapped in `@cache_page(60)` for performance; the cached HTML contains a CSRF token that gets served to multiple users (the bug).

## Intentional Vulnerabilities

Twelve planted findings, severity-graded against CVSS 3.1.

### Critical

| ID | Title | Location | CVSS | Sketch |
|---|---|---|---|---|
| BB-01 | SQL injection via `QuerySet.extra` in case search | `apps/casework/views.py:case_search` | 9.8 | `Application.objects.extra(where=[f"household_size > {request.GET['min_size']}"])` — direct interpolation. Mirrors CVE-2020-7471 (Django `StringAgg` injection) and the historical Django `extra()` warning that has been in the docs since 1.0. |
| BB-02 | Insecure pickle deserialization in Celery export task | `apps/admin_portal/tasks.py:build_export` | 9.8 | Celery configured with default `task_serializer='pickle'`; `build_export.delay(filter_blob=request.GET['q'])` accepts an attacker-controlled base64 blob that is pickled. RCE on worker. Parallels CVE-2021-23727 in `celery` and the long-standing pickle warning in Celery docs. |
| BB-03 | `DEBUG=True` reused from staging buildpack into production | `core/settings/production.py` reads `DJANGO_DEBUG` env var without coercing; cloud.gov-style `manifest.yml` for prod inherits from staging | 9.1 | When `/__debug__/` or any 500 fires, full stack trace + env vars (including `DATABASE_URL`, `SECRET_KEY`, S3 creds) render to the client. Real-world parallel: the 2019 First American Financial leak pattern and multiple `DEBUG=True` disclosures tracked by the CISA "known-exploited" advisories. |

### High

| ID | Title | Location | CVSS | Sketch |
|---|---|---|---|---|
| BB-04 | Tenant isolation bypass in `permission_required` view | `apps/casework/views.py:case_detail` | 8.1 | Decorator gates access to the *view*, but the queryset is `Application.objects.get(pk=pk)` instead of `Application.objects.for_user(request.user).get(pk=pk)`. Any caseworker in any office reads any case. Mirrors the 2022 Optus / 2018 Aadhaar / 2021 PA Department of Health pattern. |
| BB-05 | Mass assignment via `ModelForm` with `fields = '__all__'` | `apps/applications/forms.py:ApplicationForm` | 8.1 | Form whitelists every field including `status`, `monthly_benefit_cents`, `office_id`, `decided_by_id`. Applicant POSTs `status=approved&monthly_benefit_cents=99999`. Parallels the original GitHub mass-assignment incident (2012) and OWASP API #6. |
| BB-06 | Eligibility state-machine bypass via `resubmit` accepting back-transitions | `apps/applications/views.py:resubmit_application` | 7.5 | View does `app.status = 'submitted'; app.save()` from any state including `denied`, with the original (pre-denial) household snapshot. Caseworker queue picks it up as a fresh submission; auto-approve threshold fires on stale data. Parallel: the 2020 California EDD fraud pattern where state machines accepted reactivation. |
| BB-07 | PII (full SSN) rendered in admin queue without field-level redaction | `apps/admin_portal/templates/admin_portal/applicant_detail.html` | 7.5 | Template renders `{{ applicant.ssn_decrypted }}` (a property that decrypts) directly. PIIRedactionMiddleware only acts on JSON responses. Mirrors the 2018 IRS "Get Transcript" pattern and the 2023 Minnesota MNsure SSN exposure. |
| BB-08 | Session not rotated on login (fixation) | `apps/accounts/signals.py:on_user_logged_in` | 7.5 | Handler logs the event but does not call `request.session.cycle_key()`. An attacker who plants a session id (via subdomain cookie scoping or a prior MITM) keeps it post-auth. Mirrors CVE-2017-12794-class issues and OWASP ASVS V3.2.1. |

### Medium

| ID | Title | Location | CVSS | Sketch |
|---|---|---|---|---|
| BB-09 | CSRF token reuse across users via `@cache_page` on HTMX partial | `apps/casework/views.py:queue_partial` | 6.5 | `@cache_page(60)` decorates a partial that includes `{% csrf_token %}`. Cached HTML serves the same token to every caseworker for 60s; one stolen token works for any of them. Parallel: Django 1.11 docs explicitly warn against this; bug reproduces it. |
| BB-10 | Predictable presigned-URL keys on document storage | `apps/documents/storage.py:make_key` | 6.5 | S3 key = `uploads/{user_id}/{seq}.{ext}` where `seq` is monotonic per user. Combined with BB-04, an attacker enumerates `uploads/<victim_uuid>/1.pdf .. N.pdf`. Parallels the 2019 LabCorp / Quest Diagnostics presigned-URL exposures and HackerOne report #237381 (Shopify). |

### Low

| ID | Title | Location | CVSS | Sketch |
|---|---|---|---|---|
| BB-11 | Verbose error pages leak settings on 500 (DEBUG flag side-effect) | inherits from BB-03 | 5.3 | Standalone-rated for the partial-view path that throws on bad UUIDs even with DEBUG flipped off, because a custom 500 handler echoes `request.META`. |
| BB-12 | Audit log writable by application code (no append-only enforcement at app layer) | `apps/audit/models.py` | 4.3 | `AuditEvent.objects.filter(...).delete()` is reachable from a management command shipped to prod. Parallel: Sarbanes-Oxley / FISMA AU-9 control violations seen in OIG audits of state systems. |

A reference fixture loads `demo-attacker@example.com` and three caseworkers (`alex@no.gov`, `bri@no.gov`, `cory@no.gov`) across two offices ("Riverdale Regional" and "Highland County"). The attacker account is a normal applicant with no special privilege; everything in the demo is reachable from that footing plus public reconnaissance plus the engagement-letter-granted ability to authenticate.

Severity rationale, briefly: BB-01, BB-02, BB-03 are Critical because each individually yields full compromise (database, worker RCE, secret material). BB-04, BB-05, BB-06, BB-07, BB-08 are High because they yield broad PII exposure or material business-logic abuse without RCE. BB-09 and BB-10 are Medium because exploitation requires either a stolen-token primitive or composition with another finding. BB-11 and BB-12 are Low because impact is bounded (one error path, one offline command).

## Real-World Parallels

- **Django `extra()` SQLi**: CVE-2020-7471 (Django `StringAgg`), and the long-standing `extra()` deprecation warnings; HackerOne report #888410 against a Django property-management SaaS.
- **Celery pickle RCE**: CVE-2021-23727; Snyk advisory SNYK-PYTHON-CELERY-1293126; the canonical "don't use pickle" guidance in Celery 4+ release notes.
- **Government PII exposure in admin tools**: the 2018 IRS "Get Transcript" SSN exposure; the 2022 Texas DIR breach disclosure; the 2023 Minnesota MNsure incident.
- **Eligibility state-machine bypass**: California EDD fraud wave 2020-2021 (BIA OIG report); USDA OIG audit of state SNAP systems 2019.
- **Mass assignment**: the 2012 GitHub mass-assignment incident (Egor Homakov); OWASP API Security Top 10 2023 #6.
- **Session fixation**: OWASP ASVS V3.2.1; CVE-2017-12794 (Django stack-trace token leak in 500 page) — adjacent class.
- **DEBUG=True in production**: CISA KEV catalog references multiple incidents; Shodan persistently lists thousands of Django sites with DEBUG on.
- **Predictable S3 presigned URLs / direct keys**: HackerOne #237381 (Shopify); 2019 LabCorp/Quest exposures via AMCA.
- **CSRF caching**: Django ticket #16011; documented in Django security docs.
- **Permission decorator misuse**: OWASP IDOR guidance; the 2021 Pennsylvania DoH COVID contact-tracing leak.

The threat-model artifact we feed into Apex via `--threat-model` is modeled on **CISA's "Secure by Design" pledge** structure and **18F's Application Security Guide** — a real document pattern auditors recognize. We ship a sanitized copy at `demos/artifacts/benefitbridge-tm.md` describing data classifications (PII, SPII, FTI), trust boundaries, and the eligibility state machine.

## Apex Features Showcased

- **`/pentest --threat-model demos/artifacts/benefitbridge-tm.md`**: Apex ingests the threat model and uses it to (a) prioritize the eligibility state machine and PII flows, (b) reference specific control IDs (NIST 800-53 AC-3, AU-9) in findings, and (c) auto-generate a risk-acceptance memo template the patching agent can hand back.
- **Scope guard**: when Apex's attack-surface mapper finds links to `auth.newalbion.gov` and `legacy-snap.newalbion.gov`, it pauses, surfaces the out-of-scope hostnames against the engagement letter, and either skips or asks. We script the demo so that Apex politely refuses to crawl `legacy-snap.newalbion.gov` on its own.
- **Accessibility-aware reasoning**: the threat model declares Section 508 compliance as a constraint. Apex's patching agent, when proposing a fix that adds a CAPTCHA or a JS-only flow, flags the accessibility regression and proposes a server-rendered alternative. Demoed on the BB-08 fix.
- **Findings registry + CVSS + judge agent**: each of the 12 findings round-trips through the judge to dedupe BB-03 vs BB-11 (same root cause, different surface) and to re-rate BB-04 upward when Apex realizes it composes with BB-10 for full document exfiltration.
- **Sub-agent swarm**: a recon agent crawls USWDS-rendered pages, an HTMX-aware agent expands `hx-get` partials, a Django-aware agent fingerprints the admin URL prefix, and an OIDC-aware agent walks the `/.well-known/` discovery doc.
- **Patching agent**: produces minimal Django diffs — `for_user(request.user)` insertion for BB-04, `request.session.cycle_key()` for BB-08, `Meta.fields = [...]` allowlist for BB-05 — each with a unit test.
- **Persistent memory**: across a multi-day engagement, Apex remembers the eligibility-state transitions it has already mapped, and on day 2 jumps straight to BB-06 reproduction.
- **Browser/Playwright**: drives the multi-step application wizard, including USWDS file-upload widgets and HTMX `hx-confirm` modals.
- **JS endpoint extraction**: pulls HTMX `hx-get`, `hx-post`, and `hx-trigger` URLs from rendered HTML — a non-trivial superset of the JS-fetch case.
- **Kali container**: only used for `sqlmap` confirmation of BB-01 and `gobuster` against the admin prefix.
- **TUI**: live findings table groups by severity; the operator hits `r` to rerun the BB-06 race against a fresh fixture.

## Demo Storyline

A single 11-minute video, three acts.

**Act 1 — Setup (0:00 - 1:30).** Cold open on the BenefitBridge homepage. Voiceover: "This is BenefitBridge, a state benefits portal. We have an engagement letter, a threat model from the agency's CISO, and four days." Cut to terminal. `apex /pentest --target https://benefits.newalbion.gov --threat-model demos/artifacts/benefitbridge-tm.md --scope demos/artifacts/scope.txt`. Apex's TUI fans out: recon, threat-model digest, scope load.

**Act 2 — Discovery (1:30 - 7:30).**
- 1:30: Recon finds `auth.newalbion.gov` linked from login. Scope guard fires. Apex asks the operator (us) to confirm; we decline. Title card: "Scope guard."
- 2:30: Apex fingerprints Django via `/static/admin/` and the `csrftoken` cookie. JS endpoint extraction pulls 47 HTMX URLs.
- 3:15: BB-09 — Apex notices identical CSRF tokens across two simulated caseworker sessions hitting `/casework/queue/`. Cached partial. Findings registry: 1.
- 4:00: BB-04 — Apex enumerates `/casework/case/<uuid>/` with a low-privilege caseworker token from a different office. Reads cases it shouldn't. Judge agent escalates because of cross-tenant impact. Findings registry: 2.
- 4:45: BB-10 — Apex chains BB-04 with predictable S3 keys. Pulls a (synthetic) pay stub. Title card: "Compose findings."
- 5:30: BB-01 — Apex fuzzes `/casework/search/?min_size=`, sees a Postgres error reflected (DEBUG side-effect, BB-03). `sqlmap` confirms. Findings registry: 4.
- 6:15: BB-03 — full settings dump in 500. SECRET_KEY, DATABASE_URL.
- 7:00: BB-06 — Apex's state-machine reasoner walks the wizard, denies an application as caseworker, then resubmits as applicant. Eligibility re-fires. Title card: "Business-logic reasoning."

**Act 3 — Patch and report (7:30 - 11:00).**
- 7:30: Apex hands the registry to the patching agent. Diffs render in the TUI.
- 8:15: Accessibility-aware moment — for BB-08, the patching agent proposes session rotation; a naive variant adds a JS interstitial; the agent rejects it citing the threat model's 508 constraint and ships a server-side rotation instead.
- 9:00: BB-02 — almost a footnote. Apex spots `pickle` in Celery config, sends a crafted `q=` to `/admin-portal/exports/`, gets a callback on its listener. RCE on the worker. Title card: "Pickle never dies."
- 9:45: Final report renders. CVSS distribution, NIST 800-53 control mapping, exec summary the agency CISO can read. The patching agent's PR has 12 commits, each with a unit test.
- 10:30: Voiceover closes: "Apex doesn't replace the auditor. It gets you to the conversation faster." Fade.

## Build Notes

Day-by-day plan for a single engineer with Django familiarity. 3-5 days.

**Day 1 — Skeleton + auth.**
- `django-admin startproject benefitbridge`. Wire `accounts`, `applications`, `documents`, `casework`, `admin_portal`, `audit` apps.
- USWDS via `django-uswds-forms` or a vendored copy. Public Sans webfont.
- Stand up the OIDC IdP stub in docker-compose: a tiny FastAPI app that serves `/.well-known/openid-configuration` and signs JWTs with a fixed key.
- Wire allauth OIDC. Plant BB-08 by *not* adding `request.session.cycle_key()` to the `user_logged_in` signal handler. Add a TODO comment a real engineer would write: `# TODO(security): rotate session id on login -- see ticket DHS-1142`.
- Postgres, Redis, MinIO via docker-compose. `manage.py migrate`. Seed offices and caseworkers.
- Smoke test: applicant can log in, see empty dashboard.

**Day 2 — Application wizard + state machine.**
- Build the 7-step wizard: identity, household, income, expenses, residency, documents, certify.
- `Application` model with the string-typed `status` and the documented-but-unenforced transition map. Plant BB-06 by writing `resubmit_application` as a 4-line view that just sets `status='submitted'` and saves.
- `ApplicationForm` with `Meta.fields = '__all__'`. Plant BB-05. Add a comment: `# fields = '__all__' to keep this DRY -- ok per code review 2024-03-11`.
- Eligibility engine: a JSON rules file (income thresholds by household size and program) consumed by a Celery task. Make sure the auto-approve path fires on `submitted` regardless of prior state.
- Document upload posting to MinIO with key `uploads/{user_id}/{seq}.{ext}`. Plant BB-10. Presigned URLs valid for 24 hours.

**Day 3 — Casework + admin + bug planting cluster.**
- Caseworker queue with `Application.objects.for_user(user)` defined on the manager, but `case_detail` calls plain `Application.objects.get(pk=...)`. Plant BB-04.
- `case_search` uses `QuerySet.extra(where=[f"household_size > {min_size}"])`. Plant BB-01. Add a comment: `# extra() because annotate+filter was 4x slower in benchmark -- see PR #1842`.
- Admin portal templates render full SSN. Plant BB-07. PIIRedactionMiddleware exists in `core/middleware.py` but only acts on `application/json` responses — leave that asymmetry visible.
- `@cache_page(60)` on `queue_partial`. Plant BB-09.
- Audit log model + management command `cleanup_audit_events` that deletes by date. Plant BB-12.

**Day 4 — Celery, settings, and exports.**
- Celery config: `task_serializer = 'pickle'`, `accept_content = ['pickle', 'json']`. Plant BB-02.
- Build `admin_portal/exports/<uuid>` view that calls `build_export.delay(filter_blob=...)` with the raw query param.
- `core/settings/production.py`: `DEBUG = os.environ.get('DJANGO_DEBUG', 'False') == 'True'` (looks safe) but `manifest.yml` for prod inherits from staging which sets `DJANGO_DEBUG: 'True'`. Plant BB-03.
- Custom 500 handler that includes `request.META` keys for "support". Plant BB-11.
- Write the threat-model artifact at `demos/artifacts/benefitbridge-tm.md`. Sanitize from a real public 18F doc style.
- Write the engagement scope at `demos/artifacts/scope.txt` listing `benefits.newalbion.gov` in scope, the others out.

**Day 5 — Polish, fixtures, demo dry-run.**
- Realistic seed data: 200 applications across statuses, 12 caseworkers in 4 offices, 800 documents (random PDFs of pay stubs).
- A `make demo-reset` target that drops and reseeds.
- USWDS visual pass: banner, footer, accessibility statement, OMB control number in form footer.
- Run Apex against the running stack end-to-end. Capture findings. Confirm all 12 plant. Tune severities so the judge agent's output matches the spec table.
- Record a 2-minute B-roll of the queue UI for the video editor.

Plant-bug discipline: every planted vulnerability gets a one-line comment that a real engineer would have written ("optimization", "TODO", "code review approved"). No `# INTENTIONAL VULN` comments.

## Recording Notes

- Render Apex's TUI at 1920x1080, 16pt JetBrains Mono. The findings table needs to be readable on a phone.
- Use a calm, low-volume voiceover. No music under the discovery section; light ambient under the patching section.
- For the scope-guard moment, hold on the Apex prompt for a full 4 seconds — viewers need to read the out-of-scope hostnames.
- The DEBUG=True stack trace renders too much text to read; pan-and-scan with a highlight rectangle on `SECRET_KEY` and `DATABASE_URL`.
- The pickle RCE callback should land in a side-by-side terminal pane; viewers who know the genre will recognize the pattern instantly without explanation.
- Caption every NIST control reference Apex emits — these are the moments procurement officers will screenshot.
- Closing card: "BenefitBridge is fictional. The bug classes are not." Then the Apex wordmark.
- Total runtime target: 10-12 minutes. Cold open under 30 seconds. No marketing voiceover until the final 20 seconds.
- B-roll shot list: (1) USWDS banner closeup; (2) caseworker queue with the dense table; (3) document upload interaction on the wizard; (4) the engagement letter PDF rendered in a viewer with the in-scope host highlighted; (5) the threat-model artifact opened in a markdown preview; (6) terminal closeups of the findings registry sorting by severity and the patching agent's diff hunk.
- Avoid showing real SSNs even in the synthetic data. Use `900-xx-xxxx` numbers (IRS reserves the 900 block for documentation), and zoom only enough to make the redaction failure obvious without rendering a number that pattern-matches a real one.
- Keep all hostnames under `*.newalbion.gov` (an unregistered fictional state). Do not use any real state's domain even in passing.
- The recording machine should have its clock set forward to a Tuesday morning; weekday timestamps in the audit log read as more credible than weekend ones.
- Final review pass before publishing: a real federal employee on the team should watch the cut and flag any phrasing that could be read as disrespecting the population the portal serves. The bugs are the joke; the applicants are not.
