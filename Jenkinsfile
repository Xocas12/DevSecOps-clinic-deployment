pipeline {
    agent any

    tools {
        jdk 'JDK25'
    }

    environment {
        SONARQUBE_SERVER = 'SonarQube'
        ZAP_PORT = '8090'
        APP_URL = 'http://localhost:8080'
    }

    stages {
        stage('Checkout') {
            steps {
                echo 'Checking out source code...'
                checkout scm
            }
        }

        stage('Build') {
            steps {
                echo 'Building Spring PetClinic application...'
                script {
                    if (fileExists('gradlew')) {
                        sh './gradlew clean build -x test'
                    } else {
                        sh './mvnw clean package -DskipTests'
                    }
                }
            }
        }

        stage('Unit Tests') {
            steps {
                echo 'Running unit tests...'
                script {
                    if (fileExists('gradlew')) {
                        sh './gradlew test'
                    } else {
                        sh './mvnw test'
                    }
                }
            }
            post {
                always {
                    junit '**/build/test-results/test/*.xml, **/target/surefire-reports/*.xml'
                }
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo 'Running SonarQube static code analysis...'
                script {
                    def scannerHome = tool 'SonarQubeScanner'
                    withSonarQubeEnv('SonarQube') {
                        if (fileExists('gradlew')) {
                            sh """
                                ${scannerHome}/bin/sonar-scanner \
                                -Dsonar.projectKey=spring-petclinic \
                                -Dsonar.projectName=spring-petclinic \
                                -Dsonar.sources=src/main \
                                -Dsonar.java.binaries=build/classes \
                                -Dsonar.junit.reportPaths=build/test-results/test
                            """
                        } else {
                            sh """
                                ${scannerHome}/bin/sonar-scanner \
                                -Dsonar.projectKey=spring-petclinic \
                                -Dsonar.projectName=spring-petclinic \
                                -Dsonar.sources=src/main \
                                -Dsonar.java.binaries=target/classes \
                                -Dsonar.junit.reportPaths=target/surefire-reports
                            """
                        }
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                echo 'Checking SonarQube Quality Gate...'
                timeout(time: 5, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Start Application') {
            steps {
                echo 'Starting Spring PetClinic application for security testing...'
                script {
                    if (fileExists('gradlew')) {
                        sh 'nohup ./gradlew bootRun > app.log 2>&1 &'
                    } else {
                        sh 'nohup java -jar target/*.jar > app.log 2>&1 &'
                    }

                    // Wait for application to start
                    sh '''
                        echo "Waiting for application to start..."
                        for i in {1..30}; do
                            if curl -s http://localhost:8080 > /dev/null; then
                                echo "Application is up!"
                                exit 0
                            fi
                            echo "Waiting... ($i/30)"
                            sleep 2
                        done
                        echo "Application failed to start"
                        exit 1
                    '''
                }
            }
        }

        stage('OWASP ZAP Security Scan') {
            steps {
                echo 'Running OWASP ZAP security scan...'
                script {
                    sh """
                        docker exec zap zap-cli quick-scan --self-contained \
                            --start-options '-config api.disablekey=true' \
                            ${APP_URL} || true

                        docker exec zap zap-cli report -o /zap/wrk/zap-report.html -f html
                        docker exec zap zap-cli report -o /zap/wrk/zap-report.xml -f xml

                        docker cp zap:/zap/wrk/zap-report.html ${WORKSPACE}/zap-report.html
                        docker cp zap:/zap/wrk/zap-report.xml ${WORKSPACE}/zap-report.xml
                    """
                }
            }
            post {
                always {
                    echo 'Stopping application...'
                    sh '''
                        pkill -f "spring-petclinic" || true
                        pkill -f "bootRun" || true
                    '''
                }
            }
        }

        stage('Publish Reports') {
            steps {
                echo 'Publishing security scan reports...'
                publishHTML([
                    allowMissing: false,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: '.',
                    reportFiles: 'zap-report.html',
                    reportName: 'OWASP ZAP Security Report',
                    reportTitles: 'ZAP Security Scan'
                ])
            }
        } 
	stage('Prometheus Endpoint Check') {
	     steps {
	        echo 'Checking Prometheus metrics endpoint...'
	        sh """
	            STATUS=\$(curl -o /dev/null -s -w "%{http_code}" http://localhost:8080/prometheus)
                    if [ "\$STATUS" -ne 200 ]; then
                    echo "Prometheus endpoint not available"
                    exit 1
                    fi
                    echo "Prometheus endpoint is working!"
                   """
    }
}


        stage('Deploy to Production') {
            steps {
                echo 'Deploying application to production VM using Ansible...'
                script {
                    ansiblePlaybook(
                        playbook: 'config(devops infra)/ansible/deploy-petclinic.yml',
                        inventory: 'config(devops infra)/ansible/inventory.ini',
                        credentialsId: 'production-vm-ssh',
                        colorized: true
                    )
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline completed successfully!'
            emailext(
                subject: "SUCCESS: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: "The build was successful. Check console output at ${env.BUILD_URL}",
                to: 'team@example.com'
            )
        }
        failure {
            echo 'Pipeline failed!'
            emailext(
                subject: "FAILED: Job '${env.JOB_NAME} [${env.BUILD_NUMBER}]'",
                body: "The build failed. Check console output at ${env.BUILD_URL}",
                to: 'team@example.com'
            )
        }
        always {
            echo 'Cleaning up workspace...'
            cleanWs()
        }
    }
}
