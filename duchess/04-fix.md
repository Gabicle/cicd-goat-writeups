# Duchess - Root Cause and Fix

## Root Cause

**CICD-SEC-6: Insufficient Credential Hygiene**

A PyPI upload token was committed directly into a file (`.pypirc`) in the
`Wonderland/duchess` repository. Per OWASP's own description of this risk,
once credentials are pushed to any branch of an SCM repository, they remain
exposed to anyone with read access to that repository from that point on,
even after being deleted in a later commit, because they persist permanently
in the commit history unless that history is explicitly rewritten. Read
access to the repository was sufficient here; no write access, no pipeline
trigger, and no interaction with Jenkins was required at all.

## Fix

**Immediate: rotate the exposed credential.** Since the token is already
in history and was retrievable, treat it as compromised regardless of
whether it is still deleted from the current file tree. Revoke it on
PyPI's side and issue a new one.

**Purge it from history, not just the current tree.** Deleting a secret in
a new commit is not sufficient. Rewriting history with a tool such as
`git filter-repo` or BFG Repo-Cleaner, then force-pushing the rewritten
history, is required to actually remove the value from every commit that
ever contained it. This is disruptive (it changes commit hashes for every
collaborator) and is exactly why prevention matters more than cleanup.

**Prevent recurrence with automated scanning before commits land.**

- Add a pre-commit hook running gitleaks (or an equivalent secret scanner)
  locally, so a credential-shaped string is caught before it is ever
  committed, not after.
- Add the same scan as a required CI gate on every push and pull request,
  so a bypassed local hook still gets caught centrally before merge.
- Treat any file whose entire purpose is to hold a credential (`.pypirc`,
  `.npmrc`, `.env`, etc.) as high-risk by default: add such filenames to
  `.gitignore` from the start of a project, and use a secret manager or
  CI-native secret store instead of a plaintext config file for storing
  the real value.

**Broader practice worth naming:** insufficient credential hygiene in the
OWASP list also covers credentials that are still live but overly broadly
scoped, and secrets that leak through build console output rather than
through source control. Git history exposure is one delivery mechanism for
this risk class among several; the underlying failure (a secret existing
somewhere it can be read by more parties than actually need it) is the
same regardless of which mechanism surfaces it.

## Verification

Re-run the gitleaks scan against the repository after history has been
rewritten and the credential rotated. The `pypi-upload-token` finding
should no longer appear at all, since the rewritten history contains no
commit where the value was ever present. A rotated-but-not-purged secret
would still show up in the scan, which is itself a useful negative test to
demonstrate the difference between the two remediation steps.
