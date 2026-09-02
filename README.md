# CICD-Goat Security Walkthroughs

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

Three are complete so far; the remaining two will be added to this table
as they're finished.

| Challenge    | OWASP Mapping                                   | Summary                                                                                                                                      |
| ------------ | ----------------------------------------------- | -------------------------------------------------------------------------------------------------------------------------------------------- |
| White Rabbit | CICD-SEC-4 (Direct PPE), CICD-SEC-6             | Push access to a PR branch let an unreviewed Jenkinsfile pull a credential straight out of the global store.                                 |
| Duchess      | CICD-SEC-6 (Credential Hygiene)                 | A PyPI token committed and later deleted from the repo was still fully recoverable from git history.                                         |
| Caterpillar  | CICD-SEC-2 (Identity & Access Mgmt), CICD-SEC-6 | Read-only repo access was bypassed using an over-privileged service token that leaked through a low-privilege build's environment variables. |

## Structure

Each completed or in-progress challenge has its own folder:

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

Root cause and remediation are documented for each challenge as it's
completed, but actually implementing each fix in the live Jenkins/Gitea
configuration and re-running the attack to confirm it's blocked is being
done as a batch pass after all challenges are solved, rather than
challenge-by-challenge. This is a deliberate sequencing choice, not an
oversight: as long as each fix stays scoped to its own job and credential
rather than touching Jenkins-wide settings, applying it early would only
risk accidentally hardening a challenge before it's been attempted.
