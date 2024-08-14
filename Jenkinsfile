pipeline {
    agent any

    stages {
        stage('Cloning the GitHub repository') {
            steps {
                withCredentials([
                    string(credentialsId: 'GITHUB_REPOSITORY', variable: 'GITHUB_REPOSITORY'),
                    string(credentialsId: 'JENKINS_SERVER_DIRECTORY_NAME', variable: 'JENKINS_SERVER_DIRECTORY_NAME')
                ]) {
                    script {
                        sh '''
                            if [ ! -d "$JENKINS_SERVER_DIRECTORY_NAME" ]; then
                                git clone "$GITHUB_REPOSITORY" "$JENKINS_SERVER_DIRECTORY_NAME"
                            else
                                cd "$JENKINS_SERVER_DIRECTORY_NAME" && git pull
                            fi
                        '''
                    }
                }
            }
        }
        
        stage('Logging into DockerHub') {
            steps {
                withCredentials([
                    string(credentialsId: 'DOCKER_ID', variable: 'DOCKER_ID'),
                    string(credentialsId: 'DOCKER_PASSWORD', variable: 'DOCKER_PASSWORD')
                ]) {
                    sh 'echo $DOCKER_PASSWORD | docker login -u $DOCKER_ID --password-stdin'
                }
            }
        }

        stage('Building the Docker image') {
            steps {
                withCredentials([
                    string(credentialsId: 'DOCKER_IMAGE_NAME', variable: 'DOCKER_IMAGE_NAME')
                ]) {
                    dir("${env.JENKINS_SERVER_DIRECTORY_NAME}") {
                        sh 'docker build -t $DOCKER_IMAGE_NAME .'
                    }
                }
            }
        }

        stage('Testing the Docker image') {
            steps {
                withCredentials([
                    string(credentialsId: 'DOCKER_IMAGE_NAME', variable: 'DOCKER_IMAGE_NAME')
                ]) {
                    dir("${env.JENKINS_SERVER_DIRECTORY_NAME}") {
                        sh '''
                            docker run -itd -p 80:80 $DOCKER_IMAGE_NAME /bin/sh -c "netstat -antp | grep nginx || exit 1"
                            docker rm -f $(docker ps -aq)
                        '''
                    }
                }
            }
        }

        stage('Installing the AWS CLI if not installed already') {
            steps {
                sh '''
                    if ! which aws &> /dev/null; then
                        curl "https://awscli.amazonaws.com/awscli-exe-linux-x86_64.zip" -o "awscliv2.zip"
                        unzip -o awscliv2.zip
                        ./aws/install --bin-dir /usr/local/bin --install-dir /usr/local/aws-cli --update
                    else
                        aws --version
                    fi
                '''
            }
        }

        stage('Deploying the Docker image to AWS ECR') {
            steps {
                withCredentials([
                    string(credentialsId: 'AWS_REGION', variable: 'AWS_REGION'),
                    string(credentialsId: 'AWS_ECR_REGISTRY', variable: 'AWS_ECR_REGISTRY'),
                    string(credentialsId: 'AWS_ECR_REPOSITORY', variable: 'AWS_ECR_REPOSITORY'),
                    usernamePassword(credentialsId: 'aws-credentials', usernameVariable: 'AWS_ACCESS_KEY_ID', passwordVariable: 'AWS_SECRET_ACCESS_KEY'),
                    string(credentialsId: 'DOCKER_IMAGE_NAME', variable: 'DOCKER_IMAGE_NAME')
                ]) {
                    sh '''
                        aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $AWS_ECR_REGISTRY
                        docker tag $DOCKER_IMAGE_NAME:latest $AWS_ECR_REGISTRY/$AWS_ECR_REPOSITORY:latest
                        docker push $AWS_ECR_REGISTRY/$AWS_ECR_REPOSITORY:latest
                    '''
                }
            }
        }
    }
}
