pipeline {
    agent any
    
    environment {
        registryUrl = "cr152828132813.azurecr.io"
        registryCredential = "ACR"
    }
    
    stages {
        stage('Clean Workspace') {
            steps {
                cleanWs()
            }
        }
        stage('Check out') {
            steps {
                checkout scmGit(
                branches: [[name: 'main']],
                userRemoteConfigs: [[url: 'https://github.com/JuanJoseRestrepo/icesi-health-backend']])
            }
        }
        stage("Build Application"){
            steps {
                sh 'npm install'
                sh 'npm --version'
            }
        }
        stage('Run tests'){
            steps {
                sh "npm run test:unit"
            }
        }
        stage('Connect to Azure CLI and connect Kluster with ACR'){
            steps{
                script{
                    withCredentials([
                        usernamePassword(credentialsId: 'myServicePrincipal', usernameVariable: 'SP_USERNAME', passwordVariable: 'SP_PASSWORD'),
                        string(credentialsId: 'myTenant', variable: 'TENANT_ID'),
                        string(credentialsId: 'mySubscription', variable: 'SUBSCRIPTION_ID'),
                        string(credentialsId: 'acrLogin', variable: 'ACR_LOGIN')
                    ]) {
                        sh 'az login --service-principal --username $SP_USERNAME --password $SP_PASSWORD --tenant $TENANT_ID'
                        sh 'az account set --subscription $SUBSCRIPTION_ID'
                        sh 'az aks get-credentials --resource-group resources --name agic-aks-cluster --overwrite-existing'
                        sh 'az acr login --name $ACR_LOGIN'
                    }
                }
            }
        }
        stage('SonarQube'){
            steps {
             script{
    			    def scannerHome = tool 'sonar';
                    withSonarQubeEnv('sonar') { // If you have configured more than one global server connection, you can specify its name
                      sh "${scannerHome}/bin/sonar-scanner"
                    }
    			}
    		}
        }
        
        stage('Create and Pull Image to ACR'){
             steps {
                script{
                    def buildId = env.BUILD_ID // La ID de construcción en Jenkins
                    def tagName = "cr152828132813.azurecr.io/icesi-backend:${buildId}"
                    def latestTag = "cr152828132813.azurecr.io/icesi-backend"
                    sh "docker build -t icesi-backend ."
                    sh "docker tag icesi-backend ${tagName}"
                    sh "docker tag icesi-backend ${latestTag}"
                    withCredentials([usernamePassword(credentialsId: 'ACR', passwordVariable: 'password', usernameVariable: 'username')]) {
                      sh "docker login -u ${username} -p ${password} ${registryUrl}"
                      sh "docker push ${tagName}"
                      sh "docker push ${latestTag}"
                    }
                }
            }
        }
        
        stage('Delete Docker Images'){
            steps{
                script{
                    def buildId = env.BUILD_ID // La ID de construcción en Jenkins
                    def tagName = "cr152828132813.azurecr.io/icesi-backend:${buildId}"
                    def latestTag = "cr152828132813.azurecr.io/icesi-backend"
                    sh "docker rmi ${tagName}"
                    sh "docker rmi ${latestTag}"
                    sh "docker rmi icesi-backend"
                }
            }
        }
        stage('Building CD'){
            steps{
                checkout scmGit(
                branches: [[name: 'main']],
                userRemoteConfigs: [[url: 'https://github.com/JuanJoseRestrepo/deployment-back']])
            }
        }
        
        stage('deploy the yamls to CD'){
            steps{
                script{
                    def deploymentOutput = sh(returnStdout: true, script: 'kubectl get deployment icesi-backend -n icesi-health').trim()

                    if (deploymentOutput.contains('icesi-backend')) {
                        sh 'kubectl delete deployment icesi-backend -n icesi-health'
                    }
                    sh "kubectl apply -f deployment/deployment-db.yaml"
                    sh "kubectl apply -f deployment/deployment-back.yaml"
                }
            }
        }
        
     }

  }