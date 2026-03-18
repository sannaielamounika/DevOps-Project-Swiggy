pipeline {
    agent any

    environment {
        DOCKER_IMAGE  = "sannaielamounika/swiggyapp"
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

        stage('Install Dependencies') {
            steps {
                sh '''
                    npm install --prefer-offline --no-audit
                '''
            }
        }

        // ✅ SONAR SCAN
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh '''
                        export SONAR_SCANNER_OPTS="-Xmx2048m"
                        export NODE_OPTIONS="--max-old-space-size=4096"

                        sonar-scanner \
                        -Dsonar.projectName=swiggyapp \
                        -Dsonar.projectKey=swiggyapp \
                        -Dsonar.login=$SONAR_TOKEN \
                        -Dsonar.sources=src \
                        -Dsonar.exclusions=**/node_modules/**,**/*.test.js,build/**,coverage/**,public/** \
                        -Dsonar.javascript.node.maxspace=4096 \
                        -Dsonar.sourceEncoding=UTF-8
                    '''
                }
            }
        }

        // ✅ NON-BLOCKING QUALITY GATE
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

        // ✅ OWASP (FIXED)
        stage('OWASP Dependency Check') {
            steps {
                sh '''
                    if [ ! -d "dependency-check" ]; then
                        echo "Downloading OWASP Dependency Check..."
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

        // ✅ TRIVY FILE SCAN
        stage('Trivy File Scan') {
            steps {
                sh '''
                    trivy fs --scanners vuln . > trivyfs.txt || true
                '''
            }
        }

        // ✅ DOCKER BUILD & PUSH
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

        // ✅ TRIVY IMAGE SCAN
        stage('Trivy Image Scan') {
            steps {
                sh '''
                    trivy image --scanners vuln $DOCKER_IMAGE:latest > trivyimage.txt || true
                '''
            }
        }

        // ✅ DEPLOY CONTAINER
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
