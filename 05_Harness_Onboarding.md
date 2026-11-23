# 05 — Harness Onboarding (Phase 2)

## 5.1 Account, orgs and projects [C]
- Account `pC_7h33wQTeZ_j-libvF4A` on `https://app.harness.io`.
- Orgs = business units: `DIG`, `GBD`, `EDA`, `CORP`, `UEPPROD`, `BambooEnablement`, `Harness_Platform_Management` (holds `Solutions_Factory`).
- **Project identifier = APM ID** (`APM1081216`). The project *name* is separate (e.g. `unified-desktop`, `ps-digitalproducts-ehapp-azure-psgbd`, `utilization_management_decision_engine`). The name drives repo names (`<project name>-helm-charts-<medal>`).
- NextGen (Greenfield AWS) apps go to a separate project `<org>/APM<id>_NXT` (`apm<id>_nxt_*` connectors, `APP_PUSH_TYPE=ecr`).
- An APM can have an empty duplicate project in another org (`DIG/APM1081670` was empty; the real one is `GBD/APM1081670`).

## 5.2 Workflows and backend pipelines [C]
| Purpose | IDP workflow | Backend |
|---|---|---|
| Silver (creates the project) | `Harness_Onboarding_Project_Setup` ("Harness Project Onboarding Setup"), template `account.DIG.APM1009046` | `Create_and_Manage_IACM_Workspaces` → workspace `<ORG>_<project-name>`, then `helm_repo_setup`, `istio_gateway_setup`, `configure_workspace_for_da` (discovery agent) |
| Gold / Platinum (modifies the project) | `harness_project_modify_gold_platinum` | workspace `<ORG>_<APM or name>_<medal>_modify_legacy` (template lib `harness-project-medal`, 39 resources), creates `helm_repo_setup_gold_pipeline`, `istio_gateway_setup_gold_pipeline`, `fetch_istio_gateway_name_gold` |
| Migration (EHDS) | migration workflow (`migration_workflow`) | `migration_pipeline` → `Create_and_Manage_IACM_Workspaces` per app |
| Others in Solutions_Factory | — | `Plan_and_Validate_IACM_Workspace`, `Provision_Workspace` (stops at an `IACMApproval`) |

Terraform source: `custom-harness-template-library`, branch `production-onboarding` (e.g. `harness-project/locals.tf`). Stacks are launched from IDP only. [C]

**IDP form must be submitted from the UI.** API submission (`POST https://idp.harness.io/<acct>/idp/api/scaffolder/v2/tasks`) creates a task, but step 1 fails "**Token is not valid**": the workflow passes `user.token` (browser session JWT) and a PAT returns 401 when sent as a Bearer. Every attempt also stored the PAT in plain text in the task params. Workaround used on APM1080718 Gold: trigger `Create_and_Manage_IACM_Workspaces` directly with the API key, using a past execution's resolved inputs (`/pipeline/api/pipelines/execution/<id>/inputset?resolveExpressions=true`) with `RESOURCE_NAME`/`RESOURCE_VARS` swapped. [C]

## 5.3 What Silver onboarding creates (live workspace APM1080718/ARISAZ, ~251 resources) [C]
| What | Count | Purpose |
|---|---|---|
| Project | 1 | APM home |
| Delegate (ArgoCD app `apm<id>-delegate` + token) | 1 | runs every task, logs in to Vault |
| Vault secret managers | 2 | `vault` (APM ns) and `vault_bitbucket` (CORP ns) |
| Connectors | 4 | Bitbucket, Docker/Quay non-prod, Docker/Quay prod, Kubernetes |
| Secrets | 12 | pointers: Bitbucket user/token/password, Docker user/password, Artifactory user/token, Sonar, Checkmarx id/token, Wiz id/token |
| Pipelines | 15 | golden CI builds (Java, Node, React, Angular, Python, Go, monorepo, common components), CD (`cd_harness_native` or `_dynamic`), bootstrap, migration, helm_repo_setup, gateway_setup |
| Environments | 3 | dev, test, prod |
| Infrastructures | 1 per namespace | CD target per Silver namespace |
| File Store | 22 files | Dockerfiles per stack, CA bundle, CI scripts, `pip.conf`, `settings.xml`, `.npmrc` |
| User groups / roles / bindings | 9 / 9 / 20 | Admins, CICD, DEV, PO, QA, RM, SECURITY, SPE, Viewers |
| Service account, API key, token | 1 each | automation identity |
| Governance policies / sets | 5 / 3 | no hard-coded secrets, approved templates only, golden template protection, required tags, prod approval |
| Project variables | ~110 | Artifactory, Checkmarx, registries, cosign, namespaces, `dnsHost`, `onboarded_medals`… |

Outside the project: `apm<id>-gitexperience-repo` (in the "Infrastructure Code Project Key" Bitbucket project), `<tenant>-helm-charts-silver` with `values-harness.yaml`, Istio gateway `<project-name>-gateway`, service discovery agents (`<ns>-discovery`). **Not created**: Vault content, app services and input sets.

## 5.4 Silver IDP form fields (seen across runs) [C]
`project_id`, `project_name`, org, medal, `email_ids` (comma-separated), `namespace_dns_combinations` (**one row per namespace**), `discovery_namespaces`, `infra_identifiers`, delegate namespace / image / `delegate_storage_class` ("Enter storage class manually" if not in the dropdown) / `delegate_service_account_name` / `delegate_upgrader_enabled`, ArgoCD server (`hostname:port`), ArgoCD username/password (`argocd_auth_credential`, plain text, so redact), Vault fields (doc 04), Bitbucket Infrastructure Code Project Key, validation repo, Source Code Project key, Quay org, Artifactory project key (only **one**), helm options (`existing_ehds_tenant`, "Use new infra project", "Existing helm repo url"), DATA_INITIATOR / data owner (AM89727 used).
- Defaults: blank delegate namespace = APM ID. Blank SA = `argocd-application-controller`. Blank storage class = cluster default (rosa-np1: `gp3-customer-kms-v1`; Gold/Platinum/DR template default `elvh-disk-csi`). [C]
- **Values are not trimmed**: a leading space in `bitbucket_vault_namespace` went into the connector, and a trailing space in a namespace name was found. [C]

## 5.5 Execution flow of `Harness_Onboarding_Project_Setup` [C]
1. **Configuring Harness Workspace**: Terraform plan/apply on `<ORG>_<project-name>`. It creates the project, connectors, and ArgoCD app `apm<id>-delegate`, then `time_sleep.wait_for_delegate` (~4 min, **first create only**), then pipelines (remote, committed to gitexperience repo by `srcUEPHAPM<id>BIT`).
2. **Setup Helm Charts Repository**: `helm_repo_setup_pipeline`. Since Nov 2026 it requires `APPLICATION_NAME` (SNOW AD-group `application_name_associated_with_this_group`), which the workflow doesn't pass. This is a template bug.
3. **Istio gateway setup**.
4. **Configure workspace for DA** (discovery agent, ephemeral workspace).

Re-runs are **idempotent**: e.g. "0 added, 12 changed, 0 destroyed", all internal bookkeeping. [C]

## 5.6 Gold onboarding specifics [C]
- Requires Silver first (the Gold workflow modifies an existing project, and Gold pipelines are written through Silver `apm<id>_bitbucket`).
- Post-step order: `helm_repo_setup_gold_pipeline` → `fetch_istio_gateway_name_gold` (sets `ISTIO_GATEWAY_NAME`; inline; Silver delegate by design) → `istio_gateway_setup_gold_pipeline` (`MEDAL_LEVEL=gold`, `QUAY_REGISTRY_URL=https://quay-nonprod.elevancehealth.com`, `QUAY_VAULT_OVERRIDE`/`GIT_VAULT_OVERRIDE=true`).
- Helm modes: existing EHDS tenant whose project name = tenant → `EXISTING_TENANT=true, USE_NEW_INFRA_PROJECT=false` (APM1080718/arisaz). Project name ≠ tenant → `USE_NEW_INFRA_PROJECT=true` + `EXISTING_TENANT_REPO_URL=<scm>/scm/<KEY>/<tenant>-helm-charts-gold.git` (APM1081123), which also creates 5 `<project>-artifactory-*-gold` repos.
- Medal check: IACM workspace `argocd_medal_level`, `namespace_details`, and project variable `onboarded_medals` (`["silver","gold"]`).

## 5.7 Connectors (Silver and Gold) [C]
| Type | Silver | Gold |
|---|---|---|
| Kubernetes | `apm<id>_k8s_connector` | `apm<id>_gold_k8s_connector` |
| Vault | `apm<id>_vault_sm`, `apm<id>_vault_sm_bitbucket` | `apm<id>_gold_vault_sm`, `apm<id>_gold_vault_sm_docker` |
| Bitbucket | `apm<id>_bitbucket` | `apm<id>_gold_bitbucket` |
| Docker/Quay | `apm<id>_docker`, `apm<id>_docker_prod` | `apm<id>_gold_docker` |
Gold connectors run on delegate `apm<id>-gold`. Terraform has no `ignore_changes` on connectors, so **UI edits get reverted by stale tfvars**: fix the workspace inputs instead. [C]

## 5.8 Permissions and PATs [C]
- Harness PAT: personal `HARNESS_API_KEY` (user AM89727), read-only by default for Devin; executions only with your explicit OK. A PAT pasted in chat must be rotated (this happened twice).
- Running golden CI/CD needs **Templates → View + Access** (`core_template_access`) on account templates `CI_Java_Build_Pipeline`, `CD_harness_native_pipeline`. 403 on 13 Nov; granted by 16 Nov.
- Pipeline edit (`core_pipeline_edit`) was available on project pipelines; edits were saved to a branch (`devin/bootstrap-init-timeout`).
- Workspace delete has **no UI option**: `DELETE …/iacm/api/orgs/Harness_Platform_Management/projects/Solutions_Factory/workspaces/<id>` (204), only with approval, after backing up GET `<id>` and `<id>/resources`.
- `IACMApproval` in `Provision_Workspace`: `PUT …/approvals/<id>` body `{"status":"approved"}` (POST returns 405).
- Bitbucket "project admin" is needed to grant service-ID access or create repos. The org Bitbucket token is read-only on most projects and can't read permissions (401).

## 5.9 Platinum / DR [U]
The Platinum workflow is the same `harness_project_modify_gold_platinum` with medal = platinum, and Confluence marks it "not finalised yet" for some tracks. **No Platinum or DR onboarding was performed in our history.** DR-related facts seen: Pega clusters `PEGA-UD-EKS-PROD` and `PEGA-UD-EKS-DR` exist (storage classes `ebs-sc` default, `efs-sc`, `gp2`); `argcd-creds.txt` has a `platinum-dr` context; DR shares the Platinum Vault path (doc).
