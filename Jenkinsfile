pipeline {
    agent any


    environment {
        SONARQUBE = 'SonarQube'
        IMAGE_NAME = 'kowsie-devops/ci-cd-demo'
        SONAR_TOKEN = credentials('SONAR_TOKEN') // Jenkins system config name
        BUILD_TAG = "${env.BUILD_NUMBER}"
    }

    triggers {
        githubPush()
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/kowsie-devops/ci-cd-sonarqube-docker-demo.git'
            }
        }

        stage('Build & Test') {
            steps {
                sh ''' #!/bin/bash
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                    pip install pytest pytest-cov
                    pytest --maxfail=1 --disable-warnings -q
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh ''' #!/bin/bash
                        . venv/bin/activate
                        /var/lib/jenkins/tools/hudson.plugins.sonar.SonarRunnerInstallation/SonarScanner/bin/sonar-scanner \
                            -Dsonar.projectKey=ci-cd-demo \
                            -Dsonar.sources=. \
                            -Dsonar.host.url=http://172.28.93.133:9000 \
                            -Dsonar.token=$SONAR_TOKEN \
                            -Dsonar.python.version=3.12 \
                            -Dsonar.python.coverage.reportPaths=coverage.xml \
                            -Dsonar.exclusions=venv/**,**/__pycache__/**,**/*.pyc
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 20, unit: 'MINUTES') {
                  waitForQualityGate abortPipeline: true
                    script {
                        def status = sh(
                            script: "curl -s -u $SONAR_TOKEN: http://172.28.93.133:9000/api/qualitygates/project_status?projectKey=ci-cd-demo | jq -r '.projectStatus.status'",
                            returnStdout: true
                        ).trim()
                        echo "Quality Gate status: ${status}"
                        if (status != 'OK') {
                            error "❌ Quality Gate failed: ${status}"
                        }
                    }
                }
            }
        }

         stage('Docker Check') {
            steps {
                sh '''
                    echo "🔍 Checking Docker access..."
                    docker info | grep -i "Server Version"
                    docker ps
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    echo "🧱 Building Docker image (BuildKit disabled)..."
                    export DOCKER_BUILDKIT=0
                    docker build -t $IMAGE_NAME:$BUILD_TAG .
                    docker tag $IMAGE_NAME:$BUILD_TAG $IMAGE_NAME:latest
                '''
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push $IMAGE_NAME:$BUILD_TAG
                        docker push $IMAGE_NAME:latest
                    '''
                }
            }
        }

        stage('Deploy') {
            steps {
                echo '🚀 Deployment would happen here...'
            }
        }
    }

    post {
        success {
            echo '✅ Pipeline completed successfully!'
        }
        failure {
            echo '❌ Pipeline failed!'
        }
    }
}


