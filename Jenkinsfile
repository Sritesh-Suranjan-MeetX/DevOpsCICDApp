pipeline {
    agent any

    environment {
        ANDROID_HOME = '//home/jenkins/android-sdk'
        PATH = "$ANDROID_HOME/cmdline-tools/latest/bin:$ANDROID_HOME/platform-tools:$PATH"
        GRADLE_USER_HOME = "${env.WORKSPACE}/.gradle"
    }

    stages {
        stage('Checkout') {
            steps {
                checkout([
                    $class: 'GitSCM',
                    branches: [[name: 'main']],
                    extensions: [],
                    userRemoteConfigs: [[
                        url: 'https://github.com/Sritesh-Suranjan-MeetX/DevOpsCICDApp.git',
                        credentialsId: 'github-credentials'
                    ]]
                ])
            }
        }

        stage('Build Debug') {
            steps {
                sh '''
                    echo "sdk.dir=$ANDROID_HOME" > local.properties
                    chmod +x gradlew
                    ./gradlew assembleDebug --no-daemon --stacktrace
                '''
            }
        }

        stage('Test') {
            steps {
                sh '''
                    echo "sdk.dir=$ANDROID_HOME" > local.properties
                    ./gradlew testDebugUnitTest --no-daemon
                '''
            }
            post {
                always {
                    junit '**/build/test-results/**/*.xml'  // Specify a more general path
                    archiveArtifacts artifacts: '**/build/test-results/**/*.xml', allowEmptyArchive: true
                }
            }
        }

        stage('Build Release') {
            steps {
                withCredentials([
                    file(credentialsId: 'RELEASE_KEYSTORE_FILE', variable: 'RELEASE_KEYSTORE'),
                    string(credentialsId: 'RELEASE_STORE_PASSWORD', variable: 'STORE_PASS'),
                    string(credentialsId: 'RELEASE_KEY_ALIAS', variable: 'KEY_ALIAS'),
                    string(credentialsId: 'RELEASE_KEY_PASSWORD', variable: 'KEY_PASS')
                ]) {
                    sh """
                        echo "sdk.dir=$ANDROID_HOME" > local.properties
                        chmod +x gradlew
                        cp "$RELEASE_KEYSTORE" app/keystore.jks
                        ./gradlew assembleRelease \
                            -PreleaseKeystoreFile=app/keystore.jks \
                            -PreleaseStorePassword="$STORE_PASS" \
                            -PreleaseKeyAlias="$KEY_ALIAS" \
                            -PreleaseKeyPassword="$KEY_PASS"
                        rm -f app/keystore.jks
                    """
                }
            }
        }

        stage('Archive') {
            steps {
                archiveArtifacts artifacts: '**/build/outputs/apk/**/*.apk', fingerprint: true  // More specific path
                archiveArtifacts artifacts: '**/build/outputs/mapping/release/**/*', fingerprint: true  // More specific path
            }
        }

        stage('Deploy to Firebase') {
            when {
                branch 'main'
            }
            steps {
                withCredentials([string(credentialsId: 'FIREBASE_TOKEN', variable: 'FIREBASE_TOKEN')]) {
                    sh """
                        firebase appdistribution:distribute \
                            app/build/outputs/apk/release/app-release.apk \
                            --app 1:346542465534:android:32cbd1e9d400a246f060af \
                            --groups "qa-team" \
                            --token "$FIREBASE_TOKEN"
                    """
                }
            }
        }
    }

    post {
        always {
            cleanWs()
            sh 'rm -f app/keystore.jks || true'
        }
        success {
            slackSend(
                color: 'good',
                message: """✅ Build Successful: ${env.JOB_NAME} #${env.BUILD_NUMBER}
                | Debug APK: ${env.BUILD_URL}artifact/app/build/outputs/apk/debug/
                | Release APK: ${env.BUILD_URL}artifact/app/build/outputs/apk/release/
                | Test Report: ${env.BUILD_URL}testReport/""".stripMargin()
            )
        }
        failure {
            slackSend(
                color: 'danger',
                message: """❌ Build Failed: ${env.JOB_NAME} #${env.BUILD_NUMBER}
                | Logs: ${env.BUILD_URL}console
                | Test Report: ${env.BUILD_URL}testReport/""".stripMargin()
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
