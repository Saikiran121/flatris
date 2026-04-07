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
                    dependencyCheck additionalArguments: "--nvdApiKey ${NVD_API_KEY} --format HTML --scan .", odcInstallation: 'dependency-check'
                }
                dependencyCheckPublisher pattern: '**/dependency-check-report.xml'

                publishHTML([
                    allowMissing: false,
                    alwaysLinkToLastBuild: true,
                    keepAll: true,
                    reportDir: '.',
                    reportFiles: 'dependency-check-report.html',
                    reportName: 'Dependency Check HTML Report'
                ])
            }
        }

        stage('Format Check') {
            steps {
                echo 'Checking code formatting...'
                sh "yarn run prettier --check '**/*.{js,css}' '!**/{flow-typed,.next}/**'"
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
