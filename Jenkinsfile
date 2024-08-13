pipeline {
    agent any

    environment {
        GIT_REPO = 'https://github.com/cabca/Docker-Globe.git'
        AWS_REGION = 'us-east-1'
        ECR_REGISTRY = '851725429887.dkr.ecr.us-east-1.amazonaws.com'
        ECR_REPOSITORY = 'globe-image'
        DOCKER_IMAGE_NAME = 'globe-image'
        repoDir = "Docker-Globe"
    }
    
    stages {
        stage ('Git Clone') {
            steps {
                script {
                    // def repoDir = "${GIT_REPO}".replaceAll('https://', '').replaceAll('/', '-')
                    sh """
                        if [ ! -d "${repoDir}" ]; then
                            git clone ${GIT_REPO}
                        else
                            cd ${repoDir} && git pull
                        fi
                    """
                }
            }
        }
        
        stage('Build') {
            steps {
                dir("${repoDir}") {
                    sh ' docker build -t ${DOCKER_IMAGE_NAME} .'
                }
            }
        }

        stage('Test') {
            steps {
                dir("${repoDir}") {
                    sh 'docker run -itd -p 80:80 ${DOCKER_IMAGE_NAME}  /bin/sh -c "netstat -antp | grep nginx || exit 1"'
                    sh 'docker rm -f $(docker ps -aq)'
                }    
            }
        }
        
        stage('Install AWS CLI if missing') {
            steps {
                sh """
                    if ! which aws &> /dev/null; then
                        curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
                        unzip -o awscliv2.zip
                        sudo ./aws/install --bin-dir /usr/local/bin --install-dir /usr/local/aws-cli --update
                    else
                    aws --version
                    fi
                  """
                }
            }

        stage('Deploy to ECR') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'aws-credentials', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY')]) {
                    sh 'aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}'
                    sh 'docker tag ${DOCKER_IMAGE_NAME}:latest ${ECR_REGISTRY}/${ECR_REPOSITORY}:latest'
                    sh 'docker push ${ECR_REGISTRY}/${ECR_REPOSITORY}:latest'
                }
            }
        }
    }
}
