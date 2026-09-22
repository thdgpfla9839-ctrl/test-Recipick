pipeline {

    agent any

    environment {

        // =====================================================
        // Docker Hub
        // =====================================================
        IMAGE_NAME = "thdgpfla5659/recipick:latest"

        // =====================================================
        // AWS EC2
        // =====================================================
        EC2_USER = "ubuntu"
        EC2_HOST = "3.37.89.185"
        EC2_APP_DIR = "/home/ubuntu/recipick"
    }

    stages {

        stage('Git Checkout') {
            steps {
                echo "===== Git Checkout ====="
                checkout scm
            }
        }

        stage('Gradle Build') {
            steps {
                sh '''
                    chmod +x gradlew
                    ./gradlew clean build -x test
                    ls -al build/libs
                '''
            }
        }

        stage('Docker Build') {
            steps {
                sh '''
                    docker build -t ${IMAGE_NAME} .
                    docker images | grep recipick
                '''
            }
        }

        stage('Docker Hub Push') {
            steps {
                withCredentials([
                    usernamePassword(
                        credentialsId: 'dockerhub',
                        usernameVariable: 'DOCKER_USERNAME',
                        passwordVariable: 'DOCKER_PASSWORD'
                    )
                ]) {
                    sh '''
                        echo "$DOCKER_PASSWORD" | docker login -u "$DOCKER_USERNAME" --password-stdin
                        docker push ${IMAGE_NAME}
                        docker logout
                    '''
                }
            }
        }

        stage('Copy Deploy Files to EC2') {
            steps {
                sshagent(['ec2-ssh']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} "mkdir -p ${EC2_APP_DIR}"

                        scp -o StrictHostKeyChecking=no docker-compose.yml \
                            ${EC2_USER}@${EC2_HOST}:${EC2_APP_DIR}/docker-compose.yml

                        scp -o StrictHostKeyChecking=no nginx.conf \
                            ${EC2_USER}@${EC2_HOST}:${EC2_APP_DIR}/nginx.conf
                    '''
                }
            }
        }

        stage('Create EC2 .env') {
            steps {
                withCredentials([
                    string(credentialsId: 'sist_url', variable: 'SIST_URL'),
                    string(credentialsId: 'sist_username', variable: 'SIST_USERNAME'),
                    string(credentialsId: 'sist_password', variable: 'SIST_PASSWORD'),
                    string(credentialsId: 'sist_post_url', variable: 'SIST_POST_URL'),
                    string(credentialsId: 'post_username', variable: 'POST_USERNAME'),
                    string(credentialsId: 'post_password', variable: 'POST_PASSWORD'),
                    string(credentialsId: 'gen_key', variable: 'GEN_KEY'),
                    string(credentialsId: 'img_key', variable: 'IMG_KEY'),
                    string(credentialsId: 'social_key', variable: 'SOCIAL_KEY'),
                    string(credentialsId: 'jwt_secret', variable: 'JWT_SECRET'),
                    string(credentialsId: 'mail_username', variable: 'MAIL_USERNAME'),
                    string(credentialsId: 'mail_password', variable: 'MAIL_PASSWORD')
                ]) {
                    sshagent(['ec2-ssh']) {
                        sh '''
                            ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} "cat > ${EC2_APP_DIR}/.env <<EOF
SPRING_PROFILES_ACTIVE=prod
sist_url=${SIST_URL}
sist_username=${SIST_USERNAME}
sist_password=${SIST_PASSWORD}
sist_post_url=${SIST_POST_URL}
post_username=${POST_USERNAME}
post_password=${POST_PASSWORD}
GEN_KEY=${GEN_KEY}
IMG_KEY=${IMG_KEY}
social_key=${SOCIAL_KEY}
jwt_secret=${JWT_SECRET}
mail_username=${MAIL_USERNAME}
mail_password=${MAIL_PASSWORD}
EOF
chmod 600 ${EC2_APP_DIR}/.env
"
                        '''
                    }
                }
            }
        }

        stage('Deploy to EC2') {
            steps {
                sshagent(['ec2-ssh']) {
                    sh '''
                        ssh -o StrictHostKeyChecking=no ${EC2_USER}@${EC2_HOST} << EOF

                            cd ${EC2_APP_DIR}

                            docker compose config

                            docker pull ${IMAGE_NAME}

                            docker compose ps

                            docker compose up -d --scale app=2

                            docker compose ps

                            sleep 30

                            docker compose ps

                            docker exec nginx nginx -s reload || true

EOF
                    '''
                }
            }
        }
    }

    post {
        success {
            echo "===== 배포 성공 ====="
        }
        failure {
            echo "===== 배포 실패 ====="
        }
    }
}