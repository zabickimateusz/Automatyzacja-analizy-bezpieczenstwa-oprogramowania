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
                echo 'Uruchamianie skanera bibliotek (SCA) z kluczem NVD API...'
                // Pobieranie klucza API z Credentiali Jenkinsa (ID: nvd-api-key)
                withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_KEY')]) {
                    dependencyCheck additionalArguments: "--format HTML --format XML --nvdApiKey ${NVD_KEY}", odcInstallation: 'dependency-check'
                }
            }
        }

        stage('3. Analiza SAST (SonarQube)') {
            steps {
                echo 'Przesyłanie kodu do analizy statycznej w SonarQube...'
                script {
                    // Wykorzystanie zainstalowanego narzędzia sonar-scanner
                    def scannerHome = tool 'sonar-scanner'
                    withSonarQubeEnv('SonarQube') {
                        sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=dvwa-security-test -Dsonar.projectName='DVWA Security Test' -Dsonar.sources=."
                    }
                }
            }
        }

        stage('4. Bramka Jakosci (Quality Gate)') {
            steps {
                echo 'Oczekiwanie na wynik analizy bezpieczeństwa...'
                timeout(time: 15, unit: 'MINUTES') {
                    // Potok zostanie przerwany, jeśli Sonar wyśle status ERROR
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('5. Wdrożenie na VM 3 (Docker Remote)') {
            steps {
                script {
                    // --- KONFIGURACJA ---
                    def remoteIP = "192.168.0.203" // <--- TUTAJ WPISZ IP TWOJEJ 3. MASZYNY
                    def remoteUser = "root"        // Użytkownik na 3. maszynie
                    
                    echo "Budowanie obrazu kontenera na maszynie Jenkins..."
                    sh "docker build -t bezpieczne-dvwa:latest ."

                    echo "Przesyłanie obrazu na maszynę docelową ${remoteIP}..."
                    // Przesyłamy obraz "w locie" przez SSH i ładujemy go do Dockera na VM 3
                    sh "docker save bezpieczne-dvwa:latest | ssh ${remoteUser}@${remoteIP} 'docker load'"

                    echo "Uruchamianie nowej wersji aplikacji na VM 3..."
                    // Zatrzymujemy stary kontener i odpalamy nowy na porcie 80
                    sh """
                        ssh ${remoteUser}@${remoteIP} "
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
            echo 'Sukces! Aplikacja została przeskanowana i wdrożona na VM 3.'
        }
        failure {
            echo 'Błąd! Potok został przerwany (prawdopodobnie przez błędy bezpieczeństwa).'
        }
    }
}