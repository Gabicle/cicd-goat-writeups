# Hearts - Vulnerable Baseline

**OWASP mapping:** CICD-SEC-7, Insecure System Configuration

**Target:** the Jenkins server's own configuration, not a specific repo or
job. No Jenkinsfile, no Gitea repository, and no Terraform involved in
this challenge at all, unlike every prior challenge in this project.

**Context:** Jenkins' People list (`/asynchPeople/`) shows five accounts:
`49395894+Gabicle` (the account used throughout this project), `admin`,
`alice`, `knave`, and `SYSTEM`. Viewing `knave`'s individual profile page
shows a description: **"Agents admin"**. This account holds permissions
the standard `alice`/`thealice` account used in every other challenge does
not, specifically, the ability to register new Jenkins build agents.

**The target credential:** `flag8` is stored as a **System**-scoped
Jenkins credential, one tier above the job-level credentials targeted in
every prior challenge (`flag1`, `flag2`, `flag6`, `flag7`). Ordinary
pipelines cannot bind to a System-scoped credential via `withCredentials`
at all; reaching it requires actual administrative access to Jenkins
itself, not just repository or pipeline access.

**Root cause (preview):** two separate weaknesses combine here. First,
`knave`'s password turned out to be a common, real-world leaked password
(found via a straightforward automated login attempt against a well-known
wordlist), a weak credential on a privileged account. Second, once logged
in as `knave`, Jenkins' "Launch agents via SSH" feature lets that account
configure a new agent node pointing at any host and port of the
configurer's choosing, and select an existing System-scoped SSH credential
for Jenkins to use when connecting to it. Jenkins will attempt to
authenticate to whatever host is configured using that credential's real,
unmasked value, even though the value is never displayed anywhere in the
UI (`agent/*****`). Pointing that host at a server under the attacker's
control causes Jenkins to send the real password directly to it.
