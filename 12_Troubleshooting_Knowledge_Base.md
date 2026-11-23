# 12 — Troubleshooting Knowledge Base

Format per issue: **When / Component / Env-APM / Problem / Exact error / Root cause / Investigation / Fix / Why it worked / Verification / Status**. Sequence is chronological.

---
### T01 — Bootstrap started with no inputs
- **When**: 13 Nov. **Component**: `bootstrap_pipeline`. **APM**: APM1081191 (DIG).
- **Error**: `Unable to get secret information : [null]`
- **Root cause [C]**: 3 runs started with every runtime variable `null`; `GIT_PASS = <+secrets.getValue(<+pipeline.variables.git_password_secret>)>`.
- **Investigation**: compared pipeline YAML and inputs against APM1081520 run `rvTa1lUhSYmcjmr3ld4LzQ` (`/pipeline/api/pipelines/execution/{id}/inputset?resolveExpressions=true`).
- **Fix**: re-ran with the 30 reference inputs, swapped per APM. **Verified**: `KsFO8lgjQW-cfPyssUU73A` succeeded. **Status**: Resolved.

### T02 — Golden template permission
- **When**: 13 Nov. **Component**: Harness RBAC. **APM**: APM1081191.
- **Error**: `403 – Missing permission core_template_access on template CI_Java_Build_Pipeline` (same for `CD_harness_native_pipeline`).
- **Investigation**: Harness access-check API: user had `core_pipeline_execute` but not template View/Access.
- **Fix**: admin granted Templates → View + Access. **Verified** 16 Nov (templates load). **Status**: Resolved.

### T03 — Bootstrap pod-setup timeout
- **When**: 13 Nov. **APM**: APM1081670 (GBD, `aks-gbd-slvr-np01`, ns `apm1081670-devops`).
- **Error**: `POD_NOT_FOUND_ERROR`, then pod setup expired after 10 min.
- **Root cause [C/I]**: very slow image pulls from quay-prod on node `aks-linuxlarge-23837039-vmss0000gr` (`ci-addon:1.18.12` 146 MB in 4m06s).
- **Fix**: `initTimeout: 30m` in the K8s infra block, on branch `devin/bootstrap-init-timeout` (commit `e0bdd9cd`) of `MCHARNESS/apm1081670-gitexperience-repo`. **Why**: gives image pulls time to finish. **Verified**: pod setup done in 6.5 min. **Status**: Resolved (branch only).

### T04 — Step stuck "waiting"
- **When**: 13 Nov. **APM**: APM1081670 run 3. `Clone_template_repo` finished, but Harness showed it waiting 32+ min (no step timeout).
- **Root cause**: [U] (step-completion signal lost between pod and Harness). **Fix**: abort and re-run. **Status**: Resolved by retry.

### T05 — 64 KB output variable limit
- **When**: 13/16/17 Nov. **APMs**: APM1081670 (7 ns), APM1006991 (8 ns), APM1080718 (5 ns).
- **Error**: `output variable length is more than 65536 bytes` (APM1080718 showed `BrokenPipeError` / "context canceled").
- **Root cause [C]**: `Resolve_Mapping` passes `CD_INPUT_SETS_B64` (~81 KB for APM1081670) as an output variable.
- **Fix**: write the payload to a file under `/harness` and pass the path (commit `01369e09`). Re-run with a new app name (`…java02`) because the repo already existed and the push doesn't force-push. **Verified**: payload 32 bytes; runs `Kn5Kdv…`, `NbktLs…`, `cVn1qn…` succeeded. **Status**: Resolved (branch only; not merged to master).

### T06 — Checkmarx keys missing (wrong namespace)
- **When**: 16–17 Nov. **APMs**: APM1081191, APM1081670, APM1080718, APM1006991.
- **Error**: `Failed to decrypt secret [ehappcom - CheckmarxOne Access ID] … Could not retrieve secret … Shell Script execution failed` ("using secret manager [null]" is misleading wording).
- **Root cause [C]**: shared CxOne accounts were vaulted only into a "home" APM namespace (`dig-apm1081521`, `dig-apm1081668`, `dig-apm1081076`). The APM's own namespace lacked `checkmarx_access_id/token`. The Vault login worked (the same path returned `bitbucket_token`).
- **Fix**: you ran Devin Desktop KV v2 merge-patch of only the 2 keys into each APM's own namespace. **Verified**: Checkmarx login succeeded on re-run. **Status**: Resolved.

### T07 — Checkmarx "forbidden action" and Evaluate Scan Failures cut off
- **When**: 16–18 Nov. **APMs**: bootstrap apps; real services.
- **Errors**: `'stdout': 'Failed creating a project: forbidden action'` (IgnoreFailed), then `Evaluate Scan Failures` "context canceled" ~1.5 min into 20×15 s polling.
- **Root cause [C]**: the template scans CxOne project `<svc>-policy`; many services are onboarded as `<svc>-azure-policy`, and the client can't create projects. The 1.5-min cutoff cause is [U] (pod/delegate side).
- **Fixes**: temporary `SBOM_MAX_RETRYCOUNT=0` project variable (later found back at 20). Permanent fix (team, 18 Nov): `CHECKMARX_PROJECT_NAME=<svc>-azure-policy` in input sets (120 updated; template PR #212 made it overridable). **Status**: Partially resolved (services with no CxOne project under either name remain open).

### T08 — Wiz keys missing
- **When**: 16 Nov. **Error**: Build_Docker pod "Failed to decrypt" for Wiz.
- **Fix**: merge-patched `wiz_access_id/token` (`test` placeholders, as real values weren't available). **Why**: the pod needs every secret to start; Wiz itself will still fail login. **Status**: Workaround; real Wiz creds open.

### T09 — CI pods killed ("Pod was evicted" / "context canceled") or hung
- **When**: 16–18 Nov. **APMs**: APM1081191 (`ehapp-devops`, `aks-slvr-dig-np01`), APM1081670 (`apm1081670-devops`), APM1080718 (`abi-devops`).
- **Errors**: `Received signal: terminated. Canceled step execution. Possible reason: Pod was evicted.`; Maven cut off mid-download; runs #28/#6 hung ~4.5 h at `Build and Push Docker` / `Secret Scanning`.
- **Investigation**: compared resources with working APM1081520 (same cluster) and APM1081668. Same template defaults (Maven 1792Mi/1000m). ArgoCD can't show it. K8s events last ~1 h.
- **Root cause**: [U]. Eviction, OOM, or delegate losing the task. Not confirmed by kubectl.
- **Proposed fix [P]**: platform team runs `kubectl get events -n <ns>`, `kubectl get limitrange,resourcequota -n <ns>`, raises ephemeral-storage/memory, restarts the delegate. Emails drafted to Binod (APM1081191), Sumit (APM1081670), Divya Teja (APM1080718). **Status**: Unresolved (no reply recorded).

### T10 — Artifactory 401 (human-minted token)
- **When**: 17–18 Nov. **APM**: APM1006991.
- **Error**: Maven `401 Unauthorized` for `…/artifactory/public/org/springframework/boot/spring-boot-starter-parent/3.2.5/spring-boot-starter-parent-3.2.5.pom`; curl: `"Wrong username was used"`.
- **Root cause [C]**: Vault `artifactory_username=srcuephapm1006991art`, but the reference token's subject is human `am89727` (`/access/api/v1/tokens`).
- **Contradiction resolved**: Devin Desktop concluded "use Bearer"; Devin disagreed because Basic auth works when the username owns the token (APM1080718 works with the same template), and changing to Bearer would change the shared template.
- **Proper fix [P]**: mint the token logged in as `srcUEPHAPM1006991ART`, then merge-patch. Don't patch `am89727` (personal identity, policy). **Status**: Unresolved.

### T11 — Bitbucket "Not Authorized" for real services (largest CI blocker)
- **When**: 17–19 Nov. **APMs**: APM1081191 (166), APM1081123 (20), APM1080718 (13), APM1006991 (18), APM1081670 (11).
- **Error**: `Invalid request: SCMRuntimeException: SCM request failed with: UNKNOWN: Not Authorized` at Scan stage start. Related: `Resource not found…` (EHAPPCOMSC non-master branches), `Repository SYDMPROJ/api-*-medicaid-code does not exist`.
- **Root cause [C/I]**: the APM Bitbucket service account can read new `*AZ` projects but not legacy projects (EHAPPCOMSC, EHAPPGBDSC, ARISAZSC, NEXGENENG, SOADUTL, SOADMBR, SMARTHELP, S-E-A, NA). Connector tests pass, so it isn't a token problem. The Release-stage Git tag push needs **WRITE**.
- **Fix [P]**: project admins grant `PROJECT_WRITE` (or repo write) to `srcapm<id>uephbit` / `srcUEPHAPM<id>BIT`. 0 repos are duplicated in `*AZ`. **Status**: Unresolved; still failing on 18 Nov (webhook run). Only `api_claims` (branch `release/2026-08-14` missing) and `api_auth_unified_code` (no Dockerfile → `DOCKERFILE_IN_APP=false`) are self-fixable; neither was run.

### T12 — Stale IACM workspace from typo project ID
- **When**: 13 Nov. **APM**: APM1081216. **Error**: plan `Project with identifier [APM181216] not found`, surfaced as "Network operation timed out".
- **Fix**: backed up, then API `DELETE` of workspace `DIG_unified-desktop` (204; UI had no delete). All 105 tracked resources were already gone. **Status**: Resolved.

### T13 — IDP workflow via API: "Token is not valid"
- **When**: 13 and 15 Nov. **Fix**: submit via UI; for Gold, trigger `Create_and_Manage_IACM_Workspaces` directly. Rotate PATs stored in task params. **Status**: Resolved (by workaround).

### T14 — ArgoCD cluster-scope ComparisonError
- **APM1081216**, 14 Nov. Sync Unknown; "cannot list services at the cluster scope". **Fix**: in-cluster Namespaces = `pega-ud-devops`. The stuck `apm181216-delegate` app could then finish deleting. **Status**: Resolved.

### T15 — Delegate PVC storage class not found
- **APM1081216**, 14–18 Nov. `storageclass "ebs-sc-eks" not found` (Wave 2 sheet value). App team's `kubectl get storageclass`: `ebs-sc` (default, ebs.csi.aws.com, WaitForFirstConsumer), `efs-sc`, `gp2`.
- **Fix**: `ebs-sc` (manual entry), plus trimmed the leading space in `" corp-apm1002184"`. **Verified**: delegate up, 6 connectors pass (19:13 UTC 18 Nov). **Status**: Resolved.

### T16 — Re-run race: "no eligible delegates"
- **APM1081216 / APM1003706**. The wait only happens on first create. **Fix**: wait for the delegate, then re-run. **Status**: Resolved.

### T17 — Parallel remote pipeline commits time out
- **APM1081216**, 18 Nov. 11/15 created; `helm_repo_setup` "Please check if the requested Bitbucket repository … exists", `istio_gateway_setup` "Failed to create file … 'Unknown' exited with code -1". **Fix**: a plain re-run created the remaining 4 (→15). The `master`-not-default-branch theory was **not** the cause. **Status**: Resolved.

### T18 — `APPLICATION_NAME` missing (template bug)
- **APM1081216**, 18–20 Nov. `Value not provided for required variable: APPLICATION_NAME`. **Workaround**: run `helm_repo_setup_pipeline` manually with `APPLICATION_NAME=Unified Desktop`. Then T19. **Status**: Workaround; template fix pending (onboarding/template team).

### T19 — Helm setup: missing URL, then EHDS access
- **APM1081216**, 20 Nov. (a) `EXISTING_TENANT_REPO_URL is required when EXISTING_TENANT=true and USE_NEW_INFRA_PROJECT=true`. (b) With new-tenant mode: `fatal: remote error: Repository not found … EHDS/bitbucket-helm-charts-silver` (3 retries). `srcUEPHAPM1081216BIT` has no read on `EHDS`.
- **Proposed**: an EHDS project admin grants read. A workaround via a personal staging repo `~AM73599/…` was **rejected by you**. 11 of 24 successful APMs used existing-tenant mode; 13 used EHDS (their SIDs could read EHDS). **Status**: Unresolved (~75% onboarding).

### T20 — Expired delegate SA token
- **APM1081123 Silver** (15 Nov) and **APM1080718 Silver**. `Could not retrieve secret … Shell Script execution failed`; diagnostic pipeline: `invalid expiration time (exp) claim: token is expired` (issued 7 Nov, 1 h lifetime, never refreshed); K8s API 401.
- **Fix**: delete pod (you did it in ArgoCD for APM1081123); ArgoCD restart action via API for APM1080718. **Why**: the new pod gets a fresh projected token. **Status**: Resolved.

### T21 — Gold Vault connectors pointing at wrong locations
- **APM1081123 Gold** (GBD). Fixed by matching APM1081227: Bitbucket SM/secrets `VAULT_NAMESPACE=corp-apm1002184`; Docker `KV_MOUNT=secret/gld/<apm>/np02`, `VAULT_NAMESPACE=gbd-<apm>`, `SECRET_PATH=harness/harness` (connector **and** each secret). Later made permanent by fixing workspace tfvars. **Status**: Resolved.

### T22 — Shared Bitbucket token revoked across medals
- **APM1081123** (Silver token 12 Oct revoked by Gold vaulting 29 Oct). **APM1080718** (15 Nov Gold vaulting also regenerated SonarQube). Symptom: Silver `apm<id>_bitbucket` "Please ensure that the api access credentials are correct"; Bitbucket 401, Sonar `{"valid":false}`. Blocks remote pipelines for both medals and failed all 11 APM1080718 Gold migration workspaces.
- **Fix [P→?]**: copy Gold's token into Silver, or mint once and write to both. A Silver-connector→Gold-secret workaround failed (SM decrypts on the Silver delegate). Option 2 (connector on `apm1081123-gold` delegate with Gold secrets) was applied as a temporary workaround, **to be reverted** ("reverse it once our work is done"). Revert **not recorded** [U]. APM1080718 Silver connectors all passed on 17 Nov, so it was resolved by some means [I].
- **Status**: APM1080718 Resolved [I]; APM1081123 workaround still active as of 18 Nov.

### T23 — Gold role missing
- **APM1080718**: `invalid role name "harness-apm1080718-gld"` on `aks-dig-gld-np02`. The Vault side created it; Gold vaulting v2 written. **Status**: Resolved.

### T24 — DIG Gold Docker Vault copy-paste
- **APM1080718 / APM1080955**: Docker SM used `ldap/devad/static-cred` / `corp-apm1002184` / `srcuephapm<id>qua` (copied from a reference run). Fixed in workspace inputs → re-run `PyulzkqsQyWcFZPLq66aww`. APM1080955 status [U].

### T25 — Gold Helm mode wrong
- **APM1080718**: `EXISTING_TENANT=false` → "No Chart.yaml found at …/da-automation/charts/discovery-rbac". Fix `EXISTING_TENANT=true`, `USE_NEW_INFRA_PROJECT=false` (run `KebkgdlsTEyHHjuHANqXhw`).
- **APM1081123**: `USE_NEW_INFRA_PROJECT=false` failed cloning a non-existent `ps-digitalproducts-ehapp-azure-psgbd-helm-charts-gold`. Fix `true` + `EXISTING_TENANT_REPO_URL=…/EHAPPGBD/ehappgbd-helm-charts-gold.git` (run `FcxUa5BcSDWaPYlIFu5vZg`; 334 files, 25 `values-harness.yaml`). **Status**: Resolved.

### T26 — Partial Gold workspace (35/39)
- **APM1081123**: workspace `GBD_APM1081123_gold_modify_legacy` failed 29 Oct, missing helm/istio gold pipelines → no `-gold` chart repo. **Decision**: don't tear down (it would destroy shared vars and the delegate). Corrected 4 tfvars → `Plan_and_Validate_IACM_Workspace` #27: 4 add / 10 change / 0 destroy → apply; IACMApproval approved via API. **Status**: Resolved.

### T27 — Istio before fetch
- **APM1081123**: `istio_gateway_setup_gold_pipeline` "resource name may not be empty". Run `fetch_istio_gateway_name_gold` first (`ISTIO_GATEWAY_NAME=ps-digitalproducts-ehapp-azure-psgbd-gateway`). **Status**: Resolved.

### T28 — Migration trigger step "context canceled"
- **APM1081123**: `Iterate_and_Trigger_IACM` was killed after ~2 min / 30 s. Apps already fired still completed. Re-ran with missing apps (15 → 4 → 1). **Status**: Resolved by batching.

### T29 — Duplicate object key (multi-hostname namespace)
- **APM1003706**, 18 Nov. Plan `Duplicate object key` (labelled "Network operation timed out"). 28 DNS rows under 2 namespaces, plus a trailing space in `umde-services-dev `. `locals.tf` line 39 builds `namespace => dns_hostname`.
- **Fix**: one default host per namespace (`umde-services-dev1…`, `umde-services-sit1…`); per-app `dnshost` in CD input sets (Deepak). Escalated to Manjunath for APM1000159, APM1012135, APM1006894, APM1003706. **Status**: Resolved (plan passes).

### T30 — OpenShift SCC blocks delegate
- **APM1003706**, rosa-np1. Combos tried:

| # | Task | Storage class | SA | Upgrader | Result |
|---|---|---|---|---|---|
| 1 | 090887b3 | gp3-customer-kms-v1 | argocd-application-controller | on | no delegate |
| 2 | 26bb7332 | gp3 | argocd-application-controller | on | no delegate |
| 3 | 1fd0dd36 | gp3-customer-kms-v1 | argocd-argocd-application-controller | on | no delegate |
| 4 | 2500c473 | gp3-csi | argocd-argocd-application-controller | on | no delegate |
| 5 | 63cd50ff | gp3-csi | argocd-argocd-application-controller | off | no delegate |
| 6 | a93d6430 | gp3-customer-kms-v1 | argocd-argocd-application-controller | off | no delegate |
- **Real error** (from your ArgoCD screen): `pods "apm1003706-6966768f98-" is forbidden: unable to validate against any security context constraint: … restricted-v2: .spec.securityContext.fsGroup: Invalid value: []int64{1001}: 1001 is not an allowed group, … .containers[0].runAsUser: Invalid value: 0: must be in the ranges: [1001690000, 1001699999]`. Also the upgrader Role RBAC error (fixed by upgrader off).
- **Decision**: the Helm values can't be changed by us (Harness chart defaults; template-owned; ArgoCD self-heal). Followed the other team's approach (APM1007744/APM1008553 on caas01-ocp4-np1): request `anyuid`.
- **Fix**: SR to AWS OCP support. `anyuid` added to `argocd-argocd-application-controller` in `umde-services-devops` (20 Nov). **Verified**: delegate connected (pod `apm1003706-6dfd497c8c-h9xtd`, v26.06.89501). **Status**: Resolved.

### T31 — Vault role SA not authorized
- **APM1003706**, 20 Nov. Run 1bd7e6f1: `Failed to decrypt utilization-management-decision-engine - Bitbucket Username after 3 retries`; delegate log `[ERROR] Vault auth rejected (role=harness-apm1003706-slvr, auth_path=rosa-np1): {"errors":["service account name not authorized"]}`. The role was bound only to `argocd-application-controller`. The create API says "already exists".
- **Fix**: you sent Automation Hub `PATCH /api/vault/kubernetes/role/modify/` ADD (doc 04). **Verified**: `apm1003706_vault_sm_bitbucket` SUCCESS. **Status**: Resolved.

### T32 — Wrong Vault namespace (corp- vs dig-)
- **APM1003706**: `apm1003706_vault_sm` still failed; the form had `corp-apm1003706`, but the policy grants `dig-apm1003706`. Survey: 47 of 53 working DIG `<apm>_vault_sm` use `dig-<own apm>`; none use `corp-`. **Fix**: re-run dfa92a12 with `dig-apm1003706`. Vault OK. **Status**: Resolved.

### T33 — Infra project key doesn't exist
- **APM1003706**, 20 Nov. `Please check if the requested Bitbucket repository [UMINFRA/apm1003706-gitexperience-repo] exists.` Sheet row 18 Silver = `UMINFRA`; the closest real project is `MCMUMINFRA` (MCM-UM-INFRA), which has the UMDE infra repos. Devin had read-only access there. **Needed**: confirmation + an admin creates the empty `apm1003706-gitexperience-repo` with WRITE for `srcUEPHAPM1003706Bit`. Message sent to your requester. **Status**: Unresolved.

### T34 — Bootstrap service org mismatch
- **APM1006991**: service file `.harness/orgs/DIG/projects/APM1006991/services/wkstestdcsjava02.yaml` not loaded; import says "The org identifiers from yaml and request does not match". You were asked for the first 6 lines; the outcome is not recorded. **Status**: Unknown.

### T35 — Data conflicts across sheets (Wave 2)
- 16–18 Nov. Examples: APM1003706 storage class `elvh-disk-csi` (intake) vs `gp3-customer-kms-v1` (pilot); APM1006728 cluster `irx-eks-shared-np1-cluster` vs `rosa-np1`; APM1006112 `ocp-rosa-np01` vs `rosa-np1` vs `ocp-rosa-np1`, storage `gp2, gp3-customer-kms`; APM1003706 source key `srcumde` vs `MCMUMDE`; APM1006894 hostnames with `<apiname>` placeholder; APM1010417 ArgoCD URL without port, validation repo `<appname>` placeholder. **Status**: Emails sent; answers mostly not recorded [U].

### T36 — ArgoCD password verification impossible from VM
- 16 Nov, the 4 rosa APMs: DNS didn't resolve; public DNS gave `10.189.x`, timeouts on 443/8544. **Status**: Unverified (to be done from your laptop).

### T37 — Two Artifactory project keys
- **APM1010417**: the form takes one key. Program team reply: "you can add as many artifactory repos u want to in filestore → pip.conf, maven's settings.xml and nodejs npmrc". Use `aedlnp` in the form; add `aedl-maven-*` to `/ci_config/settings.xml`; the ART service ID needs read on both. **Status**: Resolved (approach).
