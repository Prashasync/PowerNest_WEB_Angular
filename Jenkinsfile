pipeline {
    agent any

    parameters {
        string(name: 'ECR_REPO_NAME', defaultValue: 'powernest_ui', description: 'Enter repository name')
        string(name: 'AWS_ACCOUNT_ID', defaultValue: '463470954735', description: 'Enter AWS Account ID')
        string(name: 'K8S_NAMESPACE', defaultValue: 'default', description: 'Enter Kubernetes namespace')
    }

    tools {
        jdk 'JDK'
        nodejs 'NodeJS_3'
    }

    environment {
        SCANNER_HOME = tool 'SonarQube Scanner'
        PYTHON = '/usr/bin/python3'
        PIP = '/usr/bin/pip3'
    }

    stages {
        stage('1. Git Checkout') {
            steps {
                git branch: 'main', url: 'https://github.com/Prashasync/PowerNest_WEB_Angular.git'
            }
        }

        stage('2. SonarQube Analysis') {
            steps {
                withSonarQubeEnv('sonar-server') {
                    sh """
                    $SCANNER_HOME/bin/sonar-scanner \
                    -Dsonar.projectName=PowerNest_UI \
                    -Dsonar.projectKey=PowerNest_UI
                    """
                }
            }
        }

        stage('3. Install npm') {
            steps {
                sh "npm install"
            }
        }

        stage('4. Build Angular Application') {
            steps {
                sh "npm run build"
            }
        }

        stage('5. Trivy Scan') {
            steps {
                sh "trivy fs . > trivy.txt"
            }
        }

        stage('6. Build Docker Image') {
            steps {
                sh "docker build -t ${params.ECR_REPO_NAME} ."
            }
        }

        stage('7. Create ECR Repo') {
            steps {
                withCredentials([string(credentialsId: 'access-key', variable: 'AWS_ACCESS_KEY'),
                                 string(credentialsId: 'secret-key', variable: 'AWS_SECRET_KEY')]) {
                    sh """
                    aws configure set aws_access_key_id $AWS_ACCESS_KEY
                    aws configure set aws_secret_access_key $AWS_SECRET_KEY
                    aws ecr describe-repositories --repository-names ${params.ECR_REPO_NAME} --region us-east-1 || \
                    aws ecr create-repository --repository-name ${params.ECR_REPO_NAME} --region us-east-1
                    """
                }
            }
        }

        stage('8. Login to ECR & Tag Image') {
            steps {
                withCredentials([string(credentialsId: 'access-key', variable: 'AWS_ACCESS_KEY'),
                                 string(credentialsId: 'secret-key', variable: 'AWS_SECRET_KEY')]) {
                    sh """
                    aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin ${params.AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com
                    docker tag ${params.ECR_REPO_NAME} ${params.AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/${params.ECR_REPO_NAME}:${BUILD_NUMBER}
                    docker tag ${params.ECR_REPO_NAME} ${params.AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/${params.ECR_REPO_NAME}:latest
                    """
                }
            }
        }

        stage('9. Push Image to ECR') {
            steps {
                withCredentials([string(credentialsId: 'access-key', variable: 'AWS_ACCESS_KEY'),
                                 string(credentialsId: 'secret-key', variable: 'AWS_SECRET_KEY')]) {
                    sh """
                    docker push ${params.AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/${params.ECR_REPO_NAME}:${BUILD_NUMBER}
                    docker push ${params.AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/${params.ECR_REPO_NAME}:latest
                    """
                }
            }
        }

        stage('10. Deploy to Kubernetes') {
            steps {
                withCredentials([string(credentialsId: 'kubeconfig', variable: 'KUBECONFIG_CONTENT')]) {
                    sh """
                    echo "$KUBECONFIG_CONTENT" > kubeconfig
                    export KUBECONFIG=kubeconfig
                    kubectl set image deployment/angular-app angular-app=${params.AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/${params.ECR_REPO_NAME}:${BUILD_NUMBER} -n ${params.K8S_NAMESPACE} || \
                    kubectl apply -f k8s_files/deployment.yaml -n ${params.K8S_NAMESPACE}
                    kubectl apply -f k8s_files/service.yaml -n ${params.K8S_NAMESPACE}
                    """
                }
            }
        }

        stage('11. Cleanup Docker Images') {
            steps {
                sh """
                docker rmi ${params.AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/${params.ECR_REPO_NAME}:${BUILD_NUMBER}
                docker rmi ${params.AWS_ACCOUNT_ID}.dkr.ecr.us-east-1.amazonaws.com/${params.ECR_REPO_NAME}:latest
                docker images
                """
            }
        }
    }
}