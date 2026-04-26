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
                echo 'Uruchamianie skanera bibliotek z kluczem API...'
                // Pobieranie sekretu nvd-api-key z magazynu Jenkinsa
                withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_KEY')]) {
                    dependencyCheck additionalArguments: "--format HTML --format XML --nvdApiKey ${NVD_KEY}", odcInstallation: 'dependency-check'
                }
            }
        }

        stage('3. Analiza SAST (SonarQube)') {
            steps {
                echo 'Przesyłanie kodu do SonarQube...'
                script {
                    // Definicja skanera
                    def scannerHome = tool 'sonar-scanner'
                    withSonarQubeEnv('SonarQube') {
                        sh "${scannerHome}/bin/sonar-scanner -Dsonar.projectKey=dvwa-security-test -Dsonar.projectName='DVWA Security Test' -Dsonar.sources=."
                    }
                }
            }
        }

        stage('4. Bramka Jakosci') {
            steps {
                echo 'Oczekiwanie na wynik z SonarQube...'
                timeout(time: 15, unit: 'MINUTES') {
                    waitForQualityGate abortPipeline: true
                }
            }
        }
    }
}