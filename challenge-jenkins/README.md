# 📘 Jenkins Challenge

Challenge :

1. Reverse Proxy frontend dan backend di handle di dalam jenkins
2. Menambahkan testing menggunakan spider di step Jenkinsfile ( Reference : [wget spider](https://www.labnol.org/software/wget-command-examples/28750/) )

---

## Reverse Proxy fe, be di handle Jenkins

Tambahkan stage berikut di Jenkinsfile sebelum deploy container.

- Frontend

```bash
stage ('Configure Nginx Reverse Proxy') {
    steps {
        sshagent([secret]) {
            sh """
                ssh ${server} << EOF
                    cat <<EOC | sudo tee /etc/nginx/sites-available/wayshub-fe.conf > /dev/null
server {
    server_name hermanto.studentdumbways.my.id;

    location / {
        proxy_pass http://103.127.137.206:3000;
    }

    listen 443 ssl;
    ssl_certificate /etc/letsencrypt/live/hermanto.studentdumbways.my.id/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/hermanto.studentdumbways.my.id/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
}

server {
    listen 80;
    server_name hermanto.studentdumbways.my.id;
    return 301 https://\$host\$request_uri;
}
EOC

                    sudo ln -sf /etc/nginx/sites-available/wayshub-fe.conf /etc/nginx/sites-enabled/wayshub-fe.conf
                    sudo nginx -t && sudo systemctl reload nginx
                    echo "✅ Nginx reverse proxy updated by Jenkins"
                    exit
                EOF
            """
        }
    }
}
```

- Back-end:

````bash
```bash
stage ('Configure Nginx Reverse Proxy') {
    steps {
        sshagent([secret]) {
            sh """
                ssh ${server} << EOF
                    cat <<EOC | sudo tee /etc/nginx/sites-available/wayshub-fe.conf > /dev/null
server {
    server_name api.hermanto.studentdumbways.my.id;

    location / {
        proxy_pass http://103.127.137.206:5000;
    }

    listen 443 ssl;
    ssl_certificate /etc/letsencrypt/live/hermanto.studentdumbways.my.id/fullchain.pem;
    ssl_certificate_key /etc/letsencrypt/live/hermanto.studentdumbways.my.id/privkey.pem;
    include /etc/letsencrypt/options-ssl-nginx.conf;
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem;
}

server {
    listen 80;
    server_name hermanto.studentdumbways.my.id;
    return 301 https://\$host\$request_uri;
}
EOC

                    sudo ln -sf /etc/nginx/sites-available/wayshub-fe.conf /etc/nginx/sites-enabled/wayshub-fe.conf
                    sudo nginx -t && sudo systemctl reload nginx
                    echo "✅ Nginx reverse proxy updated by Jenkins"
                    exit
                EOF
            """
        }
    }
}
````

---

## Menambahkan Testing Spider di Jenkinsfile

###### Jenkinsfile Front-end:

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
                        EOF
                    """
                }
            }
        }

        stage ('Build Docker Image') {
            steps {
                sshagent([secret]) {
                    sh """
                        ssh ${server} << EOF
                            cd ${directory}
                            docker build -t ${image} .
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
                            ssh ${server} << EOF
                                echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin
                                docker push ${image}
                            EOF
                        """
                    }
                }
            }
        }

        stage ('Deploy Container') {
            steps {
                sshagent([secret]) {
                    sh """
                        ssh ${server} << EOF
                            docker stop wayshub-fe || true
                            docker rm wayshub-fe || true
                            docker run -d --name wayshub-fe -p 3000:3000 ${image}
                        EOF
                    """
                }
            }
        }

        stage ('Test Deployment (Spider)') {
            steps {
                sshagent([secret]) {
                    sh """
                        ssh ${server} << EOF
                            wget --spider --no-check-certificate -q https://hermanto.studentdumbways.my.id || exit 1
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
                        EOF
                    """
                }
            }
        }

        stage ('Build Docker Image') {
            steps {
                sshagent([secret]) {
                    sh """
                        ssh ${server} << EOF
                            cd ${directory}
                            docker build -t ${image} .
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
                            ssh ${server} << EOF
                                echo \$DOCKER_PASS | docker login -u \$DOCKER_USER --password-stdin
                                docker push ${image}
                            EOF
                        """
                    }
                }
            }
        }

        stage ('Deploy Container') {
            steps {
                sshagent([secret]) {
                    sh """
                        ssh ${server} << EOF
                            docker stop wayshub-be || true
                            docker rm wayshub-be || true
                            docker run -d --name wayshub-be --network wayshub-network -p 5000:5000 ${image}
                        EOF
                    """
                }
            }
        }

        stage ('Test Deployment (Spider)') {
            steps {
                sshagent([secret]) {
                    sh """
                        ssh ${server} << EOF
                            wget --spider --no-check-certificate -q https://api.hermanto.studentdumbways.my.id || exit 1
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
