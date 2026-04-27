#!/usr/bin/env groovy

pipeline {
    agent any 

    environment{
       FEATURE_NAME = BRANCH_NAME.replaceAll('[\\(\\)_/]','-').toLowerCase()
       REGISTRY_PASSWORD = credentials('REGISTRY_PASSWORD')
       REGISTRY_USERNAME = credentials('REGISTRY_USERNAME')
       POSTGRES_PASSWORD = credentials('POSTGRES_PASSWORD')
       APP_NAME = "cicd-demo"
    }

    stages {

        stage('Docker Build & Push') {
            steps {
                sh "make dockerLogin build dockerBuild dockerPush"
            }
        }

        // 🔍 Escaneo de vulnerabilidades (Trivy)
        stage('Docker Scan') {
            steps {
                sh "make dockerScan"
            }
            post {
                cleanup {
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

                // 🧠 SonarQube (CORREGIDO)
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
                APP_DNS = util.selectAppUrl(ENV, FEATURE_NAME, APP_NAME)
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
                APP_DNS = util.selectAppUrl(ENV, FEATURE_NAME, APP_NAME)
                KUBE_SERVER = credentials("KUBE_API_SERVER")
                KUBE_TOKEN = credentials("KUBE_QA_TOKEN")
            }
            steps {
                sh "make kubeLogin deploy"
            }
        }
    }

    post {
        always {
            echo 'Limpiando entorno...'
            sh 'docker system prune -f || true'
            cleanWs()

            script {
                if(BRANCH_NAME ==~ /(master|release-[0-9]+$)/ ){
                    util.notifySlack(currentBuild.result)
                }
            }
        }

        failure {
            echo '❌ El pipeline falló'
        }

        success {
            echo '✅ Pipeline exitoso'
        }

        archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
        junit 'target/surefire-reports/*.xml'
    }
}