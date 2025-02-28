pipeline {
    agent {
        docker {
            image 'docker:20-dind'
            args '-v /var/run/docker.sock:/var/run/docker.sock'
        }
    }
    parameters {
        string(name: 'DEPLOYMENT_FILE', defaultValue: 'kubernetes/kube-catalogo-landing.yaml', description: 'Ruta al archivo de despliegue Kubernetes dentro del repositorio')
        #string(name: 'IMAGE_NAME', description: 'Nombre de la imagen')
        #string(name: 'IMAGE_TAG', description: 'Tag de la imagen')
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
        // Definir la ruta de kubectl
        KUBECTL = '/usr/local/bin/kubectl'
    }
    stages {
        stage('Install Dependencies') {
            steps {
                sh '''
                # Instalar dependencias necesarias
                apk add --no-cache python3 py3-pip curl bash

                # Instalar AWS CLI
                pip3 install awscli

                # Instalar kubectl
                curl -LO "https://dl.k8s.io/release/v1.24.0/bin/linux/amd64/kubectl"
                chmod +x kubectl
                mv kubectl /usr/local/bin/
                
                # Verificar la instalación
                /usr/local/bin/kubectl version --client
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
                    sh '''#!/bin/bash
                    set -e
                    
                    # Verificar que el archivo existe
                    ls -la ${DEPLOYMENT_FILE}
                    
                    # Crear una copia del archivo de despliegue para modificarlo
                    cp ${DEPLOYMENT_FILE} deployment-temp.yaml
                    
                    # Verificar contenido del archivo
                    echo "Contenido del archivo deployment-temp.yaml:"
                    cat deployment-temp.yaml
                    
                    # Extraer el nombre del deployment del archivo - usar una forma más robusta
                    DEPLOYMENT_NAME=$(grep -A2 "kind: Deployment" deployment-temp.yaml | grep "name:" | head -1 | awk '{print $2}')
                    echo "Nombre del deployment: ${DEPLOYMENT_NAME}"
                    
                    # Reemplazar la imagen en el archivo
                    sed -i "s|image:.*|image: ${ECR_REPO}/${IMAGE_NAME}:${IMAGE_TAG}|g" deployment-temp.yaml
                    
                    # Verificar que kubectl está instalado
                    which ${KUBECTL}
                    
                    # Crear un directorio temporal para el kubeconfig
                    TEMP_DIR=$(mktemp -d)
                    cp "$KUBECONFIG" $TEMP_DIR/config
                    
                    # Aplicar el archivo de despliegue usando el kubeconfig en el directorio temporal
                    KUBECONFIG=$TEMP_DIR/config ${KUBECTL} apply -f deployment-temp.yaml -n ${NAMESPACE}
                    
                    # Si el deployment existe, esperar a que se complete
                    if [ ! -z "${DEPLOYMENT_NAME}" ]; then
                        KUBECONFIG=$TEMP_DIR/config ${KUBECTL} rollout status deployment/${DEPLOYMENT_NAME} -n ${NAMESPACE}
                    else
                        echo "No se pudo extraer el nombre del deployment, no se esperará el rollout"
                    fi
                    
                    # Limpiar directorio temporal
                    rm -rf $TEMP_DIR
                    '''
                }
            }
        }
        stage('Cleanup') {
            steps {
                // Hacer logout de los registros por seguridad
                sh 'docker logout $SOURCE_REPO || true'
                sh 'docker logout $ECR_REPO || true'
                // Limpieza de archivos temporales
                sh 'rm -f deployment-temp.yaml || true'
            }
        }
    }
    post {
        always {
            echo 'Pipeline completado - éxito o fallo'
        }
        success {
            echo 'Pipeline completado exitosamente'
        }
        failure {
            echo 'Pipeline falló'
        }
    }
}
