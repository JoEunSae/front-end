pipeline {
    agent any

    environment {
        AZURE_SUBSCRIPTION_ID = 'c8ce3edc-0522-48a3-b7e4-afe8e3d731d9'
        AZURE_TENANT_ID = '4ccd6048-181f-43a0-ba5a-7f48e8a4fa35'
        CONTAINER_REGISTRY = 'goodbirdacr.azurecr.io'
        RESOURCE_GROUP = 'AKS'
        REPO = 'medicine/front'
        IMAGE_NAME = 'medicine/front:latest'

        TAG_VERSION = "v1.0.Beta"
        TAG = "${TAG_VERSION}${env.BUILD_ID}"
        NAMESPACE = 'front'
        GIT_CREDENTIALS_ID = 'jenkins-git-access'
    }



    stages {
        stage('Checkout') {
            steps {
                checkout scm
            }
        }

        stage('SonarQube Analysis') {
            steps {
                script {
                    def scannerHome = tool 'sonarqube_scanner'
                    withSonarQubeEnv('SonarQubeServer') {
                        // SonarScanner 실행 명령에 -X 옵션 추가
                        sh "${scannerHome}/bin/sonar-scanner -X"
                    }
                }
            }
        }
        stage('Trivy Security') {
            steps {
                sh 'chmod +x trivy-image-scan.sh' // 스크립트에 실행 권한 추가
                sh './trivy-image-scan.sh' // Trivy 이미지 스캔 실행
            }
        }
        stage('Build and Push Docker Image to ACR..') {
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
        stage('Update Kubernetes Configuration..') {
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
                        def remote = "https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/JoEunSae/front-end.git"
                        // 원격 저장소에 푸시
                        sh "git push https://${GIT_USERNAME}:${GIT_PASSWORD}@github.com/rlozi99/front_gitops.git main"
                    }
                }
            }
        }
    }
}
