pipeline {
    agent { label "Jenkins-Agent" }
    
    parameters {
        string(name: 'IMAGE_TAG', defaultValue: 'latest', description: 'The image tag to deploy')
    }

    environment {
        APP_NAME    = "accounting-app"
        DOCKER_USER = "waqaszai"
        REPO_URL    = "https://github.com/waqasalik/register-app"
    }

    stages {
        stage("Cleanup Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Checkout from SCM") {
            steps {
                git branch: 'main', credentialsId: 'github', url: "${REPO_URL}"
            }
        }

        stage("Update the Deployment Tags") {
            steps {
                sh """
                   echo "Updating deployment to tag: ${IMAGE_TAG}"
                   # This fixes the sed command to target specific image line
                   sed -i 's|image: ${DOCKER_USER}/${APP_NAME}:.*|image: ${DOCKER_USER}/${APP_NAME}:${IMAGE_TAG}|g' deployment.yaml
                   cat deployment.yaml
                """
            }
        }

        stage("Push the changed deployment file to Git") {
            steps {
                sh ""
                   git config --global user.name "waqasalik"
                   git config --global user.email "waqasalikhan086@gmail.com" 
                   git add deployment.yaml
                   git commit -m "Update image tag to ${IMAGE_TAG}" || echo "No changes to commit"
                """
                withCredentials([gitUsernamePassword(credentialsId: 'github', gitToolName: 'Default')]) {
                  sh "git push ${REPO_URL} main"
                }
            }
        }
    }
}
