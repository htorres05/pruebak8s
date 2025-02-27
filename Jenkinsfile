pipeline {
    agent { docker { image 'jenkins/ssh-agent' } }
    environment {
        KUBECONFIG_CREDENTIALS_ID = 'tu-kubeconfig-credential-id'
        KUBECONFIG_PATH = '/root/.kube/config'
    }
    stages {
        stage('Checkout') {
            steps {
                git 'URL_DE_TU_REPOSITORIO_GIT'
            }
        }
        stage('Docker actions') {
            steps{
                sh 'docker pull repo1/miimagen'
                sh 'docker tag repo1/miimagen repo2/miimagen:nuevotag'
                sh 'docker push repo2/miimagen:nuevotag'
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: env.KUBECONFIG_CREDENTIALS_ID, variable: 'KUBECONFIG_CONTENT')]) {
                    script {
                        sh "mkdir -p /root/.kube"
                        sh "echo '$KUBECONFIG_CONTENT' > ${env.KUBECONFIG_PATH}"
                        sh "kubectl --kubeconfig=${env.KUBECONFIG_PATH} apply -f deployment.yaml"
                    }
                }
            }
        }
    }
}
