pipeline {
    agent any

    environment {
        SONARQUBE_SERVER = 'sonarqube'
        IMAGE_NAME = "notes-app:${env.BUILD_NUMBER}"
    }

    stages {

        stage('Checkout') {
            steps {
                git 'https://github.com/sachitapatil2004/notes-app.git'
            }
        }

        stage('Install Dependencies') {
            steps {
                sh '''
                python3 -m venv venv
                . venv/bin/activate
                pip install -r requirements.txt
                '''
            }
        }

        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv("${SONARQUBE_SERVER}") {
                    sh '''
                    sonar-scanner \
                      -Dsonar.projectKey=notes-app \
                      -Dsonar.projectName="Notes App" \
                      -Dsonar.sources=. \
                      -Dsonar.host.url=http://sonarqube:9000 \
                      -Dsonar.login=$SONAR_TOKEN
                    '''
                }
            }
        }

        stage('Quality Gate') {
            steps {
                timeout(time: 2, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('Build Docker Image') {
            steps {
                sh 'docker build -t $IMAGE_NAME .'
            }
        }

        stage('Trivy Scan Image') {
            steps {
                sh '''
                trivy image --format json --output trivy-report.json $IMAGE_NAME
                trivy image --severity HIGH,CRITICAL --exit-code 1 $IMAGE_NAME
                '''
            }
        }

        stage('Archive Trivy Report') {
            steps {
                archiveArtifacts artifacts: 'trivy-report.json', fingerprint: true
            }
        }
    }

    post {
        success {
            withCredentials([string(credentialsId: 'slack-webhook', variable: 'WEBHOOK_URL')]) {
                sh '''
                curl -X POST -H 'Content-type: application/json' \
                --data '{"text":"✅ Notes App CI PASSED: SonarQube clean + Trivy clean"}' \
                $WEBHOOK_URL
                '''
            }
        }

        failure {
            withCredentials([string(credentialsId: 'slack-webhook', variable: 'WEBHOOK_URL')]) {
                sh '''
                curl -X POST -H 'Content-type: application/json' \
                --data '{"text":"❌ Notes App CI FAILED: Quality Gate or Trivy found issues"}' \
                $WEBHOOK_URL
                '''
            }
        }
    }
}
