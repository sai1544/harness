# 02 — Business Use Case

## What the history actually establishes [C]
- **Bamboo is being retired.** The Confluence program pages state this, as quoted in the 18 Nov KT session.
- **One standard delivery chain** for every APM: Bitbucket → build → **mandatory SonarQube + Checkmarx + Wiz** scans → JFrog Artifactory (dependencies and artifacts) → container push to Quay (or ECR for NextGen) → deploy through Harness CD / Helm (ArgoCD is used for delegate deployment) → ServiceNow change for prod.
- **Self-service onboarding**: app teams (or the onboarding team) onboard through **Harness IDP workflows** instead of tickets to a central team.
- **No secrets in Harness, sheets or YAML.** Every credential lives in Vault, and Harness stores only references. Each medal is isolated (a Silver delegate cannot read Gold secrets).
- **Definition of Done per APM** (Confluence, quoted in KT):
  - **A**: golden pipelines exist for every pattern the APM uses.
  - **B1 (EHDS)**: all apps migrated. **B2 (non-EHDS)**: one reference app migrated "white-glove".
  - **C**: the app team's DevOps are trained, with an updated runbook/RACI handed over.
- **Waves**: Wave 1 (EHDS), Wave 2 (containers, due 13 Nov), Wave 3 (EDA/GBD, started 16 Nov). [C] (KT; Wave 3 APM1010417 seen 19 Nov)

## What is NOT established [U]
- Cost, licence, or contractual reasons for choosing Harness: the KT explicitly said these aren't in the accessible resources.
- Quantified limitations of Bamboo (build times, failure rates, and so on): never discussed.
- Cutover dates or decommission dates for Bamboo/EHDS Tekton: never discussed.

## Inferred benefits [I]
These follow from the architecture we worked with, not from a business document:
- **Consistency**: the Terraform/IACM-based onboarding means every APM project gets the same 15 pipelines, connectors, policies and file store (a Silver workspace has about 251 resources).
- **Security posture**: per-APM service IDs (`srcUEPHAPM<id>BIT/ART/SON`) instead of human credentials, enforced scans, OPA-style governance policies (no hard-coded secrets, approved templates only, golden template protection, required tags, prod approval).
- **Central change control**: golden CI templates live in `HARNESSGOL/elvh-ci-cd-templates`. A single PR (e.g. #209 Gitleaks, #212 Checkmarx, #214 Sonar timeout, merged 19 Nov) changes behaviour for every APM.

## Is it CI, CD, or both? [C]
**Both.** Golden CI templates (Scan → Build → Release stages) plus CD (`cd_harness_native_pipeline` / `cd_harness_native_dynamic`, Helm-based K8s deploy). In our sessions only CI and onboarding were exercised; CD input sets were created, but **no CD execution happened**.

## Role of HashiCorp Vault [C]
Vault is the only store for real credentials (Bitbucket, Artifactory, Quay/Docker, SonarQube, Checkmarx, Wiz). The Harness delegate logs in to Vault with its own Kubernetes service-account token (Kubernetes auth method, one auth mount per cluster) and reads, at runtime, only the path its role's policy allows. See doc 04.
