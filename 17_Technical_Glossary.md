# 17 — Technical Glossary

| Term | Meaning in this project |
|---|---|
| APM | Application ID in the enterprise inventory (ZRM/App Lab); = Harness project identifier |
| Medal | Environment tier: Silver (slvr/np01), Gold (gld/np02), Platinum (plat/prod), DR |
| UEP | The program running Harness onboarding/migration (UEP Harness) |
| EHDS | Legacy container platform (Tekton + ArgoCD, "Brownfield" CaaS); main migration source |
| Brownfield | Existing CaaS clusters; also the Vault environment label `BROWNFIELD` |
| NextGen / Greenfield / `_NXT` | AWS greenfield target; separate Harness project `APM<id>_NXT`, ECR push |
| IDP | Harness Internal Developer Portal; hosts the onboarding/migration workflows (forms) |
| IACM | Harness Infrastructure as Code Management; Terraform workspaces in Solutions_Factory |
| Solutions_Factory | Project in org Harness_Platform_Management running the onboarding Terraform pipelines |
| `Create_and_Manage_IACM_Workspaces` | Pipeline the IDP forms call to create/apply workspaces |
| Workspace `<ORG>_<name>` | Terraform state per APM onboarding (Silver), `_<medal>_modify_legacy` (Gold) |
| Delegate | Harness agent pod in the APM cluster (`apm<id>`, `apm<id>-gold`); runs all tasks |
| Selector | Delegate tag (`apm<id>`) used by connectors/pipelines |
| Connector | Harness integration object (K8s, Vault SM, Bitbucket, Docker) |
| Custom Vault SM | Secret manager from `ElevanceCustomVaultSecretManagerTemplate` (shell script on the delegate) |
| Golden pipeline/template | Account-level standard CI/CD templates (`CI_Java_Build_Pipeline`, `CD_harness_native_pipeline`) |
| Input set | Saved runtime inputs per service (`<svc>_ci_inputs`, `<app>_slvr_<ns>`, `<app>_gold_uat`) |
| Git Experience / remote entity | Pipelines/input sets/services stored as YAML in `apm<id>-gitexperience-repo` |
| bootstrap_pipeline | Creates a test app repo, service and input sets for validation |
| migration_pipeline | Creates services/input sets for EHDS apps per medal |
| KV v2 | Vault key-value engine with versions; merge-patch updates only the named keys |
| Vault namespace | `<bu>-<apm>` isolation unit; `corp-apm1002184` for shared Bitbucket creds |
| Auth path | Vault Kubernetes auth mount per cluster (`rosa-np1`, `aks-dig-gld-np02`) |
| Role / policy | `harness-<apm>-<env>` bound to delegate ns+SA / `harness-<bu>-<apm>-<env>` read grant |
| sansPairings / boundClaims | ns+SA pairs allowed to log in with a role (Automation Hub create/modify) |
| Automation Hub | Internal API for Vault namespace/role management |
| vault-automation | Bitbucket repo `HARNESSADM/vault-automation` with read/put/gen-token scripts |
| Service ID | CORP account per APM and tool: `srcUEPHAPM<id>BIT/ART/SON` |
| SCC | OpenShift Security Context Constraints (`restricted-v2`, `anyuid`) |
| Storage class | K8s class for the delegate's 10Gi scratch PVC |
| IgnoreFailed | Step failure ignored by the failure strategy; not "Success" |
| `SBOM_MAX_RETRYCOUNT` | Project variable bounding Checkmarx scan-ID polling |
| `dnsHost` / `dnshost` | Project-level namespace→host map / per-app CD override |
| File Store `/ci_config` | Project files: Dockerfiles, `settings.xml`, `pip.conf`, `.npmrc`, certs |
| Devin Desktop | Devin on your laptop with Okta/AWS/kubectl access, used for Vault writes |
| Wave 1/2/3 | Onboarding batches (EHDS; containers; EDA/GBD) |
| DoD A/B1/B2/C | Golden pipelines / all EHDS apps / one white-glove app / team trained |
