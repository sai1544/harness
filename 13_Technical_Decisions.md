# 13 — Technical Decisions and Design Discussions

| # | Question | Options discussed | Decision | Why | Consequences / open concerns |
|---|---|---|---|---|---|
| D1 | Stale workspace `DIG_unified-desktop` (APM1081216) | A: support clears it; B: new project_name | A, done by Devin via API delete after backup (you couldn't see a delete button) | All 105 resources were orphaned; clean state | IDP catalog entry `dig_unified_desktop` left in place |
| D2 | How to submit IDP forms | API scaffolder (PAT), UI | UI; for Gold, a direct pipeline trigger as a bypass | PAT rejected as a Bearer session token; PAT leaked into task params | PATs had to be rotated |
| D3 | Storage class unknown without cluster access | trial classes in ArgoCD; ask app team | Ask app team ("nope, we need the app team"); no trial edits | No ad-hoc cluster changes | 4-day wait; then `ebs-sc` |
| D4 | Pipeline fixes (initTimeout, 64 KB) on master or a branch | master / branch | Branch `devin/bootstrap-init-timeout` | `bootstrap_pipeline` is shared; untested edits stay off master; no review gates | Fixes not upstreamed; master still has the bug [U] |
| D5 | Bootstrap fix policy for new APMs | pre-apply / fix on failure | "if it hits the limit, then lets do the fix" | Minimal change | APM1080718/APM1006991 needed it |
| D6 | Missing Wiz creds | dummy `test` / wait for real | `test` via merge-patch | Lets the pod start so later steps can be tested | Wiz scan will still fail; real creds needed |
| D7 | Evaluate Scan Failures cutoff | edit template / project var | `SBOM_MAX_RETRYCOUNT=0` temporarily | Template is shared; a project var only affects the APM | Must be reverted to 20 (found at 20 on 18 Nov) |
| D8 | Bitbucket token revoked across medals | re-vault Gold; re-vault both separately; mint once and write both; copy Gold token to Silver; Harness workaround | Copy Gold's token / mint once and write both. Temporary Harness option 2 for APM1081123 | One token per Bitbucket user | Workaround to be reverted; SonarQube has the same risk |
| D9 | Silver connector using Gold secret | — | Rejected (reverted) | Custom SM secrets decrypt on the consuming connector's delegate | — |
| D10 | Partial Gold workspace (APM1081123) | delete and re-onboard; fix tfvars and re-apply | Fix tfvars, plan (4/10/0), apply | Teardown destroys shared project vars and the delegate | UI hand-fixes would be reverted by stale tfvars otherwise |
| D11 | Gold chart repo for APM1081123 | B: point the mapping at existing `ehappgbd-helm-charts-gold`; run helm gold to create the `<project>`-named repo | Ran helm gold (new-infra mode) | The onboarding standard creates `<project name>-helm-charts-gold` | — |
| D12 | Artifactory key used in CI vs CD | — | CI (File Store `settings.xml`) | Dependencies are used at build time; CD never reads Maven repos | ART service ID read on both projects is unverified |
| D13 | Hostname: onboarding vs CD | many rows; one default | One default per namespace; per-app `dnshost` in CD input sets | Terraform map is namespace => single host | Per-app `dnshost` override not yet tested |
| D14 | SCC fix | change Helm security context (`runAsUser`/`fsGroup` removal, OCP team's first suggestion) vs `anyuid` | `anyuid` on the existing SA | Chart defaults are Harness-owned, the template has no field for it, ArgoCD self-heal reverts edits; consistent with APM1007744/APM1008553; delegate needs root | Must keep the SA name exactly `argocd-argocd-application-controller` |
| D15 | Vault role fix | create (fails, exists) / modify ADD | modify ADD | Additive; keeps the old SA binding | — |
| D16 | Vault namespace for DIG | corp- / dig- | `dig-<apm>` | Policy + 47/53 working connectors | — |
| D17 | Helm EHDS access workaround for APM1081216 | copy the EHDS template into AM73599's personal repo | Rejected by you | Don't use another person's account | Blocked on EHDS admin |
| D18 | APM1006991 Artifactory fix | switch to Bearer (Desktop) / patch username `am89727` / mint as service ID | Mint as service ID (recommended) | Shared template uses Basic; personal tokens violate policy | Pending |
| D19 | NextGen in reports | whole APM out / only `_NXT` apps out | Only `_NXT` services out | NextGen moves per app | — |
| D20 | Bootstrap test apps in reports | include/exclude | Exclude | Not real services | Report = 207 services |
| D21 | Gold re-onboarding when already done | re-run / don't | Don't | It would undo connector fixes and could mint new tokens | — |
| D22 | ArgoCD password sharing | — | Devin shared it once on request (14 Nov), later refused to repeat it and asked for rotation | Org rule: never echo credentials | Exposed passwords should be rotated |

### Contradictions found and resolved
- "Cluster is `slvr-dig-sharedservpega`" vs `PEGA-COMPASS-EKS-NP01-CLUSTER`: the latter is the cluster; the former is part of the DNS zone. Devin corrected this itself (16 Nov).
- "`sonarqube_access_token` was read successfully": later corrected; it was unverified, because the Scan stage doesn't use it.
- Blank delegate SA = `argocd-application-controller` default, which still conflicts with the pilot's `argocd-argocd-application-controller` (your correction, 16 Nov).
- Storage class for rosa-np1: the data sheet said `elvh-disk-csi`; the platform cluster sheet said `gp3-customer-kms-v1`. The latter is valid (cluster-specific; `elvh-disk-csi` is the generic Gold/Plat/DR default).
- Devin Desktop "Bearer, not Basic" for Artifactory: rejected (see D18).
- Bitbucket default-branch theory for the T17 failure: disproved (a re-run succeeded with the default branch still unset).
