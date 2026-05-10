pipeline {
    agent any
    tools {
        nodejs 'Node.js 22.6.0'
    }
    environment {
       MONGO_URI = "mongodb+srv://supercluster.d83jj.mongodb.net/superData"
       MONGO_DB_CREDS = credentials('mongo-db-credentials')

    }
    stages {
        stage('Unit Testing') {
            options { retry(2) }
            steps {
                sh 'eco $MONGO_DB_CREDS'
                sh 'echo $MONGO_DB_CREDS_USR'
                sh 'echo $MONGO_DB_Creds_PSW'
                sh 'npm test'
            

                junit allowEmptyResults: true, stdioRetention: '', testResults: 'test-results.xml'
            

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
                  ''', odcInstallation: 'owasp-dbcheck-10'
                  dependencyCheckPublisher failedTotalCritical: 1, pattern: 'dependency-check-report.xml', stopBuild: true

                  publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, icon: '', keepAll: true, reportDir: './', reportFiles: 'dependency-check-jenkins.html', reportName: 'HTML Report', reportTitles: '', useWrapperFileDirectly: true]) 


                  junit allowEmptyResults: true, keepProperties: true, testResults: 'dependency-check-junit.xml'


              }  
            }

          } 
        }
    }
}


