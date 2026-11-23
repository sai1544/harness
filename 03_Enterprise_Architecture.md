# 03 — Enterprise Architecture

All diagrams show only components seen in our history. Anything not observed is labelled *(proposed/illustrative)*.

## 3.1 High-level architecture
```mermaid
flowchart LR
  subgraph SCM["Bitbucket (bitbucket.elevancehealth.com)"]
    APPREPO["App repos (e.g. EHAPPGBDSC/*, *AZ projects)"]
    GITEXP["apm<id>-gitexperience-repo (pipelines, input sets, services as remote YAML)"]
    HELM["<project>-helm-charts-<medal> (values-harness.yaml)"]
    TPL["HARNESSGOL/elvh-ci-cd-templates (golden CI)"]
  end
  subgraph HARNESS["Harness SaaS app.harness.io (account pC_7h33wQTeZ_j-libvF4A)"]
    IDP["IDP workflows"]
    SF["Harness_Platform_Management / Solutions_Factory (IACM Terraform)"]
    PROJ["Org (DIG/GBD/EDA...) / Project APM<id>"]
  end
  subgraph K8S["APM cluster per medal (AKS / EKS / ROSA)"]
    ARGO["Tenant ArgoCD (namespace-scoped)"]
    DEL["Harness delegate apm<id>"]
    PODS["CI build pods / deployed apps"]
  end
  VAULT["HashiCorp Vault vault.acr.awsdns.internal.das:8200"]
  QUAY["Quay quay-nonprod / quay-prod"]
  ART["JFrog Artifactory"]
  SCAN["SonarQube / Checkmarx One / Wiz"]
  BAMBOO["Bamboo (being retired)"]
  EHDS["EHDS Tekton + ArgoCD (legacy source)"]

  IDP --> SF --> PROJ
  SF -->|creates ArgoCD app apm<id>-delegate| ARGO --> DEL
  DEL -->|outbound registration / task polling| HARNESS
  DEL -->|K8s auth login + KV read| VAULT
  DEL --> PODS
  PODS -->|clone| APPREPO
  PODS --> ART
  PODS --> SCAN
  PODS -->|push image| QUAY
  PROJ -->|remote entities| GITEXP
  PROJ -->|templates| TPL
  PODS -->|helm deploy, CD| HELM
  EHDS -.->|migration_pipeline reads app build info| PROJ
  BAMBOO -.->|"VM/Bamboo pattern (not exercised)"| PROJ
```
**Arrows explained**
- **IDP → Solutions_Factory → Project**: the IDP form only collects inputs. It triggers `Create_and_Manage_IACM_Workspaces`, which creates or updates an IACM (Terraform) workspace `<ORG>_<project-name>`. Terraform then creates the Harness project and everything in it. [C]
- **Solutions_Factory → ArgoCD → Delegate**: Terraform creates an ArgoCD Application `apm<id>-delegate` (chart `harness-delegate-ng` 1.0.32 from `https://app.harness.io/storage/harness-download/delegate-helm-chart/`, destination `in-cluster`, namespace = delegate namespace, auto-sync, `CreateNamespace`, `ServerSideApply`). ArgoCD deploys the delegate pod. [C]
- **Delegate → Harness**: the delegate registers with selector `apm<id>`. Every connector test, secret decryption and CI pod creation is executed by the delegate. [C]
- **Delegate → Vault**: Kubernetes-auth login with the pod's SA token on `auth/<cluster-auth-path>`, then KV v2 read. [C]
- **CI pods → Bitbucket / Artifactory / scanners / Quay**: the golden CI template's Scan, Build and Release stages. [C]
- **Project → gitexperience repo**: pipelines, input sets and services are stored as **REMOTE** entities in Bitbucket, read and written through the Silver `apm<id>_bitbucket` connector. [C]

## 3.2 Detailed component architecture (one APM, Silver + Gold)
```mermaid
flowchart TB
  subgraph PRJ["Harness project APM<id>"]
    subgraph SLV["Silver connectors"]
      S1["apm<id>_k8s_connector"]
      S2["apm<id>_vault_sm (ns <bu>-<apm>)"]
      S3["apm<id>_vault_sm_bitbucket (ns corp-apm1002184)"]
      S4["apm<id>_bitbucket"]
      S5["apm<id>_docker / apm<id>_docker_prod"]
    end
    subgraph GLD["Gold connectors"]
      G1["apm<id>_gold_k8s_connector"]
      G2["apm<id>_gold_vault_sm"]
      G3["apm<id>_gold_vault_sm_docker"]
      G4["apm<id>_gold_bitbucket"]
      G5["apm<id>_gold_docker"]
    end
    SEC["12 secrets (pointers to Vault)"]
    PIPE["15 pipelines: golden CI x10, cd_harness_native(_dynamic), bootstrap, migration, helm_repo_setup, gateway_setup"]
    ENV["Environments dev/test/prod + 1 infra per namespace"]
    FS["File Store /ci_config (Dockerfiles, settings.xml, pip.conf, .npmrc, certs)"]
    VARS["~110 project variables (namespaces, dnsHost, migration_baseimage_to_pipeline_mapping, SBOM_*...)"]
  end
  DS["Delegate apm<id> (Silver cluster)"] --- S1 & S2 & S3 & S4 & S5
  DG["Delegate apm<id>-gold (Gold cluster)"] --- G1 & G2 & G3 & G4 & G5
  SEC --> S2 & S3 & G2 & G3
```
- Each connector runs on **its own medal's delegate**. A custom Vault SM secret is decrypted on the delegate of the task that consumes it, so a Silver connector cannot use a Gold-SM secret. [C] (seen on APM1081123, 15 Nov)
- Silver `apm<id>_bitbucket` also loads the remote pipelines for **both** medals, so Gold depends on a healthy Silver Bitbucket connector. [C]

## 3.3 Secret-management architecture
```mermaid
sequenceDiagram
  participant Step as Pipeline step / connector test
  participant H as Harness secret ref (e.g. apm<id>_checkmarx_id)
  participant SM as Custom Vault SM (account.ElevanceCustomVaultSecretManagerTemplate)
  participant D as Delegate pod apm<id>
  participant V as Vault
  Step->>H: needs value
  H->>SM: secretManagerIdentifier = apm<id>_vault_sm
  SM->>D: run shell script on delegate (selector apm<id>)
  D->>V: POST /v1/auth/<AUTH_PATH>/login (role harness-<apm>-<medal>, SA JWT) - no namespace header
  V-->>D: client token + policy harness-<bu>-<apm>-<medal>
  D->>V: GET <ns>/secret/<medal>/<apm>/<tier>/data/harness/harness (X-Vault-Namespace)
  V-->>D: key value
  D-->>Step: value at runtime (never stored in Harness)
```
Vault location parts (example, DIG Gold): namespace `dig-apm1081520`, KV mount `secret/gld/apm1081520/np02`, path `harness/harness`, keys `artifactory_token` etc. Bitbucket exception: namespace `corp-apm1002184`, path `ldap/us/static-cred/srcueph<apm>bit`. [C]

## 3.4 Delegate communication architecture
```mermaid
flowchart LR
  TF["IACM Terraform (Solutions_Factory)"] -->|"argocd_application.delegate (ArgoCD API, admin creds from IDP form)"| AC["Tenant ArgoCD (runs in devops namespace)"]
  AC -->|"in-cluster, Namespaces scope = delegate ns"| K["K8s API"]
  K --> POD["Delegate pod: image quay-prod.../harness-delegate:26.06.89501, SA from form, 10Gi ephemeral /scratch PVC (delegate_storage_class)"]
  POD -->|"outbound HTTPS, delegate token"| HS["app.harness.io"]
  POD -->|"SA projected token"| VA["Vault K8s auth"]
  POD -->|"creates"| CI["harnessci-* build pods in same namespace"]
  TF -->|"time_sleep.wait_for_delegate ~4 min (first create only)"| HS
```
See doc 09 for the failure modes at each hop.

## 3.5 Container image staging architecture
```mermaid
flowchart LR
  SRC["Bitbucket repo"] --> SCANST["Scan stage: clone, Gitleaks, CheckmarxOne (sast,sca), Evaluate Scan Failures/SBOM"]
  SCANST --> BUILD["Build stage: Maven/Node build via Artifactory, Sonarqube_1 (ignore-fail), Build_and_Push_Docker, Wiz_Container_Scan (no ignore)"]
  BUILD -->|"push quay-nonprod.elevancehealth.com/<quay-org>/<svc>:release-<branch>-<timestamp>"| QN["Quay non-prod"]
  BUILD -.->|"NextGen _NXT projects: APP_PUSH_TYPE=ecr"| ECR["AWS ECR"]
  QN --> REL["Release stage: Git tag push"]
  QN --> CD["CD: cd_harness_native_pipeline (Silver/Gold input sets)"]
  QP["Quay prod (apm<id>_docker_prod connector; ci-addon and delegate images pulled from quay-prod)"]
```
Example tag seen: `quay-nonprod…/ehappcsbd/wkstestehappcomjava01:release-master-11172026071936`. [C]
Promotion from non-prod to prod Quay was **never exercised or discussed in detail**. [U]

## 3.6 APM-based workflow architecture
```mermaid
flowchart TB
  A["APM assigned (tracker / Wave sheet)"] --> B["Data collection and validation (intake sheet vs pilot batch vs K8s list)"]
  B --> C["SNOW service IDs srcUEPHAPM<id>BIT/ART/SON + AD groups"]
  C --> D["Vaulting: namespace, K8s role, KV secret"]
  D --> E["Silver onboarding IDP form"]
  E --> F{"Delegate up + connectors green?"}
  F -- no --> G["Fix delegate/Vault/ArgoCD, re-run same form"] --> E
  F -- yes --> H["helm_repo_setup, istio gateway, discovery agent"]
  H --> I["Validation: bootstrap, CI, Silver CD"]
  I --> J["Gold/Platinum: harness_project_modify_gold_platinum"]
  J --> K["Gold helm/istio/fetch gateway"]
  K --> L["migration_pipeline medal=gold (EHDS apps)"]
  L --> M["CI for all services + CD + handover"]
```

## 3.7 Bamboo/EHDS-to-Harness migration architecture
```mermaid
flowchart LR
  EH["EHDS migration API (app build info)"] --> MW["IDP MigrationWorkflow (identifier migration_workflow, owner group:shipsmart, project APM1078652 org DIG)"]
  MW --> MP["migration_pipeline in APM project"]
  MP -->|"Iterate_and_Trigger_IACM: webhook cd-migration-<apm>-<app> per app"| WS["Create_and_Manage_IACM_Workspaces runs (Solutions_Factory)"]
  WS --> SVC["Harness service per app"]
  WS --> CIS["<svc>_ci_inputs on golden CI pipeline"]
  WS --> CDS["<app>_gold_uat / _gold_perf (or _slvr_<ns>) on cd_harness_native_pipeline"]
  WS --> TRG["CI webhook triggers (PR-merge)"]
  BB["Bamboo plans"] -.->|"VM/Bamboo pattern: Windows delegate + runner for MSBuild (org knowledge, not exercised)"| GT["Golden templates"]
```
The onboarding unit is an **input set + trigger against a golden pipeline**, not a per-app pipeline clone. [C] (org knowledge and observed results)
