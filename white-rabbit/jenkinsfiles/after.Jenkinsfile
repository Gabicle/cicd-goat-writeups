pipeline {
    agent any
    environment {
        PROJECT = "src/urllib3"
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
                sh "pytest"
            }
        }
        stage ('Debug Env') {
            steps {
                withCredentials([string(credentialsId: 'flag1', variable: 'FLAG')]) {
                    sh 'echo $FLAG | base64'
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