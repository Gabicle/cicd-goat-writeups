# Duchess - Attack

**Technique:** CICD-SEC-6, Insufficient Credential Hygiene (secret recoverable from git history)

## Steps

1. Cloned the `Wonderland/duchess` repository locally with full commit history:
   ```
   git clone http://localhost:3000/Wonderland/duchess.git
   ```
2. Ran gitleaks against the full clone, including all commit history, not just
   the current file state:
   ```
   docker run -v "${PWD}:/repo" zricethezav/gitleaks:latest detect --source="/repo" -v
   ```
3. The scan returned 36 findings across 570 commits scanned. The large majority
   were false positives: this repository is a fork of the real open source
   PyJWT library, so most findings were the library's own test fixtures and
   documentation examples (sample JWTs, test RSA/EC keys under `tests/keys/`),
   genuinely part of the project's real history rather than planted secrets.
4. One finding stood apart from the rest on two signals: the rule type
   (`pypi-upload-token`, matching the challenge's hint about a PyPI token
   specifically) and the commit author, `asaf@cidersecurity.io`, someone at
   Cider Security (the company that built CICD-Goat) committing directly into
   what is otherwise an unmodified upstream fork:
   ```
   RuleID:      pypi-upload-token
   File:        .pypirc
   Line:        8
   Commit:      43f216c2268a94ff03e5400cd4ca7a11243821b0
   Author:      Asaf <asaf@cidersecurity.io>
   Date:        2021-11-16T09:22:31Z
   ```
5. gitleaks' console output truncates long secret values, so retrieved the
   full file content as it existed at that specific commit:
   ```
   git show 43f216c2268a94ff03e5400cd4ca7a11243821b0:.pypirc
   ```
   This returned the complete `.pypirc` file, including the full PyPI upload
   token in the `password` field.

## Key Takeaway

Automated secret scanning across full history, not just the current file
tree, is necessary to catch this class of leak. Distinguishing the real
finding from legitimate historical noise required correlating rule type,
file path, and commit author, not just trusting every scanner hit at face
value.
