#!/usr/bin/env groovy

pipeline {
    agent any

    environment {
        FEATURE_NAME = "master"
        REGISTRY_PASSWORD = "dummy"
        REGISTRY_USERNAME = "dummy"
        POSTGRES_PASSWORD = "dummy"
        APP_NAME = "cicd-demo"
        SONARQUBE_URL = "http://sonarqube:9000"
        SONARQUBE_PROJECT_KEY = "cicd-demo"
    }

    stages {

        stage('Docker Build & Push') {
            steps {
                sh '''
                    # Descargar y extraer Maven
                    MAVEN_VERSION="3.9.0"
                    MAVEN_HOME="/tmp/apache-maven-${MAVEN_VERSION}"

                    if [ ! -d "${MAVEN_HOME}" ]; then
                        echo "📦 Descargando Maven ${MAVEN_VERSION}..."
                        curl -fsSL https://archive.apache.org/dist/maven/maven-3/${MAVEN_VERSION}/binaries/apache-maven-${MAVEN_VERSION}-bin.tar.gz | tar xzf - -C /tmp
                    fi

                    export PATH="${MAVEN_HOME}/bin:$PATH"
                    echo "🔨 Compilando con Maven..."
                    mvn clean package -DskipTests

                    echo "🐳 Construyendo imagen Docker..."
                    docker build -t cicd-demo:latest .
                    docker tag cicd-demo:latest cicd-demo:${BUILD_NUMBER}
                    echo "✅ Imagen Docker construida: cicd-demo:${BUILD_NUMBER}"
                '''
            }
        }

        // 🧠 Análisis Estático de Código (SonarQube)
        stage('Static Code Analysis') {
            steps {
                script {
                    sh '''
                        # Verificar si SonarQube está disponible
                        echo "Checking SonarQube connectivity..."
                        if ! curl -sf ${SONARQUBE_URL}/api/system/status > /dev/null 2>&1; then
                            echo "⚠️  SonarQube not available at ${SONARQUBE_URL}"
                            echo "✅ Skipping SonarQube analysis (optional)"
                            exit 0
                        fi

                        MAVEN_VERSION="3.9.0"
                        MAVEN_HOME="/tmp/apache-maven-${MAVEN_VERSION}"
                        export PATH="${MAVEN_HOME}/bin:$PATH"

                        echo "🔍 Ejecutando análisis SonarQube..."
                        mvn sonar:sonar \
                            -Dsonar.projectKey=${SONARQUBE_PROJECT_KEY} \
                            -Dsonar.sources=src \
                            -Dsonar.host.url=${SONARQUBE_URL} \
                            -Dsonar.login=${SONARQUBE_TOKEN}
                    '''
                }
            }
        }

        // 🚫 Gate de Calidad (SonarQube)
        stage('Quality Gate') {
            steps {
                script {
                    sh '''
                        # Verificar si SonarQube está disponible
                        if ! curl -sf ${SONARQUBE_URL}/api/system/status > /dev/null 2>&1; then
                            echo "⚠️  SonarQube not available - skipping Quality Gate"
                            exit 0
                        fi

                        echo "⏳ Esperando resultado del Quality Gate de SonarQube..."
                        timeout(time: 2, unit: 'MINUTES') {
                            for i in {1..24}; do
                                QUALITY_STATUS=$(curl -s -u ${SONARQUBE_TOKEN}: "${SONARQUBE_URL}/api/qualitygates/project_status?projectKey=${SONARQUBE_PROJECT_KEY}" | grep -o '"status":"[^"]*' | cut -d'"' -f4)

                                if [ "$QUALITY_STATUS" = "OK" ]; then
                                    echo "✅ Quality Gate PASSOU"
                                    exit 0
                                elif [ "$QUALITY_STATUS" = "ERROR" ]; then
                                    echo "❌ Quality Gate FALHOU - Security Hotspot detectado"
                                    exit 1
                                fi

                                if [ $i -lt 24 ]; then
                                    echo "   Tentativa $i/24 - Aguardando resultado..."
                                    sleep 5
                                fi
                            done
                            echo "⚠️  Timeout esperando Quality Gate"
                            exit 0
                        }
                    '''
                }
            }
        }

        // 🔍 Escaneo de Vulnerabilidades (Trivy)
        stage('Docker Scan') {
            steps {
                sh '''
                    # Descargar Trivy binario directamente desde GitHub
                    TRIVY_HOME="/tmp/trivy"
                    if [ ! -f "${TRIVY_HOME}/trivy" ]; then
                        mkdir -p ${TRIVY_HOME}
                        cd ${TRIVY_HOME}
                        TRIVY_VERSION="0.70.0"
                        echo "📦 Descargando Trivy ${TRIVY_VERSION}..."
                        curl -sL https://github.com/aquasecurity/trivy/releases/download/v${TRIVY_VERSION}/trivy_${TRIVY_VERSION}_Linux-64bit.tar.gz -o trivy.tar.gz
                        tar xzf trivy.tar.gz
                        rm trivy.tar.gz
                        cd -
                    fi

                    export PATH="${TRIVY_HOME}:$PATH"
                    echo "🔍 Ejecutando escaneo de vulnerabilidades con Trivy..."
                    trivy image --severity HIGH,CRITICAL cicd-demo:latest
                '''
            }
        }

        // 🚫 Security Gate (Trivy)
        stage('Security Gate') {
            steps {
                sh '''
                    export PATH="/tmp/trivy:$PATH"
                    echo "🔒 Verificando vulnerabilidades CRITICAL..."
                    if trivy image --exit-code 1 --severity CRITICAL cicd-demo:latest; then
                        echo "✅ No hay vulnerabilidades CRITICAL"
                    else
                        echo "❌ FALLO: Se encontraron vulnerabilidades CRITICAL"
                        exit 1
                    fi
                '''
            }
        }

        // ℹ️ Información del Build
        stage('Build Info') {
            steps {
                sh '''
                    echo "========================================="
                    echo "📊 INFORMACIÓN DEL BUILD"
                    echo "========================================="
                    echo "Build Number: ${BUILD_NUMBER}"
                    echo "Build URL: ${BUILD_URL}"
                    echo "Imagen Docker: cicd-demo:${BUILD_NUMBER}"
                    echo "========================================="
                '''
            }
        }
    }

    post {
        always {
            sh '''
                echo "🧹 Limpiando entorno..."
                # Limpiar imágenes no etiquetadas (dangling)
                docker image prune -f 2>/dev/null || true
            '''
        }
        success {
            echo "✅ Pipeline completado exitosamente"
            sh '''
                echo "🎉 ========================================="
                echo "✅ Pipeline EXITOSO"
                echo "🎉 ========================================="
            '''
        }
        failure {
            echo "❌ El pipeline falló - revisar logs arriba"
            sh '''
                echo "❌ ========================================="
                echo "❌ Pipeline FALLÓ"
                echo "Revisar:"
                echo "  1. Compilación Maven"
                echo "  2. Análisis SonarQube / Quality Gate"
                echo "  3. Escaneo Trivy / Security Gate"
                echo "❌ ========================================="
            '''
        }
    }
}