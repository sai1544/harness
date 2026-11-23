# 14 — Your Responsibilities and Contributions (evidence-based)

Many technical actions in the history were performed **by Devin through the Harness API, under your direction and explicit approval** ("before running show me the inputs"). The lists below separate what *you* did, what you directed, and what you learned.

## 14.1 Completed by you [C]
- Set up Harness API access for Devin (personal PAT stored as a secret) and supplied all reference sheets (`excel.zip`, `data detail sheets.zip`, vault-automation zip, data sheets).
- **Submitted IDP onboarding/migration forms in the UI**: APM1081216 Silver (tasks 6972052e, 331866e4, ece38d1e, 02b1eae8, 1eecf24f), APM1003706 Silver (35df255d, 090887b3, 26bb7332, 1fd0dd36, 2500c473, 63cd50ff, a93d6430, 1bd7e6f1, dfa92a12), APM1080718 Gold migration (428aa976), APM1081123 Gold migration (0fc04152, 0493cb57, 76d2dac8).
- **Ran vaulting through Devin Desktop**: KV v2 merge-patches for Checkmarx (APM1081191, APM1081670, APM1080718, APM1006991), Wiz (APM1081191, APM1081670), Artifactory (APM1006991); APM1080718 Gold vaulting (7 keys, v2); Artifactory token owner investigation (`/access/api/v1/tokens`).
- **Fixed the Vault role binding** for APM1003706 via Automation Hub PATCH (role modify ADD), in Postman.
- **ArgoCD UI actions**: deleted the stuck APM1081123 Silver delegate pod; set in-cluster Namespaces for the pega ArgoCD (APM1081216) [I from screenshots]; collected APM1003706 delegate tree/ReplicaSet errors.
- **Raised the SNOW SR to AWS OCP support** for `anyuid` (APM1003706). Resolved 20 Nov.
- **Data validation and comms**: Silver onboarding data workbook for 11 Wave 2 APMs (one APM per tab), missing/conflict emails to app owners (Prasanna, Srinivasan), confirmation email (APM1000782, APM1003706, APM1006894), escalation to Manjunath/Deepak on multi-hostname namespaces, storage-class message to the Pega team, CI pod-kill emails (Binod, Sumit, Divya Teja), APM1003706 status messages, APM1010417 Artifactory question to the program team.
- **Gave/prepared a KT** to your team on vaulting and Silver onboarding (KT_Vaulting_Silver_Onboarding.html prepared on 18 Nov).

## 14.2 Directed and approved (executed by Devin) [C]
Bootstrap runs (APM1081191, APM1081670, APM1080718, APM1006991); 230 CI runs and the CI_issues_Analysis workbook; Gold connector repairs (APM1081123); Gold workspace repair and helm/istio gold pipelines (APM1081123); APM1080718 Gold via the direct pipeline; Silver delegate restart via the ArgoCD API (APM1080718); workspace deletion (APM1081216); manual helm runs (APM1081216); project variable `SBOM_MAX_RETRYCOUNT` changes.

## 14.3 In progress [C]
- APM1081216 Silver: Helm/Istio/DA steps (EHDS read access).
- APM1003706 Silver: infra project key / gitexperience repo.
- APM1010417 Wave 3 Silver: data and Vault role JSON prepared (`eda-apm1010417`).
- Your 4 exercise APMs: vaulting readiness (UMDE/EAPI near-ready; ACMP/DHP blocked on service IDs).
- CI blockers for Wave 1 APMs (Bitbucket access; pod kills; APM1006991 Artifactory).

## 14.4 Observed / learned
Vault K8s auth end to end; IACM/Terraform onboarding internals; ArgoCD namespace scoping; OpenShift SCC; KV v2 merge-patch; Bitbucket one-token rule; golden template structure; IDP vs pipeline triggering; reading plan logs vs surface errors.

## 14.5 Discussed but not confirmed as completed [U]
Revert of the APM1081123 Silver Bitbucket workaround; restoring `SBOM_MAX_RETRYCOUNT` (it was at 20 when checked on 18 Nov); APM1080718 Gold re-migration after the token fix; APM1006991 service org fix; any CD run; ArgoCD password rotation (you said "i'll rotate", then "not rotated").

## 14.6 Skills you are developing
HashiCorp Vault (namespaces, KV v2, Kubernetes auth, roles/policies), Harness (IDP, IACM, connectors, delegates, golden templates, input sets, Git Experience), Terraform state/plan reading, Kubernetes/OpenShift troubleshooting (PVC/storage classes, SCC, RBAC), ArgoCD, Bitbucket permissions, CI security tooling (Checkmarx, Wiz, Sonar, Gitleaks), stakeholder communication and data reconciliation.
