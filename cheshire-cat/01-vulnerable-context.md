# Cheshire Cat - Vulnerable Baseline

**OWASP mapping:** CICD-SEC-5, Insufficient Pipeline-Based Access Controls
(PBAC)

**Target:** `wonderland-cheshire-cat` Jenkins job, backed by the
`Wonderland/cheshire-cat` Gitea repository. Standard PR-triggered
multibranch pipeline, matching the same pattern as White Rabbit: `main` is
protected against direct push, but a branch plus a pull request is
sufficient to get a Jenkinsfile change built.

**Challenge-specific scope note:** the official challenge description
includes an explicit rule: _"Don't use the access gained in this challenge
to solve other challenges."_ Respected throughout; nothing obtained here
was reused elsewhere in this project.

**Context:** Jenkins in this environment has two nodes:

- `agent1`, a sandboxed build worker, where every other challenge in this
  project ran.
- **Built-In Node**, the Jenkins controller itself, the actual server
  process holding job configs, build history, and (in real deployments)
  the credential store. Confirmed via Jenkins' own REST API
  (`/computer/(built-in)/api/json`), which returned
  `"assignedLabels": [{"name": "built-in"}]` and the description
  `"the Jenkins controller's built-in node"`.

**Baseline `Jenkinsfile`:**

```groovy
pipeline {
    agent any
    environment {
        PROJECT = "sanic"
    }
    stages {
        stage ('Install_Requirements') {
            steps {
                sh """
                    virtualenv venv
                    pip3 install -r requirements.txt || true
                """
            }
        }
        stage ('Lint') {
            steps {
                sh "pylint ${PROJECT} || true"
            }
        }
        stage ('Unit Tests') {
            steps {
                sh "pytest || true"
            }
        }
    }
    post {
        always {
            cleanWs()
        }
    }
}
```

**Root cause (preview):** `agent any` leaves the choice of execution node
entirely up to whatever a pipeline's author writes, and nothing in Jenkins'
configuration here restricts a job from explicitly requesting the
controller node instead of a sandboxed agent. A build agent is, by design,
isolated from the server it reports to. The controller is not; it is the
server. Anyone who can edit a Jenkinsfile can choose to run their pipeline
there instead, with no separate authorization step required.
