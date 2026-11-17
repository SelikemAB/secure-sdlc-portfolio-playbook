# Phase 5: verify like a skeptic, then package

Run this in a FRESH Claude Code session (or a fresh subagent) that has not seen the build. A system must not grade itself.

## 1. Clean-room re-run
Extract/clone the four repos into an empty directory. For each repo run its documented targets and record exit codes:
threat-model `build check mutate`; vuln-lab `all matrix ssrf sast detect mutate`; pipeline `test gate-demo-good gate-demo-bad mutate`;
sandbox `test demo mutate`. Any non-zero exit is a bug in the repo, not in the check.

## 2. Static checks
`actionlint` on every workflow (fix its shellcheck findings too); YAML-parse every .yml/.yaml; render every Mermaid block
(`scripts/check_diagrams.sh`); `php -l` every PHP file. Note: `mermaid-cli` in Docker needs a Chrome build matching its puppeteer
version and can fail to resolve it; GitHub renders Mermaid natively, so if local rendering is blocked, say so honestly rather than
claiming "rendered" - do not move it to the verified column without a command behind it.

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

GOTCHAS when publishing to GitHub (all hit on a real account):
- Branch protection and repository rulesets are NOT available on PRIVATE repos on the Free plan (the API returns 403 "Upgrade to Pro or
  make this repository public"). Commit the ruleset as config (`.github/rulesets/protect-main.json`); it applies the moment the repo is
  public or on Pro. On a SOLO repo set `required_approving_review_count: 0` - GitHub forbids approving your own PR, so a non-zero count
  makes `main` unmergeable. Keep a repo-admin bypass actor for break-glass.
- `gh repo edit <repo> --visibility public` needs `--accept-visibility-change-consequences`. Run the full-history secret + identifier
  scan (gitleaks over all commits, not just HEAD) BEFORE going public: public exposes the entire history, not the current tree.
- CodeQL does not support PHP; it would only analyse the Python tooling. The PHP app stays covered by the custom semgrep rules.
- Enabling secret scanning / push protection / CodeQL via the API needs a token with the `security_events` scope; a default `gh` login
  usually lacks it (404). Either enable them in the repo UI, or commit a CodeQL workflow (which runs under the job's own token).

## Report format
End with two lists: VERIFIED BY RUNNING and NOT VERIFIED, with specifics. No adjectives.
