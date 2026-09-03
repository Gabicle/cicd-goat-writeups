# Cheshire Cat - Attack

**Technique:** CICD-SEC-5, Insufficient Pipeline-Based Access Controls. A
pipeline is free to specify the Jenkins controller itself as its execution
node, with no separate check preventing that choice.

## Recon

1. Found the controller node's internal label via Jenkins' own REST API,
   rather than guessing:
   ```
   http://localhost:8080/computer/(built-in)/api/json?pretty=true
   ```
   Returned `"assignedLabels": [{"name": "built-in"}]`. The label was
   also visible directly in the URL of the node's own info page
   (`/computer/(built-in)/...`), a useful shortcut worth remembering: the
   identifier is often embedded in the page's own address before you ever
   need to inspect the API response.

## Attack

1. Cloned `Wonderland/cheshire-cat`.
2. Edited `Jenkinsfile`, changing the `agent` directive from `agent any`
   (let Jenkins pick any available node) to an explicit request for the
   controller:
   ```groovy
   agent {
       node {
           label "built-in"
       }
   }
   ```
   and added a new stage reading the target file, placed first so it runs
   before anything else in the pipeline:
   ```groovy
   stage ('Read flag') {
       steps {
           sh "cat ~/flag5.txt; echo"
       }
   }
   ```
3. Attempted to push directly to `main`, rejected: `main` is a protected
   branch (`Gitea: Not allowed to push to protected branch main`),
   confirming write access to the repository exists, only direct pushes to
   `main` specifically are blocked.
4. Created a new branch and pushed it, then opened a pull request against
   `main`, same PR-based trigger mechanism as White Rabbit.

## A Real Mistake, Not a Technique

The first PR build ran successfully on the controller (confirmed by the
console output: `Running on Jenkins in /var/jenkins_home/workspace/...`,
a Jenkins-controller-specific path, not an agent workspace) but the "Read
flag" stage never appeared in the log at all, not even as skipped. Checking
the actual committed `Jenkinsfile` on that branch showed the `agent` block
change had made it in, but the new "Read flag" stage had not, a plain
copy/paste miss while editing the file locally, not a Jenkins behavior
worth documenting as a technique. Rewriting the file in full and pushing a
second commit to the same branch (Jenkins auto-rebuilt on the new commit,
no new PR needed) resolved it.

## Result

```
[Pipeline] { (Read flag)
[Pipeline] sh
+ cat /var/jenkins_home/flag5.txt
[REDACTED - see 03-flag.md]
```

The console output also confirms the flag file's real absolute path on
the controller's filesystem: `/var/jenkins_home/flag5.txt`.

A later stage (`Install_Requirements`) still failed with
`virtualenv: not found`, since the controller container does not have
Python's `virtualenv` package installed the way the agent does. This is
unrelated to the exploit and happens after the flag has already been
captured; the pipeline's final status shows red (`FAILURE`) despite the
attack having fully succeeded, worth noting so the red status isn't
mistaken for the exploit not working.
