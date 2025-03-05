pipeline {
    agent any

    stages {
        stage('Hola Mundo') {
            steps {
                echo 'Hola Mundo!'
            }
        }
    }
    post {
        success {
            echo 'Pipeline completado exitosamente'
            script {
                def summary = """
                *Despliegue exitoso* :white_check_mark:
                *Pipeline:* ${env.JOB_NAME}
                *Imagen:* ${params.IMAGE_NAME}:${params.IMAGE_TAG}
                *Desplegada en:* ${env.NAMESPACE}
                *Build:* #${env.BUILD_NUMBER}
                *Detalles:* <${env.BUILD_URL}|Ver en Jenkins>
                """
                slackSend(color: 'good', message: summary)
            }
        }
        failure {
            echo 'Pipeline falló'
            script {
                def summary = """
                *Despliegue fallido* :x:
                *Pipeline:* ${env.JOB_NAME}
                *Imagen:* ${params.IMAGE_NAME}:${params.IMAGE_TAG}
                *Build:* #${env.BUILD_NUMBER}
                *Detalles:* <${env.BUILD_URL}|Ver en Jenkins>
                """
                slackSend(color: 'danger', message: summary)
            }
        }
    }
}
        
