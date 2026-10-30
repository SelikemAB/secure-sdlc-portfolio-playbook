# Master prompt (paste into Claude Code)

Setup first (see the README):
```bash
mkdir workspace && cd workspace
git clone https://github.com/SelikemAB/secure-sdlc-portfolio-playbook playbook
cp playbook/prompts/CLAUDE.md ./CLAUDE.md
claude
```

Then paste everything inside the fence:

```text
Build a four-repository secure-SDLC portfolio in this workspace. Follow CLAUDE.md strictly.

THE STORY
One fictional app, PayLoop (PHP 8.3 mobile-money wallet API, SQLite), carried through the whole lifecycle:
  1. payloop-threat-model    what could go wrong and how coverage is proven
  2. payloop-vuln-lab        nine real flaws: found, exploited, fixed, detected
  3. payloop-secure-pipeline a fail-closed CI gate that replays the exploits
  4. payloop-secure-sandbox  the hardened build under attack, with detections and runbooks

HOW TO WORK
- First create a task list covering: the four phases, a cross-repo verification pass, a sanitization scan, packaging.
- Execute the phases in order. For each phase read the matching file in ./playbook/prompts/ IN FULL before writing code:
    ./playbook/prompts/01-threat-model.md
    ./playbook/prompts/02-vuln-lab.md
    ./playbook/prompts/03-secure-pipeline.md
    ./playbook/prompts/04-secure-sandbox.md
- Build the vulnerable baseline and the fixed tree of the app ONCE (in the vuln lab) and reuse the fixed tree in phases 3 and 4.
- A phase is done only when its Makefile targets pass from a clean checkout. Fix root causes, never weaken a test.
- Do not ask me questions. Choose sensible defaults and log them in DECISIONS.md.

VERIFY AND FINISH (read ./playbook/prompts/05-verify-and-publish.md)
- Re-run every repo's make targets from scratch, run actionlint on all workflows, render all Mermaid diagrams.
- Run the sanitization scan and the claims audit described there.
- Produce a top-level README.md, INTERVIEW-PLAYBOOK.md and PUBLISHING-CHECKLIST.md.
- Finish with a table: what was verified by running it, and what was NOT verified (be specific).

DEFINITION OF DONE
make targets in every repo exit 0; every exploit works on the baseline and fails on the fixed build; every detection case passes,
including false-positive controls; every mutation check kills 100% of its mutants; no placeholder or real-world identifier leaks.
```
