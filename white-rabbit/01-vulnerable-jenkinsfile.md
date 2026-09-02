# White Rabbit - Vulnerable Baseline

**OWASP mapping:** CICD-SEC-4 (Poisoned Pipeline Execution) - Direct PPE variant.
Secondary: CICD-SEC-6 (Insufficient Credential Hygiene) - `flag1` credential has no
scoping restricting which pipelines/jobs may reference it.

**Target:** `wonderland-white-rabbit` Jenkins multibranch pipeline, backed by
Gitea repo `Wonderland/white-rabbit`.

**Config:** Job builds any branch matching `PR-*` (WildcardSCMHeadFilterTrait).
No branch protection or required review exists before a PR's Jenkinsfile executes.

**Baseline:** see `jenkinsfiles/before.Jenkinsfile` — three benign stages
(install deps, lint, unit test), no credential access anywhere.

**Root cause (preview):** Per OWASP CICD-SEC-4, pipelines that execute
unreviewed code triggered directly off PRs or arbitrary branches are
inherently susceptible to PPE. Anyone with push access to a PR branch has
effective code execution on the Jenkins agent, including any credential
the agent is authorized to bind via `withCredentials`.
