# SpectraView
### "Yelp for haunted houses. Rate the apparitions, dispute the poltergeists, and remember: every five-star review is a BOO-ty call."

## Premise & Vibe

SpectraView is a community review platform for allegedly haunted properties. Users sign up, claim a haunted address, upload blurry "ghost photos," and write long-form reviews scored by a haunting severity rubric (Class I "creaky floorboard" through Class VII "full-torso vaporous apparition"). Premium subscribers ("Ectoplasm Plus") unlock an "Exorcise" button that flags a property for cleansing by the moderation team.

The product copy leans heavily into the bit. The signup flow is a "seance." The empty-state for the activity feed says "It's spookily quiet in here." The premium upsell modal reads "Don't get spooked by your bills — go Ectoplasm Plus." Behind the chuckles, every bug is industry-real and lifted from an actual CVE, bug bounty, or HackerOne disclosure. The vibe is a hobbyist Flask app written by an enthusiastic founder in 2018, never re-audited, now serving real traffic. That is, regrettably, the most realistic thing about it.

The demo's emotional arc: Apex pokes around, finds the kind of bugs every junior security engineer has seen at least once, then chains a Jinja2 SSTI into RCE inside the Kali container, dumps the SECRET_KEY, forges a session for the admin user, and presses the Exorcise button on a property the operator does not own. SpectraView gets exorcised. Funny on the surface, devastating on the inside — like most bug bounty triage queues.

## Why This Stack

Flask is overrepresented in the long tail of small-to-medium production web apps. Solo founders and indie hackers reach for it because the Hello World fits in five lines and the documentation is good enough that a non-expert can ship something that works. The same properties that make Flask quick to prototype make it dangerous in untrained hands: `render_template_string` is one import away from `eval`, the default `SECRET_KEY` is a footgun by omission, and the Werkzeug debugger ships with an interactive Python REPL that has been left exposed in production by companies as large as Patreon (the 2015 incident that prompted Werkzeug to add a PIN at all).

SQLAlchemy is the canonical ORM, but Flask's culture of "just write the SQL when you need to" means raw queries via `db.engine.execute` and f-strings are everywhere in real codebases — exactly the pattern that produces SQLi. Jinja2 is bundled and `|safe` and `Markup()` are the documented escape hatches, used incorrectly in roughly half the public Flask CTF writeups on the internet.

This stack maps cleanly onto Apex's classic web-audit playbook: crawl, fingerprint, route enumeration, parameter fuzzing, template injection probes, SQLi probes, debugger discovery. It is the stack Apex was first benchmarked against, and it produces the broadest spread of "named" bug classes per line of code.

## Stack Details

| Layer | Choice | Version | Notes |
|---|---|---|---|
| Language | Python | 3.12 | type hints used inconsistently on purpose |
| Web framework | Flask | 3.0.3 | factory pattern, blueprints |
| ORM | SQLAlchemy | 2.0.x | mix of ORM and raw `engine.execute` |
| DB | SQLite | bundled | single file `spectra.db`, WAL mode |
| Templates | Jinja2 | 3.1.4 | autoescape on globally; `|safe` and `Markup` used in spots |
| Auth | Flask-Login | 0.6.3 | session-cookie based |
| Forms | Flask-WTF | 1.2.1 | CSRF on most forms, missing on a few |
| Frontend JS | jQuery | 3.6.0 | served from `/static/vendor` |
| CSS | Bootstrap 5 + custom "spooky" overrides | 5.3 | dark mode default |
| Image processing | Pillow | 10.x | thumbnail pipeline |
| Background jobs | RQ | 1.x | for "exorcism" simulation tasks |
| Email | Flask-Mail | 0.10 | wired to mailhog in dev |
| Container | Docker | python:3.12-slim base | `gunicorn -w 4 -k gthread` |
| Reverse proxy | nginx | 1.27 | terminates TLS, forwards to gunicorn |

The `Dockerfile` ships `FLASK_DEBUG=1` and `FLASK_ENV=production` simultaneously because the original founder copy-pasted both lines from different Stack Overflow answers and never noticed. That single line of context is the whole reason the Werkzeug debugger is reachable in the demo.

## Architecture

```
                     +-----------------+
        Internet --> |  nginx :443     |
                     |  (TLS, gzip)    |
                     +--------+--------+
                              |
                              v
                    +---------+---------+
                    | gunicorn :8000     |
                    | 4 workers, gthread |
                    +---------+---------+
                              |
              +---------------+---------------+
              |                               |
              v                               v
       +------+-------+               +-------+------+
       | Flask app    |               | RQ worker    |
       | spectra.app  |               | exorcism.q   |
       +------+-------+               +-------+------+
              |                               |
              +---------------+---------------+
                              |
                              v
                     +--------+--------+
                     |  SQLite WAL     |
                     |  spectra.db     |
                     +-----------------+

       Static uploads at /var/spectra/uploads (bind-mounted)
       Redis at redis:6379 (job queue only)
```

The Flask app is split into blueprints: `auth`, `reviews`, `houses`, `uploads`, `admin`, `debug`, and a thin `api` blueprint serving JSON for the jQuery frontend. The `debug` blueprint is conditionally registered on `app.debug`, but because `FLASK_DEBUG=1` is set in production, it is always registered.

Static files (uploaded ghost photos) are served two ways: directly by nginx for thumbnails (with `X-Content-Type-Options: nosniff`), and through a Flask download endpoint for "originals" so the app can record download counts for the Ectoplasm Plus analytics dashboard. The Flask download endpoint is the path-traversal sink.

## Data Model

```
users                 houses                reviews
-----                 ------                -------
id PK                 id PK                 id PK
email UQ              slug UQ               house_id FK
password_hash         name                  author_id FK
display_name          address               body TEXT
is_admin BOOL         lat, lng              severity INT 1..7
is_premium BOOL       claimed_by FK users   custom_template TEXT  <-- SSTI
api_token             created_at            created_at
created_at            haunting_score
                      
ghost_photos          exorcisms             sessions (server-side mirror)
------------          ---------             --------
id PK                 id PK                 sid PK
review_id FK          house_id FK           user_id FK
filename              requested_by FK       ip
caption (HTML)        status                ua
uploaded_at           queued_at             expires_at
                      worker_log
```

Notable choices that produce vulnerabilities:

- `reviews.custom_template` is a free-text field that premium users can set to "personalize" how their review renders. The server passes it to `render_template_string`. This is the SSTI sink.
- `ghost_photos.caption` is stored raw and rendered with `|safe` because the founder wanted users to be able to bold words in captions and "didn't want to deal with a markdown library." This is the stored XSS sink.
- `users` has no field whitelist on registration. The form is bound directly via `User(**form_data)`. Adding `is_admin=true` to the POST body works.
- `sessions` exists as a server-side mirror table but is never consulted on logout, so session invalidation is purely cookie-based, and the cookie is signed with a hardcoded key.

## Key Routes / Surfaces

| Method | Path | Auth | Purpose | Notable Bug |
|---|---|---|---|---|
| GET | `/` | none | landing, top-rated hauntings | — |
| GET | `/houses` | none | search/sort by severity | SQLi via `?house=` |
| GET | `/houses/<slug>` | none | property page | — |
| POST | `/houses/<slug>/claim` | user | claim ownership | mass assignment via form |
| GET | `/reviews/<id>` | none | review detail | renders `custom_template` (SSTI) |
| POST | `/reviews/new` | user | submit review + photo | stored XSS via caption |
| GET | `/uploads/download` | user | download original photo | path traversal via `?f=` |
| GET | `/uploads/thumb/<id>` | none | served via nginx | — |
| POST | `/auth/register` | none | seance signup | mass assignment `User(**form)` |
| POST | `/auth/login` | none | login | open redirect via `?next=` |
| POST | `/auth/logout` | user | logout | does not revoke server session |
| GET | `/u/<username>` | none | profile | — |
| GET | `/admin/` | admin | dashboard | Referer-only check on subroutes |
| POST | `/admin/exorcise/<house_id>` | admin | trigger exorcism | broken auth (Referer-based) |
| GET | `/admin/templates/<name>` | admin | preview email templates | LFI via `render_template(name)` |
| GET | `/debug/console` | none | Werkzeug debugger | enabled in prod, PIN brute-forceable |
| GET | `/api/v1/houses` | none | JSON list | — |
| POST | `/api/v1/exorcise` | premium | trigger exorcism (JSON) | no CSRF token, cookie auth |
| GET | `/health` | none | k8s probe | leaks `app.debug` and version |

Static surfaces of interest:
- `/static/vendor/jquery-3.6.0.min.js` — outdated, vulnerable to known prototype pollution patterns
- `/static/uploads/` — nginx-served, no `Content-Disposition`, so HTML uploads render inline

## Auth Model

Authentication uses Flask-Login backed by a session cookie. The cookie is signed (not encrypted) with `app.config['SECRET_KEY']`, which in this codebase is the literal string `'dev-secret-please-change'` set in `config.py` and committed to git. The cookie payload includes `user_id` and `_fresh`. Apex's first move once it confirms the key is to forge a cookie for `user_id=1` (the founder/admin) using `itsdangerous` with the leaked key.

`login_required` decorates most user-facing routes. `admin_required` is a custom decorator that checks `current_user.is_admin`. However, two admin endpoints — `/admin/exorcise/<id>` and `/admin/templates/<name>` — were added late in development and use a homegrown check that reads `request.referrer` and verifies it starts with `https://spectraview.example/admin/`. This check is what the demo exploits via a forged Referer header from the SSTI-acquired RCE.

API requests can authenticate via either the session cookie (default for the jQuery frontend) or a long-lived `api_token` field on the user. The token is generated at registration with `secrets.token_urlsafe(16)` (16 bytes is acceptable; the bug is elsewhere): the token is logged in plaintext to `/var/log/spectra/access.log` by a misconfigured access-log format that includes query string parameters, and a few jQuery callsites pass the token as a GET parameter "for convenience."

CSRF protection via Flask-WTF is enabled globally but disabled on `/api/*` because "the API uses tokens." Since the API also accepts the session cookie, this is effectively a CSRF bypass for any cookie-authenticated user.

## Intentional Vulnerabilities

### Critical

| ID | Class | Location | CVSS | Real-World Parallel |
|---|---|---|---|---|
| C-1 | Jinja2 SSTI -> RCE | `GET /reviews/<id>` calls `render_template_string(review.custom_template, review=review)` | 9.8 | Uber 2016 SSTI ($10k), Vine 2016, HackerOne report 125980 |
| C-2 | Werkzeug debugger RCE | `/debug/console` enabled because `FLASK_DEBUG=1` in prod; PIN brute-forceable | 9.8 | Patreon 2015 breach; CVE-2019-14806 era weak PIN derivation |
| C-3 | Hardcoded `SECRET_KEY` -> session forgery | `config.py: SECRET_KEY = 'dev-secret-please-change'` | 9.1 | Common GitHub leak class; Flask docs explicitly warn |
| C-4 | SQL injection (UNION + boolean) | `/houses` uses `db.engine.execute(f"SELECT * FROM houses WHERE name LIKE '%{q}%' ORDER BY {sort}")` | 9.8 | Endless; closest analogue: Joomla SQLi CVE-2017-8917 pattern |

### High

| ID | Class | Location | CVSS | Real-World Parallel |
|---|---|---|---|---|
| H-1 | Mass assignment -> privilege escalation | `/auth/register` does `User(**request.form)` letting attacker set `is_admin=1` | 8.8 | GitHub 2012 mass-assignment incident (egor homakov) |
| H-2 | Path traversal | `/uploads/download?f=../../etc/passwd` via `os.path.join(uploads, request.args['f'])` | 8.6 | CVE-2018-3760 (Sprockets), generic bug bounty staple |
| H-3 | Local File Inclusion via `render_template` | `/admin/templates/<name>` calls `render_template(name)` with user-controlled name | 8.1 | CVE-2016-10516 patterns; many Flask CTFs |
| H-4 | Broken access control on exorcism | `/admin/exorcise/<id>` trusts `Referer` header instead of session | 8.1 | OWASP A01; Shopify HackerOne report 270981 vibe |
| H-5 | Stored XSS via ghost photo caption rendered with `|safe` | `templates/review.html` uses `{{ photo.caption|safe }}` | 7.4 | Rails `html_safe` misuses; common DOMPurify-less mistakes |

### Medium

| ID | Class | Location | CVSS | Real-World Parallel |
|---|---|---|---|---|
| M-1 | Open redirect | `/auth/login?next=//evil.example` redirects without origin check | 6.1 | CVE-2017-1000455 Flask-Security style; phishing chain primitive |
| M-2 | CSRF on `/api/*` | CSRF disabled on API blueprint; cookies still accepted | 6.5 | Pre-2018 Django REST framework defaults |
| M-3 | Insecure cookie flags | `SESSION_COOKIE_SECURE=False`, `SESSION_COOKIE_HTTPONLY=False`, `SAMESITE` unset | 5.4 | OWASP ASVS V3.4 violations |
| M-4 | API token leakage via access log | nginx log format includes `$args`; tokens passed as GET param | 6.5 | Twitter 2018 password-in-logs; Stripe 2020 partial-key logs |
| M-5 | Unrestricted file upload (extension allow-list bypass via double extension) | `photo.php.png` saved as PHP because nginx mime maps `.php` | 7.2 | CVE-2013-2618 nginx config vintage |

### Low

| ID | Class | Location | CVSS | Real-World Parallel |
|---|---|---|---|---|
| L-1 | Verbose `/health` leaks version + debug flag | response includes `{"debug": true, "git_sha": "..."}` | 3.7 | Spring Boot Actuator 2018 disclosures |
| L-2 | Outdated jQuery 3.6.0 served from `/static/vendor` | known prototype-pollution sinks reachable from caption HTML | 3.7 | CVE-2020-11023 |
| L-3 | Missing rate limiting on `/auth/login` | unlimited credential stuffing | 5.3 | Long tail of credential stuffing breaches |
| L-4 | Username enumeration via login response timing | bcrypt only on valid users; missing-user path returns instantly | 3.7 | OWASP ASVS V2.2.1 |

That brings the total to four Critical, five High, five Medium, and four Low — eighteen named bugs, comfortably above the floor.

## Real-World Parallels

The Werkzeug debugger exposure traces directly to the 2015 Patreon incident where attackers found `/console` open in production and used the interactive REPL to dump the database. The PIN-protection feature was added afterward, and CVE-2019-14806 (and earlier Werkzeug advisories) document weakness in the PIN derivation: it is a function of `getpass.getuser()`, the module path, the MAC address (`uuid.getnode()`), and a machine-id file. Once you have arbitrary file read or template-injection-driven Python evaluation, you can compute the PIN yourself. Apex chains C-1 (SSTI) into reading these values, computes the PIN, and unlocks `/debug/console` for a clean RCE shell on the second pass.

The hardcoded `SECRET_KEY` echoes the avalanche of GitHub leaks documented in works like Meli et al. 2019 ("How Bad Can It Git?") which found tens of thousands of secrets pushed to public repos. Flask's own documentation has, for years, warned that the default `app.secret_key` must be replaced; the demo's value `'dev-secret-please-change'` is a literal lift from a popular Real Python tutorial pattern.

The mass-assignment bug rhymes with Egor Homakov's 2012 GitHub disclosure, where Rails' default `accepts_nested_attributes_for` plus a missing `attr_accessible` allowed a researcher to add their SSH key to the rails/rails repository. The Flask version is structurally identical: `User(**request.form)` accepts whatever the client submits.

The SSTI -> RCE chain is the same primitive used in Uber's 2016 disclosure where a Jinja2 template injection in a marketing email preview earned a $10,000 bounty, and HackerOne report 125980 against Vine. The standard payload `{{ ''.__class__.__mro__[1].__subclasses__() }}` to enumerate Python classes and pivot to `Popen` or `os.system` works without modification here because the demo runs Python 3.12 and does not sandbox the environment.

The path traversal mirrors CVE-2018-3760 in Ruby's Sprockets, and is the single most common file-handling bug in bug bounty triage queues. The LFI-via-`render_template` is documented in many CTF writeups and a recurring HackerOne pattern: when the *template name* is user-controlled, the attacker can read any `.html`/`.txt` under the templates folder, and with chained traversal, files anywhere Jinja2's loader will resolve.

The stored XSS via `|safe` mirrors years of Rails `html_safe` misuse. It is also the exact mistake the original Markdown-rendering libraries warned about for a decade. The premium "exorcise" button being protected only by a Referer header is the kind of bolt-on auth that ships when a feature is rushed in a sprint; Shopify's bounty program has paid for variants of this pattern, and the OWASP Top 10 A01 (Broken Access Control) is the single most common category in their 2021 update.

## Apex Features Showcased

This demo anchors on the classic Flask audit, which Apex does very well, and ends in a marquee SSTI-to-RCE chain that exercises the Kali container's `executeCommand` primitive.

- **Attack-surface mapping**. `/pentest` first enumerates routes by combining Apex's crawler output, JS endpoint extraction (parsing the jQuery handlers in `static/js/app.js` to find `/api/v1/exorcise`), and form-action discovery. The hidden `/debug/console` and `/admin/templates/<name>` are not linked from the navigation; Apex finds them through wordlist + framework-aware fingerprinting (when it sees `Server: Werkzeug`, it adds the debug routes to its candidate set).
- **Threat model agent**. Apex builds a STRIDE-style model of the app from the home page and signup flow, predicts that a "premium feature gate" exists, and prioritizes auth-bypass probing on the exorcism endpoint.
- **Findings + CVSS + judge**. Each of the eighteen bugs becomes a finding with a Likert-confidence score, a CVSS vector, and a judge-agent re-review that filters out the noisy false positives the SQLi probe inevitably generates against the SQLite-backed search.
- **Patching agent**. After the chain succeeds, the patching agent opens a PR that fixes C-1 by replacing `render_template_string(user_input)` with a sandboxed Jinja2 environment with restricted globals, fixes C-3 by reading `SECRET_KEY` from the environment with a startup assertion, and fixes H-1 by introducing an explicit `RegistrationForm` schema. The PR also flips `FLASK_DEBUG=0` and removes the `debug` blueprint registration.
- **Memory**. The operator runs `/operator` after `/pentest`; the operator agent recalls the leaked `SECRET_KEY` and uses it to forge an admin session cookie, then chains to the Referer-trusting exorcism endpoint without re-deriving the cookie.
- **Swarm**. During the SQLi probe phase, Apex fans out three SQL-payload workers (UNION, boolean-blind, time-based) in parallel against `/houses`. The judge agent merges results and dedupes.
- **Playwright**. The stored XSS in ghost photo captions is confirmed live: Apex spins a Playwright browser, logs in as a freshly-registered user, posts a payload, then visits the review page and captures the cookie-stealing fetch in DevTools network logs.
- **Kali container `executeCommand`**. After SSTI escalation, Apex pivots the RCE shell into the Kali container and runs `nmap`, `sqlmap` (against the now-leaked DB connection string), and `gobuster` to confirm there is nothing else interesting on the local network. This is the headline "look how it pivots" moment.

## Demo Storyline

The demo is a single 18-to-22 minute recording, broken into beats. The on-screen operator narrates briefly between phases; everything else is Apex output.

**Beat 1 — Cold open (00:00 - 01:00).** SpectraView's landing page in a browser. The operator clicks around: rates a haunting, opens the Ectoplasm Plus modal, laughs at the copy. Sets the scene. The operator says, "We have permission to test this. Let's see what we find."

**Beat 2 — Recon (01:00 - 04:00).** `apex /pentest https://spectraview.demo`. The CLI streams the route map, fingerprint (`Flask 3.0`, `Werkzeug 3.0`, `gunicorn`), discovery of `/debug/console`, `/admin/`, `/api/v1/`, and the JS endpoint extractor pulling `/api/v1/exorcise` out of `app.js`. The threat-model agent prints its STRIDE summary in a side pane.

**Beat 3 — Quick wins (04:00 - 08:00).** The swarm finds, in order: open redirect on login, mass assignment on register (registers `attacker@demo` with `is_admin=1`), SQLi on `/houses?sort=`, path traversal on `/uploads/download`. Each finding flashes a card with CVSS and the judge's confidence score. The operator says, "Okay, now the fun part."

**Beat 4 — SSTI (08:00 - 12:00).** Apex registers a normal user, upgrades to Ectoplasm Plus via the dev-mode bypass it discovers (a `/dev/promote` route registered alongside the debug blueprint), submits a review with `custom_template = "{{ 7*'7' }}"` and confirms reflection. Escalates to `{{ ''.__class__.__mro__[1].__subclasses__() }}`, finds `Popen`, executes `id`. The terminal prints `uid=999(spectra) gid=999(spectra)`.

**Beat 5 — Werkzeug pivot (12:00 - 15:00).** From the SSTI shell, Apex reads `/etc/machine-id`, the MAC address, and the username, derives the Werkzeug debugger PIN, and unlocks `/debug/console`. Now it has a stable interactive shell. It dumps `app.config['SECRET_KEY']`, prints `'dev-secret-please-change'`, and the operator slowly nods.

**Beat 6 — Session forgery + Referer abuse (15:00 - 17:30).** Apex's operator agent forges a session cookie for `user_id=1`, the original founder/admin. It hits `/admin/exorcise/<id>` with a forged `Referer: https://spectraview.demo/admin/` header. The exorcism job is queued. The browser refreshes; the targeted house's status flips to "Cleansed."

**Beat 7 — Kali pivot (17:30 - 19:00).** Apex calls `executeCommand` inside the Kali container: runs `sqlmap` against the leaked DB URL, runs `gobuster` against an internal-only admin port surfaced in the debugger output. The screen shows the Kali container working alongside the findings panel.

**Beat 8 — Patch (19:00 - 21:00).** `apex /patch`. The patching agent opens a local branch, generates a PR with three commits: "harden SECRET_KEY loading," "remove render_template_string of user input," "drop debug blueprint in production." The diff renders inline. Tests pass. The operator merges.

**Beat 9 — Outro (21:00 - 22:00).** Re-run `/pentest`. The Critical and High findings are gone. Mediums remain (rate limiting, cookie flags, log redaction) — Apex doesn't fix everything, just the things that matter. Operator wraps with: "Eighteen findings, four hours, one merged PR. SpectraView has been exorcised."

## Build Notes

Three to five days, solo developer, working from a clean Flask scaffold.

**Day 1 — Skeleton + auth.** Project layout, factory app, blueprints, SQLAlchemy models, Flask-Login wiring, registration with the deliberate `User(**form_data)` mass-assignment hole, login with the `next=` open redirect. Seed script: 30 fake haunted houses across the US, 200 reviews from 25 users, three of which have `is_admin=False` but are clearly the bait. Premium tier flag with a hard-coded "Ectoplasm Plus" badge on the profile. Get the home page, signup, login, profile pages rendering with horror-flavored copy.

**Day 2 — Reviews, photos, severity.** Review model, severity enum, the `custom_template` field exposed only to premium users via a "Customize your review's frame" modal. The image upload pipeline using Pillow for thumbnails, with the nginx config that serves `/static/uploads/*.png` directly. Caption field rendered with `|safe` in `templates/review.html`. The deliberate stored-XSS sink is just `{{ photo.caption|safe }}`. The `/houses` search route with the f-string raw SQL. Confirm the SSTI fires by manually pasting `{{ 7*7 }}` into a custom template.

**Day 3 — Admin, exorcism, debug, API.** Admin dashboard with the legit `admin_required` decorator, plus the two routes that use the homemade Referer check. RQ-based exorcism worker that sleeps for 10 seconds then flips a row. The Werkzeug debug enablement: in `wsgi.py`, set `FLASK_DEBUG=1` based on a `.env` that ships with the container. The `/debug/console` route is therefore served. `/api/v1/exorcise` accepting JSON, no CSRF. Upload-download endpoint with the `os.path.join` traversal. The `/admin/templates/<name>` route that calls `render_template(name)` directly.

**Day 4 — Polish, copy, container.** Horror-pun copy pass: the seance signup flow, the BOO-ty call premium upsell, the spookily-quiet empty states. Dark mode CSS, ghostly favicons, a 404 page that reads "This page has crossed over." Dockerfile, docker-compose with redis + mailhog + the nginx reverse proxy. Verify all 18 bugs reproduce. Write a `seed.py` that idempotently creates demo data so the recording is deterministic.

**Day 5 (buffer) — Apex dry-run, fixes.** Run `/pentest` end-to-end against the live container, fix any fingerprinting friction (Apex needs `Server: Werkzeug` to be present in headers; gunicorn strips it by default — solved by setting `gunicorn --server-header`). Make sure the SSTI payload chain actually computes the Werkzeug PIN: this requires `/proc/self/cgroup` and `/etc/machine-id` to be readable, which they are in the slim image. Tune the storyline to fit the recording length. Pre-record a fallback take in case of network flakiness during the live shoot.

The repository should be public on a demo GitHub org with a `README` that says, in plain language, "this app is intentionally vulnerable." The license is MIT. There is no real user data; the seed reviews are written by the dev team and reference fictional addresses.

## Recording Notes

- Resolution 1920x1080, 30fps. Terminal at 14pt with the Apex ANSI palette, browser at default zoom.
- Two windows visible: terminal on the left, browser on the right. Apex findings panel pops up as an overlay during Beat 3.
- Background audio: subtle wind/creak loop, ducked under narration. No music during the SSTI beat — let the typing carry it.
- B-roll: a five-second close-up on the "Exorcise" button before Beat 6, so the payoff lands.
- The Werkzeug PIN computation should be visible: show the four inputs, the SHA256, the resulting PIN. Pause the recording on the "Pin code: 123-456-789" line for 2 seconds.
- Captions for all CLI output. Apex's color codes do not render in some autoplay previews; captions are the fallback.
- The patch PR diff in Beat 8 should fit on screen without scrolling. Aim for three small commits, not one large refactor.
- End screen: "SpectraView has been exorcised. Try Apex on your own stack: pensar.dev/apex." Hold for 4 seconds.
- Total target: 20 minutes. Hard ceiling 22.
- Have a recovery clip ready for the SSTI beat, since live RCE demos sometimes hang on `Popen` if stdout is buffered.
