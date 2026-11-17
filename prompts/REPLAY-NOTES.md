# Prompt completeness audit (2026-11-17)

Handoff task 9 is "replay the prompts end to end from an empty directory." A full generative replay rebuilds all four
repos (many hours) and is still outstanding. As an interim, the prompts were **completeness-audited** against the
reference repos and hardened with the lessons from the real build.

## Structure: the prompts match the reference repos
Every Makefile target, script and file the prompts name exists in the corresponding reference repo, and the acceptance
criteria match real output:

| Phase | Prompt targets | Reference `make` targets | Match |
|---|---|---|---|
| 1 threat-model | build, check, diagrams, mutate | build check diagrams mutate | yes |
| 2 vuln-lab | all, baseline, fixed, matrix, ssrf, sast, detect, docs, mutate, clean | same | yes |
| 3 pipeline | test, unit, rules, exploits, gate-demo-good, gate-demo-bad, diagrams, mutate | same | yes |
| 4 sandbox | run, demo, test, mutate, up, seed, down | same | yes |

Named artifacts like `findings/findings.yaml`, `scripts/gen_docs.py`, `scripts/build-fixed.sh`, `tools/ruletest.py` all
exist. No structural gaps that would block a builder were found.

## Lessons now baked into the prompts (were missing, would have been rediscovered the hard way)
1. **SAST vacuous pass (prompt 02):** semgrep's built-in ignore skips any `build/` dir, so scanning a fixed tree at
   `build/fixed` runs on 0 files and passes vacuously. Fix: stage outside `build/`, scan with `--json`, assert findings==0
   AND scanned-files>0 (`scripts/check_sast_clean.py`).
2. **Wazuh static fields + correlation (prompt 02), confirmed on a real 4.12 manager:**
   - reserved fields (`status`, `srcip`, `url`, `id`, `user`) are static; `<field name="status">` stops analysisd. Use `<status>`.
   - frequency correlation on `srcip` needs `<same_source_ip/>`, not `<same_field>srcip</same_field>` (same_field only
     correlates dynamic fields, so the brute-force rules silently never fire). The offline simulator hides both bugs.
3. **Base-image CVEs (prompt 03):** trivy will flag base-OS CVEs and the gate fails closed. Bumping the base tag often makes
   it worse; verify a bump helps in CI, and otherwise use time-boxed, owner-signed waivers (justify a critical on its real
   condition).
4. **Publishing on GitHub (prompt 05):** rulesets/branch protection are 403 on Free-plan PRIVATE repos (commit the ruleset
   as config; it applies on public/Pro; use 0 required approvals on a solo repo). `--visibility public` needs
   `--accept-visibility-change-consequences`, and run a full-HISTORY secret scan before going public. CodeQL has no PHP support.

## Still outstanding
A full clean-room generative execution of all four phases (the literal "replay end to end"). Best run as its own session.
