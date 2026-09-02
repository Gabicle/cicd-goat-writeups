# White Rabbit - Root Cause and Fix

## Root Cause

This challenge combines two risks from the OWASP Top 10 CI/CD Security Risks.

**CICD-SEC-4: Poisoned Pipeline Execution, Direct variant (D-PPE)**

The `wonderland-white-rabbit` Jenkins job is a multibranch pipeline configured to
automatically build any branch matching `PR-*`. Anyone with push access to the
`Wonderland/white-rabbit` Gitea repository can open a pull request, and Jenkins
will execute whatever `Jenkinsfile` exists on that PR branch with no review,
approval, or protection gate in between. Per OWASP's own definition, pipelines
that execute unreviewed code, especially those triggered directly off pull
requests, are inherently susceptible to this attack class. Since the attacker
controls the pipeline definition itself, they effectively have arbitrary code
execution on the Jenkins agent.

**CICD-SEC-6: Insufficient Credential Hygiene**

The `flag1` credential lives in Jenkins' global credential store, which makes
it readable by any pipeline that references its ID via `withCredentials`,
regardless of whether that pipeline has any legitimate reason to use it. There
is no scoping that restricts the credential to only the specific trusted job
that actually needs it.

## Fix

**For CICD-SEC-4:** require review before unreviewed code can execute.

- Configure the multibranch job to require manual approval before building
  pull requests, rather than auto-building every `PR-*` branch on push. Jenkins'
  branch source plugins support marking contributors as trusted versus
  untrusted, only auto-building PRs from trusted contributors and holding
  everyone else for approval.
- Add a Gitea branch protection rule on `main` requiring at least one review
  before merge. This does not by itself stop the initial PR build, but it
  closes the related risk of a malicious change reaching `main` unreviewed.
- Where possible, run pipelines that build unreviewed code (such as PRs from
  first-time or untrusted contributors) on isolated nodes without access to
  the credential store, rather than on the same trusted agent used for `main`
  builds.

**For CICD-SEC-6:** scope the credential instead of leaving it global.

- Move `flag1` out of the global credential store into a folder-scoped or
  job-scoped credential store, so only the specific job that legitimately
  needs it can reference the ID at all. A pipeline defined in an unrelated
  repository's PR branch should not even be able to resolve the credential ID,
  let alone bind it.
- Apply the principle of least privilege to every credential in the store,
  not just this one: audit which jobs actually need which secrets, and remove
  access by default rather than granting it by default.

## Verification

After implementing the fix, the same attack (adding a `withCredentials` stage
to a PR branch's Jenkinsfile) should either:

- Never execute at all, because the PR requires manual approval before
  Jenkins will run it, or
- Execute but fail to resolve `flag1`, because the credential is no longer
  visible outside its scoped job.

Re-run the exact PR-based attack from `02-attack.md` against the hardened
configuration and confirm one of the two outcomes above, with console output
as evidence.
