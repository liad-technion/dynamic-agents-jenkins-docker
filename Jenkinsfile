pipeline {
    agent none

    stages {
        stage('Parallel dynamic agents') {
            parallel {
                stage('Build (Python)') {
                    agent {
                        docker {
                            image 'python-java:3.11-slim@sha256:dd3cac69b6b968343ba7883b37d212f23ad7dec87f27fef9436cb0b123d4b964'
                            label 'docker-agent-python'
                            args  '-u root --memory=512m --cpus=1'
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
                            image 'node-java:20-slim@sha256:d3fa6b57ce61c94302827c57dfcd439745596a8e15b9c88cb9bb0b7ea25a5408'
                            label 'docker-agent-node'
                            args  '-u root --memory=512m --cpus=1'
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