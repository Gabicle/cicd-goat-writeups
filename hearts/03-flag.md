# Hearts - Flag

**Captured via:** the honeypot SSH server's `check_auth_password` callback,
triggered when Jenkins attempted to authenticate to the malicious agent
node using the System-scoped `agent` credential.

**Flag:** `[REDACTED]`

Redacted here, and never shared in plaintext during this project's own
working session either, unlike prior challenges where the flag was
decoded and confirmed together. In this challenge the captured value is
the actual System-scoped Jenkins credential's real password, a
genuinely more sensitive category of secret than the earlier flags
(which were dedicated, purpose-built flag values), so it was kept out of
the conversation entirely on principle, not just redacted after the fact
for the public repo.
