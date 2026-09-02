# Caterpillar - Vulnerable Baseline

**OWASP mapping:** CICD-SEC-2, Inadequate Identity and Access Management (primary),
with a CICD-SEC-6, Insufficient Credential Hygiene component in how the
escalation path is discovered.

**Target:** `Wonderland/caterpillar` Gitea repository, read-only access for the
`thealice` account. Two Jenkins jobs back this repository:

- `wonderland-caterpillar-test`, a low-privilege job that auto-builds any
  pull request. It has no access to the target credential.
- `wonderland-caterpillar-prod`, which only builds when the `main` branch
  itself is updated, and does have access to the target credential.

**Baseline Jenkinsfile:**

```groovy
pipeline {
    agent any
    environment {
        PROJECT = "loguru"
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
        stage('deploy') {
            when {
                expression {
                    env.BRANCH_NAME == 'main'
                }
            }
            steps {
                withCredentials([usernamePassword(credentialsId: 'flag2', usernameVariable: 'flag2', passwordVariable: 'TOKEN')]) {
                    sh 'curl -isSL "http://wonderland:1234/api/user" -H "Authorization: Token ${TOKEN}" -H "Content-Type: application/json" || true'
                }
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

**Root cause (preview):** unlike a repository the attacker can push directly
to, `Wonderland/caterpillar` is read-only for the low-privilege account. On
the surface this looks like it should stop a Direct PPE attack outright,
since the `deploy` stage that reaches the target credential is explicitly
gated to only run `when` the branch is `main`, which the attacker cannot
push to. The actual gap is that the automation used to bridge Gitea and
Jenkins (an access token used internally to clone repos and report build
status back to Gitea) is exposed inside every build's environment
variables, and that token turns out to hold permissions the human account
does not, specifically the ability to merge a pull request into `main`.
