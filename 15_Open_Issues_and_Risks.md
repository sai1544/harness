# 15 — Open Issues and Risks (prioritised)

| Priority | Item | APM(s) | Owner / needed from | Impact |
|---|---|---|---|---|
| P1 | Service accounts lack read/WRITE on legacy Bitbucket projects (`SCMRuntimeException … Not Authorized`) | APM1081191, APM1081123 (`EHAPPGBDSC`), APM1080718, APM1006991, APM1081670 | Bitbucket project admins | 205/207 CI runs fail; no images; blocks all CD |
| P1 | APM1081216 Helm step: `srcUEPHAPM1081216BIT` can't read `EHDS/bitbucket-helm-charts-silver`; `APPLICATION_NAME` template bug | APM1081216 | EHDS admin; template owners | Onboarding ~75%, no chart repo/Istio/DA |
| P1 | Infra project key `UMINFRA` doesn't exist; `MCMUMINFRA/apm1003706-gitexperience-repo` needed with WRITE | APM1003706 | App team + MCMUMINFRA admin | 15 pipelines not created |
| P2 | CI pods killed/hung (unproven eviction) | APM1081191, APM1081670, APM1080718 | AKS platform / app teams | CI can't finish even after Bitbucket is fixed |
| P2 | Artifactory token minted by human `am89727` | APM1006991 | you, logged in as `srcUEPHAPM1006991ART` | Maven 401 |
| P2 | Wiz real credentials missing (`test` placeholders) | bootstrap APMs | security/Wiz team | Wiz step fails (no ignore) |
| P2 | Checkmarx projects missing under both names | various | CxOne admins | Scan results not recorded |
| P2 | Temporary workaround not reverted: APM1081123 `apm1081123_bitbucket` on Gold delegate/secrets | APM1081123 | you/Devin + proper Silver token fix | Hidden dependency on Gold; config drift |
| P2 | Shared Bitbucket/Sonar accounts across medals: any future vaulting can revoke the other medal's token | all with shared SIDs | vaulting procedure | Recurring outages |
| P3 | Service IDs cancelled (Closed Incomplete) | APM1000159, APM1012135 | SNOW | Can't vault |
| P3 | Data conflicts awaiting confirmation (clusters, hostnames, source keys, Quay org, Artifactory key) | 11 Wave 2 APMs, APM1000782 | app teams | Wrong inputs → failed runs |
| P3 | Bootstrap fixes only on `devin/bootstrap-init-timeout` branches | APM1081670, APM1006991, APM1080718 | template owners | Others will hit the 64 KB / timeout issues |
| P3 | APM1006991 bootstrap service org mismatch | APM1006991 | — | Test CD can't run |
| P3 | APM1080718 Gold migration re-run after the token fix not recorded | APM1080718 | you | Gold services may not exist |
| P3 | Exposed secrets in chat (PATs, ArgoCD passwords, CxOne secrets, Automation Hub creds) | — | owners | Security |
| P3 | Per-app `dnshost` override untested | APM1003706 et al. | — | CD routing risk |
| P4 | No CD run, no Platinum/DR onboarding, no promotion/cutover/rollback procedure documented | all | program | Production readiness unknown |
| P4 | Network: Devin VM can't reach Vault/ArgoCD/Automation Hub | — | — | Dependence on your laptop / Devin Desktop |

## Decisions needing confirmation
- Is `MCMUMINFRA` the correct infra key for APM1003706?
- Clusters per medal for APM1000782 (`ocp-rosa-np1/np2/prod/dr`)?
- `MCMUMDE` vs `srcumde` source key for APM1003706?
- DEV/SIT vs UAT/PERF mapping to Silver/Gold as an official rule.

## Knowledge gaps
Bamboo plan mapping (never exercised); Platinum/DR specifics; image promotion; ServiceNow change flow; template ownership for Solutions_Factory; root cause of pod kills.
