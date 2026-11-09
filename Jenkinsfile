// =============================================================================
// Pipeline CI - Bijouterie
// Construit les 3 images Docker, les tague (numero de build + latest),
// les pousse sur Docker Hub, puis met a jour les tags dans k8s/ (GitOps).
// Argo CD detecte ensuite le commit et deploie la nouvelle version.
// =============================================================================
pipeline {

    // Le pipeline tourne sur n'importe quel agent ayant Docker et Git.
    agent any

    environment {
        // Compte Docker Hub (identifiant public, pas un secret)
        DOCKERHUB_USER  = 'cerv08'
        // Tag des images = numero de build Jenkins (jamais "latest" dans k8s/)
        IMAGE_TAG       = "${env.BUILD_NUMBER}"
        // Credential Jenkins (Username with password) : token Docker Hub
        DOCKERHUB_CREDS = 'dockerhub-credentials'
        // Credential Jenkins : token GitHub, utilise par "Update manifests"
        GIT_CREDS       = 'github-credentials'
        // Depot et branche surveilles par Argo CD
        GIT_REPO        = 'github.com/cerveau08/uadb-devops-bijouterie-microservices.git'
        GIT_BRANCH      = 'master'
    }

    options {
        // Garde seulement les 10 derniers builds
        buildDiscarder(logRotator(numToKeepStr: '10'))
        // Horodate chaque ligne de la console
        timestamps()
        // Le stage Checkout ci-dessous fait deja le checkout : on evite le doublon
        skipDefaultCheckout(true)
    }

    stages {

        // ---------------------------------------------------------------------
        // Stage 1 - Checkout : recupere le code depuis le depot Git du job.
        // On lit aussi le dernier message de commit : si c'est un commit
        // automatique de ce pipeline ("[skip ci]"), les stages suivants sont
        // ignores. Sans cela, le commit pousse par "Update manifests" serait vu
        // par le Poll SCM et relancerait un build en boucle, toutes les 5 min.
        // ---------------------------------------------------------------------
        stage('Checkout') {
            steps {
                checkout scm
                script {
                    def msg = sh(script: 'git log -1 --pretty=%B', returnStdout: true).trim()
                    env.SKIP_CI = msg.contains('[skip ci]') ? 'true' : 'false'
                    if (env.SKIP_CI == 'true') {
                        currentBuild.description = 'Ignore : commit automatique du pipeline'
                        echo "Dernier commit = [skip ci] -> build ignore pour eviter la boucle."
                    }
                }
            }
        }

        // ---------------------------------------------------------------------
        // Stage 2 - Build : construit les 3 images a partir de leurs Dockerfiles
        // (frontend/, backend/, database/). Tag temporaire ":build".
        // ---------------------------------------------------------------------
        stage('Build') {
            when { expression { env.SKIP_CI != 'true' } }
            steps {
                sh '''
                    docker build -t ${DOCKERHUB_USER}/bijouterie-frontend:build ./frontend
                    docker build -t ${DOCKERHUB_USER}/bijouterie-backend:build  ./backend
                    docker build -t ${DOCKERHUB_USER}/bijouterie-mysql:build    ./database
                '''
            }
        }

        // ---------------------------------------------------------------------
        // Stage 3 - Tag : double tag, numero de build (utilise par k8s) + latest
        // ---------------------------------------------------------------------
        stage('Tag') {
            when { expression { env.SKIP_CI != 'true' } }
            steps {
                sh '''
                    for svc in frontend backend mysql; do
                        docker tag ${DOCKERHUB_USER}/bijouterie-$svc:build ${DOCKERHUB_USER}/bijouterie-$svc:${IMAGE_TAG}
                        docker tag ${DOCKERHUB_USER}/bijouterie-$svc:build ${DOCKERHUB_USER}/bijouterie-$svc:latest
                    done
                '''
            }
        }

        // ---------------------------------------------------------------------
        // Stage 4 - Push : connexion a Docker Hub puis envoi des images.
        // Le token vient du credential Jenkins : il n'apparait jamais en clair
        // dans ce fichier ni dans les logs (masque par Jenkins).
        // ---------------------------------------------------------------------
        stage('Push') {
            when { expression { env.SKIP_CI != 'true' } }
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

        // ---------------------------------------------------------------------
        // Stage 5 (bonus GitOps) - Update manifests : remplace le tag des images
        // dans k8s/*-deployment.yaml, commite et pousse sur GitHub. Argo CD
        // detecte le commit et deploie la nouvelle version sans action manuelle.
        // Le message contient "[skip ci]" : le stage Checkout ignorera ce commit.
        // ---------------------------------------------------------------------
        stage('Update manifests') {
            when { expression { env.SKIP_CI != 'true' } }
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
        success { echo "Pipeline termine (tag ${IMAGE_TAG})." }
        failure { echo 'Le build a echoue : voir les logs du stage en erreur.' }
    }
}
