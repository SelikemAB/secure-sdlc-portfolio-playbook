# Phase 5: verify like a skeptic, then package

Run this in a FRESH Claude Code session (or a fresh subagent) that has not seen the build. A system must not grade itself.

## 1. Clean-room re-run
Extract/clone the four repos into an empty directory. For each repo run its documented targets and record exit codes:
threat-model `build check mutate`; vuln-lab `all matrix ssrf sast detect mutate`; pipeline `test gate-demo-good gate-demo-bad mutate`;
sandbox `test demo mutate`. Any non-zero exit is a bug in the repo, not in the check.

## 2. Static checks
`actionlint` on every workflow; YAML-parse every .yml/.yaml; render every Mermaid block (`scripts/check_diagrams.sh`); `php -l` every PHP file.

## 3. Claims audit
Extract every sentence in the READMEs and docs that asserts a result (blocked, detected, killed, passes, N cases, N rules, N findings).
For each, find the command that produces it and check the number against fresh output. Fix the text, not the evidence. Anything you
cannot reproduce becomes "not verified" with the reason.

## 4. Sanitization scan
```bash
grep -rniE "<employer>|<product>|<internal-domain>|@gmail|password\s*[:=]\s*['\"][^'\"]{8,}" --exclude-dir=.git .
grep -rhoE '\b([0-9]{1,3}\.){3}[0-9]{1,3}\b' . | sort -u     # only RFC 5737, loopback, reserved ranges, example.com's address
```
Replace `<...>` with your own real terms (employer, products, domains). Also check committed log samples, fixtures and screenshots.

## 5. Package
Top-level `README.md` (table of repos, one run command each, verified vs not verified), `INTERVIEW-PLAYBOOK.md` (60-second pitch, which repo
answers which question, questions to expect), `PUBLISHING-CHECKLIST.md` (replace placeholders, one repo per directory, enable branch protection,
run the workflows once, run Docker and `wazuh-logtest` and update the limits). Strip `__pycache__`, `.run`, build outputs, then archive.

## Report format
End with two lists: VERIFIED BY RUNNING and NOT VERIFIED, with specifics. No adjectives.
