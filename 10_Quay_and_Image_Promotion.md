# 10 — Quay and Image Promotion

## Confirmed [C]
- Registries: **`quay-nonprod.elevancehealth.com`** (CI push target) and **quay-prod** (source of `ci-addon`, `drone-git` and delegate images).
- Connectors: Silver `apm<id>_docker` (non-prod) and `apm<id>_docker_prod`; Gold `apm<id>_gold_docker`. Docker creds come from Vault keys `docker_username`/`docker_password` (Quay account `srcehdsquay`, standard fixed password).
- Quay org per tenant (e.g. `ehappcsbd`, `unified_desktop`, `sprk-wrklds-tekton`). It was missing from intake sheets for all 11 Wave 2 APMs.
- CI step `Build_and_Push_Docker` requires `APP_PUSH_TYPE=docker` (blank → "requires APP_PUSH_TYPE=docker; got ''"; use Run, not Re-run). `DOCKERFILE_IN_APP=false` uses the File Store Dockerfile (e.g. `/ci_config/nodejs-22-dockerfile`).
- Tag pattern: `release-<branch>-<MMDDYYYYhhmmss>` (e.g. `release-master-11172026071936`).
- After push, `Wiz_Container_Scan` checks the image (`Failed Policies: CAAS_Team_Policy` seen on an `_NXT` app).
- NextGen pushes to **ECR** (`APP_PUSH_TYPE=ecr`; bootstrap has `Create_ECR_Repo`).
- Slow pulls from quay-prod on node `aks-linuxlarge-23837039-vmss0000gr` (146 MB `ci-addon` in 4m06s) caused a pod-setup timeout. Fixed with `initTimeout: 30m`.
- Gold Docker Vault for DIG: per-APM KV (`secret/gld/<apm>/np02`, `dig-<apm>`, `harness/harness`). The copy-pasted LDAP values (`ldap/devad/static-cred`, `corp-apm1002184`, `srcuephapm<id>qua`) broke APM1080718/APM1080955.

## Not established [U]
- How images get promoted from non-prod Quay to prod Quay (a copy job, re-tag, or rebuild) was never discussed.
- Signing: a "cosign" variable group exists in the project variables, but signing was never examined.
