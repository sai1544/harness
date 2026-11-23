# 11 — Helm and ArgoCD

## 11.1 ArgoCD's roles [C]
1. **Delegate deployment** (all medals): tenant ArgoCD per devops namespace (e.g. `https://argocd-server-<ns>.apps.rosa-np1.166k.p1.openshiftapps.com`, `<tenant>-argocd-aks-silver/gold.elevancehealth.com`, `pega-ud-devops.pega-compass-nginx.slvr-dig-sharedservpega.awsdns.internal.das`). It is namespace-scoped and uses user `admin`. SA: AKS/EKS `argocd-application-controller`; OpenShift `argocd-argocd-application-controller` (`<instance>-argocd-application-controller`).
2. **Legacy EHDS app deploys** (the source of migration).
- ArgoCD **does not show** Harness CI pods and can't show node pressure/OOM. It can't confirm pod evictions.
- The form expects `hostname:port` for the ArgoCD server (e.g. `poc-devops.nginx.slvr-corp-sharedservedaiaas.awsdns.internal.das:443`).

## 11.2 Helm chart repos [C]
- `helm_repo_setup_pipeline` (Silver) / `helm_repo_setup_gold_pipeline` create `<Harness project name>-helm-charts-<medal>` in the APM's infra Bitbucket project, with `values-harness.yaml` per service and `da-automation/charts/discovery-rbac`. Also 5 `<project>-artifactory-*-<medal>` repos (new-infra mode).
- Source: existing tenant repo (`<tenant>-helm-charts-<medal>`, e.g. `soaawsad/tenxeng-silver-container-helm-chart`) or the **EHDS starter** `EHDS/bitbucket-helm-charts-silver` (service charts `nodejssvc`, `goLangsvcWithIngress`, `dotnetsvcWithIngress`, ingress and istio templates). The EHDS source is hardcoded, so the service ID needs read on `EHDS`.
- Modes:

| EXISTING_TENANT | USE_NEW_INFRA_PROJECT | Behaviour |
|---|---|---|
| true | false | add Harness files in place to an existing `<project>-helm-charts-<medal>` (project name = tenant) |
| true | true | copy from `EXISTING_TENANT_REPO_URL` into a new repo (required; blank → "EXISTING_TENANT_REPO_URL is required") |
| false | — | new tenant: copy from `EHDS/bitbucket-helm-charts-<medal>` (no da-automation/values-harness → "No Chart.yaml found at …/da-automation/charts/discovery-rbac" when the tenant actually exists) |

- New required var `APPLICATION_NAME` = the APM's SNOW app name (e.g. "Unified Desktop", "Experience Data Platform"). It raises SNOW requests for 9 AD groups `HarnessCloudCostManagement_User_APM<id>_<role>`. `SKIP_SNOW_IF_EXISTS` is a flag.
- The CD template reads `values-harness.yaml` with `<+serviceVariables.X>`; a missing service variable gives "Error while fetching values yaml…" (org knowledge, APM1081726).
- The Gold mapping `manifest_repo_name_gold` lives in `migration_baseimage_to_pipeline_mapping` (27 entries on APM1081123).

## 11.3 Istio / DNS [C]
- `istio_gateway_setup(_gold)_pipeline` creates gateway `<project-name>-gateway`; `ISTIO_GATEWAY_NAME` comes from `fetch_istio_gateway_name_gold`.
- `dnsHost` project variable = `{medal: {namespace: host}}` (one host per namespace). Per-app hostnames are set in each CD input set's `dnshost` (Deepak's guidance). Bootstrap `Resolve_Istio_Gateway` example: `sydmbrmc-gateway`.
