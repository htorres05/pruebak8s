pipeline {
    agent {
        docker {
            image 'docker:20-dind'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    parameters {
        string(name: 'DEPLOYMENT_FILE', defaultValue: 'kubernetes/kube-catalogo-landing.yaml', description: 'Ruta al archivo de despliegue Kubernetes dentro del repositorio')
        //string(name: 'IMAGE_NAME', description: 'Nombre de la imagen')
        //string(name: 'IMAGE_TAG', description: 'Tag de la imagen')
    }
    environment {
        ECR_REPO = '318518286440.dkr.ecr.us-east-1.amazonaws.com'
        SOURCE_REPO = 'srvregistry01.caredmegatone.com'
        IMAGE_NAME = 'tecnologia/jenkinsk8s'
        IMAGE_TAG = 'v1.0'
        DOCKER_REGISTRY_CRED = credentials('harbor')
        // Entorno de Kubernetes
        KUBECONFIG_CREDENTIAL_ID = 'kubeconfig1'
        NAMESPACE = 'default'
    }
    stages {
        stage('Install AWS CLI') {
            steps {
                sh '''
                apk add --no-cache python3 py3-pip
                pip3 install awscli
                '''
            }
        }
        stage('Login to Source Registry') {
            steps {
                // Login al repositorio fuente usando las credenciales
                sh 'echo $DOCKER_REGISTRY_CRED_PSW | docker login $SOURCE_REPO -u $DOCKER_REGISTRY_CRED_USR --password-stdin'
            }
        }
        stage('Login ECR') {
            steps {
                withCredentials([aws(credentialsId: 'awscarsa')]) {
                    sh 'aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin $ECR_REPO'
                }
            }
        }
        stage('Pull Image') {
            steps {
                sh "docker pull $SOURCE_REPO/$IMAGE_NAME:$IMAGE_TAG"
            }
        }
        stage('Tag Image') {
            steps {
                sh "docker tag $SOURCE_REPO/$IMAGE_NAME:$IMAGE_TAG $ECR_REPO/$IMAGE_NAME:$IMAGE_TAG"
            }
        }
        stage('Push Image to ECR') {
            steps {
                sh "docker push $ECR_REPO/$IMAGE_NAME:$IMAGE_TAG"
            }
        }
        stage('Deploy to Kubernetes') {
            steps {
                withCredentials([file(credentialsId: "${KUBECONFIG_CREDENTIAL_ID}", variable: 'KUBECONFIG')]) {
                    sh '''
                    # Crear una copia del archivo de despliegue para modificarlo
                    cp ${DEPLOYMENT_FILE} deployment-temp.yaml
                    
                    # Extraer el nombre del deployment del archivo
                    DEPLOYMENT_NAME=$(grep -A1 "kind: Deployment" deployment-temp.yaml | grep "name:" | awk '{print $2}')
                    
                    # Reemplazar la imagen en el archivo
                    sed -i "s|image:.*|image: ${ECR_REPO}/${IMAGE_NAME}:${IMAGE_TAG}|g" deployment-temp.yaml
                    
                    # Aplicar el archivo de despliegue
                    kubectl --kubeconfig=$KUBECONFIG apply -f deployment-temp.yaml -n ${NAMESPACE}
                    
                    # Esperar a que el despliegue se complete
                    kubectl --kubeconfig=$KUBECONFIG -n ${NAMESPACE} rollout status deployment/${DEPLOYMENT_NAME}
                    '''
                }
            }
        }
        stage('Cleanup') {
            steps {
                // Hacer logout de los registros por seguridad
                sh 'docker logout $SOURCE_REPO'
                sh 'docker logout $ECR_REPO'
            }
        }
    }
}
