pipeline {
    agent any

    stages {
        stage('Cloning the GitHub repository') {
            steps {
                withCredentials([string(credentialsId: 'GITHUB_REPOSITORY', variable: 'GITHUB_REPO')]) {
                    script {
                        sh """
                            if [ ! -d "${env.JENKINS_SERVER_DIRECTORY_NAME}" ]; then
                                git clone ${GITHUB_REPO} ${env.JENKINS_SERVER_DIRECTORY_NAME}
                            else
                                cd ${env.JENKINS_SERVER_DIRECTORY_NAME} && git pull
                            fi
                        """
                    }
                }
            }
        }
        
        stage('Logging into DockerHub') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'dockerhub-credentials', usernameVariable: 'DOCKER_ID', passwordVariable: 'DOCKER_PASSWORD')]) {
                    sh 'echo $DOCKER_PASSWORD | docker login -u $DOCKER_ID --password-stdin'
                }
            }
        }

        stage('Building the Docker image') {
            steps {
                withCredentials([string(credentialsId: 'DOCKER_IMAGE_NAME', variable: 'DOCKER_IMAGE')]) {
                    dir("${env.JENKINS_SERVER_DIRECTORY_NAME}") {
                        sh 'docker build -t $DOCKER_IMAGE .'
                    }
                }
            }
        }

        stage('Testing the Docker image') {
            steps {
                withCredentials([string(credentialsId: 'DOCKER_IMAGE_NAME', variable: 'DOCKER_IMAGE')]) {
                    dir("${env.JENKINS_SERVER_DIRECTORY_NAME}") {
                        sh '''
                            docker run -itd -p 80:80 $DOCKER_IMAGE /bin/sh -c "netstat -antp | grep nginx || exit 1"
                            docker rm -f $(docker ps -aq)
                        '''
                    }
                }
            }
        }
        
        stage('Installing the AWS CLI if not installed already') {
            steps {
                sh """
                    if ! which aws &> /dev/null; then
                        curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
                        unzip -o awscliv2.zip
                        ./aws/install --bin-dir /usr/local/bin --install-dir /usr/local/aws-cli --update
                    else
                        aws --version
                    fi
                """
            }
        }

        stage('Deploying the Docker image to AWS ECR') {
            steps {
                withCredentials([
                    string(credentialsId: 'AWS_REGION', variable: 'REGION'),
                    string(credentialsId: 'AWS_ECR_REGISTRY', variable: 'ECR_REGISTRY'),
                    string(credentialsId: 'AWS_ECR_REPOSITORY', variable: 'ECR_REPO'),
                    usernamePassword(credentialsId: 'aws-credentials', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY'),
                    string(credentialsId: 'DOCKER_IMAGE_NAME', variable: 'DOCKER_IMAGE')
                ]) {
                    sh '''
                        aws ecr get-login-password --region $REGION | docker login --username AWS --password-stdin $ECR_REGISTRY
                        docker tag $DOCKER_IMAGE:latest $ECR_REGISTRY/$ECR_REPO:latest
                        docker push $ECR_REGISTRY/$ECR_REPO:latest
                    '''
                }
            }
        }
    }
}
