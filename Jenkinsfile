pipeline {
    agent any

    environment {
        // GitHub variables
        GIT_REPO = 'https://github.com/cabca/Docker-Globe.git'

        // AWS variables
        AWS_REGION = credentials('AWS_REGION')
        AWS_ECR_REGISTRY = credentials('AWS_AWS_ECR_REGISTRY')
        AWS_ECR_REPOSITORY = credentials('AWS_ECR_REPOSITORY')

        // DockerHub variables
        DOCKER_ID = credentials('DOCKER_ID')
        DOCKER_PASSWORD = credentials('DOCKER_PASSWORD')
        
        // Jnekins server variables
        DOCKER_IMAGE_NAME = credentials('DOCKER_IMAGE_NAME')
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
        
        stage('DockerHub login') {
            steps {
                // This is to bypass the number of times you can pull from DockerHun without authentication
                sh 'echo $DOCKER_PASSWORD | docker login -u $DOCKER_ID --password-stdin'
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
                    sh 'aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${AWS_ECR_REGISTRY}'
                    sh 'docker tag ${DOCKER_IMAGE_NAME}:latest ${AWS_ECR_REGISTRY}/${AWS_ECR_REPOSITORY}:latest'
                    sh 'docker push ${AWS_ECR_REGISTRY}/${AWS_ECR_REPOSITORY}:latest'
                }
            }
        }
    }
}
