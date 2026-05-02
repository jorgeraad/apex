# MedVault — Healthcare Patient Portal

> "Your records, anywhere." A patient portal and provider-facing FHIR R4 API for a fictional regional health network. The kind of thing your hospital actually ships.

---

## Premise & Vibe

MedVault is the patient-facing portal for "Northbridge Health," a fictional 12-hospital regional network. Patients log in (SSO via their insurer or directly), view their records, message their care team, and request prescription refills. Providers log in to a separate "Clinician Console" that exposes a richer view: patient panels, observations, encounter notes, and an HL7 v2 / FHIR R4 import bridge from legacy EHRs. Admins configure organizations (tenants), provisioned providers, and SAML federation.

The vibe is deliberately enterprise-healthcare: pale blue chrome, "compliance-first" copy in the footer ("HIPAA-aligned audit trail"), a banner in the admin panel that says "Last SOC 2 review: 2025-11-04," and a gratuitous "Powered by FHIR R4" badge in the API docs page. Everything looks safe and audited. It is not.

This demo is intended to feel regulator-adjacent without making legal claims. We frame findings in terms of what a HIPAA-aligned shop would care about: PHI exposure, audit integrity, access controls, and federation trust. Apex's scope guard is the star here — the auditor must NOT exfiltrate real PHI, even from a synthetic dataset, because "PHI exfil" is the kind of thing that makes a buyer immediately stop watching the demo. The demo seeds the database with obviously fake records (Synthea-generated) so the redaction can be obvious on screen.

The target audience for the demo is a healthcare CISO or AppSec lead who has been bitten by Change Healthcare news cycles and wants to know: "Can this thing find what our auditors miss?"

---

## Why This Stack

The healthcare buying center has standardized hard on a few stacks. We want the demo to feel like the thing those buyers actually run in prod, not a toy. That means:

- **Java + Spring Boot.** Roughly every large hospital integration shop runs Spring. Epic's interface engines, Cerner middleware, and most homegrown patient portals in the US are JVM-based. Choosing Spring Boot 3 + Java 21 lets us model JPA-bound bugs, Spring Security misconfigurations, and Actuator exposure — all of which are the actual top findings on healthcare engagements.
- **HAPI FHIR.** HAPI is the de-facto open-source FHIR server in Java. Choosing it lets us model real chained-reference search, `_filter` parameters, and the conformance/CapabilityStatement endpoint that auditors actually probe. It also has real CVE history (CVE-2024-51132, CVE-2024-52007, CVE-2024-45294) for the XXE class.
- **PostgreSQL.** Boring and right. Lets us model row-level access patterns and PostgreSQL-flavor SQLi via JPA `@Query(nativeQuery = true)`.
- **Angular 18.** The provider console is Angular because that's what most enterprise healthcare front-ends look like. Lets us spotlight the Angular router auth-guard pattern (and how it's bypassable when the API doesn't enforce server-side).
- **Keycloak + SAML.** Healthcare loves SAML federation — provider IdPs, payer IdPs, hospital IdPs. Keycloak is the open-source default. CVE-2024-8698 (SAML signature scoping) is fresh in CISO memory, so seeing a SAML misconfig finding lands.
- **nginx in front.** Because everyone has nginx in front, and because it lets us hide one bug in a routing rule.

The stack is deliberately the boring, "we passed audit last year" stack. The point is that boring stacks are exactly where Apex earns its keep.

---

## Stack Details

| Layer | Technology | Version | Notes |
|---|---|---|---|
| Language | Java | 21 (LTS) | Records, sealed types, virtual threads enabled in `application.yaml` |
| Framework | Spring Boot | 3.3.4 | `spring-boot-starter-web`, `-data-jpa`, `-security`, `-actuator` |
| Security | Spring Security | 6.3.x | Form login + SAML2 service provider |
| ORM | Hibernate | 6.5.x | One JPA entity per FHIR resource backing table |
| FHIR | HAPI FHIR | 7.4.0 (intentionally; introduces the modern chained-search surface) | `hapi-fhir-jpaserver-base`, `hapi-fhir-server` |
| HL7 v2 | HAPI HL7v2 (`ca.uhn.hapi:hapi-base`) | 2.5 | Used for the legacy `/api/import/hl7v2` import endpoint |
| DB | PostgreSQL | 16 | Single-tenant per organization via a `tenant_id` discriminator (intentionally weak) |
| Identity | Keycloak | 25.0.x | One realm `medvault`, three clients: `portal-web`, `clinician-web`, `admin-web` |
| Frontend (patient) | Server-side Thymeleaf | 3.1 | Cheaper to build; works for the demo cold open |
| Frontend (clinician) | Angular | 18 | `@angular/router`, `@angular/material` |
| Edge | nginx | 1.27 | TLS terminator + path-based routing to portal vs clinician vs FHIR |
| Cache | Redis | 7 | Used for the password reset token store and Spring session backend |
| Mail | MailHog | latest | Captures password reset emails for the demo |
| Build | Gradle | 8.x | Single multi-module build: `:portal`, `:clinician`, `:fhir`, `:admin`, `:common` |
| Container | Docker Compose | v2 | One file boots the whole target stack |
| Seed data | Synthea | 3.3.0 | Generates ~5,000 synthetic patients with marked-fake SSNs (e.g. `999-XX-XXXX`) |

`application.yaml` highlights for the engineer:

```yaml
spring:
  jpa:
    properties:
      hibernate:
        jdbc:
          time_zone: UTC
  session:
    store-type: redis
management:
  endpoints:
    web:
      exposure:
        include: "*"   # see Vuln V8
  endpoint:
    env:
      show-values: ALWAYS
    heapdump:
      enabled: true
hapi:
  fhir:
    server_address: https://medvault.local/fhir
    cors:
      allow_credentials: true
      allowed_origin:
        - "*"           # see Vuln V11
```

---

## Architecture

```
                           +---------------------+
   Internet --> nginx --> | /                   |  Thymeleaf Patient Portal  (port 8080)
   (TLS)                  | /clinician          |  Angular SPA static + API  (port 8081)
                          | /admin              |  Admin SPA + API           (port 8082)
                          | /fhir               |  HAPI FHIR JPA Server      (port 8083)
                          | /actuator           |  (intentionally proxied)   (-> 8080)
                          +---------------------+
                                    |
                                    v
                          +----------------------+
                          | Spring Boot monolith |
                          |  (4 deployable jars) |
                          +----------------------+
                          |   |        |       |
                          v   v        v       v
                    Postgres Redis  Keycloak  MailHog
```

Service breakdown:

- **portal-svc** (`:portal`) — Patient-facing. Thymeleaf templates, form login, session-based auth. Talks to FHIR over internal HTTP using a service account.
- **clinician-svc** (`:clinician`) — Provider-facing. Serves Angular SPA + JSON API. SAML SSO via Keycloak.
- **admin-svc** (`:admin`) — Org configuration, user provisioning, SAML federation config. SAML SSO via Keycloak with the `admin` role.
- **fhir-svc** (`:fhir`) — HAPI FHIR JPA server. Bearer token (Keycloak JWT) auth via a Spring Security `OncePerRequestFilter`. Reused by all three frontends.
- **common** (`:common`) — Shared JPA entities for the audit log table, the org/user model, and a `TenantContext` ThreadLocal.

Cross-cutting:

- **AuditAspect** — Spring AOP `@Around` on every `@Controller` method writes to `audit_log`. (See Vuln V3.)
- **TenantFilter** — A servlet filter pulls `X-Org-Id` from the request and sets `TenantContext`. (See Vuln V1.)
- **ResetTokenService** — Generates password reset tokens. (See Vuln V6.)

---

## Data Model

PostgreSQL, single schema `medvault`. Selected tables (full DDL lives in `db/migrations/V1__init.sql`):

```
organization        (id PK, name, saml_entity_id, created_at)
app_user            (id PK, org_id FK, email, password_hash, role, mfa_secret nullable)
provider_profile    (user_id PK/FK, npi, specialty, panel_size)
patient             (id PK, org_id FK, mrn, given_name, family_name, dob, ssn,
                     primary_provider_id FK, created_at)
encounter           (id PK, patient_id FK, provider_id FK, encounter_type,
                     started_at, ended_at, location)
observation         (id PK, patient_id FK, encounter_id FK, code_system, code,
                     display, value_quantity, value_unit, effective_at)
medication_request  (id PK, patient_id FK, provider_id FK, rx_code, status,
                     requested_at, filled_at)
message             (id PK, sender_user_id FK, recipient_user_id FK,
                     patient_id FK nullable, subject, body, sent_at)
attachment          (id PK, message_id FK, filename, mime, storage_path)
reset_token         (token PK, user_id FK, created_at, used_at)
audit_log           (id PK, actor_user_id FK, action, target_type,
                     target_id, request_uri, request_body, response_summary,
                     occurred_at)
saml_idp_config     (org_id PK/FK, metadata_xml, sso_url, signing_cert)
fhir_resource       (id PK, resource_type, resource_id, fhir_version,
                     resource_json jsonb, last_updated)   -- HAPI's view
```

Key relationships:

- One `organization` has many `app_user`, many `patient`, many `provider_profile`.
- A `patient` is assigned exactly one `primary_provider_id` but can have many `provider_profile` linkages via `encounter`.
- `audit_log.request_body` is a `text` column — see Vuln V3.

Seed data via Synthea:

- 12 organizations (matching the 12 fictional hospitals).
- 5,000 patients distributed across orgs.
- 3 admin accounts (`admin@northbridge.local`, etc.).
- ~120 providers.
- ~20,000 observations.
- All synthetic SSNs prefixed `999-` so a screenshot can never be confused for real PHI.

---

## Key Routes / Surfaces

| # | Route | Method | Auth | Notes |
|---|---|---|---|---|
| 1 | `/login` | GET/POST | none | Patient form login; sets `JSESSIONID` |
| 2 | `/logout` | POST | session | Should rotate session — does not (Vuln V7) |
| 3 | `/portal/records` | GET | patient | Lists own records via FHIR call |
| 4 | `/portal/messages` | GET/POST | patient | Patient-provider messaging |
| 5 | `/portal/refill` | POST | patient | Prescription refill request |
| 6 | `/portal/account/reset-request` | POST | none | Email reset link |
| 7 | `/portal/account/reset` | GET/POST | reset token | Consumes reset token (Vuln V6) |
| 8 | `/clinician/saml/login` | GET | none | Initiates SAML SP-initiated SSO |
| 9 | `/clinician/saml/acs` | POST | SAML assertion | Assertion Consumer Service |
| 10 | `/clinician/api/panel` | GET | provider | Provider's patient panel |
| 11 | `/clinician/api/patient/{id}` | GET | provider | Patient detail (Vuln V1) |
| 12 | `/clinician/api/patient/{id}/note` | POST | provider | Adds clinical note |
| 13 | `/admin/api/orgs` | GET/POST | admin | Org CRUD |
| 14 | `/admin/api/orgs/{id}/saml` | PUT | admin | Update IdP metadata (Vuln V10) |
| 15 | `/admin/api/users/{id}/role` | PATCH | admin | Role edit (Vuln V2) |
| 16 | `/api/import/hl7v2` | POST | provider | Legacy HL7 v2 ingest (Vuln V4) |
| 17 | `/fhir/metadata` | GET | none | FHIR CapabilityStatement |
| 18 | `/fhir/Patient` | GET | bearer | FHIR search (Vuln V5) |
| 19 | `/fhir/Patient/{id}` | GET | bearer | FHIR read |
| 20 | `/fhir/Patient/{id}/Observation` | GET | bearer | (Vuln V5 IDOR) |
| 21 | `/fhir/$everything` | GET | bearer | Bulk export |
| 22 | `/actuator/env` | GET | none in prod profile | (Vuln V8) |
| 23 | `/actuator/heapdump` | GET | none in prod profile | (Vuln V8) |
| 24 | `/actuator/httpexchanges` | GET | none in prod profile | (Vuln V8) |
| 25 | `/static/*` | GET | none | nginx static fallthrough (Vuln V12) |

---

## Auth Model

Three roles in the system: `PATIENT`, `PROVIDER`, `ADMIN`. The intent is clean RBAC. Reality:

- **Patients** authenticate at `/login` (Spring Security form login). Session via `JSESSIONID`, backed by Redis. `SecurityContext` carries a `MedVaultUserPrincipal` with `userId`, `orgId`, `role=PATIENT`, and `linkedPatientId`. Authorization checks at the controller layer mostly look like `if (principal.getRole() != PATIENT) throw 403;` — they check role, not identity, which is the seed of multiple bugs.
- **Providers** authenticate via SAML SP-initiated SSO against Keycloak. The SAML assertion's `Role` attribute is mapped to a Spring authority. JWT issued for the `/fhir` API has `scope=patient/*.read user/*.read` — overly broad.
- **Admins** authenticate via the same Keycloak realm, different client, different group. Admin role is gated by group membership `medvault-admins` claimed in the SAML assertion.

Cross-tenant boundary: a `TenantFilter` reads `X-Org-Id` (for the clinician console) or derives it from the SAML assertion's `OrgId` attribute (for admin). Patient sessions get `orgId` from `app_user.org_id`. The intent is row-level isolation. The bugs mostly come from this boundary being too trusting.

Service-to-service: portal-svc and clinician-svc call fhir-svc using a service-account JWT issued by Keycloak (`client_credentials`). The service account has `system/*.read system/*.write` — full read/write on all FHIR resources.

---

## Intentional Vulnerabilities

Twelve bugs across Critical/High/Medium/Low. Each is a real bug class with a real-world parallel; the engineer should plant them at the exact file/method called out so Apex's findings can cite real lines.

### Critical

#### V1. BOLA / IDOR on patient detail (`/clinician/api/patient/{id}`)
- **Bug class:** Broken Object-Level Authorization (OWASP API1:2023).
- **File / route:** `clinician-svc/src/main/java/health/medvault/clinician/PatientController.java`, method `getPatient(@PathVariable UUID id, Principal p)`. Only checks role, not whether the provider's panel includes that patient.
- **Exploit storyline:** A provider in Org A authenticates, then iterates `/clinician/api/patient/{id}` against UUIDs harvested from `/clinician/api/panel` of a colleague (or from the `_history` FHIR endpoint). They retrieve patients across all 12 orgs. No `org_id` filter, no panel check.
- **Real-world parallel:** The class behind countless HHS OCR breach notifications; concrete reference is the Cerner / Oracle Health 2025 incident where an old legacy server allowed unauthorized cross-tenant access to patient data, and the genre of "EHR cross-org access" findings tracked by HHS OCR.
- **Expected Apex finding:**
  ```
  [CRITICAL] Broken Object-Level Authorization on GET /clinician/api/patient/{id}
  CWE-639 / OWASP API1:2023
  Evidence: provider account in org=NB-001 successfully retrieved patient
  id=<redacted-uuid> belonging to org=NB-007.
  Fix: enforce panel membership in PatientController#getPatient before
  returning resource. Reject 404 if (patient.org_id != principal.org_id) ||
  (patient.id NOT IN principal.panel_ids).
  ```

#### V2. Vertical privilege escalation via mass-assignment on role
- **Bug class:** Mass assignment / IDOR on role field (OWASP API3:2023).
- **File / route:** `admin-svc/.../UserController.java#updateUser(@RequestBody UserUpdateDto dto)` — DTO accepts `role` as a writable field, controller calls `userService.save(dto)` without filtering. Any authenticated user can hit `PATCH /admin/api/users/{id}/role` because `WebSecurityConfig` only gates `/admin/api/orgs/**`, not `/admin/api/users/**`.
- **Exploit storyline:** A patient logs in, captures their JWT (or session), and PATCHes their own user with `{"role": "ADMIN"}`. Re-login lands in admin console.
- **Real-world parallel:** GitHub 2012 mass-assignment incident (Egor Homakov); same pattern, contemporary stack.
- **Expected Apex finding:** Critical. Cites `WebSecurityConfig#filterChain` line where `/admin/api/orgs/**` is the only matcher, and `UserController#updateUser` for the unfiltered DTO.

#### V3. PHI written verbatim to audit log (SSN/DOB)
- **Bug class:** Sensitive data exposure via logging (CWE-532).
- **File / route:** `common/.../AuditAspect.java#logRequest(JoinPoint jp)` calls `objectMapper.writeValueAsString(jp.getArgs())` and stores it in `audit_log.request_body`. When a provider POSTs `/clinician/api/patient` with a body containing `ssn` and `dob`, those land in the audit log as plaintext. `audit_log` is also exposed to providers via `/clinician/api/audit?since=...` for "compliance review" use.
- **Exploit storyline:** Apex authenticates as a low-privileged provider in Org A, hits `/clinician/api/audit?since=2026-01-01`, and harvests SSNs and DOBs from request bodies of admin-only patient creation events across orgs. The audit log is the smoking gun — the very system meant to ensure HIPAA compliance is the leak.
- **Real-world parallel:** The "PHI in logs" finding genre — see HIPAA Journal coverage of multiple OCR enforcement actions, and the broader `CWE-532` family. Thematically aligned with the 2024 Change Healthcare incident's reminder that data inventory must include logs.
- **Expected Apex finding:** Critical. Quotes one redacted log row, names the aspect, recommends a `@RedactPHI` annotation pattern + Logback masking layout.

#### V4. XXE on HL7 v2 / FHIR XML import
- **Bug class:** XML External Entity (CWE-611).
- **File / route:** `fhir-svc/.../ImportController.java#importHl7v2(@RequestBody String xml)` parses the request via `DocumentBuilderFactory.newInstance().newDocumentBuilder().parse(...)` with neither `setFeature("http://apache.org/xml/features/disallow-doctype-decl", true)` nor disabling external entities. The endpoint accepts HL7 v2 wrapped in an XML envelope and FHIR R4 XML payloads.
- **Exploit storyline:** Apex submits a payload with `<!DOCTYPE foo [<!ENTITY xxe SYSTEM "file:///etc/passwd">]>` and observes the entity expansion in the error response. Then escalates to an out-of-band variant pulling the rendered EC2 IMDSv1 token (the demo environment runs simulated metadata at `http://169.254.169.254`).
- **Real-world parallel:** CVE-2024-51132 — XXE in HAPI FHIR via crafted XML. CVE-2024-52007 and CVE-2024-45294 are the same family in `org.hl7.fhir.core` XSLT.
- **Expected Apex finding:** Critical. Cites CVE-2024-51132 by name, shows the file-read PoC payload it generated, recommends `XMLConstants.FEATURE_SECURE_PROCESSING` plus disabling DTDs.

### High

#### V5. FHIR `_filter` chained-reference injection + IDOR on `Observation`
- **Bug class:** Improper authorization on chained references (FHIR-specific BOLA).
- **File / route:** `fhir-svc/.../FhirAuthorizationInterceptor.java` only checks the top-level `Patient/{id}` against the principal but does not scope `Observation?subject=Patient/{id}` or `_filter=subject re Patient/{id}`. Worse, `_filter` is parsed via HAPI's expression engine which permits `or` clauses, so `_filter=(subject re Patient/abc) or (subject re Patient/xyz)` returns observations for both.
- **Exploit storyline:** Apex authenticates as Patient A (allowed to read their own `Observation`s), then issues `GET /fhir/Patient/{B}/Observation` (denied), then `GET /fhir/Observation?_filter=(subject eq Patient/{A}) or (subject eq Patient/{B})` (allowed by the buggy interceptor) and dumps Patient B's labs. Scope guard limits exfil to one synthetic record for proof.
- **Real-world parallel:** General FHIR access-control class documented at `hl7.org/fhir/security.html`; aligns with the OCR-tracked pattern of EHR APIs leaking cross-patient data when chained queries skip authorization.
- **Expected Apex finding:** High. Names the interceptor, shows the curl that bypassed it, recommends `AuthorizationInterceptor` rule set with `Rule.allow().read().resourcesOfType(Observation.class).inCompartment("Patient", patientId)`.

#### V6. Predictable password reset token (timestamp + email hash)
- **Bug class:** Insufficiently random token (CWE-330).
- **File / route:** `portal-svc/.../ResetTokenService.java#mint(User u)` returns `DigestUtils.md5Hex(u.getEmail() + ":" + System.currentTimeMillis()/1000)`. Stored in Redis with 30-minute TTL.
- **Exploit storyline:** Apex requests a reset for `victim@org.example`, observes the token form via its own account, then iterates timestamps within a +/- 60s window to brute the victim's token. With `MailHog` evidence visible during the demo, the reveal lands.
- **Real-world parallel:** A textbook bug bounty finding. PortSwigger's Web Security Academy documents this exact pattern; many disclosed reports on HackerOne (e.g., the genre summarized in PortSwigger's "Vulnerabilities in other authentication mechanisms"). UUIDv1-based tokens have similar predictability (see Intruder.io "In GUID We Trust").
- **Expected Apex finding:** High. Shows the brute-force PoC (~60 candidate tokens), names the file/line, recommends `SecureRandom` 32-byte token + constant-time lookup.

#### V7. Session fixation — `JSESSIONID` not rotated on login
- **Bug class:** Session fixation (CWE-384).
- **File / route:** `portal-svc/.../WebSecurityConfig.java` — the `formLogin` builder calls `.sessionManagement(s -> s.sessionFixation().none())` (intentionally turned off, framed in a comment as "// keep sticky sessions for nginx affinity"). Spring Security's default would rotate; the override breaks it. The session id is also acceptable via URL `;jsessionid=` because `serverContainer` allows it.
- **Exploit storyline:** Apex starts an unauthenticated session, hands the victim a phishing link with `;jsessionid=...`, victim logs in, attacker now owns the authenticated session.
- **Real-world parallel:** Spring Security's documented session-fixation behavior + CVE-2023-20862 (specifically about logout context not clearing in serialized sessions). The pattern is general; the override is the defect.
- **Expected Apex finding:** High. Cites the `sessionFixation().none()` override line by line; recommends `migrateSession()` (the default).

#### V8. Spring Boot Actuator exposed without redaction
- **Bug class:** Information disclosure / sensitive endpoint exposure.
- **File / route:** `portal-svc/src/main/resources/application-prod.yaml` — `management.endpoints.web.exposure.include: "*"` and `management.endpoint.env.show-values: ALWAYS`. nginx forwards `/actuator/**` to the portal pod.
- **Exploit storyline:** Apex hits `/actuator/env` and pulls `spring.datasource.password`, the Keycloak admin credential, and the SMTP password. Then `/actuator/heapdump` produces an `.hprof` from which Apex extracts in-memory `MedVaultUserPrincipal` objects with active session IDs. Then `/actuator/httpexchanges` shows recent FHIR queries — including patient IDs (themselves PHI in some interpretations).
- **Real-world parallel:** Wiz's "Exploring Spring Boot Actuator Misconfigurations" coverage; CVE-2022-22947 (Spring Cloud Gateway) and CVE-2025-22235 are adjacent but the class is the headline. Repeatedly seen in healthcare engagements.
- **Expected Apex finding:** High. Lists each exposed endpoint with one-line consequence each. Recommends `include: health,info` plus `show-values: NEVER`, plus a Spring Security rule denying `/actuator/**` from the public chain.

### Medium

#### V9. SAML signature scoping (assertion vs response confusion)
- **Bug class:** XML signature wrapping / improper signature scope (CWE-347).
- **File / route:** `clinician-svc/.../SamlAssertionValidator.java#validate(Response r)` checks "is there a valid signature anywhere in the document" but does not verify the signature applies to the specific `<Assertion>` containing the role claim. An attacker who obtains any signed assertion (e.g., from their own org) can wrap it around a forged assertion that elevates them to `medvault-admins`.
- **Exploit storyline:** Apex captures its own SAML response, embeds a second crafted assertion with `Role=ADMIN` while preserving the signature on the original assertion, and re-submits. The validator passes; the role mapper picks up the forged assertion.
- **Real-world parallel:** CVE-2024-8698 in Keycloak — exact same class (XMLSignatureUtil determined signature scope from position rather than from `Reference` element). The demo can name this CVE.
- **Expected Apex finding:** Medium-to-High. Cites CVE-2024-8698, recommends OpenSAML's `SAMLSignatureProfileValidator` plus matching `Reference` URI to the signed assertion's ID.

#### V10. SSRF via SAML IdP metadata URL
- **Bug class:** Server-Side Request Forgery (CWE-918).
- **File / route:** `admin-svc/.../SamlIdpController.java#updateIdpMetadata(@RequestBody MetadataDto dto)` accepts `metadataUrl` and fetches it server-side via `RestTemplate`. No allowlist, no DNS resolution check, no protocol restriction.
- **Exploit storyline:** A compromised admin (or, chained with V2, an escalated patient) sets `metadataUrl=http://169.254.169.254/latest/meta-data/iam/security-credentials/medvault-prod` and the response is rendered back into the error message. AWS credentials leak.
- **Real-world parallel:** Capital One 2019 incident (SSRF to IMDS, AWS credentials, S3 bucket dump) — the canonical reference. The IdP-fetch shape also mirrors a wide class of SAML federation bugs.
- **Expected Apex finding:** Medium. Recommends RFC1918 + link-local + cloud-metadata blocklist, HTTPS-only, and IMDSv2.

### Low

#### V11. CORS misconfiguration on FHIR endpoints
- **Bug class:** Permissive CORS (`allowedOrigin: *` with `allowCredentials: true`).
- **File / route:** `application.yaml` — `hapi.fhir.cors.allowed_origin: "*"` with `allow_credentials: true`. Modern browsers reject this combo, but older Safari and many native mobile webviews honor it.
- **Exploit storyline:** Apex demonstrates that a malicious origin can fetch FHIR resources with cookies attached when reached via webview. Low because real exploitation requires a webview victim, but reportable.
- **Real-world parallel:** General CORS misconfiguration class; PortSwigger's CORS labs cover the canonical case.
- **Expected Apex finding:** Low. Notes that HAPI FHIR's own docs recommend an explicit allowlist.

#### V12. nginx alias traversal on `/static/*`
- **Bug class:** Path traversal via `alias` misconfiguration (the classic missing trailing slash).
- **File / route:** `infra/nginx/medvault.conf`:
  ```
  location /static {
      alias /var/www/medvault-static/;
  }
  ```
  (Note: `location /static` without trailing slash + `alias` ending in `/` allows `/static../` traversal.)
- **Exploit storyline:** Apex requests `/static../app.jar` and downloads the application JAR; from the JAR it extracts `application-prod.yaml` which contains the JWT signing secret.
- **Real-world parallel:** The classic nginx `alias` traversal write-up (Detectify Labs / Orange Tsai). Still found in production today.
- **Expected Apex finding:** Low/Medium depending on what the JAR leaks. Recommends `location /static/` (trailing slash).

---

## Real-World Parallels

Six widely covered industry incidents and CVE families to anchor the demo:

1. **Change Healthcare (2024).** ~190M individuals affected; entry was a Citrix portal without MFA. Frames why "patient portal security" is a CISO priority right now and why scope-guard discipline matters. Coverage: HIPAA Journal; Krebs on Security.
2. **Anthem (2015).** 78.8M individuals; the previous record-holder. Cited as the historical bar healthcare CISOs measure against.
3. **Oracle Health / Cerner (2025).** Cross-tenant access to patient data on a legacy server. Direct mirror of V1 (BOLA on `/clinician/api/patient/{id}`).
4. **Capital One (2019).** SSRF to IMDS yielding AWS creds. Mirror of V10 (SSRF via IdP metadata URL).
5. **CVE-2024-8698 — Keycloak SAML signature scope.** Mirror of V9. Same XML-signature-scoping class.
6. **CVE-2024-51132 — HAPI FHIR XXE.** Mirror of V4. Real CVE on real software the demo target embeds.

---

## Apex Features Showcased

This demo is anchored on four Apex capabilities. The script must touch each, in this order:

1. **`authenticationAgent` for multi-role login.** The portal has three distinct auth flows: form login (patient), SAML SP-initiated SSO (provider), and SAML SSO with admin group (admin). The auth agent resolves all three from a single `auth.yaml` manifest containing seed credentials. The demo shows the agent rotating between roles and producing a per-role session inventory ("3 roles, 3 active sessions, 0 manual logins required").
2. **`crawlAuthenticated` for authenticated crawl.** With each role, Apex crawls the post-login surface, including the Angular SPA (which it executes via the headless browser tool to expand client-rendered routes). The output is fed straight into the surface report. Expected count: ~60 routes for patient, ~140 for provider, ~50 for admin, with overlap deduplicated.
3. **`createAttackSurfaceReport`.** Produces a stratified surface report: public, patient-only, provider-only, admin-only, FHIR-only. The report flags the discrepancy that `/admin/api/users/**` is reachable from a non-admin token (the seed of V2). On screen this is a single dense table the camera can hold on for 4-5 seconds.
4. **Scope guard / PHI exfil prevention.** Before any FHIR or audit-log finding is reported, Apex's scope guard masks all values that match SSN, DOB, MRN, and patient name patterns. The on-screen finding for V3 shows `ssn=999-XX-XXXX [REDACTED]`. The judge agent additionally rejects findings whose only "evidence" is the raw PHI value. The demo explicitly narrates: "We could prove it harder, but we don't need real PHI on screen to prove the bug."

Secondary features that come along for the ride:

- **Findings registry + judge agent** — V11 (CORS) is a candidate to be suppressed by the judge as low-impact for browsers in scope; we surface that decision on screen as a teaching moment.
- **Patching agent** — closes the demo by proposing a unified diff for V1 and V8, then re-running the relevant probes to confirm green.
- **Persistent memory** — second engagement (off-camera) uses memory of V8's redaction settings to short-circuit re-discovery.

---

## Demo Storyline

Total runtime target: **9-11 minutes**.

### Cold open (0:00-0:45)
Camera opens on the Northbridge MedVault marketing page. Trust badges. Patient testimonial. Cut to a terminal: `apex pentest https://medvault.northbridge.local --auth ./auth.yaml --scope-guard healthcare`. Voice-over: "This is a real patient portal stack — Spring Boot 3, HAPI FHIR, Keycloak SAML. Boring. Audited. Let's see what Apex finds."

### Reveal 1 — Surface (0:45-2:30)
Apex spins up the auth agent, logs in as patient, provider, admin in parallel. Authenticated crawl runs. `createAttackSurfaceReport` renders. Camera holds on the surface table; narration calls out the `/actuator/**` row and the `/admin/api/users/**` row visible from a non-admin role — the latter foreshadows V2.

### Reveal 2 — Quick wins (2:30-4:30)
Sub-agent swarm fans out. First findings land: V8 (Actuator) and V12 (nginx alias) come back in under a minute. Apex pulls the JWT signing secret from `/actuator/env`, then re-uses it. The narrator says "Watch what we do with that secret."

### Reveal 3 — The PHI scare (4:30-6:30)
The PHI swarm finds V3 (audit log SSN/DOB) and V5 (FHIR `_filter` chained injection). Camera holds on the redacted finding output. Narrator: "If we wanted, we could pull every SSN in the system. We don't, because Apex's scope guard won't let us — and you wouldn't want a vendor that did." This is the line that lands with the CISO audience.

### Climax — Auth chain (6:30-8:30)
Apex chains V6 (predictable reset token) -> V7 (session fixation) -> V2 (mass-assignment role) -> V10 (SSRF via SAML metadata) and arrives at admin. The findings registry shows the chain visually. The judge agent suppresses one false positive (a spurious Actuator finding on a properly-secured `info` endpoint) and the narrator notes the suppression: "Apex's judge agent already filtered three noise findings."

### Close (8:30-end)
Apex's patching agent proposes a unified diff for V1 and V8, applies it in a sandbox, and re-runs the swarm. The findings registry goes from 12 to 10. The CapabilityStatement and audit log are re-checked. The terminal prints `engagement complete`. Narrator: "12 bugs. 9 minutes. No PHI exfiltrated. Ready for your real environment? `apex install`."

---

## Build Notes

Suggested build order for a 3-5 day spike (single engineer):

**Day 1 — Skeleton.**
- `docker-compose.yaml` with Postgres, Redis, Keycloak, MailHog, nginx.
- Multi-module Gradle skeleton: `:portal`, `:clinician`, `:admin`, `:fhir`, `:common`.
- Base JPA entities + Flyway migrations.
- Synthea seed pipeline (run Synthea once, dump JSON, load with a `:seed` task).

**Day 2 — Auth + FHIR.**
- Spring Security wiring (form login + SAML2 SP).
- Keycloak realm import (`medvault-realm.json`).
- HAPI FHIR JPA server stood up against the same Postgres.
- Plant V7 (session fixation), V8 (Actuator), V11 (CORS), V12 (nginx).

**Day 3 — Application surface.**
- Patient portal templates + controllers.
- Clinician Angular SPA scaffolded (one `ng generate` then static-served).
- Admin SPA (cheap; can be Thymeleaf if Angular is over-budget).
- Plant V1 (BOLA), V2 (mass-assignment), V6 (reset token).

**Day 4 — Healthcare-specific bugs.**
- HL7 v2 import endpoint + plant V4 (XXE).
- FHIR authorization interceptor + plant V5 (chained `_filter`).
- Audit aspect + plant V3 (PHI in logs).
- SAML validator + plant V9 (signature scope).
- IdP metadata controller + plant V10 (SSRF).

**Day 5 — Polish.**
- Marketing landing page with believable copy (no real hospital names).
- Demo `auth.yaml` with seeded credentials.
- `make demo-up` / `make demo-down` make targets.
- Smoke-test script that confirms each of V1-V12 is reachable end-to-end (so the demo can't fail on stage).
- README in `/demos/medvault/` listing the planted bugs and expected findings (for internal use only — do **not** ship to the public demo URL).

Engineering guardrails:

- Never ship real PHI. All seeds via Synthea. SSNs prefixed `999-`. Names from a clearly fictional list. DOBs randomized.
- Lock the demo target to internal DNS (`medvault.local`). The marketing landing page renders only over the internal CA.
- The patching agent's diffs must be reversible by a single `git checkout`; pin the demo to a tag.
- If a CVE is cited in a finding, prefer real CVE IDs for HAPI FHIR (CVE-2024-51132), Keycloak (CVE-2024-8698), Spring Cloud Gateway (CVE-2022-22947), Spring Security (CVE-2023-20862). Do not fabricate CVE IDs.

---

## Recording Notes

- **Resolution:** 2560 x 1440 capture, downscale to 1080p for delivery. The surface report table is dense; do not record at 720p.
- **Terminal theme:** dark background, 16pt monospace. Apex's findings render with ANSI color; ensure the recorder does not strip color.
- **Cuts:** we expect to cut the 9-minute story from ~14 minutes of raw footage. Reserve cut points at the end of each "Reveal" block.
- **B-roll:** capture the MedVault marketing landing page and the Angular clinician dashboard separately at 60fps for the cold open.
- **Redaction:** even though the data is synthetic, blur DOB and SSN columns in any table the camera holds for >2 seconds. This is a trust signal to the audience.
- **Voice-over hits:** scripts must say "synthetic data," "scope guard," and "no real PHI was accessed" at least once each. The CISO audience listens for those exact phrases.
- **Captions:** burn in captions for the climax (6:30-8:30); LinkedIn auto-play is muted.
- **Outro card:** CTA `apex install` + a QR code to the gated demo signup. Hold for 3 seconds.
- **Do NOT show:** the internal seed credentials file, real Keycloak admin UI, any localhost port that might suggest "this is just a docker-compose." Frame it as a deployed environment.
