pipeline {
  agent any
  environment {
    start_message = "STARTED: Job ${env.JOB_NAME} [${env.BUILD_NUMBER}]\n<${env.BUILD_URL}|Open>"
    successful_message = "SUCCESSFUL: Job ${env.JOB_NAME} [${env.BUILD_NUMBER}]\n<${env.BUILD_URL}|Open>"
    slack_token = credentials('slack.token')
  }
  stages {
    stage('Checkout') {
      steps {
        parallel(
          "Checkout": {
            checkout scm
          },
          "Send Notification": {
            script {
                if (env.BRANCH_NAME.startsWith('PR')){
                    start_message = "${start_message}\n${env.CHANGE_AUTHOR} want to merge into ${env.CHANGE_TARGET}\nTitle: ${env.CHANGE_TITLE}"
                }
                sh 'printenv'
                slackSend(message: start_message, channel: 'istqb_plus', color: '#FFFF00', teamDomain: 'innovatube', token: slack_token)
            }
          }
        )
      }
    }
    stage('Build Debug') {
      steps {
          sh 'gradle assembleDebug'
      }
    }
    stage('Build Release') {
      when {
        expression {
            return env.BRANCH_NAME == 'master'
        }
      }
      steps {
          sh 'gradle assembleRelease'
      }
    }
    stage('Sign APK') {
      environment {
        KEY_PASSWORD = credentials('jenkins.keypass')
        KEYSTORE = credentials('release.jks')
      }      
      when {
        expression {
            return env.BRANCH_NAME == 'master'
        }
      }
      steps {
            sh "rm -rf app-release-unsigned-aligned.apk"
            sh "rm -rf app-release-signed-aligned.apk"
            sh "zipalign -v -p 4 app/build/outputs/apk/app-release-unsigned.apk app-release-unsigned-aligned.apk"
            sh "apksigner sign --ks '${KEYSTORE}' --ks-key-alias release --ks-pass pass:${KEY_PASSWORD} --key-pass pass:${KEY_PASSWORD}  --out app-release-signed-aligned.apk app-release-unsigned-aligned.apk"        
      }
    }
    stage('Upload to S3') {
      steps {
        script {
            if (env.BRANCH_NAME == 'master') {
                withAWS(credentials: 'ethan.aws', region: 'ap-northeast-1') {
                    s3Upload(file: 'app-release-signed-aligned.apk', bucket: 'jenkins-archiver', path: "archive/${env.JOB_NAME}/${env.BUILD_NUMBER}/app-release-${env.BUILD_NUMBER}.apk")
                }
                successful_message = "${successful_message}\nProd APK: <https://s3-ap-northeast-1.amazonaws.com/jenkins-archiver/archive/${env.JOB_NAME}/${env.BUILD_NUMBER}/app-release-${env.BUILD_NUMBER}.apk|app-release-${env.BUILD_NUMBER}.apk>"
            }
            if (env.BRANCH_NAME == 'dev') {
                withAWS(credentials: 'ethan.aws', region: 'ap-northeast-1') {
                    s3Upload(file: 'app-debug.apk', bucket: 'jenkins-archiver', path: "archive/${env.JOB_NAME}/${env.BUILD_NUMBER}/app-debug-${env.BUILD_NUMBER}.apk")
                }
                successful_message = "${successful_message}\nProd APK: <https://s3-ap-northeast-1.amazonaws.com/jenkins-archiver/archive/${env.JOB_NAME}/${env.BUILD_NUMBER}/app-debug-${env.BUILD_NUMBER}.apk|app-debug-${env.BUILD_NUMBER}.apk>"
            }
            if (!env.BRANCH_NAME.startsWith('PR')){
                withAWS(credentials: 'ethan.aws', region: 'ap-northeast-1') {
                    s3Upload(file: 'app/build/outputs/apk/app-debug.apk', bucket: 'jenkins-archiver', path: "archive/${env.JOB_NAME}/${env.BUILD_NUMBER}/app-debug-${env.BUILD_NUMBER}.apk")
                }
                successful_message = "${successful_message}\nDebug APK: <https://s3-ap-northeast-1.amazonaws.com/jenkins-archiver/archive/${env.JOB_NAME}/${env.BUILD_NUMBER}/app-debug-${env.BUILD_NUMBER}.apk|app-debug-${env.BUILD_NUMBER}.apk>"
            }
         }
      }
    }
  }
  post {
    success {
        script {
            if (env.BRANCH_NAME.startsWith('PR')){
                successful_message = "${successful_message} \n This PR looks good, it can be merged into ${env.CHANGE_TARGET}"
            }
            slackSend(message: successful_message, channel: 'istqb_plus', color: '#00FF00', teamDomain: 'innovatube', token: slack_token)
        }

    }
    failure {
      slackSend(message: "FAILED: Job ${env.JOB_NAME} [${env.BUILD_NUMBER}]\n<${env.BUILD_URL}|Open>", color: '#FF0000', channel: 'istqb_plus', teamDomain: 'innovatube', token: slack_token)
    }
    unstable {
      slackSend(message: "UNSTABLE: Job ${env.JOB_NAME} [${env.BUILD_NUMBER}]\n<${env.BUILD_URL}|Open>", color: '#828282', channel: 'istqb_plus', teamDomain: 'innovatube', token: slack_token)
    }
  }
}
