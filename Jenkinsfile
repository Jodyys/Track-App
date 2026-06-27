pipeline {
    agent any

    environment {
        DOCKER_USER     = "jodyys"
        IMAGE_BACKEND   = "bookapp-backend"
        IMAGE_FRONTEND  = "bookapp-frontend"
    }

    stages {
        stage('Checkout') {
            steps {
                git(
                    branch: 'main',
                    credentialsId: 'github-creds',
                    url: 'https://github.com/Jodyys/Track-App.git'
                )
            }
        }

        stage('Lint') {
            steps {
                sh '''
                echo "=== Linting Frontend (Node.js) ==="
                cd frontend && npm install && npm run lint

                echo "=== Linting Backend (Python) ==="
                cd ../backend
                python3 -m pip install --user flake8 --break-system-packages || pip install flake8 --break-system-packages
                export PATH="$HOME/.local/bin:$PATH"
                flake8 . --count --select=E9,F63,F7,F82 --show-source --statistics
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                echo "=== Testing Backend (Python) ==="
                cd backend
                python3 -m pip install --user --break-system-packages -r requirements.txt pytest
                export PATH="$HOME/.local/bin:$PATH"

                if find . -name "test_*.py" -o -name "*_test.py" | grep -q .; then
                    pytest
                else
                    echo "No Python tests found. Skipping pytest."
                fi
                '''
            }
        }

        stage('Security & Quality') {
            parallel {
                stage('SonarQube Static Analysis') {
                    stages {
                        stage('SAST - SonarQube Scan') {
                            steps {
                                withSonarQubeEnv('SonarQube-TrackApp') { 
                                    script { 
                                        def scannerHome = tool 'SonarScanner' 
                                        sh "${scannerHome}/bin/sonar-scanner \
                                        -Dsonar.projectKey=book-app \
                                        -Dsonar.sources=. \
                                        -Dsonar.host.url=\$SONAR_HOST_URL \
                                        -Dsonar.token=\$SONAR_AUTH_TOKEN"
                                    }
                                }
                            }
                        }
                        stage('SonarQube Gate') {
                            steps {
                                timeout(time: 5, unit: 'MINUTES') {
                                    waitForQualityGate abortPipeline: true
                                }
                            }
                        }
                    }
                }

                // Jalur 2: Trivy FS Scan berjalan mandiri secara paralel di sebelahnya
                stage('Trivy FS Scan') {
                    steps {
                        sh '''
                        echo "=== SCA Scan Backend & Frontend Dependencies ==="
                        trivy fs --severity HIGH,CRITICAL backend/
                        trivy fs --severity HIGH,CRITICAL frontend/
                        '''
                    }
                }
            }
        }

        stage('Build Image') {
            steps {
                sh """
                # Build Docker Image Backend & Frontend
                docker build -t ${DOCKER_USER}/${IMAGE_BACKEND}:v${BUILD_NUMBER} -t ${DOCKER_USER}/${IMAGE_BACKEND}:latest backend
                docker build -t ${DOCKER_USER}/${IMAGE_FRONTEND}:v${BUILD_NUMBER} -t ${DOCKER_USER}/${IMAGE_FRONTEND}:latest frontend
                """
            }
        }

        stage('Security Scan Trivy') {
            steps {
                sh """
                trivy image --severity HIGH,CRITICAL ${DOCKER_USER}/${IMAGE_BACKEND}:v${BUILD_NUMBER}
                trivy image --severity HIGH,CRITICAL ${DOCKER_USER}/${IMAGE_FRONTEND}:v${BUILD_NUMBER}
                """
            }
        }

        stage('Push Images To DockerHub') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub-creds',
                        usernameVariable: 'DOCKERHUB_USER',
                        passwordVariable: 'DOCKERHUB_PASS'
                    )
                ]) {
                    sh """
                    echo \$DOCKERHUB_PASS | docker login -u \$DOCKERHUB_USER --password-stdin

                    # Push Backend & Frontend
                    docker push ${DOCKER_USER}/${IMAGE_BACKEND}:v${BUILD_NUMBER}
                    docker push ${DOCKER_USER}/${IMAGE_BACKEND}:latest
                    docker push ${DOCKER_USER}/${IMAGE_FRONTEND}:v${BUILD_NUMBER}
                    docker push ${DOCKER_USER}/${IMAGE_FRONTEND}:latest

                    docker logout
                    """
                }
            }
        }

        stage('Deploy K3s - EC2') {
            steps {
                withCredentials([
                    file(credentialsId: 'k3s-kubeconfig', variable: 'KUBECONFIG')
                ]) {
                    sh """
                    # Verifikasi Koneksi K3s Cluster
                    kubectl get nodes

                    # 1. Terapkan manifes dasar dari folder k8s/
                    kubectl apply -f k8s/

                    # 2. Update image secara dinamis menggunakan variabel Jenkins
                    kubectl set image deployment/backend backend=${DOCKER_USER}/${IMAGE_BACKEND}:v${BUILD_NUMBER}
                    kubectl set image deployment/frontend frontend=${DOCKER_USER}/${IMAGE_FRONTEND}:v${BUILD_NUMBER}

                    # 3. Verifikasi status deployment
                    kubectl rollout status deployment/backend
                    kubectl rollout status deployment/frontend
                    """
                }
            }
        }
    }

    post {
        success {
            echo 'Pipeline Success'
        }
        failure {
            echo 'Pipeline Failed'
        }
        always {
            sh 'docker image prune -af || true'
        }
    }
}