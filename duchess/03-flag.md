# Duchess - Flag

**Captured via:** `git show <commit>:.pypirc` on the commit identified by
gitleaks, retrieving the full PyPI upload token that was truncated in the
scanner's console output.

**Flag:** `pypi-...redacted, see 02-attack.md for the retrieval method...`

Redacted here since this is a public repo. As with White Rabbit, the value
itself is not a real secret since it is shared across everyone running the
lab. The retrieval method, correlating a scanner finding against commit
metadata to separate a real leak from legitimate historical noise, is the
part worth documenting in full.
