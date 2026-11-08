pipeline {
    agent any
    environment {
        IMAGE = 'rahmatarek/ivolve-app'
        TAG   = "${env.BUILD_NUMBER}"
    }
    stages {
        stage('Checkout') {
            steps { checkout scm }
        }
        stage('Build App') {
            steps { sh 'npm install' }
        }
        stage('Build Docker Image') {
            steps { sh 'docker build -t ${IMAGE}:${TAG} .' }
        }
        stage('Scan Docker Image') {
            steps { sh 'trivy image --exit-code 1 --severity CRITICAL --ignore-unfixed --no-progress ${IMAGE}:${TAG}' }
        }
        stage('Push to DockerHub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-creds', usernameVariable: 'DH_USER', passwordVariable: 'DH_PASS')]) {
                    sh '''
                        echo "$DH_PASS" | docker login -u "$DH_USER" --password-stdin
                        docker push ${IMAGE}:${TAG}
                    '''
                }
            }
        }
        stage('Delete Local Image') {
            steps { sh 'docker rmi ${IMAGE}:${TAG}' }
        }
        stage('Update Deployment File') {
            steps {
                sh "sed -i 's|image: ${IMAGE}:.*|image: ${IMAGE}:${TAG}|' k8s/deployment.yaml"
                sh 'grep "image:" k8s/deployment.yaml'
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: 'kubeconfig', variable: 'KUBECONFIG')]) {
                    sh '''
                        kubectl -n ivolve apply -f k8s/deployment.yaml
                        kubectl -n ivolve rollout status deployment/ivolveapp --timeout=180s
                    '''
                }
            }
        }
    }
    post {
        always  { sh 'docker logout || true'; sh 'docker rmi ${IMAGE}:${TAG} || true'; deleteDir() }
        success { echo "Pipeline succeeded: ${IMAGE}:${TAG} deployed" }
        failure { echo "Pipeline failed, check the stage logs" }
    }
}