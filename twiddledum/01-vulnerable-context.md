# Twiddledum - Vulnerable Baseline

**OWASP mapping:** CICD-SEC-3, Dependency Chain Abuse

**Target:** `wonderland-twiddledum` Jenkins job. Unlike every other challenge
in this project, neither `Wonderland/twiddledum` nor `Wonderland/twiddledee`
has a Jenkinsfile. The job is configured directly inside Jenkins (not
pipeline-as-code) to clone `twiddledum`, run `npm ci --ignore-scripts`, then
`node index.js`. It must be triggered manually with "Build Now", there is no
webhook or branch-scanning automation on this job.

**Access:** read-only on `Wonderland/twiddledum`, write access on
`Wonderland/twiddledee`.

**The dependency link:**

`twiddledum`'s `package.json` (itself a fork of the real open source
`dedupe` npm package) declares:

```json
"dependencies": {
    "twiddledee": "git+http://gitea:3000/Wonderland/twiddledee#semver:^1.1.0"
}
```

This is a git dependency, not a registry package: npm clones `twiddledee`
directly from the internal Gitea server rather than fetching it from
npmjs.org. `twiddledum`'s own `index.js` does:

```js
"use strict";
require("twiddledee");
```

`require()` executes a module's top-level code the moment it loads, so
whatever `twiddledee`'s `index.js` contains runs automatically as part of
`twiddledum`'s build, with no further action needed on `twiddledum`'s side.

**Root cause (preview):** `twiddledum` is an otherwise legitimate, unmodified
package that trusts and executes code from a second repository purely by
name. The dependency happens to point at an internal Gitea repo that a
lower-privileged account has write access to, even though that account
cannot touch `twiddledum` itself. Poisoning the dependency is functionally
equivalent to poisoning `twiddledum` directly, since nothing distinguishes
"code from a trusted first-party module" from "code from an internal
dependency with looser access controls" at execution time.
