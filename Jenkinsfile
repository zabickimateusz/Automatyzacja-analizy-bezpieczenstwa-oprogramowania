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

        stage('5. Wdrożenie na VM 3 (Docker Remote)') {
            steps {
                script {
                    def remoteIP = "192.168.0.203"
                    def remoteUser = "root"
                    
                    echo "Budowanie obrazu kontenera na maszynie Jenkins..."
                    sh "docker build -t bezpieczne-dvwa:latest ."

                    echo "Przesyłanie obrazu na VM 3..."
                    // Dodano -o StrictHostKeyChecking=no dla pelnej automatyzacji
                    sh "docker save bezpieczne-dvwa:latest | ssh -o StrictHostKeyChecking=no ${remoteUser}@${remoteIP} 'docker load'"

                    echo "Uruchamianie aplikacji na VM 3..."
                    sh """
                        ssh -o StrictHostKeyChecking=no ${remoteUser}@${remoteIP} "
                            docker stop dvwa-app || true && 
                            docker rm dvwa-app || true && 
                            docker run -d --name dvwa-app -p 80:80 bezpieczne-dvwa:latest
                        "
                    """
                }
            }
        }
    }

    post {
        success {
            echo 'System DevSecOps: Proces zakonczony sukcesem. Aplikacja wdrożona.'
        }
        failure {
            echo 'System DevSecOps: Wykryto błędy lub luki bezpieczeństwa. Wstrzymano potok.'
        }
    }
}