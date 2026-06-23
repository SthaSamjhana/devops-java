pipeline {
    agent any

    tools {
        jdk 'jdk-17'
        gradle 'gradle-814'
    }

    environment {
        APP_NAME   = 'calculator'
        JAR_NAME   = 'calculator-1.0.0.jar'
        APP_SERVER = '3.87.91.135'
    }

    stages {

        stage('Build') {
            steps {
                echo 'Building the application...'
                sh 'gradle clean compileJava'
            }
        }

        stage('Test') {
            steps {
                echo 'Running unit tests...'
                sh 'gradle test'
            }
            post {
                always {
                    junit 'build/test-results/test/*.xml'
                }
            }
        }

        stage('Security & Code Analysis') {
            parallel {

                stage('SonarQube Analysis') {
                    steps {
                        echo 'Analyzing code quality...'
                    }
                }

            }
        }

        stage('SonarQube Quality Gate') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    script {
                        echo 'Quality Gate Check'
                    }
                }
            }
        }

        stage('Package') {
            steps {
                echo 'Packaging application...'
                sh 'gradle bootJar'
                sh 'ls -l build/libs/'
            }
            post {
                success {
                    archiveArtifacts artifacts: 'build/libs/*.jar', fingerprint: true
                }
            }
        }

        stage('Approval') {
            options {
                timeout(time: 3, unit: 'MINUTES')
            }
            steps {
                input message: 'Approve deployment to Production?', ok: 'Deploy'
            }
        }

        stage('Deploy') {
            steps {
                script {

                    withCredentials([
                        sshUserPrivateKey(
                            credentialsId: 'app-server-ssh',
                            keyFileVariable: 'SSH_KEY'
                        )
                    ]) {

                        sh """
                            echo "Copying JAR to server..."

                            scp -i \$SSH_KEY \
                            -o StrictHostKeyChecking=no \
                            build/libs/${JAR_NAME} \
                            ubuntu@${APP_SERVER}:~/calculator.jar.new

                            echo "Restarting application..."

                            ssh -i \$SSH_KEY \
                            -o StrictHostKeyChecking=no \
                            ubuntu@${APP_SERVER} << 'EOF'

                            if [ -f ~/calculator.jar ]; then
                                cp ~/calculator.jar ~/calculator.jar.bak
                            fi

                            mv ~/calculator.jar.new ~/calculator.jar

                            sudo systemctl restart calculator.service

                            echo "Deployment completed"

                            echo "Service Status:"
                            sudo systemctl status calculator.service --no-pager -l

EOF
                        """
                    }
                }
            }
        }
    }

    post {

        always {
            echo 'Pipeline finished execution.'
        }

        success {
            echo 'Build, Test, Package and Deploy completed successfully!'
        }

        failure {
            echo 'Pipeline failed. Check Jenkins logs for details.'
        }
    }
}
