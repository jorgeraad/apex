# PeoplePort
### Workday-lite for the mid-market: paystubs, PTO, expenses, and an org chart that thinks it knows who you are.

## Premise & Vibe

PeoplePort is a serious-corporate HRIS / payroll portal for companies in the 500 to 5,000 employee range. It looks like an internal tool that someone at a Series-D fintech would actually use on a Tuesday afternoon: muted grays, a dense top-nav, a side panel with the user's reporting line, and a dashboard that surfaces "3 expenses awaiting your approval" in a Bootstrap-flavored card. The tone is intentionally bland, because that is how real HR software feels and that is what defenders are actually expected to harden.

Three personas drive the product:

- **Employee** logs in via their corporate IdP, views the latest paystub, requests PTO, submits expenses, and updates a personal bio.
- **Manager** does everything an employee does, plus approves PTO and expenses for direct reports, and runs a "team comp" report on people below them in the org chart.
- **HR Admin** configures the org chart, edits roles, runs the bulk employee importer, and exports payroll runs to the accounting system.

The vibe: a `/Areas/HR/`-shaped ASP.NET Core MVC + Blazor Server app with Hangfire chugging in the background, behind Azure AD / Entra ID. It feels enterprise. It is also riddled with the exact mistakes that show up in real HRIS pentest reports — stale JWT claims, deserialization on the import endpoint, IDOR on comp data, and a handful of Razor handler bugs that are basically free CVEs waiting to happen.

The demo's job is to let Apex's `/pentest` and `/operator` modes walk this codebase the way an internal red team would, and to do it in a way that produces output a CISO would forward to the engineering org without rewriting it.

## Why This Stack

Most demo apps in this series are Node or Python. PeoplePort is deliberately not, because the largest single share of in-house enterprise HRIS code is still C# / .NET, and a meaningful fraction of bounty-paying real-world targets sit on ASP.NET Core MVC or Blazor Server. We want Apex to be visibly fluent there.

Specific reasons:

- **.NET 8 + ASP.NET Core MVC + Razor Pages** gives us the Razor handler-method confusion class (`OnGet` vs `OnPost`, `[BindProperty(SupportsGet = false)]`) which is a real, frequently-missed bug class with documented incidents.
- **Blazor Server** introduces the SignalR circuit / connection-id reuse class, which almost no SAST tool currently catches. Apex showcasing it is differentiated.
- **EF Core + SQL Server** lets us stage `FromSqlRaw` interpolation SQLi cleanly, and lets the threat model talk about parameterization in a way customers recognize.
- **Azure AD / Entra OIDC** is the actual identity stack at the target customer profile, and it lets us stage stale-claim privilege escalation realistically (claims are cached in the cookie until the next sign-in; the app does its own claims augmentation on top).
- **Hangfire** gives us a believable async surface for the bulk importer — which is where we hide the deserialization sink.
- **`appsettings.json`** is where every real-world enterprise .NET app accidentally checks in a connection string at least once. Apex reads it the way a human would.

## Stack Details

- **Runtime**: .NET 8 LTS, C# 12.
- **Web**: ASP.NET Core 8 MVC + Razor Pages, mixed with Blazor Server for the interactive bits (org chart editor, expense approval queue, PTO calendar).
- **Auth**: `Microsoft.Identity.Web` against Entra ID (OIDC code flow + PKCE), cookie auth with sliding expiration. A custom `IClaimsTransformation` augments the principal with `EmployeeId`, `ManagerId`, `Role`, and `OrgPath` claims pulled from SQL on first hit per session.
- **Data**: Entity Framework Core 8, SQL Server 2022 (LocalDB in dev, full SQL in the demo container).
- **Background jobs**: Hangfire 1.8 with SQL Server storage, dashboard mounted at `/hangfire` (with deliberately weak filter — see vulns).
- **Front-end assets**: Bootstrap 5, a small amount of vanilla JS, no SPA. Blazor Server handles the interactive panes.
- **File storage**: Azure Blob Storage emulator (Azurite) for receipt uploads and import payloads.
- **Email**: MailHog in the container; password-reset / approval-notify flow goes through it.
- **Container**: `mcr.microsoft.com/dotnet/aspnet:8.0`, `mcr.microsoft.com/mssql/server:2022-latest`, Azurite, MailHog, all wired by `docker-compose.yml`.
- **Key NuGet packages (intentionally chosen)**:
  - `Newtonsoft.Json` 12.0.3 (so `TypeNameHandling.All` is available and historically used).
  - `Telerik.Web.UI` references in a legacy `/Legacy/Reports.aspx` shim, modeling CVE-2017-9248 / CVE-2019-18935 territory.
  - `System.Runtime.Serialization.Formatters` for the `BinaryFormatter` import path on the legacy importer.

## Architecture

```
                          Entra ID (OIDC)
                                |
    Browser  ----TLS----  YARP / Kestrel  ----  PeoplePort.Web (ASP.NET Core 8)
                                                 |     |       |
                                                 |     |       +-- Blazor Server hub (/_blazor)
                                                 |     +-- Razor Pages + MVC controllers
                                                 |
                                                 +-- PeoplePort.Application (services, DTOs)
                                                       |
                                                       +-- PeoplePort.Infrastructure
                                                             |   (EF Core, blob, mail)
                                                             |
                                                             +-- SQL Server (peopleport)
                                                             +-- Azurite (receipts, import-blobs)
                                                             +-- MailHog
                                                             +-- Hangfire (jobs DB)
```

Project layout:

```
src/
  PeoplePort.Web/             ASP.NET Core host, Razor Pages, MVC, Blazor components
  PeoplePort.Application/     Services, DTOs, validators
  PeoplePort.Domain/          Entities, value objects, state machines
  PeoplePort.Infrastructure/  EF Core, repositories, blob, mail, Hangfire jobs
  PeoplePort.Importers/       Legacy CSV/XML importer with the deserialization sink
tests/
  PeoplePort.Tests/           xUnit, a thin sanity suite
docker/
  docker-compose.yml
  Dockerfile.web
  init-db.sql
appsettings.json              <-- intentionally has secrets (see vulns)
```

Request lifecycle for an authenticated request:

1. Cookie middleware validates session, hydrates `ClaimsPrincipal`.
2. `OrgClaimsTransformer : IClaimsTransformation` augments principal with `EmployeeId`, `ManagerId`, `Role`, `OrgPath`. Cached in cookie, refreshed only on sign-in. (This is the stale-claims primitive.)
3. MVC / Razor Pages routes through `[Authorize(Policy = "...")]`.
4. Blazor Server: SignalR circuit established at `/_blazor`. Circuit holds the principal at connect time. (This is the circuit-hijack primitive.)
5. Service layer enforces business rules (expense state machine, PTO balance).
6. EF Core writes via `DbContext`. Some read paths use `FromSqlRaw` (vuln).

## Data Model

Core entities (EF Core, code-first, all tables in `dbo`):

| Entity | Key fields | Notes |
|---|---|---|
| `Employee` | `Id` (Guid), `EntraOid`, `Email`, `DisplayName`, `Title`, `ManagerId`, `Role` (`Employee`/`Manager`/`HRAdmin`), `Bio` (nvarchar(max)), `HireDate`, `TerminationDate`, `IsActive` | `Bio` is rendered with `@Html.Raw` (vuln). `Role` is mutable via self-service endpoint (vuln). |
| `Compensation` | `Id`, `EmployeeId`, `EffectiveFrom`, `BaseSalaryCents`, `BonusTargetPct`, `Currency`, `Notes` | Highly sensitive. IDOR target. |
| `Paystub` | `Id`, `EmployeeId`, `PeriodStart`, `PeriodEnd`, `GrossCents`, `NetCents`, `Withholdings` (json), `PdfBlobPath` | PDF served via signed-ish path that doesn't actually verify ownership (vuln). |
| `PtoRequest` | `Id`, `EmployeeId`, `StartDate`, `EndDate`, `Hours`, `Status` (`Pending`/`Approved`/`Rejected`/`Cancelled`), `ApproverId`, `DecisionAt` | State machine has a hole on re-approval. |
| `Expense` | `Id`, `EmployeeId`, `SubmittedAt`, `AmountCents`, `Currency`, `Category`, `Memo`, `ReceiptBlobPath`, `Status` (`Draft`/`Submitted`/`Approved`/`Rejected`/`Reimbursed`), `ApproverId` | Status is updated via action name in the route, not validated against current state (vuln). |
| `OrgNode` | `Id`, `EmployeeId`, `ParentNodeId`, `Path` (`/CEO/CFO/Controller/...`), `Depth` | Editable in Blazor org chart. `Path` is what authorization reads. |
| `Role` | `Id`, `Name`, `Permissions` (json) | Lookup. |
| `ImportJob` | `Id`, `SubmittedById`, `Filename`, `BlobPath`, `Status`, `Format` (`Csv`/`XmlLegacy`/`Bson`), `ResultJson` | `XmlLegacy` and `Bson` paths hit the deserialization sink. |
| `AuditEvent` | `Id`, `At`, `ActorId`, `Action`, `TargetType`, `TargetId`, `Details` (json) | Written for everything except the things we want demoable as silent (vuln: self-promotion path skips audit). |

Relationships: `Employee.ManagerId -> Employee.Id`. `OrgNode.EmployeeId -> Employee.Id`. `OrgNode.ParentNodeId -> OrgNode.Id`. `Compensation.EmployeeId -> Employee.Id` (1:N over time).

Seed data: ~120 employees in a believable corporate tree (CEO -> 4 VPs -> ~15 directors -> ~100 ICs), including a deliberately-named `intern.contractor@peopleport.test` who is the demo's starting persona, and `cfo@peopleport.test` who is the eventual escalation target.

## Key Routes / Surfaces

| Route | Method | Surface | Auth | Notes |
|---|---|---|---|---|
| `/` | GET | Razor Page | Authenticated | Dashboard. |
| `/signin-oidc` | GET | OIDC callback | Public | Entra redirect handler. |
| `/Account/PostLogin` | GET | Razor Page | Authenticated | Reads `?returnUrl=` and redirects (vuln: open redirect). |
| `/Account/SignOut` | POST | MVC | Authenticated | Clears cookie. |
| `/Profile` | GET/POST | Razor Page | Authenticated | Self-edit. `OnPostAsync` updates `Employee` (vuln: also accepts `Role` field). |
| `/Profile/Bio` | GET/POST | Razor Page | Authenticated | Bio editor; rendered with `@Html.Raw` on profile view (vuln: stored XSS). |
| `/Employees` | GET | Razor Page | Manager+ | Directory. |
| `/Employees/{id}` | GET | Razor Page | Authenticated | Public-ish profile view. |
| `/Employees/{id}/Comp` | GET | Razor Page | "ViewComp" policy | Policy only checks `Role == Manager`, not org-chart relationship (vuln: IDOR). |
| `/Paystubs` | GET | Razor Page | Authenticated | Self only — list. |
| `/Paystubs/{id}/Pdf` | GET | MVC controller | Authenticated | Streams blob; checks blob path token, not ownership (vuln). |
| `/Pto` | GET | Blazor page | Authenticated | Calendar + request form. |
| `/Pto/Request` | POST | MVC API | Authenticated | Creates request. |
| `/Pto/{id}/Approve` | POST | MVC API | Manager | Idempotency-key not enforced (vuln: state replay). |
| `/Pto/{id}/Reject` | POST | MVC API | Manager | Same. |
| `/Expenses` | GET | Blazor page | Authenticated | List + submit. |
| `/Expenses/{id}/{action}` | POST | MVC API | Manager | `action` is bound from route as string and dispatched (vuln: mass-action confusion / re-approve). |
| `/OrgChart` | GET | Blazor page | Manager+ | Tree view. |
| `/OrgChart/Edit` | GET | Blazor page | HRAdmin | Editor. SignalR circuit. |
| `/Reports/Comp` | GET | Razor Page | Manager | Calls `FromSqlRaw($"... WHERE Department = '{dept}'")` (vuln: SQLi). |
| `/Admin/Import` | GET/POST | Razor Page | HRAdmin | Upload importer file. Hangfire-enqueued job (vuln: deserialization). |
| `/Admin/Roles` | GET/POST | Razor Page | HRAdmin | Edit roles. |
| `/Legacy/Reports.aspx` | * | Web shim | HRAdmin | Telerik-style endpoint (vuln: CVE-2019-18935 family). |
| `/hangfire` | * | Dashboard | "HangfireAuth" filter | Filter only checks `IsAuthenticated`, not role (vuln). |
| `/_blazor` | WS | SignalR hub | Auth-at-connect | Circuit reuse / hijack (vuln). |
| `/api/me` | GET | API | Authenticated | Returns claims. |
| `/api/me/role` | PATCH | API | Authenticated | Self-update of profile fields (vuln: accepts `role`). |

## Auth Model

Identity is Entra ID via OIDC. The app uses cookie auth for the session and treats the OIDC tokens as pure sign-in artifacts.

Claims pipeline:

1. On sign-in, `OnTokenValidated` pulls `oid`, `preferred_username`, `name` from the ID token.
2. `OrgClaimsTransformer : IClaimsTransformation` runs and adds: `EmployeeId`, `ManagerId`, `Role`, `OrgPath`. These are read from `Employees` and `OrgNodes` and **baked into the auth cookie**. They are not refreshed until the user signs out and back in.
3. Authorization policies:
   - `"Employee"` — authenticated.
   - `"Manager"` — `Role == Manager` or `HRAdmin`.
   - `"HRAdmin"` — `Role == HRAdmin`.
   - `"ViewComp"` — `Role == Manager` or `HRAdmin`. **Does not** check `OrgPath` containment, even though it should.
   - `"ApproveFor"` — handler-based, checks `ManagerId == actor.EmployeeId`. Correct on PTO. Not used on expenses (vuln).

This is the core authz primitive the demo abuses: *the org chart is in the DB and in the claims, but most checks just look at `Role`.* Apex's org-chart-aware authorization tester explicitly walks `OrgPath` and finds endpoints that should constrain by lineage but only constrain by role.

JWT note: there is also a service-to-service JWT used by `Hangfire` jobs, signed with a symmetric key checked into `appsettings.json`. That key plus the connection string make the secrets-leak vuln load-bearing.

## Intentional Vulnerabilities

| # | Severity | Title | Location | Class |
|---|---|---|---|---|
| V1 | Critical | Privilege escalation via self-update accepting `Role` | `PATCH /api/me/role`, `Profile.cshtml.cs::OnPostAsync` | Mass assignment + missing authz |
| V2 | Critical | BinaryFormatter / Newtonsoft `TypeNameHandling.All` on import job | `PeoplePort.Importers/LegacyImporter.cs`, Hangfire `RunImportJob` | Insecure deserialization |
| V3 | Critical | Stale claims allow promoted user to retain higher role across sessions / claims set at sign-in only | `OrgClaimsTransformer.cs` | Authentication / claims caching |
| V4 | Critical | Plaintext secrets in `appsettings.json` (DB conn-string with creds, SAML signing key, Hangfire JWT key) | `src/PeoplePort.Web/appsettings.json` | Secrets management |
| V5 | High | IDOR on `/Employees/{id}/Comp` — any manager can read any employee's comp | `Pages/Employees/Comp.cshtml.cs` | Broken object-level authz |
| V6 | High | Expense state-machine abuse via route-action dispatch | `ExpensesController.Action(Guid id, string action)` | Business-logic / state machine |
| V7 | High | Razor handler confusion: `OnGet` accepts mutating params; missing `[BindProperty(SupportsGet=false)]` | `Pages/Admin/Roles.cshtml.cs` | Framework misuse |
| V8 | High | SQL injection via `FromSqlRaw` string interpolation | `Pages/Reports/Comp.cshtml.cs` | Injection |
| V9 | High | Blazor Server circuit hijack via reusable connection token | `Hub/AuthCircuitHandler.cs` | Session / circuit |
| V10 | Medium | Open redirect on SSO post-login `returnUrl` | `Pages/Account/PostLogin.cshtml.cs` | Open redirect |
| V11 | Medium | Stored XSS in employee bio rendered with `@Html.Raw` | `Pages/Employees/View.cshtml` | XSS |
| V12 | Medium | Hangfire dashboard auth filter only checks `IsAuthenticated` | `Program.cs::UseHangfireDashboard` | Misconfig |
| V13 | Medium | Paystub PDF download authorizes by blob token, not ownership | `PaystubsController.Pdf` | Broken object-level authz |
| V14 | Low | Audit log skips writes on self-update path | `EmployeeService.UpdateSelf` | Logging gap |
| V15 | Low | OIDC `nonce` / `state` cookie marked `SameSite=None` without `Secure` in dev fallback | `Program.cs` | Cookie hygiene |

### Detail on the anchor classes

**V1 — Self-promotion.** `PATCH /api/me/role` and the `Profile` Razor Page both bind a `ProfileUpdateDto` that contains `Role` because the dev who wrote it reused the admin DTO. There is no `[Bind]` allowlist and no separate authorization check on the `Role` field. Sending `{"role":"Manager"}` or `{"role":"HRAdmin"}` writes the new role to `Employees.Role`. This is the start of the privesc chain. Real-world parallel: HackerOne reports against Lattice, Gusto-style apps where `PATCH /me` accepts privileged fields.

**V2 — Deserialization.** The legacy importer accepts `Format=XmlLegacy` (XML with `xsi:type` reaching a Newtonsoft path with `TypeNameHandling = TypeNameHandling.All`) and `Format=Bson` (`BinaryFormatter.Deserialize` on the uploaded blob). Both run inside a Hangfire worker, so RCE happens off-box from the request, which is realistic and lets Apex demonstrate that the patching agent can rewrite both the format-allowlist and the serializer config. CVE references: CVE-2017-9785 (Newtonsoft `TypeNameHandling`), CVE-2019-18935 (Telerik `RadAsyncUpload` JavaScriptSerializer / BinaryFormatter — used by U.S. Census breach 2020 and others).

**V3 — Stale claims.** Because claims are baked into the auth cookie at sign-in time and `IClaimsTransformation` only refreshes from the cookie cache, a user who is *demoted* still has elevated rights until they sign out. Inverse of V1: even if HR rolls back the role in the DB, the attacker keeps the cookie. The demo flips this: attacker uses V1 to write `Role=Manager`, signs out and back in once to refresh claims, and keeps the elevated cookie.

**V4 — appsettings secrets.** `src/PeoplePort.Web/appsettings.json` contains a real-looking `Server=sql;Database=peopleport;User Id=sa;Password=Devpassword!23;` connection string plus a SAML signing PFX path *and* the Hangfire symmetric JWT signing key. Apex flags this as a recurring industrial pattern; real-world parallel: Uber 2016 (creds in private GH repo), countless Azure DevOps leaks.

**V5 — IDOR on comp.** Policy `ViewComp` only requires `Role == Manager`. Any manager (or self-promoted attacker) can `GET /Employees/{anyId}/Comp`. The controller does not check that `target.OrgPath` is a descendant of `actor.OrgPath`. This is exactly the bug class Apex's org-chart-aware authz tester is designed to find.

**V6 — Expense state-machine abuse.** `POST /Expenses/{id}/{action}` dispatches `action` against a dictionary of handlers. Each handler updates `Status` directly. There is no transition table. A rejected expense can be `POST`-ed to `/Expenses/{id}/approve` and will move to `Approved` and trigger a Hangfire payout. Real-world parallel: classic HackerOne reports against Concur, Expensify clones.

**V7 — Razor handler confusion.** `Pages/Admin/Roles.cshtml.cs` has both `OnGet` and `OnPost`. `OnGet` accepts `[BindProperty] public RoleEditModel Edit { get; set; }` without `SupportsGet = false`. Therefore `GET /Admin/Roles?Edit.Name=newRole&Edit.Permissions=...&handler=save` triggers the save path with no CSRF token because antiforgery only checks POST. Real-world parallel: documented in Microsoft Security advisory ADV-2020-0xxx and used by multiple bounty reports against ASP.NET Core 3.x apps.

**V8 — `FromSqlRaw` SQLi.** `Pages/Reports/Comp.cshtml.cs` builds:

```
var rows = await _db.Compensations
    .FromSqlRaw($"SELECT * FROM Compensations WHERE Department = '{dept}'")
    .ToListAsync();
```

`dept` comes from the query string. Classic. Apex's findings registry should flag this with both the dataflow trace and the `FromSqlRaw` usage in a way the patching agent can rewrite to `FromSqlInterpolated` plus parameter validation.

**V9 — Blazor circuit hijack.** The app stores its own `circuitId -> userId` map keyed by a cookie that is `HttpOnly=false` so the JS can render avatars. An attacker who phishes a manager into running a snippet (or who already has stored XSS via V11) can read the circuit token and reconnect to the same circuit, inheriting its server-side state including the principal. Apex's deserialization detection has a sibling check for "stateful SignalR principals." Real-world parallel: Microsoft Blazor Server circuit guidance after .NET 6 GA, `aspnet/Announcements#xxxx`.

**V10 — Open redirect.** `/Account/PostLogin` reads `returnUrl` and calls `Redirect(returnUrl)` with no host-allowlist. Useful to chain with phishing. Real-world parallel: countless OWASP sample bugs and CVE-2023-29331 territory.

**V11 — Stored XSS.** Bio is `nvarchar(max)`, persisted raw, and rendered with `@Html.Raw(Model.Employee.Bio)` on `/Employees/{id}`. Useful to chain with V9 to steal a manager circuit.

**V12 — Hangfire dashboard auth.** The `IDashboardAuthorizationFilter` is:

```
public bool Authorize(DashboardContext ctx) =>
    ctx.GetHttpContext().User.Identity?.IsAuthenticated == true;
```

Any logged-in employee can see and trigger jobs. Real-world parallel: GitHub issue threads on `HangfireIO/Hangfire` flagging this exact misconfig in the wild.

**V13 — Paystub IDOR.** `/Paystubs/{id}/Pdf` checks a HMAC over the blob path but the HMAC is generated server-side at list time and reused; sharing the URL leaks the paystub. Inversely, swapping `id` works because the handler doesn't validate `Paystub.EmployeeId == actor.EmployeeId`.

**V14 — Audit gap.** `EmployeeService.UpdateSelf` deliberately skips `_audit.Write(...)` to model real audit-coverage gaps; Apex's threat-model integration reports this as an evidence gap, not a vuln per se.

**V15 — Cookie hygiene.** Cosmetic, included so the report has a Low-severity entry that gets fixed by a single line.

## Real-World Parallels

- **CVE-2017-9785** — Newtonsoft.Json `TypeNameHandling.All` deserialization gadget chain.
- **CVE-2019-18935** — Telerik UI `RadAsyncUpload` insecure deserialization. Used by APT actors against the U.S. Census Bureau in 2020 (per OIG report OIG-21-005-A).
- **CVE-2023-36049** — ASP.NET Core handler confusion / form-bound parameter on GET.
- **CVE-2018-8284** — .NET Framework remote code execution via deserialization.
- **CVE-2017-9248** — Telerik UI Cryptographic Weakness.
- **HackerOne reports against Lattice, Gusto, Rippling-class HRIS apps**: mass-assignment on `PATCH /me` and IDOR on `/employees/{id}/compensation`. Public disclosures pay $5k–$25k.
- **Workday 2018 advisory**: handler-method exposure on internal admin pages.
- **Equifax 2017** and **Capital One 2019** as the threat-model anchor for "HR-adjacent data leak with regulator interest."
- **Uber 2016** for the appsettings-secrets-in-repo parallel.
- **U.S. Census 2020** for the deserialization-on-import parallel.

## Apex Features Showcased

- **Threat-model-driven `/pentest`.** Apex is fed a one-page TM ("PeoplePort processes PII, comp, paystubs; threat actors include a low-priv employee, a phishing-ed manager, and a contractor with admin import access"). The pentest plan walks each actor and produces a path-graph that finds V1+V3+V5 as a chain.
- **Org-chart-aware authorization testing.** Apex extracts the org chart from the seed (or from `/api/me` traversal) and replays each authorization-bearing endpoint with a peer / sibling / ancestor / random principal to find V5 and V13 mechanically.
- **Deserialization detection.** Apex's static pass flags `JsonConvert.DeserializeObject(..., new JsonSerializerSettings{ TypeNameHandling = TypeNameHandling.All })` and any reference to `BinaryFormatter`, then the swarm validates with a benign gadget.
- **Razor Pages handler analysis.** Apex's .NET pass parses `*.cshtml.cs`, identifies handler methods (`OnGet`, `OnPost`, `OnGet<X>`, named handlers via `?handler=`), and flags `[BindProperty]` without `SupportsGet = false` on pages that mutate.
- **Findings registry + CVSS + judge.** Each of V1..V15 is registered with consistent CVSS vectors; the judge dedupes V1 and the V3 follow-on.
- **Patching agent.** Generates a PR that (a) splits `ProfileUpdateDto` from `AdminProfileUpdateDto`, (b) replaces `FromSqlRaw` with `FromSqlInterpolated`, (c) replaces `BinaryFormatter` with `System.Text.Json` and removes `TypeNameHandling`, (d) tightens the Hangfire dashboard filter, (e) parameterizes the `returnUrl`.
- **Memory.** Run #2 remembers the org-chart traversal, the seeded credentials, and the discovered self-promotion chain, and uses it to short-circuit re-discovery.
- **Playwright.** Drives the Blazor Server expense-approval queue end-to-end to demonstrate V6 visually.
- **Attack-surface map.** Cleanly distinguishes Razor Pages, MVC API, Blazor circuits, and Hangfire dashboard as four sub-surfaces with different auth models.
- **Kali container.** Used for the deserialization gadget delivery, since `ysoserial.net` is the right tool and it lives there.
- **TUI.** Grouped findings by actor (Employee / Manager / HRAdmin) so the demo viewer can see the privesc ladder visually.

## Demo Storyline

The recorded demo is ~12 minutes, single take where possible.

**Cold open (0:00–0:45).** The CISO-flavored voiceover: "PeoplePort runs payroll for 5,000 people. We have 30 minutes." Camera on the dashboard signed in as `intern.contractor@peopleport.test`, role `Employee`. Apex TUI on the right.

**Act 1 — Recon (0:45–2:30).** Run `/threat-model` against the repo. Apex emits a TM JSON identifying the PII / comp / paystub assets, the three actors, and the four sub-surfaces. Then `/attack-surface` enumerates routes. Viewer sees the route table populate.

**Act 2 — Static + first findings (2:30–5:00).** Run `/pentest --plan-from-threat-model`. Apex's static pass surfaces V2 (deserialization), V8 (FromSqlRaw), V11 (Html.Raw), V4 (appsettings secrets), V7 (handler confusion), V12 (Hangfire filter). Findings registry fills. The judge merges duplicates.

**Act 3 — Dynamic privesc chain (5:00–8:30).** The swarm picks up V1 by fuzzing `PATCH /api/me/role` with `role=Manager`. Re-auths, claims update, V3 path observed. With Manager role, hits `/Employees/{cfo-id}/Comp` — V5. Hits `/Reports/Comp?dept=Engineering';--` — V8 confirmed live. Each finding gets an org-chart-overlay screenshot.

**Act 4 — Business-logic and circuit (8:30–10:30).** Apex's operator mode demonstrates V6 by submitting an expense, having the manager reject it, and the attacker re-POSTing to the `approve` action. Then it demonstrates V9 by chaining V11's stored XSS into a circuit-token exfil and reconnecting as the manager. Playwright drives the screen.

**Act 5 — Patch and re-test (10:30–12:00).** Apex emits a PR. The patching agent shows the diff: split DTO, parameterized SQL, removed `TypeNameHandling`, tightened Hangfire filter, encoded bio. CI runs, tests pass, re-pentest finds 0 critical, 0 high, 2 low. Roll credits.

## Build Notes

3 to 5 days of build, single engineer. The principle here is that every vulnerability should be planted *as the engineer is writing the feature*, not bolted on after, because the goal is for the code to look like it was written by a competent-but-distracted developer at a real HRIS company. Comments should be plausible. Variable names should be enterprise-bland. There should be no `// TODO: insecure` markers anywhere — if Apex is going to look smart, it has to be looking at code that doesn't already explain itself.

**Day 1 — scaffolding.**
- `dotnet new sln`, four projects (`Web`, `Application`, `Domain`, `Infrastructure`), one importer project, one test project.
- Wire EF Core with `peopleport` DB, code-first migrations for the entities above. Seed 120 employees + org tree + 24 months of paystubs + sample expenses + sample PTO.
- Wire Entra ID with `Microsoft.Identity.Web`. For local demo, swap to a self-hosted IdentityServer or `dotnet user-jwts` bridge so the recorded demo doesn't require a tenant. Document both modes in `README.md`.
- Wire Hangfire with SQL Server storage, dashboard at `/hangfire`, the deliberately weak filter.
- `docker-compose.yml` with web + sql + azurite + mailhog.

**Day 2 — features (happy path).**
- Razor Pages: `Profile`, `Employees` (list, view, comp), `Paystubs`, `Reports/Comp`, `Admin/Roles`, `Admin/Import`, `Account/PostLogin`.
- MVC API: `/api/me`, `/api/me/role`, `/Pto/...`, `/Expenses/...`.
- Blazor Server: `/Pto`, `/Expenses`, `/OrgChart`, `/OrgChart/Edit`. Use a single `_Host.cshtml`. Implement `AuthCircuitHandler`.
- Implement claims transformer, the four authorization policies.
- Hangfire jobs: `RunImportJob`, `IssuePaystubBatch`, `ReimburseExpense`.

**Day 3 — vulnerabilities.**
- V1: ensure `ProfileUpdateDto.Role` exists and is bound; do not filter in controller.
- V2: in `LegacyImporter.cs`, branch on `Format`. `Bson` -> `BinaryFormatter.Deserialize`. `XmlLegacy` -> Newtonsoft with `TypeNameHandling.All`. Make these run inside Hangfire so the dynamic test reflects realistic async RCE.
- V3: claims transformer only runs on sign-in (`OnTokenValidated`). Verify cookie holds claims through demotion.
- V4: write the secrets into `appsettings.json` (and a `appsettings.Development.json` that overrides them so the dev experience still works, but commit both).
- V5: `ViewComp` policy lambda checks role only.
- V6: `ExpensesController.Action(Guid id, string action)` dispatches by string. No transition table.
- V7: in `Admin/Roles.cshtml.cs`, declare `[BindProperty] public RoleEditModel Edit { get; set; }` with no `SupportsGet=false`. Have `OnGet` call `Edit.ApplyIfHandlerSpecified()`.
- V8: `Pages/Reports/Comp.cshtml.cs` uses `FromSqlRaw($"... '{dept}'")`.
- V9: implement the cookie-based `circuitId` map; expose it to JS via a non-`HttpOnly` cookie.
- V10: `Pages/Account/PostLogin.cshtml.cs` calls `Redirect(returnUrl)` directly.
- V11: bio is rendered `@Html.Raw(Model.Bio)`.
- V12: the dashboard auth filter described above.
- V13: paystub PDF authorization checks the HMAC, not the owner.
- V14: skip audit on `UpdateSelf`.
- V15: cookie config in dev.

**Day 4 — seed data, polish, demo dressing.**
- Realistic names, departments (Eng, Finance, People, GTM, Ops), believable org tree.
- Fake paystubs with plausible withholding lines.
- Five expenses pre-loaded, two pending, one rejected (the V6 target), one approved, one reimbursed.
- Bootstrap polish: muted gray-blue palette, dense rows, no rounded corners over 4px. It must look enterprise.
- README with an "Apex demo runbook" section and a `make demo` that resets the DB to a clean state.

**Day 5 — buffer / fixes / Apex dry-run.**
- Run Apex end-to-end against the container. Tune any finding messages that are confusing in the TUI. Add a `tests/` smoke suite so the post-patch re-test is green.
- Record once for timing.

Out of scope: real Entra tenant, real Azure deployment, real PDF rendering (use a stub SVG-to-PDF converter), real SAML (the SAML signing key in appsettings is decorative). Also out of scope: real PII generation libraries — use `Bogus` for names and emails and call it a day; a CFO named "Octavia Reyes" with a four-month-old expense for a $42 cab from SFO is more than convincing enough.

**Test data calibration.**

Compensation values should land in believable ranges for the seeded titles: ICs at $110k–$180k base, managers at $190k–$240k, directors at $240k–$320k, VPs at $320k–$450k, CFO at $610k. Currency mixed: USD primary, GBP and EUR for ~12% of the population (this matters because V8's SQLi parameter is `dept`, not `currency`, but realistic data makes the demo feel like a real org). Termination dates set on ~6% of `Employee` rows with `IsActive=false` to make the directory look like it has history.

**Hangfire job calibration.**

`RunImportJob` should take 4–8 seconds for a normal CSV import and 18–30 seconds for the legacy paths. This matters for the demo because the deserialization gadget needs to land while the dashboard is showing a "Processing" state — long enough for the camera to capture, short enough that the audience doesn't drift. `IssuePaystubBatch` runs nightly at 02:00 in production but for demo purposes is wired to a "Run now" button on the admin page.

## Recording Notes

- Browser at 1440x900, system font Inter, OS dark mode off (the app is intentionally light-themed to feel corporate). Apex TUI in iTerm at 16pt, dark.
- Pre-resolve all DNS and warm the SQL Server container. Cold-start lag on EF Core's first migration is real and ugly.
- Pre-seed two users:
  - `intern.contractor@peopleport.test` / `Demo!Pass1` — the attacker persona, role `Employee`.
  - `cfo@peopleport.test` / `Demo!Pass1` — the privesc target.
- For the V9 circuit-hijack scene, pre-open a manager session in a second browser profile so the camera can cut to "and now you control their queue."
- Hide the `appsettings.json` reveal until Act 2; it is the moment the audience laughs.
- During the patch act, show the diff in a side panel (split view) so the rewrite of `FromSqlRaw -> FromSqlInterpolated` and `BinaryFormatter -> System.Text.Json` is visually obvious.
- End on the re-pentest report, single screen, "0 Critical / 0 High / 2 Low," with the timestamp delta from the opening shot in the corner. That delta is the marketing line.

**Per-act camera notes.**

- *Act 1.* Hold on the dashboard. Resist the urge to mouse around. Let the viewer read "Welcome back, intern.contractor." and notice that a contractor account is in scope at all. Then cut to the Apex TUI for the threat-model invocation.
- *Act 2.* When the findings registry first populates, briefly hover on V4 (appsettings secrets) so the viewer registers that the connection string includes `User Id=sa`. Do not narrate this. The shot does the work.
- *Act 3.* During the org-chart-aware authorization replay, pin the org-chart visualization on the right half of the screen. Each finding emitted should briefly highlight the path on the chart that produced it. This is the "Apex actually understands hierarchy" moment.
- *Act 4.* The expense-state replay is the most subtle bug for a non-developer audience. Slow down. Show the expense moving from Rejected to Approved. Show the Hangfire job firing the reimbursement. This is the bug class that pays bug bounties.
- *Act 5.* The re-pentest must be on a clean container instance. Have it pre-warmed off-camera so the runtime in the lower right reads under 90 seconds. The implicit message is: full coverage, in the time it takes to grab a coffee.

**Common pitfalls during recording.**

- The Blazor circuit reconnect can fail if the managed user has the dev tools open in the second browser profile; close them before recording.
- The Newtonsoft `TypeNameHandling.All` gadget chain printed by `ysoserial.net --gadget=ObjectDataProvider --formatter=Json.Net` includes a long base64 blob; truncate it visually with a paper overlay or stage it in a paste buffer rather than typing it.
- The `FromSqlRaw` SQLi payload (`Engineering';--`) needs to be URL-encoded in the address bar but plain in the operator log; show the operator log version, not the URL bar version, to keep the audience oriented.
- If the demo container's clock drifts more than two minutes, the OIDC `nonce` validation fails and you will lose ten seconds re-authing. Pin NTP in the compose file.

**Audio direction.**

The narrator should not editorialize while findings are landing on screen. Apex's TUI is the protagonist. Narration should land in the gaps: between the threat-model emission and the static pass, between the privesc chain and the business-logic chain, and on the closing shot. The total narration target is under 90 seconds across a 12-minute cut. The remaining 10+ minutes are screen and sound design — keystrokes, the Hangfire dashboard ping, the soft tick of findings being registered. This is consistent with the rest of the series and consistent with how serious-corporate buyers want to be sold to: not by a voice, by a result.
