# Caterpillar - Root Cause and Fix

## Root Cause

**CICD-SEC-2: Inadequate Identity and Access Management**

The Jenkins-to-Gitea integration uses a service credential, `GITEA_TOKEN`,
to perform actions on behalf of the pipeline system: cloning repositories
and posting build statuses back to Gitea. Per OWASP's description of this
risk, maintaining least privilege for both human and application identities
is difficult in practice, and the average service account in an SCM or CI
system tends to be far more permissive than it needs to be, since these
accounts have not traditionally been a focus for security review the way
human accounts are.

In this case the service account was overly permissive in two separate
ways at once: it had merge rights on `main`, a capability no reasonable
build-status-reporting integration should need, and its value was reachable
from inside any build's environment variables, including a low-privilege
`test` job that any user able to open a pull request could trigger. Neither
flaw alone would have been sufficient. The excessive permission made the
token worth stealing; the exposure made it possible to steal.

## Fix

**Scope the service account to what it actually does.** A credential used
only to clone repositories and report commit statuses back to an SCM does
not need merge permissions. Reissue it as a fine-grained token restricted to
exactly those two actions, on exactly the repositories Jenkins needs to
interact with, not organization-wide merge rights.

**Do not expose the credential to job environments that do not need it.**
If a service credential must exist with elevated scope for some legitimate
job, it should be injected only into that specific job's environment, not
made globally available to every pipeline including low-privilege PR-builder
jobs. Jenkins supports scoping credentials to specific folders or jobs
rather than making every credential visible to every pipeline by default.

**Treat build environment variables as readable by the code under test.**
Any pipeline that executes code from a pull request, including linting and
test steps, should be assumed capable of reading its own environment. A
service token should never be placed somewhere that ordinary build steps
(not just an intentionally added `printenv` stage) can see it. Consider
restricting what environment variables are passed into untrusted PR builds
at all, separate from the credential scoping question.

**Review actual permission usage, not just what was originally granted.**
OWASP's broader guidance on this risk class calls for reviewing actual
usage against granted permissions and reducing access accordingly over
time. Permissions on long-lived service accounts tend to accumulate and
rarely get revisited once initially configured; a periodic audit specific
to CI/CD service identities would have caught this token's excessive scope
independent of the exposure that made it exploitable.

## Verification

After scoping `GITEA_TOKEN` down to clone and status-reporting permissions
only, re-run the same attack: leak the token from the `test` job's
environment as before, then attempt the same merge API call used in
`02-attack.md`. The Gitea API should now return a permissions error rather
than merging the PR, confirming the token can no longer be used to bypass
the merge restriction even though it is still technically exposed. Fully
closing the exposure itself (removing the token from environments that do
not need it) is the deeper fix and should be verified separately by
confirming `printenv` inside the `test` job's build no longer includes it
at all.
