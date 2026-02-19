pipeline {
    agent any

    tools {
        gradle 'Gradle-8.5'
        jdk 'JDK-21'
    }

    environment {
        SONAR_HOST_URL = 'http://localhost:9000'

        // Retrieve the "sonar-token" secret from Jenkins credentials
        SONAR_TOKEN = credentials('sonar-token')

        MAVEN_REPO_USER = 'myMavenRepo'
        MAVEN_REPO_PASSWORD = credentials('maven-repo-password')

        SLACK_CHANNEL = '#dev-notifications'
        EMAIL_RECIPIENTS = 'ma_allag@esi.dz'
    }

    stages {
        stage('Checkout') {
            steps {
                checkout scm
                echo 'Code source récupéré.'
            }
        }

        stage('Test') {
            steps {
                script {
                    echo '--- Lancement des Tests ---'
                    catchError(buildResult: 'UNSTABLE', stageResult: 'FAILURE') {
                        sh 'gradle clean test --no-daemon'
                    }

                    // Publish test results
                    junit testResults: '**/build/test-results/test/*.xml', allowEmptyResults: true

                    publishHTML([
                        allowMissing: true,
                        alwaysLinkToLastBuild: true,
                        keepAll: true,
                        reportDir: 'build/reports/tests/test',
                        reportFiles: 'index.html',
                        reportName: 'Test Report'
                    ])
                }
            }
        }

        stage('Code Analysis') {
            steps {
                script {
                    withSonarQubeEnv('SonarQube') {
                        sh '''
                            mkdir -p build/empty_dir_for_sonar

                            gradle sonar --no-daemon \
                            -Dsonar.projectKey=tp7-allag \
                            -Dsonar.projectName="TP7 Java Project - Allag" \
                            -Dsonar.host.url=http://localhost:9000 \
                            -Dsonar.token=$SONAR_TOKEN \
                            -Dsonar.java.binaries=build/empty_dir_for_sonar \
                            -Dsonar.java.source=1.8 \
                            -Dsonar.skipCompile=true
                        '''
                    }
                }
            }
        }

        stage('Quality Gate') {
            steps {
                script {
                    timeout(time: 5, unit: 'MINUTES') {
                        def qg = waitForQualityGate()
                        if (qg.status != 'OK') {
                            error "Quality Gate failed: ${qg.status}"
                        }
                    }
                }
            }
        }

        stage('Build') {
            steps {
                echo '--- Création du JAR ---'
                sh 'gradle build -x test --no-daemon'

                // Generate Javadoc
                sh 'gradle javadoc --no-daemon'

                // Archive JAR and documentation
                archiveArtifacts artifacts: '**/build/libs/*.jar', fingerprint: true, allowEmptyArchive: true
                archiveArtifacts artifacts: '**/build/docs/javadoc/**', allowEmptyArchive: true
            }
        }

        stage('Deploy') {
            steps {
                echo 'Publishing artifact to MyMavenRepo'
                sh 'gradle publish --no-daemon'
            }
        }
    }

    post {
        always {
            script {
                echo '🧹 Nettoyage du workspace...'
                cleanWs()
            }
        }
        success {
            script {
                // Slack notification on success
                withCredentials([string(credentialsId: 'SLACK_WEBHOOK', variable: 'SLACK_WEBHOOK_URL')]) {
                    sh """
                        curl -X POST -H 'Content-type: application/json' \
                        --data '{"text":"SUCCESS - ${env.JOB_NAME} #${env.BUILD_NUMBER} ${env.BUILD_URL}"}' \
                        "\$SLACK_WEBHOOK_URL"
                    """
                }

                // Email notification on success
                try {
                    emailext (
                        subject: "Succès - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                        body: "Le pipeline a réussi. Voir: ${env.BUILD_URL}",
                        to: "${env.EMAIL_RECIPIENTS}"
                    )
                } catch (Exception e) {
                    echo "Notification Email ignorée."
                }
            }
        }
        failure {
            script {
                // Email notification on failure
                try {
                    emailext (
                        subject: "Échec - ${env.JOB_NAME} #${env.BUILD_NUMBER}",
                        body: "Vérifier la console: ${env.BUILD_URL}",
                        to: "${env.EMAIL_RECIPIENTS}"
                    )
                } catch (Exception e) {
                    echo "Notification Email ignorée."
                }

                // Slack notification on failure
                try {
                    slackSend(channel: "${env.SLACK_CHANNEL}", color: 'danger', message: "Échec: ${env.JOB_NAME} #${env.BUILD_NUMBER}")
                } catch (Exception e) {
                    echo "Notification Slack ignorée."
                }
            }
        }
    }
}
