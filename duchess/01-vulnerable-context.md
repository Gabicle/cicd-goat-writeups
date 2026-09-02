# Duchess - Vulnerable Baseline

**OWASP mapping:** CICD-SEC-6, Insufficient Credential Hygiene

**Target:** `Wonderland/duchess` Gitea repository (a fork of the real open source
PyJWT Python library, ~570 commits of genuine project history).

**Context:** This challenge does not involve a live CI pipeline at all. It
tests whether a secret that was once committed to a repository, and later
removed from the current file state, can still be recovered from that
repository's commit history.

**Why this is realistic:** removing a secret from a file in a new commit does
not remove it from git. The old commit still exists, and anyone with clone
access to the repository can check out that commit or diff it directly to see
the original content. Unless history is explicitly rewritten and force-pushed
(for example with `git filter-repo` or BFG Repo-Cleaner) and the exposed
credential is rotated, the secret remains permanently recoverable.

**No baseline config to capture here** since there is no pipeline definition
being attacked. The "before" state is simply: the current, latest commit of
the repo, where the secret is no longer present in any file you can see by
browsing the repo normally.
