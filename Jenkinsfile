pipeline {
    agent any

    stages {
        stage('1. Pobieranie kodu') {
            steps {
                echo 'Pobieranie kodu z GitHub...'
                checkout scm
            }
        }

        stage('2. Analiza SCA (Dependency-Check)') {
            steps {
                echo 'Sprawdzanie bibliotek...'
                withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_KEY')]) {
                    dependencyCheck additionalArguments: "--format HTML --format XML --nvdApiKey ${NVD_KEY}", odcInstallation: 'dependency-check'
                }
            }
        }

        stage('3. Analiza SAST (SonarQube)') {
            steps {
                echo 'Skanowanie kodu w SonarQube...'
                script {
                    def scannerHome = tool 'sonar-scanner'
                    withSonarQubeEnv('SonarQube') {
                        sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=dvwa-security-test -Dsonar.projectName='DVWA Security Test' -Dsonar.sources=."
                    }
                }
            }
        }

        stage('4. Bramka Jakosci') {
            steps {
                echo 'Czekanie na wynik z SonarQube...'
                timeout(time: 15, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }

        stage('5. Budowanie i wdrożenie (Docker)') {
            steps {
                echo 'Budowanie obrazu i uruchamianie kontenera...'
                // Budujemy obraz na podstawie Twojego Dockerfile
                sh 'docker build -t bezpieczne-dvwa:latest .'
                
                // Zatrzymujemy stary kontener, jeśli działa, i odpalamy nowy
                sh 'docker stop dvwa-app || true'
                sh 'docker rm dvwa-app || true'
                sh 'docker run -d --name dvwa-app -p 8081:80 bezpieczne-dvwa:latest'
                
                echo 'Aplikacja została wdrożona i jest dostępna na porcie 8081!'
            }
        }
    }
}