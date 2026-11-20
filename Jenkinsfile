pipeline {
    agent any
    environment {
        // Secure Registry now operates on HTTPS port 443
        REGISTRY_IP = '54.242.7.221' 
        REGISTRY_PORT = '443'
        IMAGE_NAME = 'sec-reg-ec2-node-app'
        TARGET_EC2_IP = '172.31.17.89' 
    }
    stages {
        stage('Checkout Code') {
            steps { checkout scm }
        }
        
        stage('Execute Tests') {
            agent {
                docker { 
                    image 'node:alpine'; 
                    args '-u root:root'; 
                    reuseNode true 
                } 
            }
            steps {
                sh 'npm ci'
                sh 'npm test' 
            }
        }
        
        stage('Build & Push to Secure Registry') {
            steps {
                withCredentials([usernamePassword(credentialsId: 'secure-registry-creds', passwordVariable: 'REG_PASS', usernameVariable: 'REG_USER')]) {
                    sh "docker build -t ${REGISTRY_IP}:${REGISTRY_PORT}/${IMAGE_NAME}:${env.BUILD_NUMBER} ."
                    
                    // Jenkins securely logs into the TLS registry
                    sh "echo \$REG_PASS | docker login ${REGISTRY_IP}:${REGISTRY_PORT} -u \$REG_USER --password-stdin"
                    
                    // Push the image over encrypted HTTPS
                    sh "docker push ${REGISTRY_IP}:${REGISTRY_PORT}/${IMAGE_NAME}:${env.BUILD_NUMBER}"
                }
            }
        }
        
        stage('Deploy to Target EC2') {
            steps {
                sshagent(['target-ec2-key']) {
                    withCredentials([usernamePassword(credentialsId: 'secure-registry-creds', passwordVariable: 'REG_PASS', usernameVariable: 'REG_USER')]) {
                        sh """
                            ssh -o StrictHostKeyChecking=no ubuntu@${TARGET_EC2_IP} '
                                # The Target EC2 instance logs in to authorize the pull
                                echo \$REG_PASS | docker login ${REGISTRY_IP}:${REGISTRY_PORT} -u \$REG_USER --password-stdin
                                
                                docker stop node-app || true
                                docker rm node-app || true
                                
                                docker pull ${REGISTRY_IP}:${REGISTRY_PORT}/${IMAGE_NAME}:${env.BUILD_NUMBER}
                                docker run -d -p 3000:3000 --name node-app ${REGISTRY_IP}:${REGISTRY_PORT}/${IMAGE_NAME}:${env.BUILD_NUMBER}
                                
                                # Clear active authentication tokens from the server
                                docker logout ${REGISTRY_IP}:${REGISTRY_PORT}
                            '
                        """
                    }
                }
            }
        }
    }
}
