# 09 — Delegate Architecture

## 9.1 How a delegate is created [C]
- Never by pipelines. Onboarding Terraform creates the ArgoCD Application `apm<id>-delegate` in the **tenant's ArgoCD** (labels `managed_by=terraform`, `project_id=APM<id>`; project `default`; destination `in-cluster` = `https://kubernetes.default.svc`; namespace = delegate namespace).
- Chart `harness-delegate-ng` **1.0.32** from `https://app.harness.io/storage/harness-download/delegate-helm-chart/`. Image seen: `quay-prod…/harness-delegate:26.06.89501` (Silver), and Gold `26.10.90209`. Org knowledge also lists `quay-nonprod.elevancehealth.com/eks-blueprints/harness-delegate:<version>`.
- Auto-sync with self-heal (manual ArgoCD edits get reverted). Sync options `CreateNamespace`, `ServerSideApply`.
- Pod spec: Deployment `apm<id>`, SA from the form, **10Gi ephemeral scratch PVC at `/scratch`** using `delegate_storage_class`. The chart defaults to `runAsUser: 0`, `fsGroup: 1001`. An optional **upgrader** CronJob with its own Role (`apm<id>-upgrader`) is controlled by `delegate_upgrader_enabled`.
- Terraform waits `time_sleep.wait_for_delegate` (~4 min) on first create only, then creates delegate-dependent resources (connector tests, remote pipelines).

## 9.2 Communication [C]
- The delegate makes outbound HTTPS to `app.harness.io` (registers with the delegate token; selector `apm<id>`). Harness never connects inbound.
- To Vault: SA projected token → Kubernetes auth login.
- To K8s API: in-cluster, to create `harnessci-*` build pods (CI) and deploy (CD).
- Check from Harness: `delegate-setup/listDelegates`, delegate token usage, connector `testConnection`.

## 9.3 Failure modes and fixes (all seen)
| Hop | Symptom | Root cause | Fix |
|---|---|---|---|
| ArgoCD → K8s | sync **Unknown**, Health Healthy, ComparisonError "cannot list resource … at the cluster scope" | tenant ArgoCD controller is namespace-scoped; in-cluster Namespaces empty | Settings → Clusters → in-cluster → Namespaces = delegate ns → Save → Invalidate cache → Hard refresh (APM1081216) |
| Pod scheduling | Pod Pending 0/1, PVC pending `storageclass "ebs-sc-eks" not found` | class not on cluster (sheet wrong) | `ebs-sc` via "enter storage class manually" |
| OpenShift admission | ReplicaSet: `forbidden: unable to validate against any security context constraint … restricted-v2: fsGroup 1001 not allowed; runAsUser 0 must be in [1001690000, 1001699999]` | chart runs as root; SA only has `restricted-v2` | OCP team grants `anyuid` to `argocd-argocd-application-controller` in `umde-services-devops` (SR to AWS OCP support) |
| RBAC | sync failed creating Role `apm<id>-upgrader` (escalation) | ArgoCD SA can't grant cronjob update/patch | untick `delegate_upgrader_enabled` |
| Registration timing | apply: "no eligible delegates … selectors [apm<id>]" right after re-run | wait only on first create | wait for delegate Connected, re-run form |
| Vault | "token is expired" / K8s API 401 | projected SA token stopped refreshing (~7 days) | restart delegate pod (ArgoCD delete pod / Deployment restart action) |
| Vault | "service account name not authorized" | role not bound to delegate SA | Automation Hub role modify ADD SA |

## 9.4 Restart without kubectl [C]
From a working delegate (ShellScript step): ArgoCD `POST /api/v1/session`, then `POST /api/v1/applications/apm<id>-delegate/resource/actions?namespace=<ns>&resourceName=apm<id>&version=v1&kind=Deployment&group=apps` with body `"restart"`. Pass the password through a temporary Harness secret and delete it after. The working ArgoCD password may be the `argocd_password` on the Silver IACM workspace (the sheet can be stale). Used for APM1080718 Silver.

## 9.5 Network constraints [C]
ArgoCD hosts (`*.awsdns.internal.das`, `argocd-server-<ns>.apps.rosa-np1…` → `10.189.x`), Vault and Automation Hub are **not reachable from the Devin cloud VM**. The rosa-np1 port mapping was 8544. Work there was done through your browser, Devin Desktop, or a delegate.
