# Twiddledum - Root Cause and Fix

## Root Cause

**CICD-SEC-3: Dependency Chain Abuse**

`twiddledum` declares `twiddledee` as a dependency sourced directly from an
internal Gitea server, using a floating semver range (`^1.1.0`) rather than
a fully pinned commit. Anyone with write access to `twiddledee`, regardless
of whether they have any relationship to `twiddledum` itself, can cause
their code to execute inside `twiddledum`'s build the moment `twiddledum`
requires and loads it. The two projects share no ownership or trust
boundary beyond one `package.json` line, yet a compromise of one is a full
compromise of the other's build execution.

A partial control was already in place and did work correctly: the build
runs `npm ci --ignore-scripts`, which blocks the npm lifecycle-script
injection vector specifically. It does not, and cannot, block plain module
execution via `require()`, which is a fundamentally different code path
npm has no equivalent flag to disable.

## Fix

**Fetch internal packages only from an internal, access-controlled
registry, not directly from a source repository.** OWASP's own guidance on
this risk is specific: clients should be forced to fetch packages under an
organization's scope solely from an internal package registry (an npm
Artifactory or Verdaccio instance, for example), rather than resolving
git dependencies straight from a source control server. A registry adds a
publish step as a deliberate checkpoint, rather than making every commit
to a source repository instantly consumable by every downstream project
that depends on it.

**Pin the exact dependency, not a floating range.** Even without a full
internal registry, replacing `#semver:^1.1.0` with a fully pinned commit
hash removes the ability to redirect the dependency via a new tag at all.
This is a meaningfully weaker fix than an internal registry (an attacker
with commit access to the pinned repo could still eventually get a
malicious commit approved and re-pinned through normal change review), but
it closes the specific "just push a new tag" attack path demonstrated
here.

**Isolate install-script execution from the rest of the build, per
OWASP's guidance, even though `--ignore-scripts` was already doing this
correctly here.** Worth stating explicitly since it's easy to assume a
partial control failed just because the overall attack still succeeded
through a different path.

**Never assume read-only access to the primary repository is sufficient
isolation on its own,** the same lesson from Caterpillar applies here in
a different form: `twiddledum` being unwritable did not actually protect
its build, because build-time trust extended transitively to a second
repository with looser access controls.

## Verification

Re-run the same attack after pinning `twiddledee` to a specific commit
hash instead of a semver range. Pushing a new tag to `twiddledee` (by
either the Gitea web UI or git CLI) should no longer change what the next
`wonderland-twiddledum` build resolves and executes, since there is no
longer a floating range for a new tag to satisfy. Confirm by checking that
the build's `twiddledee - <version>` output line stays constant across
builds even after `twiddledee`'s `main` branch and tags continue to
change.
