# Hearts - Root Cause and Fix

## Root Cause

**CICD-SEC-7: Insecure System Configuration**

OWASP frames this risk broadly: flaws in the security settings and
hardening of any system across the CI/CD pipeline, not limited to a
specific job or repository, that hand an attacker a low-hanging foothold.
This challenge demonstrates two separate configuration weaknesses on the
Jenkins server itself compounding into one exploit chain:

1. **Weak credential hygiene on a privileged account.** `knave`, an
   account explicitly described as "Agents admin," had a password
   present in a well-known leaked-password wordlist. No account lockout,
   rate limiting, or multi-factor requirement stopped an automated,
   sequential login attempt from succeeding.
2. **Overly permissive agent-configuration capability.** Once
   authenticated as `knave`, Jenkins allowed configuring a new SSH-based
   agent pointing at an arbitrary host and port, while separately
   allowing selection of an existing, more sensitive System-scoped
   credential to authenticate with. Nothing restricted which credentials
   a given class of node configuration may use, and nothing validated
   that the configured host was a legitimate, expected agent before
   Jenkins attempted a real authentication handshake against it.

Neither weakness alone was catastrophic. A brute-forceable password on
an account with no special privileges would matter far less. The
combination, weak authentication on an account that can also redirect
where a sensitive credential gets used, is what turned a guessable
password into full System-credential exfiltration.

## Fix

**Enforce strong, non-default credential hygiene on every account, not
only ones perceived as high-value.** Account lockout after repeated
failed attempts, minimum password complexity, and ideally multi-factor
authentication would have stopped the brute-force step entirely,
regardless of how the account's permissions were configured afterward.

**Apply least-privilege network access, per OWASP's core recommendation
for this risk.** Restrict which hosts a Jenkins-managed agent
configuration may target, ideally to a pre-approved allowlist, so that
even an authenticated "Agents admin" cannot redirect Jenkins' outbound
SSH authentication toward an arbitrary attacker-controlled destination.

**Scope credentials to the specific nodes and jobs that legitimately need
them**, rather than making a System-level credential selectable from any
agent-configuration screen an account with agent-admin rights can reach.
The same principle already documented for Caterpillar (CICD-SEC-2/6)
applies here at a higher privilege tier: broad credential availability
combined with any single compromised account is a much larger blast
radius than either weakness alone.

**Periodically review system configuration itself, not just code**, per
OWASP's guidance: establish a recurring process specifically for
reviewing settings that affect a system's security posture (who can
configure agents, what those agents can authenticate as, what network
destinations are reachable), separate from ordinary code review, since
these settings rarely change and are easy to leave unexamined once
initially configured.

## Verification

After enforcing account lockout on repeated failed logins, re-run the
same brute-force script against `knave`'s account and confirm it is
locked out well before reaching a working password, rather than allowed
to continue indefinitely. Separately, after restricting agent-host
targets to an allowlist, attempt to configure a new SSH agent pointing at
an arbitrary host (such as the honeypot's address used here) and confirm
Jenkins rejects the configuration outright, rather than accepting it and
only failing at connection time.
