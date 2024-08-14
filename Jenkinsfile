pipeline {
    agent any

    environment {
        // GitHub variables
        GITHUB_REPOSITORY = credentials('GITHUB_REPOSITORY')

        // AWS variables
        AWS_REGION = credentials('AWS_REGION')
        AWS_ECR_REGISTRY = credentials('AWS_ECR_REGISTRY')
        AWS_ECR_REPOSITORY = credentials('AWS_ECR_REPOSITORY')

        // DockerHub variables
        DOCKER_ID = credentials('DOCKER_ID')
        DOCKER_PASSWORD = credentials('DOCKER_PASSWORD')
        
        // Jenkins server variables
        DOCKER_IMAGE_NAME = credentials('DOCKER_IMAGE_NAME')
        JENKINS_SERVER_DIRECTORY_NAME = credentials('JENKINS_SERVER_DIRECTORY_NAME')
    }
    
    stages {
        stage ('Cloning the GitHub repository') {
            steps {
                script {
                    // def JENKINS_SERVER_DIRECTORY_NAME = "${GITHUB_REPOSITORY}".replaceAll('https://', '').replaceAll('/', '-')
                    sh """
                        if [ ! -d "${JENKINS_SERVER_DIRECTORY_NAME}" ]; then
                            git clone ${GITHUB_REPOSITORY}
                        else
                            cd ${JENKINS_SERVER_DIRECTORY_NAME} && git pull
                        fi
                    """
                }
            }
        }
        
        stage('Loging into DockerHub') {
            steps {
                // This is to bypass the number of times you can pull from DockerHun without authentication
                sh 'echo $DOCKER_PASSWORD | docker login -u $DOCKER_ID --password-stdin'
            }
        }

        stage('Building the Docker image') {
            steps {
                dir("${repoDir}") {
                    sh ' docker build -t ${DOCKER_IMAGE_NAME} .'
                }
            }
        }

        stage('Testing the Docker image') {
            steps {
                dir("${repoDir}") {
                    sh 'docker run -itd -p 80:80 ${DOCKER_IMAGE_NAME}  /bin/sh -c "netstat -antp | grep nginx || exit 1"'
                    sh 'docker rm -f $(docker ps -aq)'
                }    
            }
        }
        
        stage('Installing the AWS CLI if not installed already') {
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

        stage('Deploying the Docker image to AWS ECR') {
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
