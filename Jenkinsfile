pipeline {
    agent any

    environment {
        SONARQUBE = 'SonarQube' // Matches name in Jenkins config
        IMAGE_NAME = 'kowsie-devops/ci-cd-demo' // Your DockerHub repo
    }

    triggers {
        githubPush() // Auto trigger on commit
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/kowsie-devops/ci-cd-sonarqube-docker-demo.git'
            }
        }

        stage('Build & Test') {
            steps {
                // ✅ Use python3 & pip3 explicitly (Jenkins runs in its own environment)
                sh '''
                    python3 -m venv venv
                    . venv/bin/activate
                    pip install --upgrade pip
                    pip install -r requirements.txt
                    pytest --maxfail=1 --disable-warnings -q
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('SonarQube') {
                    sh '''
                        sonar-scanner \
                          -Dsonar.projectKey=ci-cd-demo \
                          -Dsonar.sources=. \
                          -Dsonar.host.url=http://172.28.93.133:9000 \
                          -Dsonar.login=$SONARQUBE \
                          -Dsonar.python.version=3.12
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 1, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Docker Build') {
            steps {
                sh "docker build -t ${IMAGE_NAME}:${BUILD_NUMBER} ."
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-cred', usernameVariable: 'USER', passwordVariable: 'PASS')]) {
                    sh """
                        echo $PASS | docker login -u $USER --password-stdin
                        docker push ${IMAGE_NAME}:${BUILD_NUMBER}
                        docker tag ${IMAGE_NAME}:${BUILD_NUMBER} ${IMAGE_NAME}:latest
                        docker push ${IMAGE_NAME}:latest
                    """
                }
            }
        }

        stage('Deploy') {
            steps {
                sh """
                    docker rm -f ci_cd_demo || true
                    docker run -d --name ci_cd_demo -p 8080:8080 ${IMAGE_NAME}:latest
                """
            }
        }
    }

    post {
        always {
            echo '✅ Pipeline completed successfully.'
        }
    }
}

