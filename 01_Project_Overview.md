# 01 — Project Overview

> **Evidence legend** (used in every document)
> - **[C] Confirmed**: seen directly in Harness, Bitbucket, Vault, ArgoCD or API output during our sessions, or stated by you or a colleague in chat.
> - **[I] Inferred**: concluded from evidence but never directly verified.
> - **[P] Proposed**: an approach that was suggested but not carried out, or not confirmed as carried out.
> - **[U] Unverified / Unknown**: discussed but never established.
>
> **Sources**: 11 Devin sessions, 13–20 Nov 2026 (listed in `00_Master_Index.md`), plus Devin's long-term memory notes for you. Secrets are replaced with `[REDACTED]`.

## 1. What the project is
The **UEP Harness program** at Elevance Health moves every application (identified by an **APM ID**) onto one CI/CD platform: **Harness**. It uses golden (standard) pipelines, Kubernetes delegates and **HashiCorp Vault–backed secrets**. [C] (Confluence *Self-Service Onboarding & Migration guide* v18 and *Onboarding/Migration engagement page* v36, as cited in the KT session on 18 Nov.)

The documented source platforms ("migration patterns") are: [C]
1. **Container EHDS**: Tekton builds plus ArgoCD deploys on the "Brownfield" CaaS clusters. Bulk-migrated through the IDP `MigrationWorkflow`.
2. **Container non-EHDS**: other containerised apps.
3. **VM / Bamboo**: classic Bamboo plans (`bamboo.elevancehealth.com`, which is being retired).
4. **APIGEE**.

**Important reality check [C]:** almost all of the hands-on work in our history is the **EHDS / container** track (Silver/Gold onboarding, `migration_pipeline`, bootstrap, golden CI). The Confluence docs list "Bamboo apps to Harness" readiness as **ETA TBD**. **No Bamboo plan was migrated in any session we have.** Bamboo-specific mapping (jobs, tasks, variables) was never worked on hands-on; see doc 06 for what is and isn't known.

## 2. Your team's role
- Your team is the **onboarding team**, and for Vault you said "**we are the vault team**" (20 Nov). [C] (your words, 18 Nov: "we are the onboarding team and we are supposed to fix this mess")
- Work is assigned **per APM** (for example, your exercise tickets APM1003706, APM1012135, APM1006894, APM1000159). [C]
- You stated your typical load is about **two APMs per day**. [C] (your statement in the extraction prompt; not measured)

## 3. The three phases
| Phase | What it produces | Main tooling |
|---|---|---|
| 1. Vaulting | Vault namespace `<bu>-<apm>`, K8s auth role `harness-<apm>-<slvr/gld/plat>`, KV secret `secret/<medal>/<apm>/<tier>/harness/harness` holding the service-ID tokens | Bitbucket `HARNESSADM/vault-automation` scripts, Automation Hub API, Devin Desktop on your laptop |
| 2. Harness onboarding | Harness project `APM<id>`, delegate `apm<id>`, connectors, secrets, 15 pipelines, environments/infras, gitexperience repo, Helm chart repo, Istio gateway, discovery agents | IDP workflows `Harness_Onboarding_Project_Setup` (Silver) and `harness_project_modify_gold_platinum` (Gold/Platinum), which run IACM/Terraform through `Create_and_Manage_IACM_Workspaces` |
| 3. Migration and validation | Harness services and CI/CD input sets per app, CI/CD runs, CI issue reports | `migration_pipeline` (IDP migration workflow), `bootstrap_pipeline`, golden CI templates, `cd_harness_native_pipeline` |

## 4. Timeline (chronological, from accessible history)
| Date (2026) | Session | Main events |
|---|---|---|
| 13 Nov | Bootstrap pipeline (6d57…) | Harness API access set up. APM1081191 bootstrap run, CI blocked by `core_template_access` 403. APM1081670 bootstrap hit pod-setup timeout and the 64 KB limit. |
| 13–14 Nov | APM1081216 Silver (a600…) | Stale workspace `DIG_unified-desktop` (typo `APM181216`) deleted. IDP API submission fails "Token is not valid". ArgoCD in-cluster namespace scope fixed. Delegate Pending: storage class `ebs-sc-eks` not found. |
| 15 Nov | APM1081123 Gold (05bf…) | Gold Vault connectors re-pointed. Silver delegate SA token expired (pod restart). Bitbucket one-token-per-user problem found. Temporary Silver→Gold connector workaround. |
| 15 Nov | APM1080718 Gold (d63d…) | Gold run directly via `Create_and_Manage_IACM_Workspaces`. Silver delegate restarted through ArgoCD API. Gold role missing, then Gold vaulting done. Docker Vault path fixed. Helm `EXISTING_TENANT=true`. Gold done, but migration failed because Gold vaulting revoked the Silver Bitbucket token. |
| 16 Nov | Bootstrap (6d57…) | Checkmarx keys vaulted into the wrong namespaces (shared CxOne accounts). Wiz keys missing. `SBOM_MAX_RETRYCOUNT=0` workaround. Pod "evicted" / "context canceled". |
| 16 Nov | Remember tickets (cea8…), DATA validation (1944…) | Your 4 rosa-np1 APMs recorded. ArgoCD passwords could not be tested (no route). Vaulting readiness report. Silver data-validation workbook for 11 APMs and emails. |
| 17 Nov | CI runs (3fbc…) | 230 CI runs across 5 APMs. 227 failed `SCMRuntimeException … Not Authorized`. CI_issues_Analysis report. NextGen (`_NXT`) excluded. |
| 17–18 Nov | Bootstrap (6d57…) | APM1080718 and APM1006991 bootstraps (64 KB fix). APM1006991 Artifactory 401: token minted by human `am89727`. |
| 18 Nov | APM1081123 Gold (05bf…) | Gold workspace repaired (4 add, 10 change, 0 destroy). Gold Helm repo created. Istio order fixed. Gold migration of 20 apps in 3 batches. 40 Gold input sets. |
| 18 Nov | KT sessions (065e…, d343…) | Bamboo→Harness KT, Silver onboarding/vaulting KT guide (HTML). APM1081123 CI: 20/20 Not Authorized on `EHAPPGBDSC`. |
| 18–20 Nov | APM1003706 (7ee4…) | Duplicate object key (28 DNS rows). Delegate SCC rejection on rosa-np1; `anyuid` granted. Vault role SA binding patched. Vault namespace `corp-` changed to `dig-`. Infra project key `UMINFRA` doesn't exist. |
| 18–20 Nov | APM1081216 (a600…) | `ebs-sc` fix, delegate up, 15 pipelines. `APPLICATION_NAME` template bug. Manual Helm runs fail on EHDS repo access. About 75% done. |
| 19 Nov | APM1010417 Wave 3 (725e…) | Two Artifactory project keys; CI File Store `settings.xml` approach confirmed by the program team. Vault role JSON. |

## 5. Current state in one paragraph
Several Wave 1 EHDS APMs (APM1081191, APM1081123, APM1080718, APM1081520, APM1006991, APM1081670) are onboarded and migrated, but **no golden CI run succeeded for real app services**. 205 of 207 failed at the Bitbucket checkout (service accounts lack access to legacy `*SC` projects). Bootstrap test apps hit Vault-key, Checkmarx, pod-kill and Artifactory problems. **No CD deployment was run in any session.** Wave 2 Silver onboardings: APM1081216 is about 75% done (Helm step blocked). APM1003706 has the delegate and Vault fixed but is blocked on the Bitbucket infra project key. APM1010417 (Wave 3) has its data prepared. [C]
