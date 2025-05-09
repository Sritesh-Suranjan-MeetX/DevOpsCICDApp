pipeline {
    agent any

    environment {
        // Android SDK configuration (update path for your EC2 instance)
        ANDROID_HOME = '/usr/local/android-sdk'
        PATH = "$ANDROID_HOME/cmdline-tools/bin:$ANDROID_HOME/platform-tools:$PATH"
        
        // Gradle cache configuration
        GRADLE_USER_HOME = "${env.WORKSPACE}/.gradle"
    }

    stages {
        // Stage 1: Checkout Code
        stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: 'main']],
                    extensions: [],
                    userRemoteConfigs: [[
                        url: 'https://github.com/Sritesh-Suranjan-MeetX/DevOpsCICDApp.git',
                        credentialsId: 'github-credentials' // Replace with your Jenkins credentials ID
                    ]]
                ])
            }
        }

        // Stage 2: Build Debug APK
        stage('Build Debug') {
            steps {
                sh './gradlew assembleDebug --no-daemon --stacktrace'
            }
        }

        // Stage 3: Run Unit Tests
        stage('Test') {
            steps {
                sh './gradlew testDebugUnitTest --no-daemon'
            }
            post {
                always {
                    junit 'app/build/test-results/**/*.xml'
                }
            }
        }

        // Stage 4: Build and Sign Release APK
        stage('Build Release') {
            steps {
                withCredentials([
                    file(credentialsId: 'RELEASE_KEYSTORE_FILE', variable: 'RELEASE_KEYSTORE'),
                    string(credentialsId: 'RELEASE_STORE_PASSWORD', variable: 'STORE_PASS'),
                    string(credentialsId: 'RELEASE_KEY_ALIAS', variable: 'KEY_ALIAS'),
                    string(credentialsId: 'RELEASE_KEY_PASSWORD', variable: 'KEY_PASS')
                ]) {
                    sh """
                        # Securely handle keystore
                        cp "$RELEASE_KEYSTORE" app/keystore.jks
                        ./gradlew assembleRelease \
                            -PreleaseKeystoreFile=app/keystore.jks \
                            -PreleaseStorePassword="$STORE_PASS" \
                            -PreleaseKeyAlias="$KEY_ALIAS" \
                            -PreleaseKeyPassword="$KEY_PASS"
                        # Clean up keystore immediately
                        rm -f app/keystore.jks
                    """
                }
            }
        }

        // Stage 5: Archive Artifacts
        stage('Archive') {
            steps {
                archiveArtifacts artifacts: 'app/build/outputs/**/*.apk', fingerprint: true
                archiveArtifacts artifacts: 'app/build/outputs/mapping/release/**/*', fingerprint: true
            }
        }

        // Stage 6: Optional Firebase Distribution
        stage('Deploy to Firebase') {
            when {
                branch 'main' // Only deploy from main branch
            }
            steps {
                withCredentials([string(credentialsId: 'FIREBASE_TOKEN', variable: 'FIREBASE_TOKEN')]) {
                    sh """
                        firebase appdistribution:distribute \
                            app/build/outputs/apk/release/app-release.apk \
                            --app 1:1234567890:android:abcdef123456 \
                            --groups "qa-team" \
                            --token "$FIREBASE_TOKEN"
                    """
                }
            }
        }
    }

    post {
        always {
            // Clean workspace to save disk space
            cleanWs()
            
            // Delete temporary keystore if still exists
            sh 'rm -f app/keystore.jks || true'
        }
        success {
            slackSend(
                color: 'good',
                message: """✅ Build Successful: ${env.JOB_NAME} #${env.BUILD_NUMBER}
                | *Debug APK*: ${env.BUILD_URL}artifact/app/build/outputs/apk/debug/
                | *Release APK*: ${env.BUILD_URL}artifact/app/build/outputs/apk/release/
                | *Test Report*: ${env.BUILD_URL}testReport/""".stripMargin()
            )
        }
        failure {
            slackSend(
                color: 'danger',
                message: """❌ Build Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}
                | *Logs*: ${env.BUILD_URL}console
                | *Test Report*: ${env.BUILD_URL}testReport/""".stripMargin()
            )
            
            emailext (
                subject: "FAILED: ${env.JOB_NAME} Build #${env.BUILD_NUMBER}",
                body: """Check the failed build:
                       |Build URL: ${env.BUILD_URL}
                       |Console Logs: ${env.BUILD_URL}console""".stripMargin(),
                to: 'devops-team@yourcompany.com'
            )
        }
    }
}
