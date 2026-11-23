# 16 — Interview Preparation (based only on our project history)

## 1. 60-second introduction
"I work on the UEP Harness program at Elevance Health, on the onboarding and Vault team. We move applications, each tracked by an APM ID, from legacy EHDS (Tekton + ArgoCD) and Bamboo onto Harness CI/CD with golden pipelines. For each APM I validate the onboarding data and get the service IDs in place. Then I vault their credentials into HashiCorp Vault and onboard the APM to Silver, and later to Gold, through Harness IDP workflows that run Terraform. After that I validate with a bootstrap test app and golden CI runs. Most of my time goes into troubleshooting the chain between Harness delegates, Kubernetes, Vault and Bitbucket, and working with app and platform teams to clear blockers."

## 2. Two-minute technical explanation
Each APM gets one Harness project. The Silver IDP form (`Harness_Onboarding_Project_Setup`) triggers `Create_and_Manage_IACM_Workspaces` in Solutions_Factory, which applies a Terraform workspace. That workspace creates the project, about 250 resources, and an ArgoCD app that deploys the Harness delegate into the APM's devops namespace. The delegate is the key component. It registers outbound to Harness, and it logs in to Vault with its Kubernetes service-account token on the cluster's auth path. Every connector and secret is a pointer to Vault KV v2 at `secret/<medal>/<apm>/<tier>/harness/harness` in namespace `<bu>-<apm>`. Pipelines and input sets are remote YAML in a gitexperience Bitbucket repo. CI uses account-level golden templates (Scan with Gitleaks/Checkmarx, Build with Maven/Sonar/Docker/Wiz, Release with the Quay push and a Git tag). Gold modifies the same project, using its own cluster, delegate and Vault path. EHDS apps are bulk-migrated by `migration_pipeline`, which creates services and input sets per app.

## 3. Five-minute architecture explanation
Walk through doc 03 in this order: diagram 3.1 (flows), then 3.3 (secret sequence), then 3.4 (delegate), then 3.5 (image path). For each one, give a real failure from that hop: the ArgoCD namespace scope, the storage class, the SCC, the expired SA token, the role binding, the wrong Vault namespace, Bitbucket access, the one-token rule.

## 4. End-to-end APM example: APM1003706 (UMDE, rosa-np1)
1. Data validation: conflicts found in storage class, SA and source key; email sent.
2. Vaulting: namespace `dig-apm1003706`, role `harness-apm1003706-slvr` (`ocp`, `rosa-np1`).
3. Silver run 1: plan failed with `Duplicate object key` (28 hosts under 2 namespaces). Fixed with one default per namespace, after guidance from the platform owner.
4. Delegate never registered. Six storage class and SA combinations failed. The ArgoCD ReplicaSet showed the SCC `restricted-v2` rejecting `runAsUser 0`/`fsGroup 1001`. Upgrader turned off for the RBAC error. A SR to OCP got `anyuid` granted, and the delegate connected.
5. Vault login error "service account name not authorized". Fixed with an Automation Hub role modify that ADDs the SA.
6. `corp-apm1003706` changed to `dig-apm1003706`, after checking 47/53 working peers.
7. Final blocker: infra key `UMINFRA` doesn't exist; `MCMUMINFRA` is being confirmed with the app team.

## 5. How Vault, Harness and Kubernetes work together
See doc 03 §3.3. In short: the Harness secret points to a custom Vault SM. The delegate runs the script, logs in with its SA JWT on `auth/<cluster>`, the role binds ns+SA, the policy allows the KV path, and the value is returned at runtime.

## 6. Ten resume bullets (verified contributions)
1. Onboarded APMs to Harness Silver/Gold through IDP/Terraform (IACM) workflows on AKS, EKS and OpenShift ROSA clusters.
2. Vaulted per-APM service-ID credentials into HashiCorp Vault KV v2 using additive merge-patch, with no loss of existing keys.
3. Resolved an OpenShift SCC block on the Harness delegate (root UID vs `restricted-v2`) by driving an `anyuid` grant through ServiceNow.
4. Fixed a Vault Kubernetes-auth role binding through the Automation Hub API, restoring secret access for Harness connectors.
5. Diagnosed cross-medal credential outages caused by Bitbucket's one-token-per-user rule, and defined a mint-once/write-both procedure.
6. Validated Harness onboarding with bootstrap test apps and golden CI templates across 5 APMs and 230+ CI runs.
7. Produced CI failure-analysis reports (root-cause grouping, per-service errors) for 207 services, isolating a Bitbucket access gap that affected 205 of them.
8. Reconciled onboarding data across intake, pilot and Kubernetes sheets for 11 Wave 2 APMs, and coordinated corrections with app owners.
9. Unblocked delegate deployments by resolving ArgoCD namespace scoping and storage-class mismatches.
10. Delivered a KT session for the team on vaulting and Silver onboarding.

## 7. Thirty Q&As
1. **Why must Silver come before Gold?** Silver creates the project; Gold only modifies it, and Gold's remote pipelines are written through Silver's Bitbucket connector.
2. **Gold Vault path?** `secret/gld/<apm>/np02/harness/harness` in `<bu>-<apm>`.
3. **What is the Kubernetes auth path for?** It tells Vault which cluster to validate the delegate's SA token against. A wrong path means the login is rejected.
4. **Connector passes but CI says "Could not retrieve secret"?** A key is missing at the path; CI stops at the first missing one.
5. **"Failed to decrypt … after 3 retries"?** Missing key, expired delegate token, or a role/namespace mismatch. Trace secret → SM → templateInputs → delegate.
6. **Why did re-vaulting Gold break Silver?** Bitbucket allows one token per user, so minting a new one revokes the token Silver held.
7. **Can a Silver connector use a Gold secret?** No. Decryption happens on the Silver delegate, which can't log in to Gold.
8. **Why can't you edit connectors in the UI permanently?** Terraform manages them without `ignore_changes`, so the next apply reverts the edit. Fix the tfvars instead.
9. **Why not submit IDP forms via API?** The workflow needs a browser session token; a PAT fails "Token is not valid" and is stored in plain text.
10. **What happens on re-running the onboarding form?** It's idempotent (0 destroy). But the delegate wait only runs on first create, so you can get "no eligible delegates".
11. **"Network operation timed out" in onboarding?** It's a label. Read the plan log (e.g. "Duplicate object key", "Project … not found").
12. **Multiple hostnames per namespace?** One default in the form; per-app `dnshost` in CD input sets.
13. **Two Artifactory keys?** One in the form, the rest in File Store `settings.xml`; it's a CI concern.
14. **Delegate Pending with PVC?** The storage class doesn't exist. Use the cluster's real class.
15. **ArgoCD sync Unknown, cluster-scope error?** The controller is namespace-scoped; set in-cluster Namespaces.
16. **SCC rejection?** Delegate wants UID 0/fsGroup 1001; `restricted-v2` forbids that. Grant `anyuid` to the SA.
17. **"token is expired" from Vault?** The projected SA token stopped refreshing. Restart the delegate.
18. **"invalid role name"?** The role was never created for that auth path.
19. **"service account name not authorized"?** The role isn't bound to the delegate SA. Modify ADD.
20. **CI Not Authorized but the connector passes?** The service ID lacks project permission on that repo's Bitbucket project. It needs WRITE for the Git tag.
21. **Checkmarx "forbidden action"?** The project name is `<svc>-azure-policy`; set `CHECKMARX_PROJECT_NAME`.
22. **Why is Evaluate Scan Failures slow?** It polls for the scan ID; `SBOM_MAX_RETRYCOUNT` bounds it.
23. **Bootstrap 64 KB error?** The output variable limit. Write the payload to a file.
24. **Bootstrap pod setup timeout?** Slow image pulls. Use `initTimeout: 30m`.
25. **Run bootstrap per medal?** No. Run it once; CI once; CD per input set.
26. **Does CI need Gold?** No. It runs on the Silver delegate and connectors.
27. **Artifactory 401 "Wrong username"?** The token's owner differs from the username. Mint the token as the service ID.
28. **Helm "No Chart.yaml … discovery-rbac"?** Wrong `EXISTING_TENANT` mode for an existing tenant.
29. **Istio "resource name may not be empty"?** Run `fetch_istio_gateway_name_gold` first.
30. **Migration step "context canceled"?** Apps already fired finish; re-run the missing apps in batches.

## 8. Ten troubleshooting scenarios
Use T15, T14, T30, T31, T32, T20, T22, T11, T05 and T10 from doc 12. Tell each one as: symptom → what I looked at → root cause → fix → how I verified it.
