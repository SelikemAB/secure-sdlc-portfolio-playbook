# Phase 4: payloop-secure-sandbox (the flagship)

**Deliverable:** the hardened build running in a locked-down stack, ten attacks fired at it, and a generated table showing for each one
whether it was blocked and which Wazuh rules fired. Plus runbooks and an interview pack.

## Contents
- `app/`: the fixed tree plus database constraints as the last line of defence: `CHECK(balance_minor >= 0)`,
  `CHECK(amount_minor > 0)`, foreign key on the destination wallet.
- `Dockerfile`, `docker-compose.yml`, `docker/` (nginx-unprivileged reverse proxy with rate limiting and security headers; read-only fs;
  non-root; `cap_drop: ALL`; `no-new-privileges`). Docker is optional; the demo runs on PHP's built-in server.
- `monitoring/wazuh/payloop_rules.xml` (29 rules) and `ossec-localfile.xml` (agent snippet that reads the JSON log).
- `tools/ruletest.py`, `tools/mutation_check.py`, `tests/` (lab.py, ssrf_guard_test.php, detections.yml, samples).
- `attack-sim/simulate.py`: scenarios S1 password spraying, S2 brute force on ONE account (use `dave`), S3 SQLi, S4 path traversal,
  S5 SSRF (loopback, metadata, file://), S6 IDOR enumeration, S7 transfer from another customer's wallet, S8 negative/zero amount,
  S9 replayed payment, S10 forged token signed with the old leaked secret. Options `--only`, `--url`, `--log`, `--report`, `--quiet`.
  Each scenario sends real requests, records HTTP outcomes (compressed like `401x10`), replays the app's own log through the rules,
  and prints blocked/detected. Exit non-zero if any attack succeeds or any expected detection is missing.
  Design lesson: S2 once locked out `carol`, which broke S9's login (429). Scenarios must not depend on accounts an earlier scenario burns.
- `docs/`: `architecture.md` (Mermaid), `sdlc-security-map.md` (stage -> control -> evidence -> threat ids), `runbooks.md` (one per detection
  family: triage questions, queries, containment, false positives, escalation), `demo-script.md` (a 3 to 5 minute screen share),
  `interview-qa.md`, `demo-output.md` (generated).
- `scripts/embed_demo.py` copies the generated table into README between `<!-- demo:start -->` and `<!-- demo:end -->`, so the README
  can never drift from what the code does.
- Makefile: `run demo test mutate up seed down`. Workflow `ci.yml` runs `make test` and `make mutate`.
- README: story across the four repos (Mermaid), run instructions three ways, the live results table, SDLC table, honest limits, including that
  the per-user login throttle can lock a victim out (accepted risk TM-17).

## Done when
`make demo` ends with "All attacks blocked; every expected detection fired."; `make test` and `make mutate` pass; the README table is populated.
