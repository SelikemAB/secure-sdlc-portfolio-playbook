# Project rules (copy this file to the root of your build workspace as CLAUDE.md)

You are building a four-repository security portfolio around ONE fictional app ("PayLoop", a mobile-money wallet API).
The goal is a body of work a hiring manager can run in minutes and a candidate can defend line by line.

## Hard rules
1. **Everything is synthetic.** No real company, product, hostname, domain, person, ticket or incident. IPs come only from
   RFC 5737 (203.0.113.0/24, 198.51.100.0/24, 192.0.2.0/24) or loopback. Demo passwords look like `alice-demo-pw`.
2. **Evidence is generated, never typed.** Any sentence of the form "X was blocked / detected / exploitable" must come from a
   command's output that a Makefile target reproduces. If you cannot run it, write "not verified" and say why.
3. **Every claim has a command.** README claims map to `make <target>`. No target, no claim.
4. **Tests must be able to fail.** Each exploit/detection suite is run against the vulnerable AND the fixed build, and each repo
   ships a mutation check that breaks the system on purpose and expects the suite to go red.
5. **Fail closed.** Missing report, crashed scanner, unparseable file, expired waiver: the gate fails, it never passes silently.
6. **Honest limits.** Every README ends with what was and was not verified (rules on a live SIEM, workflows on GitHub,
   container build). Never imply otherwise.
7. **Zero budget.** Open-source tools only. Docker and heavy scanners are optional; the core demo must run with PHP's built-in
   server and Python. Degrade gracefully and report exactly what was skipped.
8. **Stack.** PHP 8.3 (no framework), SQLite via PDO, Python 3.10+ with PyYAML only for tooling, GNU Make, Bash.
9. **Small, reviewable diffs.** One finding = one patch = one PR write-up.
10. **No secrets in the repo.** Token signing secret comes from the environment in the fixed build; the committed secret exists
    only in the deliberately vulnerable baseline and is documented as compromised.

## Working method
- Create a task list first; mark each task done only after its verification command passes.
- After each phase run that repo's `make` targets from a clean state and paste the final lines into the phase log.
- Prefer the Read/Edit/Write tools for files and the shell for running things.
- Do not ask questions mid-run; pick the most reasonable default and record it in `DECISIONS.md`.
- Never use em dashes in documentation.
