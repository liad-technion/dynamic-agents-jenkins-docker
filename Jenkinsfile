pipeline {
    agent none

    stages {
        stage('Parallel dynamic agents') {
            parallel {
                stage('Build (Python)') {
                    agent { label 'docker-python-agent' }
                        //docker {
                        //    image 'devtwist/jenkins-py:3.11'
                        //    label 'docker-python-agent'
                        //    args '-u root --memory=512m --cpus=1 -v /var/run/docker.sock:/var/run/docker.sock'
                        //}
                    steps {
                        sh 'hostname'
                        sh 'python3 --version'
                        sh 'pip install --quiet pytest && pytest --version'
                    }
                }
                stage('Tools (Node)') {
                    agent { label 'docker-node-agent' }
                        //docker {
                        //    image 'devtwist/jenkins-node:20'
                        //    label 'docker-node-agent'
                        //    args '-u root --memory=512m --cpus=1 -v /var/run/docker.sock:/var/run/docker.sock'
                        //}
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