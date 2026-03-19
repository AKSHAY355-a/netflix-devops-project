pipeline {
    agent any
    tools {
        jdk 'jdk17'
        nodejs 'node18'
    }
    environment {
        SCANNER_HOME = tool 'sonar-scanner'
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
                script { waitForQualityGate abortPipeline: false }
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
                        sh 'docker build --build-arg TMDB_V3_API_KEY=b1d4988f6cd31131db31a41c3523b9c4 -t netflix .'
                        sh 'docker tag netflix akshay23007/netflix:latest'
                        sh 'docker push akshay23007/netflix:latest'
                    }
                }
            }
        }
        stage('TRIVY Image Scan') {
            steps { sh 'trivy image akshay23007/netflix:latest > trivyimage.txt' }
        }
        stage('Deploy to EKS') {
            steps {
                script {
                    withCredentials([
                        string(credentialsId: 'aws-access', variable: 'AWS_ACCESS_KEY_ID'),
                        string(credentialsId: 'aws-secret', variable: 'AWS_SECRET_ACCESS_KEY')
                    ]) {
                        sh '''
                            export AWS_DEFAULT_REGION=ap-south-1
                            aws eks update-kubeconfig --region ap-south-1 --name cloudnetflix
                            kubectl apply -f deployment.yml
                            kubectl get pods
                            kubectl get svc
                        '''
                    }
                }
            }
        }
    }
}
