pipeline {
    agent any
    tools {
        nodejs 'Node.js 22.6.0'
    }
    environment {
       MONGO_URI = "mongodb+srv://supercluster.d83jj.mongodb.net/superData"
       MONGO_DB_CREDS = credentials('mongo-db-credentials')
       SONAR_SCANNER_HOME = tool name: 'sonar-scanner', type: 'hudson.plugins.sonar.SonarRunnerInstallation'
       SONAR_TOKEN = credentials('solar-system-token')
       SONAR_SCANNER_OPTS = "-Xmx4096m"
    }
    stages {
        stage('Git Checkout') {
         steps {
           checkout scm
         }
        } 
        stage('Unit Testing') {
            options { retry(2) }
            steps {
                sh 'echo $MONGO_DB_CREDS'
                sh 'echo $MONGO_DB_CREDS_USR'
                sh 'echo $MONGO_DB_CREDS_PSW'
                sh 'npm install'
                sh 'npm test'
                sh 'npm run coverage'
                sh 'ls -la coverage'
                sh 'test -f coverage/lcov.info'

                junit allowEmptyResults: true, testResults: 'test-results.xml'
            

                publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true, reportDir: 'coverage/lcov-report', reportFiles: 'index.html', reportName: 'Code Coverage HTML Report', reportTitles: '', useWrapperFileDirectly: true])                

            }

        }
        
        stage('Dependency Scanning') { 
          parallel {  
            stage('NPM Dependency Audit') { 
              options { timestamps() } 
              steps {
                  sh '''
                     npm audit --audit-level=critical
                     echo $?
                  ''' 
              }
            }

           
            stage('OWASP Dependency Check') {
              steps {
                  dependencyCheck additionalArguments: '''
                    --scan './'
                    --out './'
                    --format HTML XML JUNIT JENKINS
                    --prettyPrint 
                    --disableYarnAudit
                  ''', odcInstallation: 'owasp-dbcheck-10'
                  dependencyCheckPublisher failedTotalCritical: 1, pattern: 'dependency-check-report.xml', stopBuild: true

                  publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, icon: '', keepAll: true, reportDir: './', reportFiles: 'dependency-check-jenkins.html', reportName: 'HTML Report', reportTitles: '', useWrapperFileDirectly: true]) 


                  junit allowEmptyResults: true, keepProperties: true, testResults: 'dependency-check-junit.xml'


              }  
            }
            
            stage('SAST-SonarQube') {
              steps {
                timeout(time: 15, unit: 'MINUTES') {
                 withSonarQubeEnv('SonarQube') {

                   sh "echo $SONAR_SCANNER_HOME"
                   sh '''
                      $SONAR_SCANNER_HOME/bin/sonar-scanner \
                      -Dsonar.projectKey=Solar-System-Project \
                      -Dsonar.sources=. \
                      -Dsonar.host.url=http://localhost:9000 \
                      -Dsonar.javascript.lcov.reportPaths=./coverage/lcov.info \
                      -Dsonar.token=$SONAR_TOKEN 
                   
                   '''
                 }
                }
                 //waitForQualityGate abortPipeline: true
              }
            } 
         }
       }
       
       stage('Build Docker Image') {
         steps {
           sh'printenv'
           sh 'docker build -t m0hamedzaki/solar-system:$GIT_COMMIT .'
         }
       }





    }
}
 



