pipeline {
  agent {
    label 'centos-latest'
  }

  triggers {
     cron('H 23 * * *')
  }

  options {
    buildDiscarder(logRotator(numToKeepStr: '10'))
    disableConcurrentBuilds()
    skipDefaultCheckout false
  }

  tools {
    maven 'apache-maven-latest'
    jdk 'temurin-jdk17-latest'
  }

  environment {
    PUBLISH_LOCATION = 'updates'
    BUILD_TIMESTAMP = sh(returnStdout: true, script: 'date +%Y%m%d%H%M').trim()
  }

  parameters {
    choice(
      name: 'BUILD_TYPE',
      choices: ['nightly', 'milestone', 'release'],
      description: '''
        Choose the type of build.
        Note that a release build will not promote the build, but rather will promote the most recent milestone build.
        '''
    )

    choice(
      name: 'TARGET_PLATFORM',
      choices: [ 'eclipse_2023_03-xtext_2_31_0', 'eclipse_2023_09-xtext_nightly' ],
      description: '''
        Choose the named target platform against which to compile and test.
        This is relevant only for nightly and milestone builds.
        '''
    )

    booleanParam(
      name: 'ECLIPSE_SIGN',
      defaultValue: true,
      description: '''
        Choose whether or not the bundles will be signed.
        This is relevant only for nightly and milestone builds.
      '''
    )

    booleanParam(
      name: 'PROMOTE',
      defaultValue: true,
      description: 'Whether to promote the build to the download server.'
    )
  }

  stages {
    stage('Display Parameters') {
      steps {
        script {
          env.BUILD_TYPE = params.BUILD_TYPE
          env.TARGET_PLATFORM = params.TARGET_PLATFORM

          if (env.BRANCH_NAME == 'master') {
            // Only sign the master branch.
            env.ECLIPSE_SIGN = params.ECLIPSE_SIGN
          } else {
            env.ECLIPSE_SIGN =  false
          }

            // Only promote signed builds.
          env.PROMOTE = params.PROMOTE && (env.ECLIPSE_SIGN == 'true')
        }
        echo """
BUILD_TIMESTAMP=${env.BUILD_TIMESTAMP}
BUILD_TYPE=${env.BUILD_TYPE}
TARGET_PLATFORM=${env.TARGET_PLATFORM}
ECLIPSE_SIGN=${env.ECLIPSE_SIGN}
PROMOTE=${env.PROMOTE}
BRANCH_NAME=${env.BRANCH_NAME}
"""
      }
    }

    stage('Build') {
      steps {
        sshagent(['projects-storage.eclipse.org-bot-ssh']) {
          wrap([$class: 'Xvnc', useXauthority: true]) {
            dir('.') {
              sh '''
                if [[ $PROMOTE != true ]]; then
                  promotion_argument='-P!promote'
                fi
                mvn  \
                  --fail-at-end \
                  --no-transfer-progress \
                  --update-snapshots \
                  $promotion_argument \
                  -Dmaven.repo.local=xpect-local-maven-repository \
                  -Dmaven.artifact.threads=16 \
                  -Dbuild.id=${BUILD_TIMESTAMP} \
                  -Dgit.commit=$GIT_COMMIT \
                  -Declipsesign=${ECLIPSE_SIGN} \
                  -Dbuild.type=$BUILD_TYPE \
                  -Dtycho-version=4.0.3 \
                  -Dtarget-platform=${TARGET_PLATFORM} \
                  -Dorg.eclipse.justj.p2.manager.build.url=$JOB_URL \
                  -Dorg.eclipse.justj.p2.manager.relative=$PUBLISH_LOCATION \
                  clean \
                  verify
                '''
            }
          }
        }
      }
    }
  }

  post {
    failure {
      mail to: 'ed.merks@gmail.com',
      subject: "[EMF CI] Build Failure ${currentBuild.fullDisplayName}",
      mimeType: 'text/html',
      body: "Project: ${env.JOB_NAME}<br/>Build Number: ${env.BUILD_NUMBER}<br/>Build URL: ${env.BUILD_URL}<br/>Console: ${env.BUILD_URL}/console"
      archiveArtifacts allowEmptyArchive: true, artifacts: '**'
    }

    fixed {
      mail to: 'ed.merks@gmail.com',
      subject: "[EMF CI] Back to normal ${currentBuild.fullDisplayName}",
      mimeType: 'text/html',
      body: "Project: ${env.JOB_NAME}<br/>Build Number: ${env.BUILD_NUMBER}<br/>Build URL: ${env.BUILD_URL}<br/>Console: ${env.BUILD_URL}/console"
    }

    success {
      archiveArtifacts artifacts: 'org.eclipse.xpect.releng/p2-repository/target/repository/**/*.*,org.eclipse.xpect.releng/p2-repository/target/org.eclipse.xpect.repository-*.zip'
    }

    always {
      junit allowEmptyResults: true, testResults: '**/TEST-*.xml'
    }

    cleanup {
      deleteDir()
      postAlways()
    }
  }
}

def postAlways() {
  def curResult = currentBuild.currentResult
  def lastResult = 'NEW'
  if (currentBuild.previousBuild != null) {
    lastResult = currentBuild.previousBuild.result
  }

  echo "matrix result check: curResult=${curResult} lastResult=${lastResult}"

  if (curResult != 'SUCCESS' || lastResult != 'SUCCESS') {
    def color = ''
    switch (curResult) {
      case 'SUCCESS':
        color = '#00FF00'
        break
      case 'UNSTABLE':
        color = '#FFFF00'
        break
      case 'FAILURE':
        color = '#FF0000'
        break
      default: // e.g. ABORTED
        color = '#666666'
    }

    echo "matrix send message"
    matrixSendMessage https: true,
      hostname: 'matrix.eclipse.org',
      accessTokenCredentialsId: "matrix-token",
      roomId: '!aFWRHMCLJDZBzuNIRD:matrix.eclipse.org',
      body: "${lastResult} => ${curResult} ${env.BUILD_URL} | ${env.JOB_NAME}#${env.BUILD_NUMBER}",
      formattedBody: "<div><font color='${color}'>${lastResult} => ${curResult}</font> | <a href='${env.BUILD_URL}' target='_blank'>${env.JOB_NAME}#${env.BUILD_NUMBER}</a></div>"
  }
}