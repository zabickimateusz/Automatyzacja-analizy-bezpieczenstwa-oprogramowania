pipeline {
    agent any

    stages {
        stage('1. Pobieranie kodu') {
            steps {
                echo 'Pobieranie kodu z repozytorium GitHub...'
                checkout scm
            }
        }

        stage('2. Analiza SCA (Dependency-Check)') {
            steps {
                echo 'Uruchamianie skanera bibliotek (SCA)...'
                withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_KEY')]) {
                    dependencyCheck additionalArguments: "--format HTML --format XML --nvdApiKey ${NVD_KEY}", odcInstallation: 'dependency-check'
                }
            }
        }

        stage('3. Analiza SAST (SonarQube)') {
            steps {
                echo 'Przesyłanie kodu do analizy w SonarQube...'
                script {
                    def scannerHome = tool 'sonar-scanner'
                    withSonarQubeEnv('SonarQube') {
                        sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=dvwa-security-test -Dsonar.projectName='DVWA Security Test' -Dsonar.sources=."
                    }
                }
            }
        }

        stage('4. Bramka Jakosci (Quality Gate)') {
            steps {
                echo 'Weryfikacja wyniku skanowania...'
                timeout(time: 15, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('5. Wdrożenie na VM 3 (Docker Compose Remote)') {
            steps {
                script {
                    def remoteIP = "192.168.0.203"
                    def remoteUser = "root"
                    def sshArgs = "-o StrictHostKeyChecking=no"
                    
                    echo "Przygotowanie folderu na VM 3..."
                    sh "ssh ${sshArgs} ${remoteUser}@${remoteIP} 'mkdir -p /opt/dvwa'"
                    
                    echo "Kopiowanie plików źródłowych na VM 3..."
                    sh "scp ${sshArgs} -r . ${remoteUser}@${remoteIP}:/opt/dvwa/"

                    echo "Uruchamianie aplikacji i bazy danych przez Docker Compose..."
                    sh """
                        ssh ${sshArgs} ${remoteUser}@${remoteIP} "
                            cd /opt/dvwa && 
                            docker-compose down || true && 
                            docker-compose up -d --build
                        "
                    """
                }
            }
        }
    }

    post {
        success {
            echo 'System DevSecOps: Proces zakonczony sukcesem. Aplikacja wdrożona na VM 3.'
        }
        failure {
            echo 'System DevSecOps: Wykryto błędy lub luki bezpieczeństwa. Wstrzymano potok.'
        }
    }
}