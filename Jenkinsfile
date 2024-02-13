pipeline {
    agent any

    environment {
        DOCKER_CREDENTIALS = credentials('docker')
        REPOSITORY_PREFIX = 'ender0'
    }

    stages {
        stage('Preparation') {
            steps {
                script {
                    try {
                        echo 'Mise à jour du système et configuration Docker...'
                        sh '''
                            set -x
                            sudo apt update
                            sudo apt-get install python3.10-venv -y
                            sudo chmod 666 /var/run/docker.sock
                            sudo usermod -aG docker $USER
                        '''
                    } catch (Exception e) {
                        echo "Erreur lors de la préparation : ${e.getMessage()}"
                    }
                }
            }
        }

        stage('Setup /app Directory') {
            steps {
                script {
                    try {
                        echo 'Configuration du répertoire /app...'
                        sh '''
                            set -x
                            sudo rm -rf /app
                            sudo mkdir -p /app
                            sudo chown $USER:$USER /app
                        '''
                    } catch (Exception e) {
                        echo "Erreur lors de la configuration du répertoire /app : ${e.getMessage()}"
                    }
                }
            }
        }

        stage('Verify and Set Permissions for /app') {
            steps {
                script {
                    try {
                        echo 'Vérification et configuration des permissions pour /app...'
                        sh '''
                            set -x
                            ls -ld /app
                            sudo chmod -R 777 /app
                            ls -ld /app
                        '''
                    } catch (Exception e) {
                        echo "Erreur lors de la vérification / configuration des permissions : ${e.getMessage()}"
                    }
                }
            }
        }

        stage('Setup Python Environment') {
            steps {
                script {
                    try {
                        echo 'Configuration de l\'environnement virtuel Python...'
                        sh '''
                            set -x
                            python3 -m venv venv
                            . venv/bin/activate
                            pip install pytest selenium webdriver_manager faker pytest-html
                        '''
                    } catch (Exception e) {
                        echo "Erreur lors de la configuration de l'environnement virtuel Python : ${e.getMessage()}"
                    }
                }
            }
        }


        stage('Clean Docker') {
            steps {
                script {
                    try {
                        echo 'Nettoyage des conteneurs et images Docker...'
                        sh '''
                            set -x
                            docker stop $(docker ps -aq) || true
                            docker rm -f $(docker ps -aq) || true
                            # docker rmi -f $(docker images -q) || true
                        '''
                    } catch (Exception e) {
                        echo "Erreur lors du nettoyage Docker : ${e.getMessage()}"
                    }
                }
            }
        }

        stage('Clone Repository') {
            steps {
                script {
                    try {
                        echo 'Clonage du dépôt Git...'
                        sh '''
                            set -x
                            git clone https://github.com/aineops/CI-pet.git /app
                            sudo chown -R $USER:$USER /app
                            sudo find /app -type d -exec chmod 755 {} \\;
                            sudo find /app -type f -exec chmod 644 {} \\;
                        '''
                    } catch (Exception e) {
                        echo "Erreur lors du clonage du dépôt : ${e.getMessage()}"
                    }
                }
            }
        }

        stage('Build and Push Docker Images') {
            steps {
                script {
                    try {
                        echo 'Construction et déploiement des images Docker...'
                        withCredentials([usernamePassword(credentialsId: 'docker', usernameVariable: 'DOCKER_USER', passwordVariable: 'DOCKER_PASSWORD')]) {
                            sh '''
                                set -x
                                echo $DOCKER_PASSWORD | docker login -u $DOCKER_USER --password-stdin
                                cd /app
                                sudo chown $USER:$USER /app
                                sudo usermod -aG docker $USER
                                mvn clean
                                chmod +x mvnw
                                ./mvnw clean install -P buildDocker -DskipTests
                                docker images --format "{{.Repository}}:{{.Tag}}" | grep 'springcommunity' | while read image; do
                                    echo "Image originale: $image"
                                    new_image=$(echo $image | sed "s/springcommunity/ender0/")
                                    echo "Nouvelle image: $new_image"
                                    if [ ! -z "$new_image" ]; then
                                        docker tag $image $new_image
                                        docker rmi $image
                                    else
                                        echo "Erreur: Le nom de la nouvelle image est vide."
                                    fi
                                done
                                ./pushImages.sh

                            '''
                        }
                    } catch (Exception e) {
                        echo "Erreur lors de la construction et du déploiement : ${e.getMessage()}"
                    }
                }
            }
        }
        
        stage('Deploy Services') {
            steps {
                script {
                    try {
                        echo 'Déploiement des services...'
                        sh '''
                            set -x
                            cd /app
                            docker-compose up -d --build
                             # Vérification que tous les conteneurs sont en cours d'exécution
                            while [ "$(docker-compose ps -q | wc -l)" != "$(docker ps -q | wc -l)" ]; do
                                echo "En attente que tous les conteneurs soient lancés..."
                                sleep 10
                            done
                        '''
                    } catch (Exception e) {
                        echo "Erreur lors du déploiement des services : ${e.getMessage()}"
                    }
                }
            }
        }
        stage('Run Selenium Tests and Update Report') {
            steps {
                script {
                    try {
                        echo 'Vérification de la disponibilité de localhost:8080...'
                        sh '''
                            #!/bin/bash
                            set -x
                            max_attempts=30
                            for ((i=1;i<=max_attempts;i++)); do
                                if curl -s http://localhost:8080; then
                                    echo "localhost:8080 est accessible. Exécution des tests Selenium."
                                    break
                                else
                                    echo "Attente de la disponibilité de localhost:8080... tentative $i de $max_attempts"
                                    sleep 10
                                fi
                            done
        
                            if [ $i -gt $max_attempts ]; then
                                echo "Échec : localhost:8080 n'est pas accessible après $max_attempts tentatives."
                                exit 1
                            fi
                        '''
        
                        echo 'Exécution des tests Selenium et mise à jour du rapport...'
                        sh '''
                            . venv/bin/activate
                            ./venv/bin/pytest --html=report.html > /reports/selenium_tests.txt
                            cd /reports
                            git pull origin master
                            git add selenium_tests.txt
                            git commit -m "Update Selenium test results"
                            git push origin master
                        '''
                    } catch (Exception e) {
                        echo "Erreur lors des tests Selenium : ${e.getMessage()}"
                    }
                }
            }
        }
    }

    post {
        always {
            echo "Le processus est terminé"
        }
    }
}
