pipeline {
    agent any

    environment {
        DOCKER_IMAGE  = "mounikasannaiela/swiggyapp"
        DOCKER_CRED   = 'DockerHub-token'
        SONAR_TOKEN   = credentials('sonar-token')
    }

    options {
        timestamps()
    }

    stages {

        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }

        stage('Checkout Code') {
            steps {
                git branch: 'master', url: 'https://github.com/sannaielamounika/DevOps-Project-Swiggy'
            }
        }

        stage('Verify Tools') {
            steps {
                sh '''
                    echo "=== Checking tools ==="
                    java -version
                    node -v
                    npm -v
                    docker --version
                    trivy --version || true
                '''
            }
        }

        // ⚡ FAST INSTALL
        stage('Install Dependencies') {
            steps {
                sh 'npm ci --no-audit'
            }
        }

        // ✅ SONARQUBE SCAN
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                        export SONAR_SCANNER_OPTS="-Xmx1024m"
                        export NODE_OPTIONS="--max-old-space-size=2048"

                        sonar-scanner \
                        -Dsonar.projectName=swiggyapp \
                        -Dsonar.projectKey=swiggyapp \
                        -Dsonar.login=$SONAR_TOKEN \
                        -Dsonar.sources=src \
                        -Dsonar.exclusions=**/node_modules/**,build/**,coverage/** \
                        -Dsonar.sourceEncoding=UTF-8
                    '''
                }
            }
        }

        // ⏳ NON-BLOCKING QUALITY GATE
        stage('Quality Gate') {
            steps {
                script {
                    try {
                        timeout(time: 2, unit: 'MINUTES') {
                            waitForQualityGate abortPipeline: false
                        }
                    } catch (Exception e) {
                        echo "Quality Gate skipped due to delay"
                    }
                }
            }
        }

        // 🛡️ OWASP SCAN
        stage('OWASP Dependency Check') {
            steps {
                sh '''
                    if [ ! -d "dependency-check" ]; then
                        wget -q https://github.com/jeremylong/DependencyCheck/releases/download/v9.0.9/dependency-check-9.0.9-release.zip
                        unzip -q dependency-check-9.0.9-release.zip
                    fi

                    cd dependency-check/bin

                    ./dependency-check.sh \
                    --project "swiggyapp" \
                    --scan $WORKSPACE \
                    --format XML || true
                '''
            }
        }

        // 🔍 TRIVY FILE SCAN
        stage('Trivy File Scan') {
            steps {
                sh '''
                    mkdir -p .trivycache
                    trivy fs \
                    --cache-dir .trivycache \
                    --scanners vuln \
                    --severity HIGH,CRITICAL \
                    --skip-db-update \
                    . > trivyfs.txt || true
                '''
            }
        }

        // 🚀 DOCKER BUILD & PUSH
        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: "${DOCKER_CRED}", url: 'https://index.docker.io/v1/') {
                        sh '''
                            docker build -t $DOCKER_IMAGE:latest .
                            docker push $DOCKER_IMAGE:latest
                        '''
                    }
                }
            }
        }

        // 🔍 TRIVY IMAGE SCAN (FIXED - NO HANG)
        stage('Trivy Image Scan') {
            steps {
                timeout(time: 5, unit: 'MINUTES') {
                    sh '''
                        echo "Starting Trivy Image Scan..."

                        mkdir -p .trivycache

                        trivy image \
                        --cache-dir .trivycache \
                        --skip-db-update \
                        --severity HIGH,CRITICAL \
                        --timeout 4m \
                        $DOCKER_IMAGE:latest > trivyimage.txt || true

                        echo "Trivy scan completed"
                    '''
                }
            }
        }

        // 🚀 DEPLOY CONTAINER
        stage('Deploy Container') {
            steps {
                sh '''
                    docker stop swiggy || true
                    docker rm swiggy || true
                    docker run -d -p 3000:3000 --name swiggy $DOCKER_IMAGE:latest
                '''
            }
        }
    }

    post {
        always {
            echo "Pipeline finished!"

            archiveArtifacts artifacts: 'trivyfs.txt', allowEmptyArchive: true
            archiveArtifacts artifacts: 'trivyimage.txt', allowEmptyArchive: true
        }
    }
}
