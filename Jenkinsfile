pipeline {
    agent any

    environment {
        SONARQUBE = 'SonarQube' // matches Jenkins config name
        IMAGE_NAME = 'kowsie-devops/ci-cd-demo'
        SONAR_TOKEN = credentials('SonarQube-Token')
    }

    triggers {
        githubPush()  // ✅ Trigger pipeline automatically on every GitHub push
    }

    stages {
        stage('Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/kowsie-devops/ci-cd-sonarqube-docker-demo.git'
            }
        }

        stage('Build & Test') {
            steps {
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
                        /var/lib/jenkins/tools/hudson.plugins.sonar.SonarRunnerInstallation/SonarScanner/bin/sonar-scanner \
                        -Dsonar.projectKey=ci-cd-demo \
                        -Dsonar.sources=. \
                        -Dsonar.host.url=http://172.28.93.133:9000 \
                        -Dsonar.token=$SonarQube-Token \
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
                sh 'docker build -t $IMAGE_NAME .'
            }
        }

        stage('Docker Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'docker-hub', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                    sh '''
                        echo "$DOCKER_PASS" | docker login -u "$DOCKER_USER" --password-stdin
                        docker push $IMAGE_NAME
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

