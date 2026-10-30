# Phase 1: payloop-threat-model

**Deliverable:** a threat model that is data, not a Word document: one YAML file generates every table, diagram and ATT&CK layer,
and a validator proves nothing is left uncovered or stale.

## System to model (fictional, synthetic)
PayLoop: customer clients (mobile app and USSD gateway), partner systems (merchants/aggregators calling and receiving webhooks),
an ops administrator, reverse proxy with TLS, the PHP API, an outbound webhook dispatcher, a CI/CD pipeline and image build,
wallet DB (users, wallets, transfers), receipt file store, JSON-lines security log, and a SIEM (Wazuh).
Trust boundaries: TB1 internet, TB2 edge/proxy, TB3 application zone, TB4 outbound/partner zone, TB5 build zone.

## Build
1. `data/threats.yaml` is the single source of truth with sections: `meta` (system, methodology, ATT&CK version), `risk_bands`
   (score = likelihood x impact, each 1-5; critical>=20, high>=12, medium>=6), `stride`, `elements` (external/process/store with a
   boundary), `flows` (data flows crossing boundaries), and `threats`.
2. Write 25 threats, ids TM-01..TM-25, STRIDE per element. Each threat has: element, STRIDE letter, title, scenario, ATT&CK technique
   id and name, likelihood, impact, `controls` (each with status implemented | partial | open), `detections` (Wazuh rule ids, may
   be empty with a stated reason), `tests` (exploit or suite names), `findings` (links to F1..F9 from phase 2), and an explicit
   `accepted_risk` reason when a control is deliberately absent. Cover: spoofing of tokens, IDOR, injection, path traversal, SSRF
   via partner webhooks, credential attacks, transfer integrity and replay, log tampering, DoS, admin abuse, secrets in repo,
   vulnerable dependencies, malicious CI change, supply chain (image/base).
3. `scripts/build.py` has three jobs:
   - **validate**: unique ids; every threat references real elements; technique ids match `^T\d{4}(\.\d{3})?$`; scores in 1..5;
     every threat has a control OR an accepted_risk reason; every detection id exists in `detections/payloop_rules.xml`; findings
     and tests referenced exist.
   - **generate** into `docs/`: 01-system-description, 02-data-flow (Mermaid DFD with trust boundaries), 03-stride-analysis,
     04-attack-mapping (+ `attack-navigator-*.json` layers: coverage by detection, and by risk), 05-risk-register (sorted by score,
     banded), 06-detection-traceability (threat -> rule -> test matrix, with gaps called out).
   - `--check`: regenerate in memory and fail if the committed files differ (stale-docs gate).
4. `scripts/check_diagrams.sh` extracts every ```mermaid block from README and docs and renders it with mermaid-cli
   (support `CHROME_PATH` for an existing Chromium; pass `--no-sandbox` via a temp puppeteer config created with `printf`).
5. `scripts/mutation_check.py` breaks the data in six realistic ways (duplicate id, dangling element, bad technique id, control
   removed without accepted_risk, detection id that does not exist, stale generated file) and requires validation to fail each time.
6. `detections/payloop_rules.xml`: copy from phase 2 once it exists, or create stubs and replace later; the validator must pass either way.
7. Makefile targets: `build`, `check`, `diagrams`, `mutate`. Workflow `validate.yml` runs check + mutate with least privilege.
8. README: scope, method (STRIDE per element + ATT&CK + L x I), how to read the outputs, honest gaps (MFA, signed partner callbacks,
   TLS inside the lab, runtime telemetry are recorded as open/partial, not hidden).

## Done when
`make build check mutate` exit 0; output contains `ok: 25 threats` and `6/6 mutants killed`; all diagrams render.
