# 📘 Jenkins Challenge

Challenge :

1. Reverse Proxy frontend dan backend di handle di dalam jenkins
2. Menambahkan testing menggunakan spider di step Jenkinsfile ( Reference : [wget spider](https://www.labnol.org/software/wget-command-examples/28750/) )

---

## Reverse Proxy fe, be di handle Jenkins

Jenkinsfile Front-end:

```bash
def secret = 'totywan-vps'
def server = 'totywan@103.127.137.206'
def directory = '/home/totywan/dumbways-app/wayshub-frontend'
def branch = 'main'
def image = 'totywan/wayshub-frontend13:1.0'

pipeline {
    agent any
    environment {
        DOCKERHUB_CRED = 'totywan-dockerhub'
    }
    stages {
        stage ('Pull Latest Code from GitHub') {
            steps {
                sshagent([secret]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${server} << EOF
                            cd ${directory}
                            git pull origin ${branch}
                            echo "✅ Pulled latest code"
                            exit
                        EOF
                    """
                }
            }
        }

        stage ('Build Docker Image on VPS') {
            steps {
                sshagent([secret]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${server} << EOF
                            cd ${directory}
                            docker build -t ${image} .
                            echo "✅ Image built successfully"
                            exit
                        EOF
                    """
                }
            }
        }


        stage ('Push to Docker Hub') {
            steps {
                sshagent([secret]) {
                    withCredentials([usernamePassword(credentialsId: "${DOCKERHUB_CRED}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh """
                            ssh -o StrictHostKeyChecking=no ${server} << EOF
                                echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                                docker push ${image}
                                echo "📦 Image pushed ke Docker Hub"
                                exit
                            EOF
                        """
                    }
                }
            }
        }

        stage ('Deploy Container on VPS') {
            steps {
                sshagent([secret]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${server} << EOF
                            docker stop wayshub-fe || true
                            docker rm wayshub-fe || true
                            docker run -d --name wayshub-fe -p 3000:3000 ${image}
                            echo "🚀 Deployed frontend!"
                            exit
                        EOF
                    """
                }
            }
        }
    }
}
```

Jenkinsfile Back-end:

```bash
def secret = 'totywan-vps'
def server = 'totywan@103.127.137.206'
def directory = '/home/totywan/dumbways-app/wayshub-backend'
def branch = 'main'
def image = 'totywan/wayshub-backend13:1.0'

pipeline {
    agent any
    environment {
        DOCKERHUB_CRED = 'totywan-dockerhub'
    }
    stages {
        stage ('Pull Latest Code from GitHub') {
            steps {
                sshagent([secret]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${server} << EOF
                            cd ${directory}
                            git pull origin ${branch}
                            echo "✅ Pulled latest code"
                            exit
                        EOF
                    """
                }
            }
        }

        stage ('Build Docker Image on VPS') {
            steps {
                sshagent([secret]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${server} << EOF
                            cd ${directory}
                            docker build -t ${image} .
                            echo "✅ Image built successfully"
                            exit
                        EOF
                    """
                }
            }
        }


        stage ('Push to Docker Hub') {
            steps {
                sshagent([secret]) {
                    withCredentials([usernamePassword(credentialsId: "${DOCKERHUB_CRED}", usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASS')]) {
                        sh """
                            ssh -o StrictHostKeyChecking=no ${server} << EOF
                                echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
                                docker push ${image}
                                echo "📦 Image pushed ke Docker Hub"
                                exit
                            EOF
                        """
                    }
                }
            }
        }

        stage ('Deploy Container on VPS') {
            steps {
                sshagent([secret]) {
                    sh """
                        ssh -o StrictHostKeyChecking=no ${server} << EOF
                            docker stop wayshub-be || true
                            docker rm wayshub-be || true
                            docker run -d --name wayshub-be -p 5000:5000 ${image}
                            echo "🚀 Deployed Backend!"
                            exit
                        EOF
                    """
                }
            }
        }
    }
}
```

---

---
