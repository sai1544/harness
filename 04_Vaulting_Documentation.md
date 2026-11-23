# 04 — Vaulting (Phase 1)

## 4.1 How Vault is hosted and reached
- Vault endpoint: `https://vault.acr.awsdns.internal.das:8200`. [C] It is **not reachable from the Devin cloud VM**. It is reachable from your laptop / Devin Desktop. [C]
- Vault Enterprise **namespaces** are used per APM: `<bu>-<apm>`, e.g. `dig-apm1081191`, `gbd-apm1081123`, `eda-apm1010417`, `csbd-apm1012135`. [C]
- A shared CORP namespace holds Bitbucket LDAP static creds: `corp-apm1002184`, path `ldap/us/static-cred/srcueph<apm>bit`. This is "an intentional shared space" (your answer for APM1081216). [C]
- The **write path** (vault-automation README): your Okta login (`portalsso.elevancehealth.com`) → AWS → a write pod (`buildah-debug`) on the `workos-tools-platinum-container` EKS cluster. Reads use a read pod (`apm-read-pod.yaml`) in the APM's cluster. [C]
- Vault environment label in Automation Hub responses: `"vaultEnvironment": "BROWNFIELD"`. [C]

## 4.2 Secret engine and layout
- Engine: **KV v2** (paths include `/data/`; versions are tracked, e.g. "pre-patch version was 11, rollback via KV history available"). [C]
- Path pattern: `secret/<env>/<apm>/<tier>/harness/harness`

| Medal | env | tier | Example |
|---|---|---|---|
| Silver | `slvr` | `np01` | `secret/slvr/apm1081191/np01/harness/harness` |
| Gold | `gld` | `np02` | `secret/gld/apm1080718/np02/harness/harness` |
| Platinum (+DR shares it) | `plat` | `prod` | `secret/plat/<apm>/prod/harness/harness` [C from Confluence via KT, never exercised] |

- Keys CI needs (the template fails at the first missing key): `bitbucket_token`, `bitbucket_username`, `checkmarx_access_id`, `checkmarx_access_token`, `wiz_access_id`, `wiz_access_token`, `artifactory_username`, `artifactory_token`, `docker_username`, `docker_password`, `sonarqube_access_token` (Silver also had `sonarqube_token` referenced on APM1080718). [C]
- APM1080718 Gold vaulting wrote 7 keys: `bitbucket_token`, `bitbucket_username`, `sonarqube_access_token`, `artifactory_username`, `artifactory_token`, `docker_username`, `docker_password`. No Checkmarx/Wiz keys, which were added later with merge-patch. [C]

## 4.3 Authentication method and paths
- Method: **Kubernetes auth**. There is one auth mount per cluster, holding that cluster's API address and CA. The delegate presents its **service-account JWT**. [C]
- Auth paths seen:

| Auth path | Cluster | Flavor | APMs |
|---|---|---|---|
| `aks-slvr-dig-np01` | DIG Silver AKS | aks | APM1081191, APM1080718, APM1081520 |
| `aks-dig-gld-np02` | DIG Gold AKS (eastus2) | aks | APM1080718, APM1081191, APM1081520, APM1080955 |
| `gbd-slvr-np01` | `aks-gbd-slvr-np01` | aks | APM1081123, APM1081670 |
| `aks-gbd-gld-np02` | GBD Gold AKS | aks | APM1081123 |
| `rosa-np1` | OpenShift ROSA `rosa-np1` | ocp | APM1003706, APM1000159, APM1012135, APM1006894 |
| `eks-pega-compass-np01` | `PEGA-COMPASS-EKS-NP01-CLUSTER` | eks | APM1081216 |
| `eda-np1` | `EDA-EKS-NP01-Cluster` | eks | APM1010417 |
| `sydneymbr-gold-container` | (EKS) | eks | APM1006991 Gold (your JSON example) |

- Pass the mount **without** the `auth/` prefix to scripts (`-A aks-dig-gld-np02`); the API builds `/v1/auth/<path>/login`. Automation Hub **modify** responses show `"authPath": "auth/rosa-np1"`. [C]
- The Harness template logs in **without** the namespace header and adds `X-Vault-Namespace` only for the KV read. An early Devin Desktop prompt got this wrong and was corrected (15 Nov). [C]

## 4.4 Roles and policies
- Role name: `harness-<apm>-<slvr|gld|plat>` (e.g. `harness-apm1080718-gld`). [C]
- Policy name: `harness-<bu>-<apm>-<env>` (e.g. `harness-dig-apm1080718-slvr`). It grants read on `<ns>/secret/<env>/<apm>/<tier>/data/harness/*`, plus the Bitbucket LDAP static cred. [C]
- Role binding = `bound_service_account_names` + `bound_service_account_namespaces` (the delegate namespace and the SA the delegate runs as). [C]
- Created through the **Automation Hub API** (`https://automationhub-dev.916044042746.awsdns.internal.das`):
  - create: `vault_kubernetes_role_harness_create` (`/api/vault/kubernetes/role/harness/`). Fails with "Role … already exists" for an existing role.
  - modify: `PATCH /api/vault/kubernetes/role/modify/` with `boundClaims` `action: ADD`. Additive. [C]
  - namespace: `vault_namespace_create` (no adGroup). [C]
- Role-create JSON used by the team (example, APM1010417): [C]
```json
{
  "kubernetesFlavor": "eks",
  "authPath": "eda-np1",
  "environment": "slvr",
  "sansPairings": [{ "ns": "poc-devops", "sa": "argocd-application-controller" }],
  "vaultNamespace": "eda-apm1010417"
}
```
  APM1080718 Gold: `aks`, `aks-dig-gld-np02`, `gld`, ns `abi-devops`, SA `argocd-application-controller`, `dig-apm1080718`. APM1003706 Silver: `ocp`, `rosa-np1`, `slvr`, ns `umde-services-devops`, `dig-apm1003706`.
- Role-modify body that fixed APM1003706 (20 Nov): [C]
```json
{
  "vaultEnvironment": "BROWNFIELD",
  "authPath": "auth/rosa-np1",
  "roleName": "harness-apm1003706-slvr",
  "boundClaims": [
    { "action": "ADD", "namespace": "umde-services-devops", "serviceAccount": "argocd-argocd-application-controller" }
  ]
}
```
  Response: "Kubernetes/JWT role updated successfully". `bound_service_account_names` became `["argocd-application-controller","argocd-argocd-application-controller"]`.

## 4.5 Service IDs and credential sourcing
- Per-APM CORP service IDs created through SNOW: `srcUEPHAPM<id>BIT` (Bitbucket), `…ART` (Artifactory), `…SON` (SonarQube). Older/Wave 1 naming is `srcapm<id>uephbit`. Status is tracked in the SNOW Master xlsx (create + AD-group RITMs). [C]
- Quay: `srcehdsquay` with a standard fixed password, set manually. [C]
- Checkmarx (CxOne): from a "CxOne creds" sheet. Column A **Service ID** → `checkmarx_access_id`; column B **Secret** → `checkmarx_access_token`. Several APMs share **combined accounts** for duplicate DLs (e.g. `SrvDEPEGBDCSBDAdvSoaCx1`, `SrvDEPEGBDOptimizersCx1`, `SrvDEPENLPBenefitCx1`). The sheet's "vault NS" column pointed to *another* APM's namespace (`dig-apm1081521`, `dig-apm1081668`, `dig-apm1081076`). [C]
- Wiz: real values not available to you; `test` placeholders were patched temporarily for bootstrap apps. [C]
- Tokens must be minted **while logged in as the service ID**. `onboard-tool.sh` warns when a token is minted under a human identity. APM1006991's Artifactory token was minted by human `am89727`, causing 401 "Wrong username was used". [C]

## 4.6 vault-automation repo (Bitbucket `HARNESSADM/vault-automation`) [C]
| Script | Use |
|---|---|
| `read-apm-secret.sh -N <ns> -A <authpath> -e <env> -t <tier>` | back up / read current secret as JSON (from a read pod in the APM cluster) |
| `put-apm-secret.sh -N … -k <key> --base current.json [--dry-run]` | additive write that keeps existing keys |
| `gen-bitbucket-token.sh -r <SID>` | **revokes** the existing token, then mints a new one (one token per Bitbucket user) |
| `gen-sonar-token.sh -r <SID>` | same for SonarQube |
| `onboard-tool.sh --base … [--bitbucket-replace]` | orchestrated vaulting |
| skill `vault-onboard` | used by Devin Desktop prompts |

Harness has a pipeline `vault_k8s_role_registration` that registers roles; **it does not add secret keys**. [C]

## 4.7 Rules you set for vaulting [C]
- "**we should not remove any of the usefull things**": additive or patch only, back up first (`--base current.json`, or true KV v2 `PATCH` with `Content-Type: application/merge-patch+json` naming only the new keys).
- Never print token values; clean up local token files and pod work dirs afterwards. (Your Devin Desktop runs reported doing this.)
- Use only the APM's own credentials.

## 4.8 Harness ↔ Vault wiring
- Connector type: custom secret manager built from account template `ElevanceCustomVaultSecretManagerTemplate`. Inputs: `VAULT_NAMESPACE`, `KV_MOUNT`, `SECRET_PATH`, `AUTH_PATH`, `ROLE`, delegate selector. [C]
- **Each secret carries its own copy of these env vars**, so a fix means updating the connector **and** each secret. A secret's `secretManagerIdentifier` cannot be changed after creation (HTTP 400). [C]
- IDP form Vault fields: `vault_auth_path`, `vault_role`, `vault_namespace`, `vault_kv_mount`, `bitbucket_vault_auth_path`, `bitbucket_vault_namespace`, `docker_vault_kv_mount`, `docker_vault_namespace`, `docker_vault_secret_path`, overrides `QUAY_VAULT_OVERRIDE`/`GIT_VAULT_OVERRIDE`. [C]

## 4.9 Silver vs Gold vs Platinum/DR auth
| | Silver | Gold | Platinum / DR |
|---|---|---|---|
| Auth path | cluster np01 path (e.g. `aks-slvr-dig-np01`) | np02 path (e.g. `aks-dig-gld-np02`) | prod / DR cluster paths [U] |
| Role | `harness-<apm>-slvr` | `harness-<apm>-gld` | `harness-<apm>-plat` [I] |
| KV | `secret/slvr/<apm>/np01` | `secret/gld/<apm>/np02` | `secret/plat/<apm>/prod` (DR shares) [C doc / U practice] |
| Bitbucket ns | `corp-apm1002184` | `corp-apm1002184` | [U] |
| DIG Gold Docker | n/a | `docker_vault_kv_mount=secret/gld/<apm>/np02`, `docker_vault_namespace=dig-<apm>`, path `harness/harness` (never `ldap/devad/static-cred`) | [U] |
Clusters for APM1000782 were proposed per medal as `ocp-rosa-np1/np2/prod/dr` (sent to the app team for confirmation; answer not seen). [P]

## 4.10 Validations after vaulting [C]
1. `POST /ng/api/connectors/testConnection/{id}` on `apm<id>_vault_sm`, `_vault_sm_bitbucket`, `_bitbucket`, `_docker`, and the Gold equivalents.
2. If the connector passes but CI says "Could not retrieve secret", a key is missing (CI stops at the first missing key).
3. Diagnostic from inside the delegate: temporary inline pipeline (Custom stage, ShellScript step, `delegateSelectors: [apm<id>]`) that performs the template's login and lists **key names only**. Delete it afterwards (`tmp_devin_vault_diag` was used on APM1080718).
4. Credential validity: Bitbucket 200/401, SonarQube `{"valid":…}`, Artifactory HTTP code on a known POM URL.

## 4.11 Real examples (redacted)
- **APM1081191 Checkmarx**: Harness secret `apm1081191_checkmarx_id` → `apm1081191_vault_sm` → ns `dig-apm1081191`, mount `secret/slvr/apm1081191/np01`, path `harness/harness`, key `checkmarx_access_id`. Fixed by merge-patching `checkmarx_access_id/token` = `SrvDEPEGBDCSBDAdvSoaCx1` / `[REDACTED]`.
- **APM1081123 Silver/Gold Bitbucket**: Gold vaulting on 29 Oct minted a new token for `srcapm1081123UEPHBit`, revoking Silver's 12 Oct token. Fix: copy Gold's live token into Silver (`put-apm-secret.sh -k bitbucket_token --base current.json`). Do not mint again.
- **APM1080718 Gold**: role `harness-apm1080718-gld` didn't exist ("invalid role name"). The Vault team / your Desktop run created it and wrote `secret/gld/apm1080718/np02/harness/harness` v2. That run regenerated the Bitbucket and SonarQube tokens, which broke Silver.
