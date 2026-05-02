# Atlas LMS — When Whitebox Reads What Blackbox Can't

> A Canvas-style learning management platform for universities, riddled with the kind of Rails-flavored bugs that only show up when your tooling can read the source. Apex switches from blackbox crawling to whitebox `--cwd` mode and the demo turns from "polite recon" into "find, fix, re-test."

---

## Premise & Vibe

Atlas LMS is the kind of mid-sized EdTech platform you would expect a state university or a private liberal-arts college to deploy on a five-year contract. Students log in through their institution, enroll in courses, submit assignments to instructors, take quizzes (with a "proctored exam mode" that locks the browser tab via Turbo Stream heartbeats), receive grades, and download PDF certificates of completion at the end of each course. Instructors author content using a rich-text editor, can build their own quiz banks, and — critically for our story — can edit ERB-flavored "certificate templates" that get rendered server-side when a student finishes a course. Admins manage tenancy across departments: the College of Engineering and the Music Department both live on the same Rails monolith, separated only by a `tenant_id` scope.

The vibe is intentionally serious and a little drab — government-grant-funded EdTech, lots of beige, a 2014 Bootstrap aesthetic refreshed with Hotwire in 2023, a banner across the top reminding faculty about FERPA. The bugs are not hacker-CTF-flag bugs; they are the bugs you actually find in audited Rails apps. The point of the demo is to show Apex peeling them out of the source code, not to wow viewers with neon "PWNED" art.

This is the spec where **whitebox mode shines**. Three of the highest-impact findings in this target are effectively invisible to a blackbox crawler — the SSTI in instructor-authored certificate templates lives behind a permissioned route, the Marshal deserialization is hidden inside a Sidekiq job triggered by a specific MIME type, and the mass-assignment is only reachable through a deeply nested form. When Apex is invoked with `--cwd ./atlas-lms`, the agent reads the Rails source, follows the controllers into the models, and finds the bugs by reading them.

---

## Why This Stack

Ruby on Rails remains the dominant framework for higher-ed and EdTech monoliths. Canvas LMS itself is a Rails app at Instructure's scale; Coursera, Codecademy, and Github (yes, the famous 2012 mass-assignment incident happened on Rails) are or were Rails-shaped. Choosing Rails 7.1 + Hotwire gives us:

- A real, recognizable codebase shape that mirrors what auditors see in production EdTech engagements.
- A rich library of well-documented, real-world Rails CVE classes to seed (mass assignment, ERB SSTI, Active Storage variant RCE, Marshal deserialization, raw-string SQLi).
- Hotwire (Turbo Streams) gives us a modern, realistic vector for an exam-mode bypass that looks like 2026 code, not a 2014 SQLi tutorial.
- Sidekiq + Redis lets us showcase Apex tracing a vulnerability across an async boundary — a known weak spot for blackbox tooling.
- Devise is the de facto auth gem in Rails-land, with its own well-known footguns (`paranoid` mode off by default, timing-attack surface on resets).

The stack also lets the demo lean into one of Apex's biggest differentiators: when the **whitebox agent reads `app/controllers`, `app/jobs`, and `config/initializers`**, it finds a class of bug that no blackbox fuzzer can practically discover.

---

## Stack Details

| Layer | Choice | Notes |
|---|---|---|
| Language | Ruby 3.3.5 | matches current LTS-ish for Rails 7.1 |
| Framework | Rails 7.1.3 | intentionally a minor behind 7.1.5.2 (CVE-2025-24293 fix) |
| Frontend | Hotwire (Turbo 8 + Stimulus 3.2) | Turbo Streams over ActionCable |
| Background | Sidekiq 7.2 | grading, certificate rendering, file scanning |
| Database | PostgreSQL 16 | one DB, `tenant_id` column scoping |
| Cache / queue | Redis 7.2 | Sidekiq + ActionCable backend |
| Auth | Devise 4.9 | `:database_authenticatable, :recoverable, :rememberable, :validatable` |
| Authorization | Pundit 2.3 | policies present but inconsistently applied |
| File storage | Active Storage on MinIO (S3-compat) | with `image_processing` + `mini_magick` |
| Image transforms | image_processing 1.12 + mini_magick 4.12 | vulnerable transform allowlist (see Vulns) |
| Templating | ERB (default) | + a custom `CertificateTemplate` model that calls `ERB.new(user_input).result(binding)` |
| Mail | letter_opener (dev), SMTP (prod) | reset emails go through ActionMailer |
| Container | Docker Compose, Ruby 3.3 slim base + ImageMagick 7.1 | ImageMagick policy.xml left at distro default |
| Observability | Lograge + Sentry stub | useful for the demo to show Apex correlating a 500 with a payload |

---

## Architecture

```
                        ┌──────────────────────────────┐
                        │   Browser (student/faculty)  │
                        │   Turbo + Stimulus           │
                        └───────────────┬──────────────┘
                                        │ HTTPS / WSS
                          ┌─────────────▼────────────┐
                          │   Rails 7.1 monolith     │
                          │   Puma, 4 workers        │
                          │  ┌────────┬───────────┐  │
                          │  │Routes  │Controllers│  │
                          │  ├────────┴───────────┤  │
                          │  │ Pundit policies    │  │
                          │  │ Devise sessions    │  │
                          │  │ ActionCable (WSS)  │  │
                          │  └────────────────────┘  │
                          └──┬─────────────┬─────────┘
                             │             │
                ┌────────────▼──┐    ┌─────▼──────────┐
                │  PostgreSQL   │    │     Redis      │
                │  atlas_lms    │    │ (Sidekiq + AC) │
                └───────────────┘    └─────┬──────────┘
                                           │
                                ┌──────────▼──────────┐
                                │   Sidekiq workers   │
                                │  GradingJob         │
                                │  CertificateJob     │
                                │  AssignmentScanJob  │
                                └──────────┬──────────┘
                                           │
                                ┌──────────▼──────────┐
                                │   MinIO (S3-compat) │
                                │   uploads/...       │
                                └─────────────────────┘
```

Three async paths matter for the demo:

1. **Assignment upload → AssignmentScanJob** runs ImageMagick variant analysis on any image-shaped attachment. This is where the Active Storage transform RCE lives.
2. **Quiz submission → GradingJob** uses an ActiveSupport cache (`Rails.cache.fetch`) keyed on a serialized exam-context blob. The cache is configured with `:file_store` and `Marshal` serialization — the deserialization sink.
3. **Course completion → CertificateJob** renders the instructor-authored ERB template against the student record. Direct ERB SSTI to RCE.

---

## Data Model

```
tenants ──┬─< departments ──< courses ──< enrollments >── users
          │                       │
          │                       ├──< assignments ──< submissions
          │                       │
          │                       ├──< quizzes ──< quiz_attempts ──< answers
          │                       │
          │                       └──< certificate_templates
          │
          └─< lti_integrations  (api_key plaintext)
```

Key tables (abbreviated):

| Table | Notable columns | Notes |
|---|---|---|
| `users` | `id, email, encrypted_password, role, tenant_id, suspended` | `role` enum: `student, instructor, admin`. Targeted by mass-assignment bug. |
| `courses` | `id, title, description, instructor_id, tenant_id, published` | search action uses raw string interpolation |
| `enrollments` | `user_id, course_id, status` | the "did this student take this course" join |
| `assignments` | `id, course_id, title, instructions, due_at` | rich-text via Action Text |
| `submissions` | `id, assignment_id, user_id, attached_file_id` | Active Storage attachment |
| `quizzes` | `id, course_id, exam_mode, time_limit_seconds` | `exam_mode=true` triggers Turbo Stream lockdown channel |
| `quiz_attempts` | `id, quiz_id, user_id, started_at, locked` | `locked` flips on heartbeat |
| `certificate_templates` | `id, course_id, body_erb, signature_image_id` | instructor-authored ERB |
| `lti_integrations` | `id, tenant_id, name, consumer_key, shared_secret` | `shared_secret` stored plaintext in DB **and** mirrored into `config/lti.yml` |
| `api_keys` | `id, user_id, token` | for the mobile app; `token` is SHA1, not bcrypt |

Active Storage uses the default `active_storage_blobs` / `active_storage_attachments` tables on MinIO.

---

## Key Routes / Surfaces

| Method | Path | Controller#action | Auth | Notes |
|---|---|---|---|---|
| GET | `/` | `Pages#home` | public | tenant landing page |
| GET | `/users/sign_in` | `Devise::SessionsController#new` | public | Devise default |
| POST | `/users/password` | `Devise::PasswordsController#create` | public | timing leak |
| GET | `/courses` | `CoursesController#index` | session | search via `?q=` |
| GET | `/courses/:id` | `CoursesController#show` | session | |
| GET | `/courses/:id/grades` | `GradesController#index` | session + Pundit (broken) | IDOR — accepts `?student_id=` |
| GET | `/courses/:id/grades/:student_id` | `GradesController#show` | session | IDOR target |
| POST | `/courses/:id/assignments/:aid/submissions` | `SubmissionsController#create` | session | file upload → Sidekiq |
| GET | `/quizzes/:id/take` | `QuizzesController#take` | session | exam mode handshake |
| WSS | `/cable` channel `ExamModeChannel` | `ExamModeChannel#receive` | session | broadcasts to `course:<id>` — no recipient filtering |
| GET | `/courses/:id/certificate` | `CertificatesController#show` | session | renders ERB template |
| POST | `/admin/certificate_templates/:id` | `CertificateTemplatesController#update` | session, instructor | stores raw ERB |
| GET | `/admin/users` | `Admin::UsersController#index` | admin | |
| PUT | `/users/:id` | `UsersController#update` | session | mass-assignment sink |
| GET | `/lti/launch` | `LtiController#launch` | none (signed) | signed with plaintext shared secret |
| GET | `/uploads/:signed_id/variant/:transform` | `Rails::ActiveStorage::Variants` | session | vulnerable variant params |
| GET | `/api/v1/courses` | `Api::V1::CoursesController#index` | bearer token | accepts SHA1 token |
| GET | `/healthz` | `Pages#healthz` | public | exposes git SHA + Rails env |

---

## Auth Model

- **Authentication** is Devise with database sessions. Passwords are bcrypt. Devise's `paranoid` mode is **off** (default), and `confirm_within` is unset. Reset emails go out via `Devise::Mailer`. There is a separate bearer-token path for the mobile app, where the token is stored as `SHA1(token)` in `api_keys.token`.
- **Authorization** is Pundit. Policies exist for `CoursePolicy`, `SubmissionPolicy`, `CertificateTemplatePolicy`. They are applied via `authorize @course` in some controllers but **forgotten in `GradesController` and `UsersController#update`** — that is what makes the IDOR and the mass-assignment land.
- **Roles** are an integer enum on `users`: `0 = student, 1 = instructor, 2 = admin`. Role assignment goes through `User#role=` with no callback validation, which is why the mass-assignment escalates straight to admin.
- **Tenancy** is enforced by a `current_tenant` helper that reads from the subdomain. Several controllers correctly scope by `tenant_id`; a few (notably `LtiController`) do not.
- **CSRF** is on globally; the API namespace uses `protect_from_forgery with: :null_session`.
- **Sessions** are cookie-based, signed with the Rails secret. The cookie store uses `MessageEncryptor` with the Rails 7.0-style `Marshal` serializer because the app was upgraded from 7.0 and `config.action_dispatch.cookies_serializer` was left at `:marshal`.

---

## Intentional Vulnerabilities

Twelve seeded bugs, ordered by severity. Every class maps to a real, public CVE or HackerOne report — no inventions.

### Critical

| # | Title | Class | Where | Real-world parallel |
|---|---|---|---|---|
| C1 | Mass assignment via `permit!` on `UsersController#update` lets a student promote themselves to admin | A04 / IDOR-mass-assign | `app/controllers/users_controller.rb` line ~42, `params.require(:user).permit!` | GitHub 2012 (Homakov), CVE-2025-2304 (Camaleon CMS) |
| C2 | ERB SSTI in `CertificateTemplate#body_erb` rendered with `ERB.new(template.body_erb).result(binding)` in `CertificateJob` | A03 / SSTI → RCE | `app/jobs/certificate_job.rb` | RailsGoat ERB sink; Invicti-documented ERB SSTI class |
| C3 | Active Storage variant RCE: `params[:t]` and `params[:v]` flow into `blob.variant(t => v)` in `VariantsController#show` | A03 / RCE | image_processing transform allowlist not pinned; Rails 7.1.3 uses pre-fix list | CVE-2025-24293 |
| C4 | Marshal deserialization in `GradingJob`: `Marshal.load(Rails.cache.read(key))` where the key is partly user-controlled via `quiz_attempt.context_blob` | A08 / Insecure Deserialization → RCE | `app/jobs/grading_job.rb` | CVE-2017-0903 (rubygems.org), Trail of Bits "Marshal madness", HackerOne #473888 |

### High

| # | Title | Class | Where | Real-world parallel |
|---|---|---|---|---|
| H1 | Old-school SQLi in course search: `Course.where("title LIKE '%#{params[:q]}%'")` in `CoursesController#index` | A03 / SQLi | `app/controllers/courses_controller.rb` | Rails Brakeman ruleset; OWASP RailsGoat |
| H2 | IDOR on `/courses/:id/grades/:student_id` — controller scopes course but never checks that `student_id == current_user.id` and skips Pundit | A01 / BOLA | `app/controllers/grades_controller.rb` | HackerOne report #2487889; OWASP IDOR cheat sheet |
| H3 | Exam-mode bypass via Turbo Stream injection: `ExamModeChannel#receive` re-broadcasts any `data[:html]` to the course channel without sanitization, letting a malicious classmate unlock another student's locked attempt | A04 / broadcast injection | `app/channels/exam_mode_channel.rb` | Hotwire forum thread on Turbo Stream security; analogous to Slack/Discord webhook abuse patterns |
| H4 | Cookie-store Marshal serializer + leaked `secret_key_base` (committed to `config/credentials/development.key` and copied into `Dockerfile`) gives RCE via signed cookie deserialization | A02 / A08 | `config/application.rb` `cookies_serializer = :marshal`, secret leaked | Public Rails 7.0 default; documented in Greg Molnar's CVE write-ups |

### Medium

| # | Title | Class | Where | Real-world parallel |
|---|---|---|---|---|
| M1 | Devise password-reset user enumeration via timing — paranoid mode off, valid emails get bcrypt-rehashed token + SMTP, invalid emails return immediately. ~700ms delta. | A07 | `config/initializers/devise.rb` | CVE-2024-47057 (Mautic), CVE-2026-26185 (Directus), CVE-2026-33877 (ApostropheCMS) |
| M2 | LTI shared secret stored plaintext in DB and duplicated into `config/lti.yml`, which is shipped in the Docker image and readable via the `/healthz`-adjacent `/uploads/...` directory traversal | A02 | `config/lti.yml`, `LtiIntegration` model | Canvas LMS issue #1545 (LTI key/secret handling); 1EdTech LTI 1.1 deprecation guidance |
| M3 | Stored XSS in assignment rich-text via Action Text where instructors can embed raw `<iframe>` because the sanitizer allowlist was customized to permit `iframe` for "embedded H5P content" | A03 | `config/initializers/action_text.rb` | Canvas XSS (andrew-healey/canvas-lms-vuln); historic jQuery-based Canvas XSS |

### Low

| # | Title | Class | Where | Real-world parallel |
|---|---|---|---|---|
| L1 | `/healthz` exposes git SHA, Rails env, and the count of Sidekiq queues — useful for fingerprinting | A05 | `app/controllers/pages_controller.rb#healthz` | OWASP A05 misconfig; standard pre-engagement fingerprint |
| L2 | API tokens stored as SHA1 (no salt) in `api_keys.token`, so a DB read recovers tokens via rainbow tables | A02 | `app/models/api_key.rb` | Generic weak-hash class; well-trodden audit finding |

Total: 12 findings. C1, C2, C3, C4, H4 are the ones that are dramatically easier to find with whitebox source-reading than with a blackbox crawl.

---

## Real-World Parallels

Each bug is tethered to a real reference so an Apex finding can cite something a viewer can google after the demo:

- **Mass assignment escalation (C1):** GitHub's famous 2012 incident where Egor Homakov pushed a commit to `rails/rails` by mass-assigning `user_id` on an SSH-key form. CVE-2025-2304 (Camaleon CMS) is the recent equivalent: `permit!` instead of explicit allowlist.
- **ERB SSTI (C2):** OWASP RailsGoat documents ERB injection as an "extras" RCE vector. Invicti and TrustedSec both publish writeups on `<%= 7*7 %>`-style payloads and how an instructor-authored template surface is the canonical real-world sink.
- **Active Storage variant RCE (C3):** CVE-2025-24293 (disclosed by OPSWAT Unit 515, August 2025). Apex flagging Rails 7.1.3 with `image_processing` + `mini_magick` is exactly the pattern the CVE describes; Rails 7.1.5.2 patches it.
- **Marshal deserialization (C4):** CVE-2017-0903 (rubygems.org RCE). Trail of Bits' August 2025 "Marshal madness" post is the modern survey. HackerOne #473888 is the canonical Rails report.
- **SQLi via interpolation (H1):** the textbook example in Rails Brakeman documentation; the kind of finding Brakeman has shipped a check for since 2010.
- **IDOR on grades (H2):** HackerOne report #2487889 is the widely-cited "IDOR allows viewing other users' data" archetype. GitHub's own bounty page documents IDOR as their top-paid class.
- **Turbo Stream injection (H3):** the Hotwire community forum thread "Turbo Stream Security" raises this exact class. Snyk has tracked broadcast-injection issues in `@hotwired/turbo-rails`.
- **Cookie-store Marshal RCE (H4):** Greg Molnar's blog and the Rails 7.1 release notes call out the move from `Marshal` to `JSON` as the default `MessageEncryptor` serializer specifically to close this class.
- **Devise reset timing (M1):** CVE-2024-47057 (Mautic), CVE-2026-26185 (Directus), CVE-2026-33877 (ApostropheCMS). Devise's own docs recommend `paranoid` mode for this reason.
- **LTI plaintext secrets (M2):** Canvas LMS GitHub issue #1545 (SpeedGrader launching LTI apps with the wrong consumer key/secret) and 1EdTech's own LTI 1.1 → 1.3 migration guidance both name this anti-pattern.
- **Stored XSS via custom sanitizer (M3):** the andrew-healey/canvas-lms-vuln writeup and the historical jQuery-era Canvas XSS issues.

---

## Apex Features Showcased

This is the spec where whitebox carries the demo. Anchor features:

1. **Whitebox `--cwd ./atlas-lms`** — the agent reads `app/`, `config/`, and `Gemfile.lock`, builds a route map by parsing `config/routes.rb`, and seeds its threat model from the source. This is the headline.
2. **Cross-mode value** — Apex first runs `/pentest --blackbox` and finds H1, H2, M1, L1, L2, M3 — solid but expected. Then it re-runs with `--cwd` and surfaces C1, C2, C3, C4, H3, H4. The demo explicitly contrasts the two finding lists to show that whitebox is finding things blackbox literally cannot reach.
3. **Sub-agent swarm** — `surface-mapper`, `auth-prober`, `source-reader`, `ssti-specialist`, `deserialization-specialist`, and `judge` agents fan out. The source-reader and SSTI specialist are the heroes here.
4. **Findings registry + judge** — duplicates collapse (e.g. variant transform RCE and `image_processing` Gemfile pin become one finding with two evidence trails), and the judge agent assigns CVSS scores citing the matching real CVE.
5. **Patching agent — Rails-flavored** — for the mass-assignment bug, Apex generates a minimal patch:
   - Replaces `params.require(:user).permit!` with an explicit allowlist (`:email, :name, :time_zone`).
   - Adds a `before_action :authorize_self!` guard.
   - Generates a request spec proving a student can no longer set `role: 2`.
   - Re-runs the original exploit, confirms it now 403s.
6. **Persistent memory** — the engagement remembers, between runs, that this codebase uses Pundit policies and that several controllers skip them. On the second run Apex starts by greping for missing `authorize` calls.
7. **Browser/Playwright** — used during the blackbox phase to drive the Devise login and exercise the Turbo Stream channel for H3.
8. **JS endpoint extraction** — finds `/api/v1/courses` and `/cable` from the compiled Stimulus bundle.
9. **Threat modeling** — Apex generates a STRIDE-flavored model from the route map and cross-refs it against findings.
10. **Headless CLI / Kali container** — the variant RCE (C3) is demonstrated with a payload generated in the Kali container via ImageMagick + `magick identify`.

---

## Demo Storyline

Total runtime target: 7-9 minutes of recorded video.

**Act 1 — Blackbox (0:00 - 2:30).** Apex launches `/pentest https://atlas-lms.demo` with no source access. The surface mapper finds the Devise endpoints, the course catalog, and the API. Within ~90 seconds Apex reports six findings: the SQLi in course search (H1), the grades IDOR (H2), the Devise reset timing leak (M1), the `/healthz` info disclosure (L1), the SHA1-token weakness inferred from the mobile API response shape (L2), and the Action Text iframe XSS (M3). The narrator pauses on the IDOR finding and the patch-ready summary. So far, normal blackbox engagement.

**Act 2 — Switching to whitebox (2:30 - 3:00).** The narrator says: "this is a real engagement — the customer gave us read-only source access." The operator runs:

```
apex /pentest https://atlas-lms.demo --cwd ./atlas-lms --resume
```

Apex resumes the engagement, hashes the source tree, and the source-reader sub-agent kicks off. Within seconds the routes map fills in with controller names and policies-or-not.

**Act 3 — The whitebox haul (3:00 - 5:30).** Findings start landing that no blackbox tool would have surfaced:

- The SSTI specialist reads `app/jobs/certificate_job.rb`, sees `ERB.new(template.body_erb).result(binding)`, traces back to the instructor route, and proves RCE end-to-end with a `<%= \`id\` %>` payload submitted as a certificate template. (C2)
- The deserialization specialist greps for `Marshal.load`, finds the Sidekiq cache path, reads `config/application.rb`, sees `cookies_serializer = :marshal`, and notices `config/credentials/development.key` was copied into the Docker image. (C4 + H4)
- The source-reader notices `params.require(:user).permit!` in `UsersController#update` and proves a student can become an admin in one PUT. (C1)
- The Active Storage variant RCE is identified by reading `Gemfile.lock` (Rails 7.1.3, image_processing 1.12) and matching the controller code against CVE-2025-24293. (C3)
- The Turbo Stream broadcast injection (H3) is recognized by reading `app/channels/exam_mode_channel.rb` and noticing the channel rebroadcasts payloads without recipient filtering.

**Act 4 — The patch (5:30 - 7:00).** The narrator picks the mass-assignment bug as the demo target for the patching agent. Apex:

1. Drafts a unified diff.
2. Runs the diff through the project's RSpec suite locally (the spec is included in the seed repo).
3. Generates a new request spec that asserts `expect { put :update, params: { user: { role: "admin" } } }.not_to change { user.reload.role }`.
4. Re-executes the original exploit and shows the 403.
5. Saves the patch to `findings/C1.patch` for the engagement report.

**Act 5 — Wrap (7:00 - end).** Apex prints the final report: 12 findings, CVSS-scored, deduped, each linked to a real public CVE or H1 report. The narrator highlights the contrast: 6 in blackbox, 12 with whitebox, and one of those (C2) was a clean RCE that no fuzzer would have found.

---

## Build Notes

Engineer scaffolding the target should plan for **3 to 5 days** of work. The skeleton is a stock `rails new` so most of the time is in seed data and bug placement, not framework wiring.

**Day 1 — skeleton.**

- `rails new atlas_lms -d postgresql --css=tailwind --javascript=esbuild --hotwire`.
- Add Devise, Pundit, Sidekiq, ActiveStorage with MinIO, image_processing + mini_magick.
- Models: `Tenant, Department, User, Course, Enrollment, Assignment, Submission, Quiz, QuizAttempt, Answer, CertificateTemplate, LtiIntegration, ApiKey`.
- Devise install with `:database_authenticatable, :recoverable, :rememberable, :validatable`. Do **not** enable `:confirmable` or `paranoid`.
- Pundit install. Generate policies for Course, Submission, CertificateTemplate, User. Wire `authorize` correctly in 70% of controllers.

**Day 2 — surfaces.**

- Course catalog with the `?q=` search using raw interpolation (H1).
- Grades pages with the IDOR (H2).
- Assignment upload with Active Storage and a `SubmissionsController#create` that schedules `AssignmentScanJob` (C3 path).
- Quiz take page with Turbo Stream lockdown channel `ExamModeChannel` (H3).
- Devise reset email, default config, no paranoid mode (M1).
- Action Text initializer customized to allow `iframe` (M3).

**Day 3 — the dramatic bugs.**

- `CertificateTemplate` with `body_erb` text column. `CertificatesController#show` and `CertificateJob` both call `ERB.new(...).result(binding)` on it. (C2)
- `UsersController#update` with `params.require(:user).permit!` and no Pundit `authorize`. (C1)
- `GradingJob` with `Marshal.load(Rails.cache.read(...))`. Rails.cache configured with `config.cache_store = :file_store`. (C4)
- `config/application.rb`: `config.action_dispatch.cookies_serializer = :marshal`. Commit `config/credentials/development.key` to the repo. (H4)
- `LtiIntegration` model + `config/lti.yml` with plaintext shared_secret. (M2)
- `api_keys.token` populated with `Digest::SHA1.hexdigest(token)`. (L2)
- `/healthz` returns `{ rails_env:, git_sha:, sidekiq_queues:, ruby_version: }`. (L1)

**Day 4 — seeds and Docker.**

- Seed three tenants (Engineering, Music, Continuing Ed), 25 instructors, 1,500 students, 80 courses, 400 assignments, 200 quizzes, 6,000 submissions, 8,000 quiz attempts. Use realistic course titles ("CS 343 — Operating Systems", "MUSC 220 — Counterpoint", "BIOL 101").
- One real-looking LTI integration ("Perusall", "Pearson MyLab") per tenant.
- Build a `Dockerfile` that pins Rails 7.1.3 explicitly (do not let `bundle update` bump it past 7.1.5.2). Pin `image_processing 1.12.2`, `mini_magick 4.12.0`. Leave `/etc/ImageMagick-7/policy.xml` at distro default.
- `docker-compose.yml` with services: `web`, `worker`, `db`, `redis`, `minio`, `mailhog`. Expose 3000.

**Day 5 — polish, tests, recording prep.**

- Write a passing RSpec suite that does not exercise the bugs (so the patching agent can show "tests still pass after fix").
- Add Brakeman to the Gemfile but do not run it in CI — the demo's punchline is that Apex finds things Brakeman would also flag plus things it would miss (especially cross-job dataflow).
- Write a `seed_bugs.rb` rake task that resets the vulnerable state if a previous demo modified it (e.g., re-seeds the certificate template body).
- Create `bin/demo-reset` that wipes Sidekiq queues, re-seeds, restarts containers.
- Record one rehearsal pass end-to-end and time each act.

**Bug placement guidance.**

- Do not collapse all bugs into one file. Real Rails apps spread misconfig across `config/`, `app/controllers`, `app/jobs`, `Gemfile.lock`, and `Dockerfile`. The whitebox demo is more compelling when Apex has to read across files.
- Make the SSTI bug discoverable but not screaming — the `body_erb` column should look like a normal templating surface; the bug is that it is rendered with `result(binding)` instead of through a sandboxed renderer like Liquid.
- Keep at least one bug intentionally subtle: H4 (cookie-store Marshal + leaked key) requires Apex to correlate three files. That is the moment where viewers say "oh, that is what whitebox buys you."

**Out of scope.**

- Real OAuth-based LTI 1.3. Stick with LTI 1.1 + plaintext shared secret. That is the bug.
- Real proctoring (camera, etc.). The "exam mode" is just the Turbo Stream lockdown channel.
- Multi-region / production-grade infra. One Docker host, one MinIO, one Postgres.

---

## Recording Notes

- **Branding.** Atlas LMS logo is a flat blue mountain glyph; favicon should match. The login page should look like Canvas-meets-mid-2010s-bootstrap. Resist the urge to make it pretty — boring is on-brand.
- **Tenant for the demo.** Use the "College of Engineering" tenant. Seed a believable demo student account (`student@atlas-demo.edu`) and instructor (`prof.kim@atlas-demo.edu`).
- **Camera path.** The viewer should never see the seed data setup. Always start from a `bin/demo-reset` clean state. The first thing on screen is the Apex CLI prompt.
- **CLI sizing.** Run the terminal at 120x36 for the recording. Apex's finding cards render best at that width.
- **Source-tree shot.** When Apex switches to whitebox mode, briefly show the file tree in a side pane (`tree -L 2 atlas-lms`). This sells the "we have the source now" beat.
- **Diff readability.** When the patching agent emits the strong-params fix, render the diff with syntax highlighting at >=14pt. The before/after should fit on one screen.
- **Pacing.** Act 3 (the whitebox haul) is the act viewers will replay. Slow down to 0.85x narration speed and let each new finding card land for at least a beat before the next one appears.
- **Avoid fake noise.** No fake "5,000 requests/sec" overlays, no Matrix rain. The differentiator is that Apex reads code; show it reading code.
- **End card.** Final frame: blackbox count vs whitebox count side by side (6 vs 12), with the line "whitebox isn't a mode — it's a different conversation with the codebase."
- **Captions.** Caption every finding title. Many viewers watch muted on Slack / LinkedIn.
- **Length target.** 7:30 - 8:30 final cut. If it runs long, trim the SQLi blackbox demo (H1) — it is the least surprising finding and the audience will not miss it.

---
