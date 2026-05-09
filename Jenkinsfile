pipeline {
    agent any
    tools {
        nodejs 'Node.js 22.6.0'
    }
    stages {
        stage('Installing Dependency') {
            steps {
                sh 'npm install --no-audit'
            }
        }
    }
}


