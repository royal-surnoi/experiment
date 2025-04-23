pipeline{
    agent any
    options{
        disableConcurrentBuilds()
        disableResume()
        timeout(time: 1, unit: "HOURS")
    }
    environment {
        BACKEND_VERSION = ''
        FRONTEND_VERSION = ''
        RELEASE_VERSION = 'V1.0.1'
        APP_VERSION = 'V1.0.1'
        AWS_ACCESS_KEY_ID = credentials('AWS_ACCESS_KEY_ID')
        AWS_SECRET_ACCESS_KEY = credentials('AWS_SECRET_ACCESS_KEY')
        AWS_DEFAULT_REGION = "us-east-1"
    }
    parameters{
        choice(name: 'ENVIRONMENT', choices: ['DEV', 'UAT', 'PROD'], description: 'Choose the environment')
        choice(name: 'OPERATION', choices: ['Release', 'Roleback'], description: 'Choose the operation')
        choice(name: 'DEPLOYMENT_ENVIRONMENT', choices: ['BLUE', 'GREEN'], description: 'Choose the deployment environment')
        booleanParam(name: 'SWITCH_TRAFFIC', defaultValue: false, description: 'Switch traffic between Blue and Green')

    }
    stages{
        stage('Setup'){
            steps{
                script {
                    // Validate parameters
                    if (!params.ENVIRONMENT || !params.OPERATION) {
                        error "Environment or Operation not specified!"
                    }
                    echo "Running ${params.OPERATION} for ${params.ENVIRONMENT}"
                }
            } 
        }

        stage('Versions-Imports'){   
            steps {
                // This copies the `image-version.txt` from the last successful build of project-build
                copyArtifacts(
                    projectName: 'project-build-backend',
                    filter: 'image-version.txt',
                    selector: lastSuccessful()
                )
                script {
                    BACKEND_VERSION = readFile('image-version.txt').trim()
                    echo "backend:version: ${BACKEND_VERSION}"
                    // Use this version for Kubernetes deploy, tagging, etc.
                }
                copyArtifacts(
                    projectName: 'project-build-frontend',
                    filter: 'image-version.txt',
                    selector: lastSuccessful()
                )
                script {
                    FRONTEND_VERSION = readFile('image-version.txt').trim()
                    echo "frontend:version: ${FRONTEND_VERSION}"
                    // Use this version for Kubernetes deploy, tagging, etc.
                }
            }
        }

        stage('EKS Login') {
                steps {
                    script{
                        sh """
                            aws eks update-kubeconfig --region us-east-1 --name fusioniq-${params.ENVIRONMENT.toLowerCase()}
                            kubectl get nodes
                            kubectl apply -f namespace.yaml
                        """
                    }
                }
        }

        stage('Deploy') {
            when {
                expression { params.OPERATION == 'Release' }
            }
            steps {
                script {
                    if (params.ENVIRONMENT == 'PROD') {
                        input message: "Approve for Release in PROD environment?"
                    }

                    def envTag = params.DEPLOYMENT_ENVIRONMENT.toLowerCase()
                    withKubeConfig(caCertificate: '', clusterName: 'fusioniq-prod', contextName: '', credentialsId: 'k8-token', namespace: 'fusioniq', restrictKubeConfigAccess: false, serverUrl: 'https://6ca4aab3d35ce1273f4a110793eaf5f3.gr7.us-east-1.eks.amazonaws.com') {
                    // Deploy backend
                    sh """
                        sed -i "s/^appVersion: .*/appVersion: ${APP_VERSION}/" ./Helm/backend/Chart.yaml
                        helm upgrade --install backend-${envTag} ./Helm/backend \
                            --set image.tag=${BACKEND_VERSION} \
                            --set releaseVersion=${RELEASE_VERSION} \
                            --set deployment.environment=${envTag} \
                            --namespace fusioniq
                    """

                    // Deploy frontend
                    sh """
                        sed -i "s/^appVersion: .*/appVersion: ${APP_VERSION}/" ./Helm/frontend/Chart.yaml
                        helm upgrade --install frontend-${envTag} ./Helm/frontend \
                            --set image.tag=${FRONTEND_VERSION} \
                            --set releaseVersion=${RELEASE_VERSION} \
                            --set deployment.environment=${envTag} \
                            --namespace fusioniq
                    """
                    }
                }
            }
        }

        stage('Switch Traffic') {
            when {
                expression { return params.SWITCH_TRAFFIC }
            }
            steps {
                script {
                    def envTag = params.DEPLOYMENT_ENVIRONMENT.toLowerCase()
                    echo "Switching traffic to ${envTag}"
                    withKubeConfig(caCertificate: '', clusterName: 'fusioniq-prod', contextName: '', credentialsId: 'k8-token', namespace: 'fusioniq', restrictKubeConfigAccess: false, serverUrl: 'https://d0dbde6088e5eda9db7e0ce79cfcf11c.sk1.us-east-1.eks.amazonaws.com/') {
                    // Switch service selector to new deployment
                    sh """
                        kubectl patch svc frontend-service -n fusioniq -p '{
                            "spec": {
                                "selector": {
                                    "deployment.environment": "${envTag}"
                                }
                            }
                        }'
                        kubectl patch svc backend-service -n fusioniq -p '{
                            "spec": {
                                "selector": {
                                    "deployment.environment": "${envTag}"
                                }
                            }
                        }'
                    """
                     }
                }
            }
        }

        stage('Deployment-test') {
            when {
                expression { return params.SWITCH_TRAFFIC }
            }
            steps {
                script {
                    withKubeConfig(caCertificate: '', clusterName: 'fusioniq-prod', contextName: '', credentialsId: 'k8-token', namespace: 'fusioniq', restrictKubeConfigAccess: false, serverUrl: 'https://6ca4aab3d35ce1273f4a110793eaf5f3.gr7.us-east-1.eks.amazonaws.com/') {
                     sh """
                        kubectl get pods -l -n fusioniq
                        kubectl get svc backend-service -n fusioniq
                        kubectl get svc frontend-service -n fusioniq
                    """
                    }
                }
            }        
        }

        stage('Rollback') {
            when {
                expression { params.OPERATION == 'Rollback' }
            }
            steps {
                script {
                    if (params.ENVIRONMENT == 'PROD') {
                        input message: "Approve rollback in PROD environment?"
                    }

                    sh '''
                        # Backend rollback
                        CURRENT_BE_REV=$(helm history backend-blue -n fusioniq -o json | jq '.[] | select(.status == "deployed") | .revision')
                        PREV_BE_REV=$(helm history backend-blue -n fusioniq -o json | jq --argjson cur "$CURRENT_BE_REV" '.[] | select(.revision < $cur) | .revision' | sort -nr | head -1)
                        if [ -n "$PREV_BE_REV" ]; then helm rollback backend-blue "$PREV_BE_REV" -n fusioniq; fi

                        # Frontend rollback
                        CURRENT_FE_REV=$(helm history frontend-blue -n fusioniq -o json | jq '.[] | select(.status == "deployed") | .revision')
                        PREV_FE_REV=$(helm history frontend-blue -n fusioniq -o json | jq --argjson cur "$CURRENT_FE_REV" '.[] | select(.revision < $cur) | .revision' | sort -nr | head -1)
                        if [ -n "$PREV_FE_REV" ]; then helm rollback frontend-blue "$PREV_FE_REV" -n fusioniq; fi
                    '''
                }
            }
        }
    }
}