pipeline {
    agent any

    environment {
        APP_NAME  = "stephdeve/reddit-clone-pipeline-ci"
        // Tag dynamique basé sur le numéro de build Jenkins
        IMAGE_TAG = "1.0.0-${env.BUILD_NUMBER}"
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
                    cat deployment.yaml | grep image || true

                    sed -i "s|image: .*|image: ${APP_NAME}:${IMAGE_TAG}|g" deployment.yaml

                    echo 'Après modification:'
                    cat deployment.yaml | grep image || true
                """
            }
        }

        stage("Push the changed deployment file to GitHub") {
            steps {
                withCredentials([usernamePassword(credentialsId: 'github', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
                    sh """
                        git config --global user.name "stephdeve"
                        git config --global user.email "stephdeve6@gmail.com"
                        git add deployment.yaml || true

                        if git diff --cached --quiet; then
                          echo 'Aucun changement à committer, on continue sans erreur.'
                        else
                          git commit -m "Updated Deployment Manifest to ${IMAGE_TAG}"
                          git push https://${GIT_USER}:${GIT_PASS}@github.com/stephdeve/reddit-gitops main
                        fi
                    """
                }
            }
        }
    }
}