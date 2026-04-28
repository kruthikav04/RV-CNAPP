pipeline {
    agent any

    environment {
        IMAGE_NAME = "airowire.azurecr.io/notes-app"
        RESOURCE_GROUP = "Airowire-RG"
        AKS_CLUSTER = "Airowire-aks"
        ACR_NAME = "airowire"
        TENANT_ID = "981439d1-88ac-4c7c-bd5d-d5df66bc0f4c"
        SUBSCRIPTION_ID = "Kruthika's-Subscription"
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

                    az account set --subscription "$SUBSCRIPTION_ID"
                    '''
                }
            }
        }

        stage('Login to ACR') {
            steps {
                sh '''
                echo "Logging into ACR..."
                az acr login --name $ACR_NAME
                '''
            }
        }

        stage('Build & Push Image') {
            steps {
                sh '''
                echo "Building Docker image..."

                docker build -t $IMAGE_NAME:${BUILD_NUMBER} .
                docker tag $IMAGE_NAME:${BUILD_NUMBER} $IMAGE_NAME:latest

                echo "Pushing image..."
                docker push $IMAGE_NAME:${BUILD_NUMBER}
                docker push $IMAGE_NAME:latest
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

                kubectl get nodes

                echo "Deploying application..."

                kubectl apply -f k8s/deployment.yaml
                kubectl apply -f k8s/service.yaml

                echo "Updating image..."

                kubectl set image deployment/notes-app \
                    notes-app=$IMAGE_NAME:${BUILD_NUMBER}

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
            echo "❌ FAILED: Check logs"
        }
    }
}
