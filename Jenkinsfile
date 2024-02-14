pipeline {
    agent any

    environment {
        AZURE_SUBSCRIPTION_ID = 'c8ce3edc-0522-48a3-b7e4-afe8e3d731d9'
        AZURE_TENANT_ID = 'bac4b78b-fcc2-4614-a32b-b69330b1af9f'
        CONTAINER_REGISTRY = 'goodacr.azurecr.io'
        RESOURCE_GROUP = 'AKS'
        REPO = 'kwujio/front'
        IMAGE_NAME = 'kwujio/front:latest'
        TAG_VERSION = "v1.0.Beta"
        TAG = "${TAG_VERSION}${env.BUILD_ID}"
        NAMESPACE = 'front'
        GIT_CREDENTIALS_ID = 'jenkins-git-access'
        GIT_REPO_URL = 'https://github.com/rlozi99/front-end'
    }

    stages {
        stage('Checkout Frontend') {
            steps {
                checkout scm
            }
        }

        stage('Build and Push Docker Image to ACR') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'acr-credential-id', passwordVariable: 'ACR_PASSWORD', usernameVariable: 'ACR_USERNAME')]) {
                        // Log in to ACR
                        sh "az acr login --name $CONTAINER_REGISTRY --username $ACR_USERNAME --password $ACR_PASSWORD"

                        // Build and push Docker image to ACR
                        // 변경: 이미지 이름을 $CONTAINER_REGISTRY/$IMAGE_NAME으로 수정
                        sh "docker build -t $CONTAINER_REGISTRY/$REPO:$TAG ."
                        sh "docker push $CONTAINER_REGISTRY/$REPO:$TAG"
                    }
                }
            }
        }

        stage('Checkout GitOps') {
            steps {
                // 'front_gitops' 저장소에서 파일들을 체크아웃합니다.
                git branch: 'main',
                    credentialsId: 'jenkins-git-access',
                    url: 'https://github.com/rlozi99/front_gitops'
            }
        }

        stage('Update Kubernetes Configuration') {
            steps {
                script {
                    // kustomize를 사용하여 Kubernetes 구성 업데이트
                    // dir('gitops') 블록을 제거합니다.
                    sh "kustomize edit set image ${CONTAINER_REGISTRY}/${REPO}=${CONTAINER_REGISTRY}/${REPO}:${TAG}"
                    sh "git add ."
                    sh "git commit -m 'Update image to ${TAG}'"
                }
            }
        }

        stage('Push Changes to GitOps Repository') {
            steps {
                script {
                    withCredentials([usernamePassword(credentialsId: 'jenkins-git-access', passwordVariable: 'GIT_PASSWORD', usernameVariable: 'GIT_USERNAME')]) {
                        // 현재 브랜치 확인 및 main으로 체크아웃
                        def currentBranch = sh(script: "git rev-parse --abbrev-ref HEAD", returnStdout: true).trim()
                        if (currentBranch != "main") {
                            sh "git checkout main"
                        }

                        // 원격 저장소에서 최신 변경사항 가져오기
                        sh "git pull --rebase origin main"

                        def remote = "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/rlozi99/front-end.git"
                        // 원격 저장소에 푸시
                        sh "git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/rlozi99/front_gitops.git main"
                    }
                }
            }
        }
    }
}