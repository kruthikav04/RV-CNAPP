pipeline {
    agent any

    environment {
        IMAGE_NAME = "airowire.azurecr.io/notes-app"
        RESOURCE_GROUP = "Airowire-RG"
        AKS_CLUSTER = "Airowire-aks"
        ACR_NAME = "airowire"
        TENANT_ID = "981439d1-88ac-4c7c-bd5d-d5df66bc0f4c"
    }

    stages {

        stage('Checkout Code') {
            steps {
                checkout scm
            }
        }

        stage('Azure Login') {
            steps {
                withCredentials([usernamePassword(
                    credentialsId: 'azure-sp-creds',
                    usernameVariable: 'AZURE_CLIENT_ID',
                    passwordVariable: 'AZURE_CLIENT_SECRET'
                )]) {
                    sh '''
                    echo "Logging into Azure..."

                    az login --service-principal \
                        --username $AZURE_CLIENT_ID \
                        --password $AZURE_CLIENT_SECRET \
                        --tenant $TENANT_ID

                    az account set --subscription "Kruthika's-Subscription"
                    '''
                }
            }
        }

        stage('Login to ACR') {
            steps {
                sh '''
                echo "Logging into Azure Container Registry..."
                az acr login --name $ACR_NAME
                '''
            }
        }

       
        stage('Build & Push Multi-Arch Image') {
            steps {
                sh '''
                echo "Setting up buildx..."

                docker buildx create --use || true
                docker buildx inspect --bootstrap

                echo "Enabling cross-architecture support..."
                docker run --privileged --rm tonistiigi/binfmt --install all

                echo "Building and pushing multi-arch image..."

                docker buildx build \
                    --platform linux/amd64,linux/arm64 \
                    -t $IMAGE_NAME:${BUILD_NUMBER} \
                    -t $IMAGE_NAME:latest \
                    --push .
                '''
            }
        }

        stage('Deploy to AKS') {
            steps {
                sh '''
                echo "Getting AKS credentials..."

                az aks get-credentials \
                    --resource-group $RESOURCE_GROUP \
                    --name $AKS_CLUSTER \
                    --overwrite-existing

                echo "Checking cluster..."
                kubectl get nodes

                echo "Deploying Kubernetes manifests..."
                kubectl apply -f k8s/deployment.yaml
                kubectl apply -f k8s/service.yaml

                echo "Updating image in deployment..."
                kubectl set image deployment/notes-app \
                    notes-app=$IMAGE_NAME:${BUILD_NUMBER}

                echo "Waiting for rollout..."
                kubectl rollout status deployment/notes-app --timeout=120s
                '''
            }
        }
    }

    post {
        success {
            echo "✅ SUCCESS: App deployed to AKS"
        }
        failure {
            echo "❌ FAILED: Check Jenkins logs"
        }
    }
}
