# 08 — Gold, Platinum and Silver Environments

## 8.1 Purpose [C unless marked]
- **Medal = environment tier**, each with its **own cluster, delegate, Vault path and connectors**.
- **Silver** (`slvr`, `np01`): base non-prod onboarding. It creates the Harness project. Typical namespaces are dev/sit/devops (e.g. `abi-dev`, `abi-sit`, `abi-devops`). CI runs here.
- **Gold** (`gld`, `np02`): second non-prod tier added to the existing project. Typical namespaces are uat/perf (e.g. `abi-uat`, `abi-perf`, `ehappgbd-uat`, `ehappgbd-perf`). The UAT/PERF mapping is seen in practice; the Confluence docs don't formally define the DEV/SIT vs UAT/PERF split [U].
- **Platinum** (`plat`, `prod`): production. **DR** shares the Platinum Vault path (doc) [C doc / U practice]. Never onboarded in our sessions.

## 8.2 Comparison table
| Aspect | Silver | Gold | Platinum | DR |
|---|---|---|---|---|
| Creates project? | Yes | No (modifies) | No (modifies) | [U] |
| IDP workflow | `Harness_Onboarding_Project_Setup` | `harness_project_modify_gold_platinum` | same, medal=platinum ("not finalised" for some tracks) | [U] |
| IACM workspace | `<ORG>_<project-name>` | `<ORG>_<APM/name>_gold_modify_legacy` | `<…>_platinum_modify_legacy` [I] | [U] |
| Vault KV | `secret/slvr/<apm>/np01` | `secret/gld/<apm>/np02` | `secret/plat/<apm>/prod` | same as Platinum [C doc] |
| Vault role | `harness-<apm>-slvr` | `harness-<apm>-gld` | [I] `harness-<apm>-plat` | [U] |
| Example DIG cluster / auth path | `aks-slvr-dig-np01` | `aks-dig-gld-np02` | [U] | [U] |
| Example GBD | `aks-gbd-slvr-np01` / `gbd-slvr-np01` | `aks-gbd-gld-np02` | [U] | [U] |
| ROSA (APM1000782 proposal) | `ocp-rosa-np1` | `ocp-rosa-np2` | `ocp-rosa-prod` | `ocp-rosa-dr` [P, awaiting confirmation] |
| Pega UD | `PEGA-COMPASS-EKS-NP01` | `PEGA-UD-EKS-NP2` | `PEGA-UD-EKS-PROD` | `PEGA-UD-EKS-DR` |
| Delegate | `apm<id>` | `apm<id>-gold` | [U] | [U] |
| ArgoCD (example) | `ehappgbd-argocd-aks-silver.elevancehealth.com` | `ehappgbd-argocd-aks-gold.elevancehealth.com` | [U] | `platinum-dr` context in `argcd-creds.txt` |
| Connectors | `apm<id>_*` | `apm<id>_gold_*` | [U] | [U] |
| Bitbucket Vault ns | `corp-apm1002184` | `corp-apm1002184` | [U] | [U] |
| Docker/Quay (DIG) | `apm<id>_docker` (non-prod), `_docker_prod` | `apm<id>_gold_docker` from `secret/gld/<apm>/np02` | [U] | [U] |
| Helm repo | `<project>-helm-charts-silver` | `<project>-helm-charts-gold` | [U] | [U] |
| CD input sets | `<app>_slvr_<ns>` | `<app>_gld_<ns>` (bootstrap) / `<app>_gold_uat`/`_gold_perf` (migration) | [U] | [U] |
| Default storage class | cluster default (rosa: `gp3-customer-kms-v1`) | `elvh-disk-csi` (template default for Gold/Plat/DR) | `elvh-disk-csi` | `elvh-disk-csi` |

## 8.3 Different clusters or namespaces?
Both [C]. Silver and Gold are different clusters (`aks-slvr-dig-np01` vs `aks-dig-gld-np02`), and each has its own devops namespace for the delegate (sometimes the same name on both clusters, e.g. `abi-devops`, `ehappgbd-devops`).

## 8.4 Same Harness project? [C]
Yes. One project per APM holds both medals' connectors, infrastructures (in environments dev/test/prod, e.g. Gold infra in env `test`) and variables. The exception is NextGen `_NXT` projects.

## 8.5 Credential differences [C]
- Separate Vault KV per medal. The Silver delegate cannot read Gold secrets.
- **Shared Bitbucket/SonarQube account across medals** (e.g. `srcapm1081123UEPHBit`, `srcuephapm1080718bit`) combined with one token per user means Gold vaulting can revoke Silver's token.

## 8.6 Deployment and release differences [U]
CD was not run in any session, so deployment differences between medals are only structural (different infra, input sets, chart repos, gateways). Prod change control (ServiceNow) is referenced in docs but not exercised.
