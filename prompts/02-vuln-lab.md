# Phase 2: payloop-vuln-lab (the core of the portfolio)

**Deliverable:** a small PHP API with nine deliberate flaws. Each flaw has a scripted exploit, a one-file patch, a PR write-up and a
Wazuh detection. The same exploit harness proves "vulnerable" on the baseline and "mitigated" on the fixed tree.

## 1. The app (baseline = vulnerable)
PHP 8.3, no framework. `app/public/index.php` routes: `GET /health`, `POST /login`, `GET /wallet?id=`, `GET /statement?wallet=&q=`,
`GET /receipt?file=`, `POST /partner/webhook-test`, `POST /transfer`, `GET /admin/users`.
Structure: `src/{autoload,Config,Auth,PasswordHasher,Db,Logger,Request,Response,HttpException,Wallets}.php`,
`src/Handlers/{Login,Wallet,Statement,Receipt,WebhookTest,Transfer,Admin}Handler.php`, `db/schema.sql`, `db/seed.php`.
Stateless HMAC-signed tokens. SQLite via PDO. `Logger` writes one JSON object per line (ts, event, src_ip, user, path, status,
plus event-specific fields) to `PAYLOOP_LOG`. Seed (fictional): alice, bob, carol, dave (customers, wallets 1 to 4) and admin
(wallet 5), passwords `<name>-demo-pw`. Wallet ids follow insertion order.

## 2. The nine findings (baseline behaviour -> fix)
| ID | Flaw | Fix |
|---|---|---|
| F1 | SQL injection: `q` concatenated into the statement query (UNION reads the users table) | prepared statements, escape LIKE wildcards, cap length |
| F2 | IDOR: `/wallet?id=` authenticates but never checks ownership | shared `Wallets::assertAccess()` (owner or admin); non-owner gets 404 and a logged denial |
| F3 | Path traversal in `/receipt?file=` | allow-list filename pattern + `realpath` containment inside the receipts dir |
| F4 | SSRF in webhook test (loopback, metadata 169.254.169.254, `file://`) | `SsrfGuard`: scheme allow-list first, resolve every A/AAAA, reject non-public ranges, IPv4-only `isPublic` (found: `::ffff:127.0.0.1` bypass), timeouts, no body echo |
| F5 | Unsalted MD5 passwords | Argon2id via `PasswordHasher`, rehash-on-login migration path |
| F6 | Hard-coded token signing secret committed in code | secret from env, fail closed when missing or short, document rotation |
| F7 | No login throttling; user enumeration via timing/messages | per-IP and per-user throttle with 429, identical error for unknown user and bad password, verify against a DUMMY hash so timing matches |
| F8 | Transfer integrity: negative/zero amount, overdraft, replay | amount validation, balance check, ownership of source wallet, idempotency key, `BEGIN IMMEDIATE` transaction |
| F9 | Stack traces returned to clients | generic error body with a request id; details only in the log |
Keep each fix self-contained. If a fix needs a helper from another fix (it happened with F7 needing F5's hash helper), move the
helper into the fix that needs it so patches stay independent.

## 3. Evidence engine
- `tests/lab.py --app <dir> --expect vulnerable|mitigated`: starts the app with PHP's built-in server on a free port with a throw-away DB,
  runs one exploit per finding, prints a JSON/text result per finding and exits non-zero if reality differs from `--expect`.
- `scripts/build-fixed.sh`: copies `app/` to a fixed tree and applies `patches/F*.patch`.
- `tests/patch_matrix.py`: apply each patch ALONE to a fresh baseline copy and require that its finding flips to MITIGATED and the
  others stay VULNERABLE. This proves patches are independent and minimal.
- `tests/ssrf_guard_test.php`: table-driven bypass cases (decimal/octal/hex IPs, IPv6 loopback, IPv4-mapped IPv6, userinfo tricks,
  `localhost` variants, link-local, CGNAT, metadata, non-http schemes). Every bypass idea from a review becomes a row.
- `findings/findings.yaml` (metadata) + lab output -> `scripts/gen_docs.py` -> `findings/FINDINGS.md` and `pull-requests/*.md`.
  Findings carry CWE, OWASP 2021, ATT&CK, reproduce commands, root cause, fix, generated evidence, SAST rule, Wazuh rule ids, blind spots,
  and an "AI-proposed fix review" note. Severity is qualitative (impact x ease), not CVSS.
- `scripts/make-pr-branches.sh`: one branch and commit per patch (`fix/f1-sql-injection`, ...) in a fresh clone.

## 4. SAST
`semgrep/payloop-php.yml`: custom taint-mode rules (request input to SQL, to filesystem, to HTTP client; md5/sha1 for passwords;
hard-coded secret; error disclosure). They must fire on the baseline and stay silent on the fixed tree. State plainly that
F2, F7 and F8 are logic flaws that static analysis cannot see; they were found by reading handlers against the threat model.

## 5. Detection as code
- `detections/wazuh/payloop_rules.xml`: ids 110100 to 110199, JSON decoder, MITRE ids on every rule, levels that match impact.
  Cover: brute force, spraying (different_field), success-after-failures, throttle ignored, SQLi, traversal, SSRF blocked/succeeded/metadata,
  IDOR enumeration (different_field wallet), authz-denial probing, negative amount accepted/rejected, duplicate idempotency key,
  invalid-token bursts, repeated 5xx.
- `tools/ruletest.py` with `lint`, `replay`, `suite`: an offline simulator of the rule subset used (decoded_as json, pcre2 fields,
  if_sid, frequency/timeframe, same_field, different_field, `$(field)` interpolation). Document its differences from the real engine.
  `wazuh-logtest` on a live manager stays the final authority (write `docs/WAZUH-VALIDATION.md` with the exact procedure).
- `tests/detections.yml`: cases with `events`, `expect`, `expect_absent`, `min_level`, `max_level`. Include FALSE-POSITIVE CONTROLS
  (same wallet viewed 8 times is not enumeration; distinct idempotency keys are normal; an ordinary customer day stays <= level 5).
- Replay REAL captured logs: run the exploit suite against baseline and against fixed, save both JSONL logs as samples, and assert
  different alert sets (the baseline shows successful SSRF and accepted negative transfers; the fixed run shows blocks and denials).
- `tools/mutation_check.py`: six mutants (raise a threshold 8 to 80, loosen a regex, move a rule to a sibling position, silence a
  rule by renaming its event, drop a same_field condition, make an escalation threshold unreachable). Baseline must pass, every mutant must fail.

## 6. Docs and CI
`docs/REMEDIATION-NOTES.md` (what the patch does not fix: rotate the secret, migrate MD5 hashes, egress filtering, least-privilege DB
account, nightly reconciliation), `docs/WAZUH-VALIDATION.md`, `docs/AI-ASSISTED-WORKFLOW.md` (verify AI output: exploit first, ask for
bypasses after the fix, review the diff like a human PR). Makefile: `all baseline fixed matrix ssrf sast detect docs mutate clean`.
Workflow `lab.yml`. README with a findings table and the honest limits.

## Done when
`make baseline` shows 9 VULNERABLE, `make fixed` shows 9 MITIGATED, `make matrix` 9/9, `make ssrf` passes, `make sast` is clean on the
fixed tree, `make detect` passes, `make mutate` kills 6/6.
