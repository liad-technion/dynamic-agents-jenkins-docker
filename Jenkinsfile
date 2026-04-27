pipeline {
    agent none

    stages {
        stage('Parallel dynamic agents') {
            parallel {
                stage('Build (Python)') {
                    agent {
                        docker {
                            image 'python-java:3.11-slim@sha256:9314202733ad6509beb5793094fa22beed6d333c8c92d9b7de0d664231a2af95'
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
                            image 'node-java:20-slim@sha256:306016f4ed677cf741df197f382f80cc659a77e5efa0eff91bee00dddf87df85'
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