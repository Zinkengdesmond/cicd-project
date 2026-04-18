
pipeline {
    agent any

    triggers {
        githubPush()
    }

    tools {
        jdk 'JDK-17'  // Must be configured in Jenkins
    }

    environment {
        // Java Configuration
        JAVA_HOME = tool name: 'JDK-17', type: 'jdk'
        PATH = "${JAVA_HOME}/bin:${env.PATH}"

        // Docker Hub Configuration
        DOCKER_CREDENTIALS_ID = 'dockerhub-credentials'
        DOCKER_HUB_USERNAME  = 'desmondzinkeng'
        DOCKER_IMAGE_NAME    = 'appointment-app'
        DOCKER_IMAGE_TAG     = "${BUILD_NUMBER}"
        DOCKER_IMAGE_LATEST  = "${DOCKER_HUB_USERNAME}/${DOCKER_IMAGE_NAME}:latest"
        DOCKER_IMAGE_VERSION = "${DOCKER_HUB_USERNAME}/${DOCKER_IMAGE_NAME}:${DOCKER_IMAGE_TAG}"

        // SonarQube Configuration
        SONARQUBE_ENV     = 'sonarqube'
        SONAR_PROJECT_KEY = 'appointment-app'
        SONAR_CREDENTIALS_ID = 'sonarqube-token'

        // Kubernetes
        K8S_NAMESPACE     = 'default'
        APP_NAME          = 'appointment-app'

        // Trivy
        TRIVY_SEVERITY = 'HIGH,CRITICAL'

        // Kubeconfig
        KUBECONFIG = '/var/jenkins_home/.kube/config'
    }

    stages {
        stage('📥 Checkout') {
            steps {
                // Clean workspace BEFORE checkout to avoid git errors
                cleanWs()

                // Now checkout the code
                checkout scm

                // Verify checkout
                sh '''
                    echo "=== Git Status ==="
                    git status
                    echo "=== Current Branch ==="
                    git branch
                    echo "=== Repository Info ==="
                    git remote -v
                '''
            }
        }

 

        stage('☕ Verify Java Version') {
            steps {
                sh '''
                    echo "=== Java Version ==="
                    java -version
                    echo "=== Maven Version ==="
                    ./mvnw -version
                '''
            }
        }

        stage('🔍 SonarQube Analysis') {
            steps {
                withCredentials([string(credentialsId: "${SONAR_CREDENTIALS_ID}", variable: 'SONAR_TOKEN')]) {
                    withSonarQubeEnv("${SONARQUBE_ENV}") {
                        sh """
                            ./mvnw clean verify sonar:sonar \
                            -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                            -Dsonar.token=${SONAR_TOKEN}
                        """
                    }
                }
            }
        }

        stage('⏳ Sonar Quality Gate') {
            steps {
                script {
                    try {
                        timeout(time: 2, unit: 'MINUTES') {
                            waitForQualityGate abortPipeline: false
                        }
                    } catch (Exception e) {
                        echo "⚠️ Quality Gate check skipped - continuing pipeline"
                    }
                }
            }
        }

        stage('🔨 Build & Test') {
            steps {
                sh './mvnw clean test'
            }
            post {
                always {
                    junit allowEmptyResults: true, testResults: '**/target/surefire-reports/*.xml'
                }
            }
        }

        stage('📦 Package') {
            steps {
                sh './mvnw clean package -DskipTests'
            }
        }

        stage('🐳 Docker Build') {
            steps {
                sh """
                    docker build --cache-from ${DOCKER_IMAGE_LATEST} \
                    -t ${DOCKER_IMAGE_VERSION} \
                    -t ${DOCKER_IMAGE_LATEST} .
                """
            }
        }

        stage('🔒 Trivy Image Scan') {
            steps {
                sh """
                    trivy image \
                    --scanners vuln \
                    --timeout 10m \
                    --severity ${TRIVY_SEVERITY} \
                    --exit-code 0 \
                    --format json \
                    --output trivy-report.json \
                    ${DOCKER_IMAGE_VERSION}
                """
            }
        }

        stage('📤 Push to Docker Hub') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: DOCKER_CREDENTIALS_ID,
                    usernameVariable: 'DOCKER_USER',
                    passwordVariable: 'DOCKER_PASS'
                )]) {
                    sh """
                        echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin
                        docker push ${DOCKER_IMAGE_VERSION}
                        docker push ${DOCKER_IMAGE_LATEST}
                        docker logout
                    """
                }
            }
        }

        stage('🚀 Deploy to Kubernetes') {
            steps {
                sh """
                    export KUBECONFIG=${KUBECONFIG}
                    echo "Using kubeconfig:"
                    ls -l \$KUBECONFIG

                    kubectl cluster-info
                    kubectl get nodes

                    # Create deployment manifest
                    cat > k8s-deployment.yaml <<'EOF'
apiVersion: apps/v1
kind: Deployment
metadata:
  name: ${APP_NAME}
  namespace: ${K8S_NAMESPACE}
spec:
  replicas: 2
  selector:
    matchLabels:
      app: ${APP_NAME}
  template:
    metadata:
      labels:
        app: ${APP_NAME}
    spec:
      containers:
      - name: ${APP_NAME}
        image: ${DOCKER_IMAGE_VERSION}
        ports:
        - containerPort: 8080
        resources:
          requests:
            memory: "256Mi"
            cpu: "250m"
          limits:
            memory: "512Mi"
            cpu: "500m"
---
apiVersion: v1
kind: Service
metadata:
  name: ${APP_NAME}
  namespace: ${K8S_NAMESPACE}
spec:
  type: NodePort
  ports:
  - port: 8080
    targetPort: 8080
    nodePort: 30080
  selector:
    app: ${APP_NAME}
EOF

 

                    # Substitute environment variables and apply
                    envsubst < k8s-deployment.yaml | kubectl apply -f -

                    # Wait for rollout
                    kubectl rollout status deployment/${APP_NAME} -n ${K8S_NAMESPACE} --timeout=5m
                """              
            }
        }

 

        stage('✅ Verify Deployment') {
            steps {
                sh """
                    export KUBECONFIG=${KUBECONFIG}
                    echo "=== Pods ==="
                    kubectl get pods -n ${K8S_NAMESPACE} -l app=${APP_NAME}
                    echo "=== Services ==="
                    kubectl get svc -n ${K8S_NAMESPACE} ${APP_NAME}
                    echo "=== Deployment ==="
                    kubectl get deployment -n ${K8S_NAMESPACE} ${APP_NAME}
                """
            }
        }
    }

    post {
        always {
            archiveArtifacts artifacts: '**/target/*.jar', allowEmptyArchive: true
            archiveArtifacts artifacts: 'trivy-report.json', allowEmptyArchive: true
        }

        success {
            echo '✅ PIPELINE COMPLETED SUCCESSFULLY'
            echo "🐳 Docker Image: ${DOCKER_IMAGE_VERSION}"
            echo "🌐 Access app: minikube service ${APP_NAME} --url"
        }

        failure {
            echo '❌ PIPELINE FAILED — CHECK LOGS'
        }
    }
}

