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
                            sudo chmod 777 /app
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
                            docker rmi -f $(docker images -q) || true
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
                                mvnw clean install -P buildDocker
                                find . -type f -name "*.jar" -exec chmod 755 {} \\;

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
                        echo 'Exécution des tests Selenium et mise à jour du rapport...'
                        sh '''
                            set -x
                            . venv/bin/activate
                            ./venv/bin/pytest --html=report.html > /reports/selenium_tests.txt
                            cd
                            cd /reports
                            git pull origin master
                            git add selenium_tests.txt
                            git commit -m "Update Selenium test results"
                            git push origin master
                            cd
                            cd /app
                            ./scripts/pushImages.sh
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
