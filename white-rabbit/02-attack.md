# White Rabbit - Attack

**Technique:** CICD-SEC-4, Direct Poisoned Pipeline Execution (D-PPE)

## Steps

1. Edited `Jenkinsfile` on `Wonderland/white-rabbit` (Gitea) via the web UI, adding a `Debug Env` stage using `withCredentials([string(credentialsId: 'flag1', variable: 'FLAG')])` to bind the target credential to an environment variable.
2. Committed as a new branch and pull request, not directly to `main`. This was required because the Jenkins job's `WildcardSCMHeadFilterTrait` only builds branches matching `PR-*`.
3. Jenkins auto-scanned the multibranch job, detected the new `PR-1` branch, and ran the pipeline (confirmed by `Obtained Jenkinsfile from <commit sha>` in the console log).
4. `echo $FLAG | base64` bypassed Jenkins' credential log masking. Masking only matches the literal plaintext value of the credential, so encoding it first produced a different string that passed through untouched.
5. Decoded the output locally with `base64 -d` to recover the credential.

## Console Evidence

Confirmation that masking was active and attempted to catch the raw value:

```
[Pipeline] { (Debug Env)
[Pipeline] withCredentials
Masking supported pattern matches of $FLAG
[Pipeline] sh
+ base64
+ echo ****
```

The line immediately after is the base64-encoded output that bypassed masking:

```
MDYxNjVERjItQzA0Ny00NDAyLThDQUItMUM4RUM1MjZDMTE1Cg==
```

## Diff

See `jenkinsfiles/before.Jenkinsfile` versus `jenkinsfiles/after.Jenkinsfile` for the full change.
