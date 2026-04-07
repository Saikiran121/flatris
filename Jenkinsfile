pipeline {
    agent { label 'agent' }

    stages {
        stage('Version Check') {
            steps {
                echo 'Verifying tool versions...'
                sh 'node -v'
                sh 'npm -v'
                sh 'yarn -v'
            }
        }

        stage('Install Dependencies') {
            steps {
                echo 'Installing project dependencies...'
                sh 'yarn install'
            }
        }

        stage('Dependency Check') {
            steps {
                echo 'Running OWASP Dependency-Check...'
                withCredentials([string(credentialsId: 'nvd-api-key', variable: 'NVD_API_KEY')]) {
                    dependencyCheck additionalArguments: '--nvdApiKey ' + NVD_API_KEY + ' --format HTML --scan . --exclude "**/node_modules/**" --exclude "**/bower_components/**" --disableAssembly', odcInstallation: 'dependency-check'
                }
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'

                archive includes: 'dependency-check-report.html'
            }
        }

        stage('Format Check') {
            steps {
                echo 'Checking code formatting...'
                sh "yarn run prettier --check '**/*.{js,css}' '!**/{flow-typed,.next,coverage}/**'"
            }
        }

        stage('Test') {
            steps {
                echo 'Running tests with coverage...'
                sh 'yarn test --coverage'
            }
        }

        stage('SonarQube Analysis') {
            steps {
                echo 'Performing SonarQube analysis...'
                withSonarQubeEnv('sonar-server') {
                    sh 'npx sonar-scanner -Dsonar.projectKey=flatris -Dsonar.sources=. -Dsonar.javascript.lcov.reportPaths=coverage/lcov.info'
                }
            }
        }

        stage('SonarQube Quality Gate') {
            steps {
                echo 'Waiting for SonarQube Quality Gate (Non-blocking)...'
                timeout(time: 1, unit: 'HOURS') {
                    
                    waitForQualityGate abortPipeline: false
                }
            }
        }

        stage('Docker Build') {
            steps {
                echo 'Building Docker image...'
                sh 'docker build -t saikiran8050/flatris:${BUILD_NUMBER} -t saikiran8050/flatris:latest .'
            }
        }

        stage('Trivy Scan') {
            steps {
                echo 'Performing Trivy container scan on both images...'

                sh 'trivy image --format json -o trivy-build-report.json saikiran8050/flatris:${BUILD_NUMBER}'
                
                sh 'trivy image saikiran8050/flatris:latest'
                
                archive includes: 'trivy-build-report.json'
            }
        }
    }

    post {
        always {
            echo 'Pipeline execution finished.'
        }
        success {
            echo 'Build and tests passed successfully!'
        }
        failure {
            echo 'Pipeline failed. Please check the logs.'
        }
    }
}
