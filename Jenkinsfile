pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node18'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
        DOCKERHUB_USER = 'akshay23007'
        IMAGE_NAME = "${DOCKERHUB_USER}/netflix"
    }
    stages {
        stage('Clean Workspace') {
            steps { cleanWs() }
        }
        stage('Checkout from Git') {
            steps {
                git branch: 'main',
                    credentialsId: 'github-token',
                    url: 'https://github.com/AKSHAY355-a/DevSecOps-Project.git'
            }
        }
        stage('SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonarqube') {
                    sh '''$SCANNER_HOME/bin/sonar-scanner \
                        -Dsonar.projectName=netflix \
                        -Dsonar.projectKey=netflix'''
                }
            }
        }
        stage('Quality Gate') {
            steps {
                script { waitForQualityGate abortPipeline: true }
            }
        }
        stage('Install Dependencies') {
            steps { sh 'npm install' }
        }
        stage('TRIVY FS Scan') {
            steps { sh 'trivy fs . > trivyfs.txt' }
        }
        stage('Docker Build & Push') {
            steps {
                script {
                    withDockerRegistry(credentialsId: 'docker', url: 'https://index.docker.io/v1/') {
                        withCredentials([string(credentialsId: 'tmdb-api-key', variable: 'TMDB_KEY')]) {
                            sh "docker build --build-arg TMDB_V3_API_KEY=${TMDB_KEY} -t netflix ."
                            sh "docker tag netflix ${DOCKERHUB_USER}/netflix:${BUILD_NUMBER}"
                            sh "docker push ${DOCKERHUB_USER}/netflix:${BUILD_NUMBER}"
                        }
                    }
                }
            }
        }
        stage('TRIVY Image Scan') {
            steps { sh "trivy image ${DOCKERHUB_USER}/netflix:${BUILD_NUMBER} > trivyimage.txt" }
        }
        stage('Deploy to EKS') {
            steps {
                script {
                    withCredentials([
                        string(credentialsId: 'aws-access', variable: 'AWS_ACCESS_KEY_ID'),
                        string(credentialsId: 'aws-secret', variable: 'AWS_SECRET_ACCESS_KEY')
                    ]) {
                        sh """
                            export AWS_DEFAULT_REGION=ap-south-1
                            aws eks update-kubeconfig --region ap-south-1 --name cloudnetflix
                            sed -i 's|akshay23007/netflix:latest|akshay23007/netflix:${BUILD_NUMBER}|g' deployment.yml
                            kubectl apply -f deployment.yml
                            kubectl get pods
                            kubectl get svc
                        """
                    }
                }
            }
        }
    }
    post {
        always {
            echo 'Pipeline execution complete.'
        }
        success {
            echo 'Pipeline succeeded! Application deployed to EKS.'
        }
        failure {
            echo 'Pipeline failed. Check logs for details.'
            emailext(
                subject: "FAILED: Pipeline '${env.JOB_NAME}' [${env.BUILD_NUMBER}]",
                body: "Build failed. Check console output at: ${env.BUILD_URL}",
                to: 'your-email@gmail.com'
            )
        }
    }
}
