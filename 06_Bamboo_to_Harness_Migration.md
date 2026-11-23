# 06 — Bamboo / EHDS to Harness Migration (Phase 3)

## 6.1 Scope reality [C]
- The migration we actually performed was **EHDS (Tekton + ArgoCD) → Harness**, driven by the IDP migration workflow and `migration_pipeline`.
- **Bamboo**: the org knowledge says classic Bamboo plans (MSBuild/.NET Framework, scripts) go to Harness CI on delegates. .NET Framework needs a **Windows delegate + Windows runner** (e.g. `mcr.microsoft.com/dotnet/framework/sdk:4.8-windowsservercore-ltsc2022`, multi-GB; mirror internally and keep nodes warm). Delegates for that track live in `apm<n>-bamboo-delegates-ns`. **None of this was exercised in our sessions.** [C as org knowledge / U as practice]
- How Bamboo jobs, tasks, variables and credentials map to Harness was **never worked through**. [U] The general principle that *is* established: credentials become Vault keys referenced by Harness secrets; build logic becomes golden-template stages; per-app variables become input-set values. [I]

## 6.2 How apps and pipelines are identified [C]
- `migration_pipeline` reads each app's build info from the **EHDS migration API** and maps it to a golden CI pipeline via project variable `migration_baseimage_to_pipeline_mapping`. This includes per-medal chart repos such as `manifest_repo_name_gold`.
- Your inputs: target org/project (e.g. `DIG` / `APM1080718`. A stale "Medisys" value from another APM's form was caught), medal, `app_list`.
- Trackers: EHDS-Tenant-intake-tracker.xlsx (Master Data, Wave-1 APM Mapping), the UEP Master Tracker (status like "02g – Migration Complete – Gold", "07a – Migration Complete – Silver").

## 6.3 What migration creates [C]
Per app, a `Create_and_Manage_IACM_Workspaces` run named `cd-migration-<apm>-<app>` creates:
- the Harness service (remote YAML in the gitexperience repo),
- `<svc>_ci_inputs` on the language's golden CI pipeline,
- CD input sets `<app>_gold_uat` / `<app>_gold_perf` (Gold) or `<app>_slvr_<ns>` (Silver) on `cd_harness_native_pipeline`,
- CI webhook triggers (PR-merge). On APM1081123 Gold the CI trigger steps were **skipped** in all 3 runs.

## 6.4 Manual vs automated [C]
**Hybrid.** Bulk creation is automated (IDP + Terraform). The onboarding team manually: validates data, submits forms, runs Gold post-steps, re-runs failed batches, runs bootstrap/CI tests, and fixes Vault/Bitbucket/cluster issues.

## 6.5 YAML maintenance [C]
- Pipelines, input sets and services are **REMOTE** entities in `<KEY>/apm<id>-gitexperience-repo`, branch `master`. Harness syncs them (it may take minutes). The inputSets `searchTerm` misses them, so list with `pageSize=500` and filter client-side. GET with `&branch=master`.
- Project pipelines are thin wrappers over account golden templates (`CI_Java_Build_Pipeline`, `CI_NodeJS_Build_Pipeline`, `CI_ReactJS_Container_Build_Pipeline`, `CI_Python_Build_Pipeline`, `CI_Go_Build_Pipeline`, `CI_AngularJS_Container_Build_Pipeline`, `CI_Monorepo_NodeJS_Container_Build_Pipeline`) with no pinned version (stable). Template source: `HARNESSGOL/elvh-ci-cd-templates`.
- To use new template variables (e.g. `CHECKMARX_PROJECT_NAME`, `GITLEAKS_FULL_HISTORY`), **refresh/reconcile the project pipeline from the template first**.
- Pipeline fixes during testing were saved to branch `devin/bootstrap-init-timeout` and never merged to `master`. Your rule: only fix if a master run fails ("if it hits the limit, then lets do the fix").

## 6.6 Golden CI stages (Java template, as observed) [C]
| Stage | Steps (observed) | Failure behaviour |
|---|---|---|
| Scan | clone (codebase connector `apm<id>_bitbucket`), TLS setup, Gitleaks, `CheckmarxOne_1` (`--scan-types sast,sca`, project `<repo>-policy`), `Evaluate Scan Failures` (`cx-scan-id.sh` polls `SBOM_MIN/MAX_RETRYCOUNT`, 15 s apart), Secret Scanning | Checkmarx is IgnoreFailed; Evaluate polls |
| Build (Build_Docker) | `Maven_Build` (Artifactory via `settings.xml`), `Sonarqube_1` (ignore-fail), `Build_and_Push_Docker` (needs `APP_PUSH_TYPE=docker`), `Wiz_Container_Scan` | Wiz has **no** ignore-failure |
| Release | push image, **Git Tag** push (needs WRITE on the repo) | Java/Go/React/Angular/Monorepo have no failure strategy |
Default resources: Maven 1792Mi / 1000m, Wiz 1792Mi / 500m. [C]

## 6.7 Validation method: bootstrap [C]
Colleague guidance (Surya, and another colleague): "run the bootstrap pipeline, it creates a test app, input set and service object; then trigger the pipelines with them". Reference: APM1081520 run `rvTa1lUhSYmcjmr3ld4LzQ`.
- 30 inputs. Swap `app_name`/`repo_name` (`wkstest<tenant>java01`), `project_id`, `bb_project_name` (the Bitbucket project of the APM's gitexperience repo / app code), `git_username_secret`/`git_password_secret` (`apm<id>_bitbucket_username/_password`), `gitexp_connector` (`apm<id>_bitbucket`).
- Steps: `Clone_template_repo` → `CookieCutter_code_generator` → `CreateRepo` → `DirectPush_Custom` → `Apply_Branch_Restrictions` → `Set_Repo_Permissions` → `Create_ECR_Repo` → `Resolve_Istio_Gateway` → `Resolve_Mapping` → `Commit_Input_Sets`.
- Run **once** per APM; it creates the CI input set and one CD input set per namespace per medal. Then CI once, CD per input set. A re-run after a partial run needs a **new app name** (no force-push; `master` is protected).

| APM | Result | Notes |
|---|---|---|
| APM1081191 (ehappcom) | Success `KsFO8lgjQW-cfPyssUU73A` | 5 CD input sets |
| APM1081670 (sydmbrmc, GBD) | Success on run 5 `Kn5Kdv…` (`java02`) | needed `initTimeout: 30m` + 64 KB file fix; 7 CD sets |
| APM1080718 (arisaz) | Success `NbktLs…` (`java02`) | 64 KB fix |
| APM1006991 (dcs) | Success `cVn1qn…` (`java02`) | service file rejected "org identifiers from yaml and request does not match" [U final] |

## 6.8 Execution validation and production readiness
- Order: Silver onboard → CI → Silver CD → Gold onboard → Gold CD (Gold can run in parallel with Silver CI testing). [C]
- Report rule: per service, show run link, final status (**never call IgnoreFailed "Success"**), image tag, failed or ignored step, exact error line, root cause. [C]
- **No CD run and no production cutover** happened in our history. Cutover, rollback and ServiceNow change steps were not discussed. [U]

## 6.9 Migration examples
- **APM1081123 Gold (18 Nov)** [C]: 3 workflow runs (15 + 4 + 1 apps) because `Iterate_and_Trigger_IACM` died with "context canceled". All 20 `cd-migration-apm1081123-<app>` runs succeeded, giving 40 Gold input sets. CI then failed 20/20 on `EHAPPGBDSC` Not Authorized.
- **APM1080718 Gold (15 Nov)** [C]: `migration_pipeline` generated 20 Gold CD input sets for 11 apps, but all 11 workspace runs failed: "Please check if your Bitbucket credentials in connector [apm1080718_bitbucket] are valid" (Silver token revoked by Gold vaulting). Re-migration after the token fix is **not recorded** [U].
