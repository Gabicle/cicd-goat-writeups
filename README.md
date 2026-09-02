# CICD-Goat Security Walkthrough

A hands-on DevSecOps portfolio project working through
[CICD-Goat](https://github.com/cider-security-research/cicd-goat), a
deliberately vulnerable CI/CD environment covering the
[OWASP Top 10 CI/CD Security Risks](https://owasp.org/www-project-top-10-ci-cd-security-risks/).

Each challenge is documented as a full cycle: capture the vulnerable
baseline, exploit it, retrieve the flag, then document the root cause and
a concrete fix mapped to the relevant OWASP CICD-SEC risk. The goal is to
demonstrate not just "found the flag" but the reasoning an AppSec or
DevSecOps engineer would actually apply: why the flaw exists, what makes
it exploitable, and what a real remediation looks like.

## Environment

- CICD-Goat run locally via Docker (9 containers: Gitea, Jenkins,
  Jenkins agent, GitLab, GitLab runner, LocalStack, CTFd, and two
  Docker-in-Docker services), unmodified `docker-compose.yaml` pulled
  from the upstream project's `main` branch.
- Attacker perspective throughout: the low-privilege `alice` / `thealice`
  accounts provisioned by the lab, not the break-glass admin accounts.

## Scope

Five challenges selected out of CICD-Goat's full set, chosen to cover five
distinct risk categories rather than several variations on the same one.
All five are complete.

| Challenge    | OWASP Mapping                                   | Summary                                                                                                                                                          |
| ------------ | ----------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| White Rabbit | CICD-SEC-4 (Direct PPE), CICD-SEC-6             | Push access to a PR branch let an unreviewed Jenkinsfile pull a credential straight out of the global store.                                                     |
| Duchess      | CICD-SEC-6 (Credential Hygiene)                 | A PyPI token committed and later deleted from the repo was still fully recoverable from git history.                                                             |
| Caterpillar  | CICD-SEC-2 (Identity & Access Mgmt), CICD-SEC-6 | Read-only repo access was bypassed using an over-privileged service token that leaked through a low-privilege build's environment variables.                     |
| Twiddledum   | CICD-SEC-3 (Dependency Chain Abuse)             | A dependency pulled from an internal repo with looser access controls let its code execute inside a build for a project that could never be written to directly. |
| Dodo         | CICD-SEC-1 (Insufficient Flow Control)          | A security scanner correctly detected a policy violation on every run; nothing enforced that a failed scan actually stopped the deployment.                      |

## Structure

Each challenge has its own folder:

```
<challenge-name>/
├── 01-vulnerable-context.md   baseline config and why it's exploitable
├── 02-attack.md                exact steps taken, with console evidence
├── 03-flag.md                  flag captured (redacted for this public repo)
└── 04-fix.md                   root cause and remediation, mapped to OWASP CICD-SEC
```

Flag values are intentionally redacted throughout. CICD-Goat flags are
shared across everyone running the lab, so they carry no real
confidentiality value, but publishing them verbatim in a public repo isn't
good practice regardless. The retrieval method, which is the actual skill
being demonstrated, is documented in full in each `02-attack.md`.

## Fix-and-Verify

Root cause and remediation are documented for each challenge, but actually
implementing each fix in the live Jenkins/Gitea configuration and
re-running the attack to confirm it's blocked is being done as a batch
pass now that all five are solved, rather than challenge-by-challenge.
This was a deliberate sequencing choice, not an oversight: as long as
each fix stays scoped to its own job and credential rather than touching
Jenkins-wide settings, applying it earlier would only have risked
accidentally hardening a challenge before it had been attempted.
