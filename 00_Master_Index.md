# 00 — Master Index: UEP Harness Project Knowledge Base

| Doc | Topics |
|---|---|
| 01 Project Overview | scope, phases, timeline 13–20 Nov, current state, evidence legend |
| 02 Business Use Case | Bamboo retirement, standard chain, DoD, waves, CI+CD, Vault role |
| 03 Enterprise Architecture | 7 Mermaid diagrams: high-level, component, secrets, delegate, image, APM workflow, migration |
| 04 Vaulting | namespaces, KV v2 layout, auth paths, roles/policies, Automation Hub, vault-automation, validations |
| 05 Harness Onboarding | orgs/projects, IDP workflows, what Silver creates, form fields, Gold flow, connectors, permissions/PAT, Platinum/DR |
| 06 Migration | EHDS migration_pipeline, golden CI stages, bootstrap validation, Bamboo gaps |
| 07 APM Execution Workflow | APM definition, assignment, lifecycle, blockers, reusable checklist |
| 08 Environments | Silver/Gold/Platinum/DR comparison table |
| 09 Delegate Architecture | creation, communication, failure modes, restart without kubectl |
| 10 Quay | registries, connectors, tags, ECR, unknown promotion |
| 11 Helm and ArgoCD | ArgoCD roles, helm repo modes, APPLICATION_NAME, Istio/dnsHost |
| 12 Troubleshooting | T01–T37 issues with errors, causes, fixes, status |
| 13 Decisions | D1–D22 plus contradictions resolved |
| 14 Responsibilities | done by you / directed / in progress / learned / unconfirmed |
| 15 Open Issues and Risks | prioritised list, decisions needed, knowledge gaps |
| 16 Interview Prep | intros, APM1003706 walkthrough, resume bullets, 30 Q&A, scenarios |
| 17 Glossary | terms |

## Sources used
Devin sessions for user Ashwitha Boddu: 6d57f695 (Bootstrap pipeline, 13–19 Nov), a60073e6 (APM1081216 Silver, 13–20 Nov), 05bffe6a (APM1081123 Gold, 15–20 Nov), d63d0759 (APM1080718 Gold, 15–18 Nov), cea88c9e (Assigned tickets, 16–17 Nov), 19444104 (Data validation, 16–18 Nov), 3fbcc88b (CI runs, 17–19 Nov), 065eaf00 (Bamboo-to-Harness KT, 18–22 Nov), d343ee9c (Silver onboarding KT, 18–19 Nov), 7ee4bba9 (APM1003706, 18–20 Nov), 725eb5ea (APM1010417 Wave 3, 19 Nov); plus Devin memory notes and org Harness knowledge.

## Historical information I could NOT recover
- Anything before 13 Nov 2026, or work done outside Devin (Teams chats, emails, calls, Devin Desktop sessions on your laptop; only what you pasted back is known).
- The contents of your attached sheets (excel.zip etc.) beyond the values quoted in chat; the files weren't re-read for this document.
- The KT HTML guide and the Excel/CSV reports themselves (only their descriptions in chat).
- Message bodies longer than the history API returns were truncated in places; some middle portions of long sessions were read only in summary.
- Outcomes after the last message of each session (e.g. MCMUMINFRA repo creation, APM1081216 EHDS access, pod-kill root cause, replies to emails, workaround reverts).
- Any hands-on Bamboo plan migration, Platinum/DR onboarding, CD execution or prod cutover: none occurred in the accessible history.
