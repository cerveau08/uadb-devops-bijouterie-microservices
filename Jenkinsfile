pipeline {

    agent any

    environment {
        DOCKERHUB_USER = 'cerv08'
        IMAGE_TAG      = "${env.BUILD_NUMBER}"
        DOCKERHUB_CREDS = 'dockerhub-credentials'
        GIT_CREDS      = 'github-credentials'
        GIT_REPO       = 'github.com/cerveau08/uadb-devops-bijouterie-microservices.git'
        GIT_BRANCH     = 'master'
    }

    options {
        buildDiscarder(logRotator(numToKeepStr: '10'))
        timestamps()
    }

    stages {

        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('Build') {
            steps {
                sh '''
                    docker build -t ${DOCKERHUB_USER}/bijouterie-frontend:build ./frontend
                    docker build -t ${DOCKERHUB_USER}/bijouterie-backend:build  ./backend
                    docker build -t ${DOCKERHUB_USER}/bijouterie-mysql:build    ./database
                '''
            }
        }

        stage('Tag') {
            steps {
                sh '''
                    for svc in frontend backend mysql; do
                        docker tag ${DOCKERHUB_USER}/bijouterie-$svc:build ${DOCKERHUB_USER}/bijouterie-$svc:${IMAGE_TAG}
                        docker tag ${DOCKERHUB_USER}/bijouterie-$svc:build ${DOCKERHUB_USER}/bijouterie-$svc:latest
                    done
                '''
            }
        }

        stage('Push') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${DOCKERHUB_CREDS}",
                                                  usernameVariable: 'DH_USER',
                                                  passwordVariable: 'DH_TOKEN')]) {
                    sh '''
                        echo "$DH_TOKEN" | docker login -u "$DH_USER" --password-stdin
                        for svc in frontend backend mysql; do
                            docker push ${DOCKERHUB_USER}/bijouterie-$svc:${IMAGE_TAG}
                            docker push ${DOCKERHUB_USER}/bijouterie-$svc:latest
                        done
                        docker logout
                    '''
                }
            }
        }

        stage('Update manifests') {
            steps {
                withCredentials([usernamePassword(credentialsId: "${GIT_CREDS}",
                                                  usernameVariable: 'GH_USER',
                                                  passwordVariable: 'GH_TOKEN')]) {
                    sh '''
                        sed -i -E "s#(image: ${DOCKERHUB_USER}/bijouterie-[a-z]+):[^ ]+#\\1:${IMAGE_TAG}#" k8s/*-deployment.yaml
                        git config user.email "jenkins@bijouterie.local"
                        git config user.name  "Jenkins"
                        git add k8s/
                        git diff --cached --quiet || git commit -m "ci: images en version ${IMAGE_TAG} [skip ci]"
                        git push https://${GH_USER}:${GH_TOKEN}@${GIT_REPO} HEAD:${GIT_BRANCH}
                    '''
                }
            }
        }
    }

    post {
        success { echo "Images publiees : ${DOCKERHUB_USER}/bijouterie-*:${IMAGE_TAG}" }
        failure { echo 'Le build a echoue : voir les logs du stage en erreur.' }
    }
}
