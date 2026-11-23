# Harness Project Modify – Gold/Platinum Onboarding: block-by-block guide

**What it is:** a Harness IDP **Workflow**, not a normal CI/CD pipeline. It is the form you fill in the Internal Developer Portal. When you press **Modify**, it runs a series of Harness pipelines one after another.

**Purpose:** take a Harness project that is **already onboarded to Silver** and add a **Gold** or **Platinum (+ Platinum-DR)** medal to it. It creates the delegate, connectors, Vault secret manager, secrets, helm chart repo, Istio gateway and discovery agent for that medal.

**Where it lives:** org `DIG` → project `APM1009046` (Atlas), identifier `harness_project_modify_gold_platinum`.

**Big picture: what runs when you click Modify**

| # | Step id | Runs which pipeline | Where the pipeline lives | Gold | Platinum |
|---|---|---|---|---|---|
| 1 | configure_workspace | Create_and_Manage_IACM_Workspaces (Terraform) | Harness_Platform_Management / Solutions_Factory | Yes | Yes |
| 2 | configure_workspace_dr | same, for platinum-dr | Solutions_Factory | – | Yes |
| 3 | release_process_setup | release_process_setup_pipeline | APM project (gitexperience repo) | – | Yes |
| 4 | setup_helm_repo | helm_repo_setup_<medal>_pipeline | APM project | Yes | Yes |
| 5 | setup_helm_repo_dr | helm_repo_setup_platinum_dr_pipeline | APM project | – | Yes |
| 6 | fetch_istio_gateway_name | fetch_istio_gateway_name_<medal> | APM project | Yes | Yes |
| 7 | fetch_istio_gateway_name_dr | fetch_istio_gateway_name_platinum_dr | APM project | – | Yes |
| 8 | setup_gateway | istio_gateway_setup_<medal>_pipeline | APM project | Yes | Yes |
| 9 | setup_gateway_dr | istio_gateway_setup_platinum_dr_pipeline | APM project | – | Yes |
| 10 | configure_workspace_for_da | Create_and_Manage_IACM_Workspaces (discovery agent) | Solutions_Factory | Yes | Yes |
| 11 | configure_workspace_for_da_dr | same, for platinum-dr | Solutions_Factory | – | Yes |

So a **Gold** run executes 5 steps and a **Platinum** run executes all 11. Steps run in order. If one step fails, the steps after it do not run.

The YAML has three main parts:
1. **Header / metadata**: name, owner, tags (Block 1)
2. **spec.output**: what you see on the result page after the run (Block 2)
3. **spec.parameters**: the form pages you fill in (Blocks 3–12)
4. **spec.steps**: the pipelines it triggers (Blocks 13–19)

---

## Block 1 – Header and metadata

```yaml
apiVersion: harness.io/v1
kind: Workflow
name: Harness Project Modify - Gold/Platinum Onboarding
identifier: harness_project_modify_gold_platinum
type: harness_project
owner: No Owner
projectIdentifier: APM1009046
orgIdentifier: DIG
metadata:
  description: A template to request a new Harness Project with ArgoCD-managed delegate deployment
  tags: [account-management, solutions-factory, harness, argocd]
```

- `kind: Workflow`: tells Harness IDP this is a self-service form, not a pipeline.
- `identifier`: the unique ID. Use it to search for the workflow, or in API calls.
- `orgIdentifier/projectIdentifier`: the workflow entity is stored in DIG/APM1009046 (Atlas). It can still act on **any** org/project you pick in the form.
- `description`: it says "request a new Harness Project", but that text was copied from the Silver workflow. This workflow does **not** create a project. It modifies an existing one.

---

## Block 2 – `spec.output` (result page)

```yaml
spec:
  output:
    links:
      - title: Access Harness Project
        url: ${{ steps.configure_workspace.output['pipeline.stages.Provision.spec.execution.steps.Provision.steps.apply.output.outputVariables.project_url']}}
      - title: View Delegate in ArgoCD
        url: ${{ ...outputVariables.argocd_application_url }}
      # ... CI Pipeline, Bitbucket/GitHub/Docker/K8s connector, Vault SM, Secrets, File Store, User Groups
    text:
      - title: Outputs
        content: |
          Harness Project Details
          organization_identifier: ${{ ...outputVariables.organization_identifier }}
          project_identifier: ${{ ...outputVariables.project_identifier }}
          ...
          Delegate Details
          argocd_application: ${{ ...outputVariables.argocd_application_name }}
          delegate_namespace: ${{ ...outputVariables.delegate_namespace }}
  lifecycle: production
  ui:options:
    customCreateButtonText: Modify
```

- This builds the "Outputs" box from your screenshot (`organization_identifier: DIG`, `project_identifier: APM1082588`, `argocd_application: apm1082588-gold-delegate` ...).
- Each value is read from **step 1 (configure_workspace)**. That step's Terraform `apply` step publishes output variables such as `project_url` and `argocd_application_url`. The workflow picks them up through the long path `pipeline.stages.Provision...apply.output.outputVariables.<name>`.
- If step 1 fails, these links stay empty.
- `customCreateButtonText: Modify`: the submit button shows "Modify" instead of "Create".

---

## Block 3 – Form page 1: "Request Harness Project"

```yaml
  parameters:
    - title: Request Harness Project
      required: [organization_id, project_name, medal_to_onboard]
      properties:
        token:
          ui:widget: password
          ui:field: HarnessAuthToken
        organization_id:
          ui:field: HarnessOrgPicker
        project_name:
          title: Project Identifier
          ui:field: SelectFieldFromApi
          ui:options:
            path: proxy/harness-api/projects?accountIdentifier=pC_7h33wQTeZ_j-libvF4A&orgIdentifier={{ parameters.organization_id }}&pageSize=2000
            valueSelector: project.identifier
            labelSelector: project.name
        medal_to_onboard:
          enum: [gold, platinum]
          enumNames: [Gold, Platinum (includes Platinum-DR)]
        existing_ehds_tenant:
          type: boolean
          default: false
```

- `token`: hidden field. `HarnessAuthToken` fills it with **your logged-in session token**. Every step sends this token as its `apikey`, so the pipelines run **as you**. You need permission to run pipelines in both Solutions_Factory and the APM project.
- `organization_id`: dropdown of orgs (DIG, GBD, EDA ...).
- `project_name`: dropdown loaded live from the Harness projects API for the chosen org. The label shows the project name, but the **stored value is the project identifier** (e.g. `APM1082588`). Everything later uses this value: delegate selector, connector names, Vault role names.
- `medal_to_onboard`: the main switch. `platinum` turns on all the `_dr` steps.
- `existing_ehds_tenant`: tick this if the app already has an EHDS tenant with existing helm chart repos.

### Block 3b – conditional fields on page 1 (`dependencies`)

```yaml
      dependencies:
        existing_ehds_tenant:
          oneOf:
            - properties: { existing_ehds_tenant: { const: true },
                            use_new_infra_project: { type: boolean, default: false } }
        use_new_infra_project:
          oneOf:
            - properties: { use_new_infra_project: { const: false }, existing_tenant_name: {...} }
              required: [existing_tenant_name]
            - properties: { use_new_infra_project: { const: true },  existing_helm_repo_url: {...} }
        medal_to_onboard:
          oneOf:
            - properties: { medal_to_onboard: { const: platinum },
                            harness_api_key: {...password...},
                            deployment_type: { enum: [non-bg, bg] } }
              required: [harness_api_key, deployment_type]
```

`dependencies` + `oneOf` is JSON-schema for "show this field only when that one has a certain value":

| You choose | Extra field that appears | Used later by |
|---|---|---|
| Existing EHDS tenant = true | Use new infra project? | helm repo step: `USE_NEW_INFRA_PROJECT` |
| …and Use new infra project = **false** | Existing Tenant Name(s), required (e.g. `syeus` or `syeus,syeus-dr`) | helm step: `EXISTING_TENANT_NAME` |
| …and Use new infra project = **true** | Existing Helm Repo URL(s) | helm step: `EXISTING_TENANT_REPO_URL` |
| Medal = platinum | Harness PAT for Release Processes + Deployment Type (BG / non-BG) | release_process_setup step |

For Platinum, these fields take **two comma-separated values**: primary first, DR second. Step 4 uses `split(',')[0]` and step 5 uses `split(',')[1]`.

Two pitfalls:
- If you tick "Use new infra project" and leave the URL blank, the helm pipeline stops with "EXISTING_TENANT_REPO_URL is required".
- If the Harness project name differs from the EHDS tenant name, `false` fails because it tries to clone `<project>-helm-charts-gold`. Use `true` + the tenant's repo URL instead.

---

## Block 4 – Form page 2: "Secret Manager Configuration" (Vault)

```yaml
    - title: Secret Manager Configuration
      properties:
        secret_manager_type: { default: vault, enum: [vault] }
      dependencies:
        secret_manager_type:
          oneOf:
            - required: [vault_namespace]
              properties:
                vault_addr:        { default: https://vault.acr.awsdns.internal.das:443 }
                vault_kv_mount:    { description: "blank → Gold secret/gld/{project}/np02, Platinum secret/plat/{project}/prod" }
                vault_auth_path:   { description: "e.g. aks-dig-apm1009046-gold-np01" }
                vault_role:        { description: "blank → harness-{project}-gld / -plat" }
                vault_namespace:   { description: "required, e.g. dig-apm1009046" }
                vault_secret_path: { description: "blank → harness/harness" }
                bitbucket_username_vault_key: { default: username }
                bitbucket_password_vault_key: { default: password }
                bitbucket_token_vault_key:    { default: bitbucket_token }
                docker_username_vault_key:    { default: username }
                docker_password_vault_key:    { default: password }
                github_username_vault_key:    { default: github_username }
                github_token_vault_key:       { default: github_token }
                gh_dst_auth_type: { default: token, enum: [token, githubapp] }
                gh_dst_app_id / gh_dst_installation_id / gh_dst_app_private_key_vault_key
                bitbucket_vault_namespace_default: { default: corp-apm1002184, ui:widget: hidden }
                bitbucket_vault_kv_mount_default:  { default: ldap/us/static-cred, ui:widget: hidden }
                docker_vault_namespace_default:    { default: corp-apm1002184, ui:widget: hidden }
                docker_vault_kv_mount_default:     { default: ldap/devad/static-cred, ui:widget: hidden }
                docker_nonprod_username_vault_key / docker_nonprod_password_vault_key   # Platinum only
```

This page tells Terraform how the medal's **Vault Secret Manager connector** logs in to Vault and where the secrets are stored.

- **Shared Vault (the APM's own store):** `vault_namespace` (e.g. `dig-apm1082588`), `vault_auth_path` (the Kubernetes auth path of the medal cluster, e.g. `aks-dig-gld-np02`), `vault_role` and `vault_kv_mount`.
  - You can leave the role and KV mount blank. The step then builds them from the project ID and medal.
- **`*_vault_key` fields:** the key **names** inside the Vault secret, e.g. the key that holds the Bitbucket token is `bitbucket_token`. They are not the values.
- **GitHub fields:** only matter if the project's SCM is GitHub.
- **Hidden `*_default` fields:** the corporate LDAP static-credential store `corp-apm1002184` is where the CORP service IDs (srcueph…bit/qua) live. The hidden defaults aren't actually used in the steps, because the step expressions hard-code the same strings again. They are only for display/documentation.

### Block 4b – Per-service Vault overrides

```yaml
                vault_overrides:
                  properties:
                    bitbucket:
                      properties: { enabled: { default: false } }
                      # enabled=true → bitbucket_vault_kv_mount / auth_path / role / namespace / secret_path
                    docker:
                      properties:
                        use_shared_vault: { default: false }   # use the APM's own vault for Quay
                        enabled:          { default: false }   # fully custom Quay vault config
                    github:
                      properties: { enabled: { default: false } }
```

By default, each credential type reads from a fixed place. You can change that here:

| Credential | Default when override is OFF | Override options |
|---|---|---|
| Bitbucket | ns `corp-apm1002184`, mount `ldap/us/static-cred`, path `srcueph<apm>bit`, auth path = shared auth path, role `harness-<apm>-gld/plat` | `enabled=true` → type your own |
| Docker/Quay | ns `corp-apm1002184`, mount `ldap/devad/static-cred` (Gold) or `ldap/us/static-cred` (Platinum), path `srcueph<apm>qua` | `use_shared_vault=true` → sends empty values, so Terraform uses the shared APM vault; `enabled=true` → type your own |
| GitHub | ns `dig-<apm>`, mount = shared KV mount, path `harness/harness` | `enabled=true` → type your own |

> From earlier DIG Gold work: the working APMs (APM1081191, APM1081520) read Quay from the **per-APM store** (`secret/gld/<apm>/np02`, ns `dig-<apm>`, path `harness/harness`), **not** the `ldap/devad/static-cred` default. Leaving the Docker override off is what broke APM1080718/APM1080955. For DIG Gold, set Docker → custom (or shared vault) on purpose.

---

## Block 5 – Form page 3: "Vault Configuration (DR)" – Platinum only

```yaml
    - title: Vault Configuration (DR)
      dependencies:
        medal_to_onboard:
          oneOf:
            - required: [dr_vault_namespace]
              properties:
                medal_to_onboard: { const: platinum }
                dr_vault_kv_mount:  { description: "blank → secret/plat/{project}/prod" }
                dr_vault_auth_path: ...
                dr_vault_role:      { description: "blank → harness-{project}-plat" }
                dr_vault_namespace: ...   # required
                dr_*_vault_key ...
                dr_vault_overrides: { bitbucket, docker, github }   # same pattern as Block 4b
            - properties: { medal_to_onboard: { const: gold } }     # Gold → page shows nothing
```

This page mirrors Block 4 with a `dr_` prefix, for the DR cluster. With Gold, the page is empty.

---

## Block 6 – Form pages 4–5: "ArgoCD Configuration" (+ DR)

```yaml
    - title: ArgoCD Configuration
      properties:
        argocd_enabled:         { default: true, ui:widget: hidden }
        argocd_server:          { default: shipsmart-devops.elevancehealth.com:443 }
        argocd_username:        { default: admin }
        argocd_auth_credential: { ui:widget: password }
        argocd_project:         { default: default }
    - title: ArgoCD Configuration (DR)
      # platinum only: dr_argocd_server (default shipsmart-devops-dr...), dr_argocd_username, dr_argocd_auth_credential, dr_argocd_project
```

- The delegate is **not** installed with kubectl. Terraform creates an **ArgoCD Application** (`apm<id>-<medal>-delegate`) on the medal cluster's ArgoCD, and ArgoCD deploys the delegate pod.
- `argocd_server` must be the tenant's ArgoCD for that medal, e.g. `psdpccaz-argocd-aks-gold.elevancehealth.com:443` in your screenshot. Don't keep the default `shipsmart-devops`.
- The password comes from the ArgoCD creds sheet. If that is stale, use the `argocd_password` variable on the tenant's Silver IACM workspace.
- **Warning:** the password field is masked in the form, but it is stored in **plain text** in the run's Input tab (`argocd_auth_credential`). Don't share screenshots of that tab.

---

## Block 7 – Form pages 6–7: "Namespace Configuration" (+ DR)

```yaml
    - title: Namespace Configuration
      dependencies:
        medal_to_onboard:
          oneOf:
            - properties:
                medal_to_onboard: { const: gold }
                namespace_details_input:
                  type: array
                  maxItems: 3
                  items: { properties: { namespace: ..., dns_hostname: ... }, required: [namespace, dns_hostname] }
            - properties:
                medal_to_onboard: { const: platinum }
                namespace: ...
                dns_hostname: ...
    - title: Namespace Configuration (DR)
      # platinum only: dr_namespace, dr_dns_hostname
```

- These are the app namespaces on the medal cluster, plus the DNS host for each.
- **Gold** takes up to **3** namespace/DNS rows. **Platinum** takes **one** (plus one for DR).
- Terraform turns them into the `dnsHost` project variable and the environments/infra definitions. The discovery agent step (Block 18) reuses them.
- Give each namespace only one row, and trim trailing spaces. Duplicate namespaces make Terraform fail with "Duplicate object key".

---

## Block 8 – Form pages 8–9: "Delegate Configuration" (+ DR)

```yaml
    - title: Delegate Configuration
      properties:
        delegate_enabled:        { default: true, hidden }
        delegate_replicas:       { default: 1, min 1, max 10 }
        delegate_cpu_request:    { default: "0.5" }
        delegate_cpu_limit:      { default: "4" }
        delegate_memory_request: { default: 2048Mi }
        delegate_memory_limit:   { default: 2048Mi }
        delegate_storage_size:   { default: 10Gi }
        delegate_image:          { default: quay-prod.elevancehealth.com/elvh_harness_cicd/harness-delegate:26.06.89501 }
        delegate_namespace:      { description: defaults to project_name if empty }
        delegate_upgrader_enabled:     { default: true, hidden }
        delegate_service_account_name: { default: argocd-application-controller }
        enter_storageclass_manually:   { default: false }
      dependencies:
        enter_storageclass_manually:
          oneOf:
            - false → delegate_storage_class dropdown [elvh-disk-csi, elvh-disk-csi-retain, ebs-sc-eks, gp2, gp3]
            - true  → delegate_storage_class free text
```

- These are the size and settings of the new medal delegate pod. The delegate registers with selector `<apm>-<medal>` (e.g. `apm1082588-gold`). All later steps run on that delegate.
- `delegate_namespace`: where the pod runs. In your screenshot it is `psdigitalcommon-devops`. If left blank, it uses the project ID.
- Storage class: pick from the list. If the cluster uses another class, tick "Add storage class manually" and type it in.
- DR page: same fields with a `dr_` prefix. **Note:** the DR image default is a different, older image (`eks-blueprints/harness-delegate:26.02.88404`) than the primary (`elvh_harness_cicd/...:26.06.89501`). Check which one you want.

---

## Block 9 – Form page 10: "Image Registry Connector"

```yaml
    - title: Image Registry Connector
      properties:
        docker_connector_enabled: { default: true, hidden }
        docker_registry_type:     { default: Quay, enum: [Quay, ECR] }
      dependencies:
        docker_registry_type:
          oneOf:
            - Quay → docker_registry_url (e.g. https://quay-nonprod.elevancehealth.com), dr_docker_registry_url
            - ECR  → docker_registry_url (ECR URL), dr_docker_registry_url
```

This creates the medal's Docker registry connector, used to pull and push images. If the DR URL is left blank, it uses the primary URL. The Istio step also gets this URL as `QUAY_REGISTRY_URL`.

---

## Block 10 – Form page 11 (hidden): "Solutions Factory Connection"

```yaml
    - title: Solutions Factory Connection
      properties:
        solutions_factory_details:
          properties:
            harness_account_url:        { default: https://app.harness.io, hidden }
            harness_account_id:         { default: pC_7h33wQTeZ_j-libvF4A, hidden }
            harness_org_id:             { default: Harness_Platform_Management, hidden }
            harness_project_id:         { default: Solutions_Factory, hidden }
            template_library_connector: { default: org.Custom_Harness_Template_Library_Repo, hidden }
            template_library_repo:      { default: https://git.harness.io/.../custom-harness-template-library.git, hidden }
            template_library_branch:    { default: production-onboarding, hidden }
            template_library_directory: { default: harness-project-medal, hidden }
            da_automation_*:            same repo/branch, directory da-automation
        solutions_factory_opts:
          properties:
            repo_source:       { default: custom }
            workspace_type:    { default: harness-project }
            is_ephemeral:      { default: "false" }
            requires_approval: { default: "false" }
            use_mini_factory:  { default: "false" }
```

You never see these fields, but they are important:
- The Terraform code lives in the **custom-harness-template-library** repo (Harness Code), branch `production-onboarding`.
- Folder `harness-project-medal` holds the medal Terraform (delegate, connectors, secrets, pipelines). Folder `da-automation` holds the discovery agent Terraform.
- The Terraform runner is the `Create_and_Manage_IACM_Workspaces` pipeline in `Harness_Platform_Management/Solutions_Factory`.
- `is_ephemeral=false`: the medal workspace is **kept** after the run, so you can re-plan or re-apply it later.
- `use_mini_factory=false`: if it were true, the step would use a per-org `hws_<org>` project instead of Solutions_Factory.

---

## Block 11 – Step 1: `configure_workspace` (the main Terraform step)

```yaml
  steps:
    - id: configure_workspace
      name: Configure Workspace (Gold/Platinum)
      action: trigger:harness-custom-pipeline
      input:
        url: https://app.harness.io/ng/account/<acct>/all/orgs/Harness_Platform_Management/projects/Solutions_Factory/pipelines/Create_and_Manage_IACM_Workspaces/pipeline-studio?storeType=INLINE
        inputset:
          GIT_REPOSITORY_CONNECTOR: org.Custom_Harness_Template_Library_Repo
          GIT_REPOSITORY_NAME: .../custom-harness-template-library.git
          GIT_REPOSITORY_BRANCH: production-onboarding
          GIT_REPOSITORY_PATH: harness-project-medal
          RESOURCE_NAME: ${{ parameters.organization_id }}_${{ parameters.project_name|replace(" ","-") }}_${{ parameters.medal_to_onboard }}_modify_legacy
          RESOURCE_OWNER: group:default/Project_Admins
          RESOURCE_VARS:
            organization_id / project_id / project_name / medal_level
            argocd_*            ← Block 6
            namespace_details   ← Block 7, converted to JSON
            delegate_*          ← Block 8
            docker_*            ← Block 9
            vault_*             ← Block 4 (with auto-derive defaults)
            bitbucket_vault_* / docker_vault_* / github_vault_*  ← Block 4b override logic
          RESOURCE_VARS_SECRETS:
            HARNESS_TEMPLATE_API_KEY: account.account-template-sa-token
          INCLUDE_HARNESS_ENVS: "true"
          WORKSPACE_TAGS: { source: custom, type: harness-project }
          IS_EPHEMERAL: "false"
          REQUIRES_APPROVAL: "false"
        apikey: ${{ parameters.token }}
        showOutputVariables: true
```

This is the most important step.

- `action: trigger:harness-custom-pipeline`: runs a Harness pipeline (the one in `url`) with `inputset` as its runtime inputs, then waits for it to finish.
- It creates or updates an **IACM workspace** named e.g. `DIG_APM1082588_gold_modify_legacy`. It then runs Terraform `plan` + `apply` on `harness-project-medal`.
- Terraform creates about 39 resources in the APM project, including:
  - the ArgoCD delegate app
  - the K8s connector `<apm>_gold_k8s_connector`
  - the Vault SM connector(s), Docker/Bitbucket/GitHub connectors and their secrets
  - project variables and the gold environments/infra
  - remote pipelines `helm_repo_setup_gold_pipeline`, `istio_gateway_setup_gold_pipeline` and `fetch_istio_gateway_name_gold`, written to `<apm>-gitexperience-repo` through the Silver `apm<id>_bitbucket` connector
- `HARNESS_TEMPLATE_API_KEY`: a shared service-account token that Terraform uses to call the Harness API.
- `showOutputVariables: true`: makes the Terraform outputs available to Block 2 and to step 8 (`devops_namespace`).

**How the main expressions resolve** (example APM1082588, Gold):

| Variable | Expression logic | Result if left blank |
|---|---|---|
| vault_kv_mount | input, else gold→`secret/gld/<apm>/np02`, plat→`secret/plat/<apm>/prod` | `secret/gld/apm1082588/np02` |
| vault_role | input, else `harness-<apm>-gld` / `-plat` | `harness-apm1082588-gld` |
| vault_secret_path | input, else `harness/harness` | `harness/harness` |
| bitbucket_vault_* | override if enabled, else corp LDAP | ns `corp-apm1002184`, mount `ldap/us/static-cred`, path `srcuephapm1082588bit` |
| docker_vault_* | custom if enabled; `''` if use_shared_vault; else corp LDAP | ns `corp-apm1002184`, mount `ldap/devad/static-cred`, path `srcuephapm1082588qua` |
| github_vault_namespace | override, else `dig-<apm>` | `dig-apm1082588` (hard-coded `dig-`, wrong for GBD/EDA orgs) |
| namespace_details | Gold: `[{"medal_level":"gold","namespace_dns_combinations":<your rows as JSON>}]` | – |

> Terraform owns everything this step creates. If you fix a connector by hand in the UI, the next re-apply puts the workspace values back. Make permanent fixes in the IACM workspace variables (or re-submit this form). The step is idempotent: re-submitting the same form updates the workspace in place.

---

## Block 12 – Step 2: `configure_workspace_dr` (Platinum only)

```yaml
    - id: configure_workspace_dr
      if: ${{ parameters.medal_to_onboard == "platinum" }}
      input:
        url: .../Create_and_Manage_IACM_Workspaces...
        inputset:
          RESOURCE_NAME: <org>_<project>_platinum-dr_modify_legacy
          RESOURCE_VARS:
            medal_level: platinum-dr
            argocd_* ← dr_argocd_*
            namespace_details: '[{"medal_level":"platinum-dr","namespace_dns_combinations":[{"namespace":"<dr_namespace>","dns_hostname":"<dr_dns_hostname>"}]}]'
            delegate_* ← dr_delegate_*
            docker_registry_url: dr url, else primary url
            vault_* ← dr_vault_* (defaults secret/plat/<apm>/prod, harness-<apm>-plat)
            environment_exists: "false"
```

- `if:`: the step is skipped unless the medal is Platinum.
- It does the same job as step 1, but for the DR cluster. It creates a second workspace, `..._platinum-dr_modify_legacy`, and a second delegate `<apm>-platinum-dr`.

---

## Block 13 – Step 3: `release_process_setup` (Platinum only)

```yaml
    - id: release_process_setup
      if: ${{ parameters.medal_to_onboard == "platinum" }}
      input:
        url: .../orgs/<org>/projects/<apm>/pipelines/release_process_setup_pipeline/pipeline-studio?storeType=REMOTE&repoName={{ parameters.project_name | lower }}-gitexperience-repo&branch=<main|master>
        inputset:
          DEPLOYMENT_TYPE: ${{ parameters.deployment_type }}   # bg / non-bg
          HARNESS_API_KEY: ${{ parameters.harness_api_key }}
```

- From here on, the pipelines run **inside the APM project**, not in Solutions_Factory. They are REMOTE pipelines stored in Bitbucket/GitHub repo `<apm>-gitexperience-repo`. The branch is `main` for GitHub and `master` for Bitbucket; step 1's output `scm_type` decides which.
- This step sets up the Harness **Release Management** processes (RMG) for prod: blue-green or not. It needs a separate PAT because the RMG API requires one.
- **Possible bug:** in this URL, `repoName={{ ... }}` is missing the `$` that every other step has (`${{ ... }}`). The repo name may therefore not resolve. I haven't tested it. If Platinum runs fail at this step with a "pipeline/repo not found" error, check this first.

---

## Block 14 – Steps 4–5: `setup_helm_repo` / `setup_helm_repo_dr`

```yaml
    - id: setup_helm_repo
      input:
        url: .../projects/<apm>/pipelines/helm_repo_setup_${{ medal | replace("-","_") }}_pipeline/...REMOTE...
        inputset:
          MEDAL_LEVEL: ${{ parameters.medal_to_onboard }}
          DATA_OWNER: SYSTEM
          DATA_INITIATOR: SYSTEM
          SKIP_SNOW_IF_EXISTS: "true"
          DELEGATE_SELECTOR: <apm>-<medal>                   # e.g. apm1082588-gold
          K8S_CONNECTOR_REF: <apm>_<medal>_k8s_connector     # e.g. apm1082588_gold_k8s_connector
          EXISTING_TENANT:        'true' if existing_ehds_tenant else 'false'
          USE_NEW_INFRA_PROJECT:  'true' if use_new_infra_project else 'false'
          EXISTING_TENANT_REPO_URL: first value of existing_helm_repo_url  (only if tenant=true AND new infra=true)
          EXISTING_TENANT_NAME:     first value of existing_tenant_name    (only if tenant=true AND new infra=false)
    - id: setup_helm_repo_dr      # platinum only, MEDAL_LEVEL platinum-dr, uses the SECOND comma value
```

- This runs the `helm_repo_setup_gold_pipeline` that step 1 just created.
- It creates or prepares the Bitbucket repo `<project>-helm-charts-gold`, with `values-harness.yaml` and the `da-automation/` charts.
- It raises ServiceNow requests for it (or skips them if the repo already exists: `SKIP_SNOW_IF_EXISTS`). For a new tenant it can also create the Artifactory repos.
- It runs on the **new gold delegate** (`DELEGATE_SELECTOR`). If that delegate didn't come up, this step fails with "no eligible delegates".
- What the tenant flags do:

| EXISTING_TENANT | USE_NEW_INFRA_PROJECT | Behaviour |
|---|---|---|
| false | – | New tenant: copies from `EHDS/bitbucket-helm-charts-<medal>` |
| true | false | Updates the existing `<tenant>-helm-charts-<medal>` in place (project name = tenant name) |
| true | true | Copies the given repo URL into a new `<project>-helm-charts-<medal>` (project name ≠ tenant) |

---

## Block 15 – Steps 6–7: `fetch_istio_gateway_name` (+ DR)

```yaml
    - id: fetch_istio_gateway_name
      input:
        url: .../projects/<apm>/pipelines/fetch_istio_gateway_name_<medal>/...
        inputset:
          PLACEHOLDER: "true"
```

- This pipeline looks up the tenant's Istio ingress gateway name on the cluster and saves it as the project variable `ISTIO_GATEWAY_NAME`.
- It has no real inputs. `PLACEHOLDER` is there only because the trigger action needs at least one input.
- It **must run before** step 8. The Istio setup pipeline reads `<+variable.ISTIO_GATEWAY_NAME>`, and if the variable is missing it fails with "resource name may not be empty". This step is new in this version; older runs, like the 9-action run in your screenshot, didn't have it.

---

## Block 16 – Steps 8–9: `setup_gateway` / `setup_gateway_dr`

```yaml
    - id: setup_gateway
      input:
        url: .../projects/<apm>/pipelines/istio_gateway_setup_<medal>_pipeline/...
        inputset:
          DEVOPS_NAMESPACE: step1 output devops_namespace, else delegate_namespace
          MEDAL_LEVEL: <medal>
          DELEGATE_SELECTOR: <apm>-<medal>
          K8S_CONNECTOR_REF: <apm>_<medal>_k8s_connector
          QUAY_REGISTRY_URL: docker_registry_url
          QUAY_VAULT_OVERRIDE: 'true' if a custom Docker vault mount was given, else ...
          GIT_VAULT_OVERRIDE:  'true' if a custom Bitbucket vault mount was given, else ...
```

- This sets up the Istio Gateway / VirtualService resources so the apps are reachable on the DNS hosts from Block 7.
- `DEVOPS_NAMESPACE` is the first place a later step **reads an output from step 1**.
- `QUAY_VAULT_OVERRIDE` / `GIT_VAULT_OVERRIDE` tell the pipeline whether to read Quay/Git credentials from the custom Vault paths you entered in Block 4b or from the corp defaults.

---

## Block 17 – Steps 10–11: `configure_workspace_for_da` (+ DR)

```yaml
    - id: configure_workspace_for_da
      input:
        url: .../Solutions_Factory/pipelines/Create_and_Manage_IACM_Workspaces/...
        inputset:
          GIT_REPOSITORY_PATH: da-automation
          RESOURCE_NAME: <org>_<project>_<medal>_modify_legacy_da_automation
          RESOURCE_VARS:
            org_identifier / project_identifier
            discovery_namespaces: Gold "ns1,ns2,ns3" (from namespace rows) | Platinum: the single namespace
            infra_identifiers:    same list with '-' removed
            env_identifier: 'test' if gold else 'prod'   # DR → prod_dr
            collector_image_tag: 0.70.0
            log_watcher_image_tag: 1.90.0
          WORKSPACE_TAGS: { type: da-automation }
          IS_EPHEMERAL: "true"
```

- This step goes back to Solutions_Factory Terraform, this time for the folder `da-automation`. It installs a Harness **Service Discovery agent** for each app namespace (`<ns>-discovery`), so Harness can map the services running there. These are used for chaos/resilience and the service map.
- `IS_EPHEMERAL: "true"`: the workspace is **deleted after the run**. You won't find it later. To check the result, look at the Service Discovery agents in the project, or use the servicediscovery API.
- For Gold it attaches to environment `test`. For Platinum it uses `prod`, and `prod_dr` for DR.

---

## Things worth knowing

| # | Point | Why it matters |
|---|---|---|
| 1 | Runs as **you** (`apikey: ${{ parameters.token }}`) | You need execute rights in Solutions_Factory and the APM project. A PAT can't submit this form through the API ("Token is not valid"). |
| 2 | Steps run in order and stop at the first failure | If step 1 fails, nothing after it runs. If the delegate isn't up, steps 4–9 fail with "no eligible delegates". |
| 3 | Re-submitting is safe (idempotent) | Fix the input and submit again. The same IACM workspace is updated, not duplicated. |
| 4 | Terraform owns connectors | UI hand-fixes are reverted on re-apply. Fix the workspace variables instead. |
| 5 | Docker Vault default = corp LDAP | For DIG Gold, use the Docker override to point at `secret/gld/<apm>/np02` / `dig-<apm>` / `harness/harness`. |
| 6 | `github_vault_namespace` default hard-codes `dig-` | For non-DIG orgs (GBD, EDA) enable the GitHub override if GitHub is used. |
| 7 | ArgoCD password stored in plain text in run inputs | Don't share screenshots of the Input tab. |
| 8 | DR delegate image default differs from primary | Check it before a Platinum run. |
| 9 | release_process_setup URL is missing a `$` in `repoName={{…}}` | Possible Platinum failure point (not tested). |
