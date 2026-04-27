#!/usr/bin/env groovy

pipeline {
    agent any 

    environment {
        FEATURE_NAME = "master"
        REGISTRY_PASSWORD = "dummy"
        REGISTRY_USERNAME = "dummy"
        POSTGRES_PASSWORD = "dummy"
        APP_NAME = "cicd-demo"
    }

    stages {

        stage('Docker Build & Push') {
            steps {
                sh 'sed -i "s/\\r//" mvnw'
                sh 'chmod +x mvnw'
                sh './mvnw clean package -DskipTests'
                sh 'docker build -t cicd-demo:latest .'
                sh 'docker tag cicd-demo:latest cicd-demo:${BUILD_NUMBER}'
                echo "Docker build completado"
            }
        }

        // 🔍 Escaneo de vulnerabilidades (Trivy)
        stage('Docker Scan') {
            steps {
                sh '''
                    curl -sfL https://raw.githubusercontent.com/aquasecurity/trivy/main/contrib/install.sh | sh -s -- -b /usr/local/bin
                    trivy image --timeout 10m --exit-code 0 --severity HIGH,CRITICAL cicd-demo:latest || true
                '''
            }
            post {
                always {
                    sh "docker-compose down -v || true"
                }
            }
        }

        // 🚫 Gate de seguridad (Trivy)
        stage('Security Gate') {
            steps {
                sh 'trivy image --exit-code 1 --severity CRITICAL cicd-demo:latest'
            }
        }

        // ⚙️ Tests en paralelo
        stage('Parallel Tests') {
            failFast true            
            parallel {                  

                // 🧠 SonarQube
                stage('Static Code Analysis') {
                    when {
                        anyOf { branch 'master'; branch 'release' }
                    }    
                    steps {
                        withSonarQubeEnv('SonarQube') {
                            sh "mvn sonar:sonar -Dsonar.projectKey=cicd-demo"
                        }
                    }
                }

                stage('Integration Tests') {
                    steps {
                        sh "make integrationTest"
                    }
                }
            }
        }

        // 🚫 Gate de calidad (SonarQube)
        stage('Quality Gate') {
            when {
                anyOf { branch 'master'; branch 'release' }
            }
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Push Latest Tag') {
            when { branch 'master' }
            steps {
                sh "make dockerPushLatest"
            }
        }

        stage('Deploy To dev') {
            environment { 
                ENV = "dev"
                KUBE_SERVER = credentials("KUBE_API_SERVER")
                KUBE_TOKEN = credentials("KUBE_DEV_TOKEN")
            }
            steps {
                sh "make kubeLogin deploy"
            }
        }

        stage('Deploy To qa') {
            when { expression { BRANCH_NAME ==~ /(master|release-[0-9]+$)/ }}
            environment {
                ENV = "qa"
                KUBE_SERVER = credentials("KUBE_API_SERVER")
                KUBE_TOKEN = credentials("KUBE_QA_TOKEN")
            }
            steps {
                sh "make kubeLogin deploy"
                sh 'docker system prune -f || true'
                sh 'docker rm -f cicd-demo || true'
            }
        }
    }

    post {
        always {
            echo 'Limpiando entorno...'
        }
        failure {
            echo '❌ El pipeline falló - revisar logs arriba'
        }
        success {
            echo '✅ Pipeline completado exitosamente'
        }
    }
}