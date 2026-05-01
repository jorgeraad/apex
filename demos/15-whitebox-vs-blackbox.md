# Demo 15 — Whitebox + blackbox = 1+1=3

**Length:** 4 minutes
**Format:** Side-by-side screencast w/ report comparison
**Audience:** Sec-eng, dev-leads, anyone choosing how to integrate
**Series:** Standalone

---

## Purpose

Show that running Apex with codebase access (`--cwd`) doesn't just *add*
findings — it improves the precision and depth of every finding. The
viewer should walk away knowing the right answer to "should I run blackbox
or whitebox?": *both, in that order.*

This demo also pairs naturally with Demo 03 (`/threat-model`), which
produces a whitebox artifact that further sharpens later runs.

---

## Requirements

- **Asset:** Threat-model target (reused from Demo 03). It needs at least:
  - 2 findings reachable from blackbox observation alone (e.g., a missing
    auth check that returns 200 to a probe).
  - 2 additional findings only reachable with codebase access (e.g., a
    secret accidentally committed in a config file, or a SQL query with
    string concatenation that's safe at runtime today but fragile on
    inspection).
  - 1 finding both modes find but with vastly different evidence quality
    (blackbox: "looks like SQLi"; whitebox: file:line with the offending
    template literal).
- **Two recordings, identical model + temperature**, only differing on
  `--cwd` flag.
- **Report comparison overlay** — show both reports side-by-side, scrolling
  in sync.
- **Theme:** `apex` dark.

---

## Things to check first

- [ ] Both runs reproduce reliably. The blackbox-only finding count and
      the whitebox-additional finding count must each be stable across at
      least two dry runs.
- [ ] Whitebox findings cite file:line. If they don't, the precision
      argument is weaker. Confirm in a dry run that file paths show up in
      evidence.
- [ ] Reports render in the same theme so they look comparable.
- [ ] Same swarm size in both runs to make the comparison fair.
- [ ] No real secrets in the codebase. The "accidentally committed
      secret" should be obviously fake (`API_KEY=DEMO_NOT_A_REAL_KEY_42`).
- [ ] Same target URL, same auth, same scope. The only variable is
      `--cwd`.

---

## Full script

### Scene 1 — Cold open (0:00–0:20)

**Visual:** Dark frame. Two boxes labeled "blackbox" and "whitebox".

**VO:** "Two ways to run a pentest. Blackbox — only the URL. Whitebox —
URL plus codebase. Most tools force a choice. Apex doesn't. Today: why
you should just do both."

---

### Scene 2 — Run blackbox (0:20–1:15)

```bash
pensar pentest --target https://staging.threat-model-target.local
```

Time-lapse the run. End-of-run summary:

```
✓ 3 findings
  - 🟠 High Auth bypass on /admin
  - 🟡 Medium Reflected XSS on /search
  - 🟡 Medium SQL injection (suspected) on /products?id=
```

Highlight the *suspected* qualifier on the SQLi finding.

**VO:** "Blackbox run. Three findings. Auth bypass and XSS confirmed
through behavior. The SQLi is *suspected* — error message looks SQL-
shaped, but the agent never got a clean confirmation from outside."

---

### Scene 3 — Run whitebox (1:15–2:15)

```bash
pensar pentest --target https://staging.threat-model-target.local \
               --cwd ./threat-model-target
```

Time-lapse the run. End-of-run summary:

```
✓ 5 findings
  - 🔴 Critical Hardcoded API key in src/config/secrets.ts:14
  - 🔴 Critical SQL injection (confirmed) in src/db/products.ts:47
  - 🟠 High Auth bypass on /admin (router middleware not applied)
  - 🟡 Medium Reflected XSS on /search (template literal, src/views/search.ts:22)
  - 🟢 Low Missing security headers
```

**VO:** "Whitebox run. Five findings. Two of them — the hardcoded key and
the missing security headers — only show up because the agent could grep
the codebase. The other three? Same bugs as before, but look at the
quality of the evidence: file paths, line numbers, the exact unsafe
template literal. That SQLi is no longer *suspected* — the agent showed
the unsafe code and confirmed the runtime behavior."

---

### Scene 4 — Side-by-side (2:15–3:00)

**Visual:** Reports tile side-by-side. Scroll in sync. Highlight matching
findings.

| Finding         | Blackbox                  | Whitebox                                    |
| --------------- | ------------------------- | ------------------------------------------- |
| Auth bypass     | 200 OK on /admin probe    | + middleware not registered in app.ts:18    |
| XSS             | reflected payload         | + offending template literal at search.ts:22 |
| SQLi            | suspected (error)         | confirmed + line:47 with concat             |
| Hardcoded key   | not found                 | secrets.ts:14                               |
| Missing headers | not found                 | listed                                      |

**VO:** "Same 3 bugs, sharper evidence — and 2 extra bugs the agent only
sees with the codebase. That's the 1+1=3."

---

### Scene 5 — Why the developer cares (3:00–3:30)

**Visual:** Cut to a dev's editor. Open `src/db/products.ts`, jump to
line 47. The unsafe code is right there:

```ts
return db.query(`SELECT * FROM products WHERE id=${id}`);
```

**VO:** "And this is why developers care about whitebox findings. The fix
isn't a paragraph in a report. It's a file and a line. Open the file, fix
the line, push. Done."

---

### Scene 6 — Best of both (3:30–3:50)

**Visual:** Show a pipeline diagram:

```
1.  /threat-model       → ./threat-model.md
2.  pentest --cwd ...    → whitebox findings, deep
3.  pentest (no --cwd)   → blackbox sanity check, fast
4.  CI gate on (2) or (3)
```

**VO:** "The pattern we recommend: whitebox first to find everything,
blackbox in CI to catch regressions. Both share the threat model. Each
makes the other better."

---

### Scene 7 — Outro (3:50–4:00)

```
WHITEBOX + BLACKBOX
docs.pensar.dev/apex/modes
```

**VO:** "Run them both. Link in the description."

---

## Editing notes

- The side-by-side report scroll in Scene 4 is the deliverable for this
  video. Smooth, synchronized scroll is critical — pre-render if needed.
- The dev's-editor scene (Scene 5) gives the video a human moment.
  Don't skip it.
- Avoid making blackbox sound bad. The argument is "both is better,"
  not "blackbox is broken." Many users will run blackbox-only because
  they don't have source — keep them onside.
- This video pairs cleanly with Demo 03. Cross-link in descriptions.
