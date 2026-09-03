# Cheshire Cat - Root Cause and Fix

## Root Cause

**CICD-SEC-5: Insufficient Pipeline-Based Access Controls (PBAC)**

OWASP's own framing of this risk: a piece of code able to run within a
pipeline execution node has the full permissions of whatever that node
can reach. The severity of that risk scales directly with what the node
actually has access to. A sandboxed build agent (`agent1` in this
environment) is deliberately limited: it can build and test code, but has
no access to the Jenkins server's own filesystem, configuration, or
credential store. The Jenkins controller has all of that, by definition,
it is the server.

Nothing in this environment's configuration distinguished those two
targets from a pipeline author's perspective. `agent any` and
`agent { node { label "built-in" } }` are both syntactically ordinary
Jenkinsfile instructions; nothing rejected the second one, and no
additional authorization step existed for a job that requests
execution on the controller versus one that requests a normal agent.
Anyone able to edit a Jenkinsfile, the same access every legitimate
contributor already has, could redirect their entire pipeline to run in
the single most sensitive location in the whole system.

## Fix

**Disable execution on the controller entirely**, which is Jenkins' own
official recommendation, not a project-specific suggestion: set the
Built-In Node's executor count to 0. With no executors available there,
no pipeline, however it's configured, can ever be scheduled to run on
the controller, removing the vulnerable choice rather than trying to
police it after the fact.

**Isolate nodes by trust level and sensitivity**, per OWASP's broader
PBAC guidance: use dedicated, more restricted execution environments for
any pipeline or stage that genuinely needs elevated access, rather than
leaving a single shared setting (which node a job may request) available
to every pipeline uniformly.

**Limit what a pipeline author can specify in the first place.**
Jenkins supports restricting which labels a given job or folder is
permitted to target, so that even a compromised or malicious Jenkinsfile
cannot request a node outside its intended, pre-approved set. Applied
here, `Wonderland/cheshire-cat`'s job could have been scoped to only ever
consider `agent1`, regardless of what its own Jenkinsfile requested.

**Treat "which node runs this" as a decision requiring the same rigor
as credential access**, since, as demonstrated here, controlling
execution location is often equivalent to controlling what the pipeline
can ultimately reach.

## Verification

After setting the Built-In Node's executor count to 0, re-run the same
attack: push a Jenkinsfile requesting `agent { node { label "built-in" } }`
again. The build should fail to schedule at all (no available executor
matching that label), rather than running on the controller and printing
the flag. Confirm this distinctly from a simple job failure, the build
should show as stuck/unschedulable, not as a failed pipeline run.
