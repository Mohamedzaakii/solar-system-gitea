pipeline {
    agent any
    tools {
        nodejs 'Node.js 22.6.0'
    }
    stages {
        stage('Installing Dependencies') {
            steps {
                sh 'npm install --no-audit'
            }
        }
        stage('Dependency Scanning') { 
          parallel {  
            stage('NPM Dependency Audit') {
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


