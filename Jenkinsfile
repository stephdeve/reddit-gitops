pipeline {
    agent any

    environment {
        APP_NAME   = "stephdeve/reddit-clone-pipeline-ci"
        IMAGE_TAG  = "1.0.0-13" 
    }

    stages {
        stage("Cleanup Workspace") {
            steps {
                cleanWs()
            }
        }

        stage("Checkout from SCM") {
            steps {
                git branch: 'main',
                    credentialsId: 'github',
                    url: 'https://github.com/stephdeve/reddit-gitops'
            }
        }

        stage("Update the Deployment Image Tag") {
            steps {
                sh """
                    echo 'Avant modification:'
                    cat deployment.yaml | grep image

                    sed -i "s|image: .*|image: ${APP_NAME}:${IMAGE_TAG}|g" deployment.yaml

                    echo 'Après modification:'
                    cat deployment.yaml | grep image
                """
            }
        }

        stage("Push the changed deployment file to GitHub") {
            steps {
                sh """
                    git config --global user.name "stephdeve"
                    git config --global user.email "stephdeve6@gmail.com"
                    git add deployment.yaml
                    git commit -m "Updated Deployment Manifest to ${IMAGE_TAG}"
                """
                withCredentials([gitUsernamePassword(credentialsId: 'github', gitToolName: 'Default')]) {
                    sh "git push https://github.com/stephdeve/reddit-gitops main"
                }
            }
        }
    }
}