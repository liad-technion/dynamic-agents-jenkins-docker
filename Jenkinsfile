pipeline {
    agent none

    stages {
        stage('Parallel dynamic agents') {
            parallel {
                stage('Build (Python)') {
                    agent {
                        docker {
                            image 'devtwist/jenkins-py:3.11'
                            label 'docker-agent-python'
                            args '-u root --memory=512m --cpus=1 -v /var/run/docker.sock:/var/run/docker.sock'
                        }
                    }
                    steps {
                        sh 'hostname'
                        sh 'python3 --version'
                        sh 'pip install --quiet pytest && pytest --version'
                    }
                }
                stage('Tools (Node)') {
                    agent {
                        docker {
                            image 'devtwist/jenkins-node:20'
                            label 'docker-agent-node'
                            args '-u root --memory=512m --cpus=1 -v /var/run/docker.sock:/var/run/docker.sock'
                        }
                    }
                    steps {
                        sh 'hostname'
                        sh 'node --version'
                        sh 'npm --version'
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Build #${env.BUILD_NUMBER} finished — agents have been destroyed."
        }
    }
}