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
       CHECKS_API_SKIP = "true"
    }
    stages {
        stage('Git Checkout') {
         steps {
           checkout scm
         }
        } 
        stage('Unit Testing') {
            
            steps {                
                sh 'npm install'
                withCredentials([usernamePassword(
                  credentialsId: 'mongo-db-credentials',
                  usernameVariable: 'MONGO_USERNAME',
                  passwordVariable: 'MONGO_PASSWORD'
               )]) {
                   sh '''
                      export MONGO_URI="mongodb+srv://supercluster.d83jj.mongodb.net/superData"
                      npm test
                  '''
              }
                
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
                  sh 'mkdir -p /var/lib/jenkins/owasp-data'               
                  dependencyCheck additionalArguments: '''
                    --scan './'
                    --out './'
                    --format HTML XML JUNIT JENKINS
                    --prettyPrint 
                    --disableYarnAudit
                    --disableRubyGemsAnalyzer
                    --disableNugetConfAnalyzer
                    --disableCentralAnalyzer
                    --disableExperimental
                    --data /var/lib/jenkins/owasp-data
                  ''', odcInstallation: 'owasp-dbcheck-10'
                  dependencyCheckPublisher failedTotalCritical: 1, pattern: 'dependency-check-report.xml', stopBuild: true

                  publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, icon: '', keepAll: true, reportDir: './', reportFiles: 'dependency-check-jenkins.html', reportName: 'HTML Report', reportTitles: '', useWrapperFileDirectly: true]) 


                  junit allowEmptyResults: true, keepProperties: true, testResults: 'dependency-check-junit.xml'


              }  
            }
            
            stage('SAST - SonarQube') {
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
       stage('Trivy vulnerability scanning') {
         steps {
           
           sh 'trivy image --severity LOW,MEDIUM --exit-code 0 --quiet --format json --output trivy-medium.json m0hamedzaki/solar-system:$GIT_COMMIT'
           sh 'trivy image --severity HIGH --exit-code 0 --quiet --format json --output trivy-high.json m0hamedzaki/solar-system:$GIT_COMMIT'
           sh 'trivy image --severity CRITICAL --exit-code 1 --quiet --format json --output trivy-critical.json m0hamedzaki/solar-system:$GIT_COMMIT'
         }
         post {
          always {
            sh '''
                
                mkdir -p $HOME/.trivy/templates/
                
                # Download HTML template if missing
                if [ ! -f "$HOME/.trivy/templates/html.tpl" ]; then
                    curl -sSL -o $HOME/.trivy/templates/html.tpl \
                        https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/html.tpl
                fi
                
                # Download JUnit template if missing
                if [ ! -f "$HOME/.trivy/templates/junit.tpl" ]; then
                    curl -sSL -o $HOME/.trivy/templates/junit.tpl \
                        https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/junit.tpl
                fi
                
                # Now convert the JSON reports
                for severity in medium high critical; do
                    if [ -f "trivy-${severity}.json" ]; then
                        trivy convert --format template \
                            --template "@$HOME/.trivy/templates/html.tpl" \
                            --output "trivy-${severity}.html" \
                            "trivy-${severity}.json" || echo "Failed to convert ${severity} to HTML"
                        
                        trivy convert --format template \
                            --template "@$HOME/.trivy/templates/junit.tpl" \
                            --output "trivy-${severity}.xml" \
                            "trivy-${severity}.json" || echo "Failed to convert ${severity} to JUnit"
                    fi
                done
            '''
            
            publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true, reportDir: '.', reportFiles: 'trivy-critical.html', reportName: 'Trivy - CRITICAL Vulnerabilities'])
            publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true, reportDir: '.', reportFiles: 'trivy-high.html', reportName: 'Trivy - HIGH Vulnerabilities'])
            publishHTML([allowMissing: true, alwaysLinkToLastBuild: true, keepAll: true, reportDir: '.', reportFiles: 'trivy-medium.html', reportName: 'Trivy - MEDIUM/LOW Vulnerabilities'])
            junit allowEmptyResults: true, keepLongStdio: true, testResults: 'trivy-*.xml'
         }
        }
      }

      stage('Push Docker Image') {
        steps {
          withDockerRegistry(credentialsId: 'docker-hub', url: "") {
          sh 'docker push m0hamedzaki/solar-system:$GIT_COMMIT'
          }
        }
      }

      stage('Deploy to EC2') {
        steps {
          script {
           sshagent(['ec2-ssh']) {
            sh """
                ssh -o StrictHostKeyChecking=no ubuntu@54.242.124.114 "
                if sudo docker ps -a | grep -q 'solar-system'; then
                   echo "Container found. Stopping..."
                   sudo docker stop "solar-system" && sudo docker rm "solar-system"
                   echo "Container stopped and removed"
                fi
                
                   sudo docker run --name solar-system \\
                     -e MONGO_URI="$MONGO_URI" \\
                     -e MONGO_USERNAME="$MONGO_DB_CREDS_USR" \\
                     -e MONGO_PASSWORD="$MONGO_DB_CREDS_PSW" \\
                     -p 3000:3000 -d m0hamedzaki/solar-system:$GIT_COMMIT
                "
            """  
           } 
          }
        }
      }
      stage('Integration Testing - AWS EC2') {
        steps {
          withAWS(credentials: 'AWS-S3-EC2-Lambda', region: 'us-east-1') {
            sh ''' 
                bash integration-testing-ec2.sh
            '''
          }
        }
      }

    }
}
 



