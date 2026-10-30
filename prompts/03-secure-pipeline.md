# Phase 3: payloop-secure-pipeline

**Deliverable:** a CI/CD security gate for the PayLoop fixed tree. Scanners only gather evidence; one gate decides, and it fails closed.

## Contents
- The fixed app tree from phase 2, a hardened `Dockerfile` (multi-stage, pinned tags, non-root user, no secrets) and
  `docker-compose.yml` (read-only fs, `cap_drop: ALL`, `no-new-privileges`, tmpfs, healthcheck, env-supplied secret).
- `policy/gate-policy.yml`: severity thresholds per tool, pinned checkov check ids that block (root container, exposed SSH) vs warn
  (unpinned tags), required reports list, waiver rules (owner, reason, expiry <= 90 days).
- `tools/gate.py`: reads `reports/*.json` (semgrep, gitleaks, trivy-fs, trivy-image, checkov, exploit-regression, ssrf, detection)
  and prints a Markdown verdict. Exit 0 pass, 1 policy failure, 2 missing/unparseable required report. A skipped tool yields
  `PASS [INCOMPLETE: skipped ...; not equivalent to a full run]`, never a bare PASS. Expired waivers fail.
- `tools/needs_check.py`: used in CI to turn `needs` results into a failure unless every upstream job succeeded.
- Workflows: `security-gate.yml` (jobs: secrets/gitleaks, sast/semgrep, container-and-iac/trivy+checkov, sbom/syft, exploit-regression,
  then `gate` with `if: always()` as the single required check) and `detection-ci.yml`. Least privilege: `permissions: contents: read`,
  `persist-credentials: false`, no `pull_request_target`, untrusted values passed via `env`.
- `.github/CODEOWNERS` protecting `policy/`, `tools/gate.py`, workflows and detections; `dependabot.yml`; `.gitleaks.toml` with a
  narrow allowlist for the documented demo credentials only.
- `samples/vulnerable-app` and `samples/bad-container`: intentionally bad inputs excluded from scans but used by unit tests to prove
  the gate FAILS when it should.
- `tests/test_gate.py`: unit tests on real semgrep/checkov output stored in `tests/fixtures`; cases include missing report, malformed
  JSON, expired waiver, blockers vs warnings, INCOMPLETE verdict. `scripts/local-gate.sh` runs whatever scanners are installed.
- `docs/pipeline-design.md` (stage table linking each job to threat ids, the "why", hardening still on the list) and
  `docs/branch-protection.md` (require only `gate`, CODEOWNERS review, push protection).
- `make test unit rules exploits gate-demo-good gate-demo-bad diagrams mutate`; `mutate` breaks the gate (drops the missing-report check,
  ignores waiver expiry, treats a skipped tool as clean, lets a high finding through, and so on) and requires the unit tests to fail.

## Done when
`make test` passes, `gate-demo-good` prints PASS (INCOMPLETE if scanners are absent, listing which), `gate-demo-bad` prints FAIL,
`make mutate` kills 6/6, `actionlint` is clean on every workflow. State that the workflows were linted, not run on GitHub, until you run them.
