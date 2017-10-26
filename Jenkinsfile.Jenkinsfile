#!groovy

pipeline {
  agent { label 'master' }
  triggers {
    pollSCM('')
  }
  options {
    timestamps()
    timeout(time: 1, unit: 'HOURS')
    buildDiscarder(logRotator(numToKeepStr: '30'))
    skipStagesAfterUnstable()
  }
  stages {
    stage('Initialize') {
      environment {
        PROPS_FILE = 'jenkins.properties'
      }
      steps {
        script {
          loadProperties()
          SCM_BRANCH = scm.branches[0].name
          SCM_URL = sh(returnStdout: true, script: 'git config --get remote.origin.url').trim()
        }
      }
    }
    stage('Start') {
      environment {
        SLACK_CHANNEL = "${NOTIFICATION_SLACK_CHANNEL}"
        SCM_BRANCH = "${SCM_BRANCH}"
      }
      steps {
        notifyStart()
        notifyStash('INPROGRESS')
      }
    }
    stage('Maven Build & Test') {
      environment {
        BUILD_RUNNER = "${JENKINS_RUNNER_JOB_NAME}"
        DOCKER_ECR_SERVER = "${DOCKER_ECR_SERVER}"
        SCM_BRANCH = "${SCM_BRANCH}"
        SCM_URL = "${SCM_URL}"
      }
      steps {
        echo "------------- Cleaning up local docker containers and images -------------"
        cleanupDockers()
        echo "------------- Running maven build & test -------------"
        triggerMavenBuild()
      }
    }
    stage('Build and publish docker image') {
      environment {
        BUILD_RUNNER = "${JENKINS_RUNNER_JOB_NAME}"
        DOCKER_APP_PATH = "${DOCKER_APP_PATH}"
        DOCKERFILE_PATH = "${DOCKER_DOCKERFILE_PATH}"
        DOCKER_ECR_SERVER = "${DOCKER_ECR_SERVER}"
        DOCKER_ECR_CREDENTIAL = "${DOCKER_ECR_CREDENTIAL}"
      }
      /*
      when {
        expression { currentBuild.currentResult == 'SUCCESS' }
      }
      */
      steps {
        script {
          publishDocker()
        }
      }
    }
    stage('Integration Test') {
      environment {
        INTEGRATION_JOB = "${JENKINS_INTEGRATION_JOB_NAME}"
        INTEGRATION_URL = "${JENKINS_INTEGRATION_SERVER_PROTOCOL}://${JENKINS_INTEGRATION_SERVER}:${JENKINS_INTEGRATION_SERVER_PORT}"
        SERVICE_NAME = "${JOB_BASE_NAME}"
      }
      /*
      when {
        expression { currentBuild.currentResult == 'SUCCESS' }
      }
      */
      steps {
        script {
          echo "------------- Run integration test -------------"
          triggerIntegrationTest()

          echo "------------- Update build description -------------"
          if (currentBuild.currentResult == 'UNSTABLE') {
            currentBuild.description = "<b><font color='gold'>${currentBuild.currentResult}</font><br>" +
              " - " +
              "<a href='${INTEGRATION_URL}/job/${INTEGRATION_JOB}'>IntegrationTest</b></a>"
          }
          if (currentBuild.currentResult == 'FAILURE') {
            currentBuild.description = "<b><font color='red'>${currentBuild.currentResult}</font><br>" +
              " - " +
              "<a href='${INTEGRATION_URL}/job/${INTEGRATION_JOB}'>IntegrationTest</b></a>"
          }
        }
      }
    }
  }
  post {
    always {
      echo "current build result is ${currentBuild.result}"
      publishBuildQualityAnalysis()
      notifyResult()
      notifyStash("${currentBuild.result}")
      }
    /*
    success {
      echo "current build result is ${currentBuild.result}"
      publishBuildQualityAnalysis()
    }
    unstable {
      echo "current build result is ${currentBuild.result}"
      publishBuildQualityAnalysis()
    }
    failure {
      echo "current build result is ${currentBuild.result}"
    }
    */
  }
}

def triggerMavenBuild() {
  build job: "${BUILD_RUNNER}", parameters: [
    string(name: 'REPO_URL', value: "${SCM_URL}"),
    string(name: 'BRANCH', value: "${SCM_BRANCH}"),
    string(name: 'UPSTREAM_JOB_NAME', value: "${JOB_NAME}"),
    string(name: 'UPSTREAM_BUILD_NUMBER', value: "${BUILD_NUMBER}")
  ]
}

def publishBuildQualityAnalysis() {
  stage('Publish Quality Analysis Reports') {
    dir("${JENKINS_HOME}/workspace/${JENKINS_RUNNER_JOB_NAME}/${JOB_NAME}") {
      stash allowEmpty: false, includes: "${QUALITY_TEST_RESULT}", name: 'testReport'
      stash includes: "${QUALITY_CHECKSTYLE_RESULT}, ${QUALITY_FINDBUGS_RESULT}, ${QUALITY_PMD_RESULT}", name: 'qualityAnalysisReport'
    }
    unstash 'testReport'
    unstash 'qualityAnalysisReport'

    def qualityChecks = [:]

    qualityChecks["test result"] = {
      junit allowEmptyResults: true, healthScaleFactor: 10.0, testResults: "${QUALITY_TEST_RESULT}"
    }

    qualityChecks["code coverage"] = {
      dir ("${JENKINS_HOME}/workspace/${JENKINS_RUNNER_JOB_NAME}/${JOB_NAME}") {
        jacoco classPattern: "${JACOCO_CLASS}",
          inclusionPattern: "${JACOCO_INCLUDE}",
          exclusionPattern: "${JACOCO_EXCLUDE}",
          execPattern: "${JACOCO_EXEC}",
          changeBuildStatus: true,
          maximumLineCoverage: "${JACOCO_THREASHOLD}"
      }
    }

    qualityChecks["checkstyle"] = {
      checkstyle \
        canComputeNew: false,
        canRunOnFailed: false,
        defaultEncoding: '',
        healthy: '0',
        unHealthy: '',
        pattern: "${QUALITY_CHECKSTYLE_RESULT}",
        unstableTotalHigh: "${QUALITY_THRESHOLD_HIGH_UNSTABLE}",
        failedTotalHigh: "${QUALITY_THRESHOLD_HIGH_FAILURE}",
        unstableTotalNormal: "${QUALITY_THRESHOLD_NORMAL_UNSTABLE}",
        failedTotalNormal: "${QUALITY_THRESHOLD_NORMAL_FAILURE}",
        unstableTotalLow: "${QUALITY_THRESHOLD_LOW_UNSTABLE}",
        failedTotalLow: "${QUALITY_THRESHOLD_LOW_FAILURE}",
        unstableTotalAll: "${QUALITY_THRESHOLD_TOTAL_UNSTABLE}",
        failedTotalAll: "${QUALITY_THRESHOLD_TOTAL_FAILURE}"
    }
    qualityChecks["pmd"] = {
      pmd \
        canComputeNew: false,
        canRunOnFailed: false,
        defaultEncoding: '',
        healthy: '0',
        unHealthy: '',
        pattern: "${QUALITY_PMD_RESULT}",
        unstableTotalHigh: "${QUALITY_THRESHOLD_HIGH_UNSTABLE}",
        failedTotalHigh: "${QUALITY_THRESHOLD_HIGH_FAILURE}",
        unstableTotalNormal: "${QUALITY_THRESHOLD_NORMAL_UNSTABLE}",
        failedTotalNormal: "${QUALITY_THRESHOLD_NORMAL_FAILURE}",
        unstableTotalLow: "${QUALITY_THRESHOLD_LOW_UNSTABLE}",
        failedTotalLow: "${QUALITY_THRESHOLD_LOW_FAILURE}",
        unstableTotalAll: "${QUALITY_THRESHOLD_TOTAL_UNSTABLE}",
        failedTotalAll: "${QUALITY_THRESHOLD_TOTAL_FAILURE}"
    }
    qualityChecks["findbugs"] = {
      findbugs \
        canComputeNew: false,
        canRunOnFailed: false,
        defaultEncoding: '',
        healthy: '0',
        unHealthy: '',
        pattern: "${QUALITY_FINDBUGS_RESULT}",
        unstableTotalHigh: "${QUALITY_THRESHOLD_HIGH_UNSTABLE}",
        failedTotalHigh: "${QUALITY_THRESHOLD_HIGH_FAILURE}",
        unstableTotalNormal: "${QUALITY_THRESHOLD_NORMAL_UNSTABLE}",
        failedTotalNormal: "${QUALITY_THRESHOLD_NORMAL_FAILURE}",
        unstableTotalLow: "${QUALITY_THRESHOLD_LOW_UNSTABLE}",
        failedTotalLow: "${QUALITY_THRESHOLD_LOW_FAILURE}",
        unstableTotalAll: "${QUALITY_THRESHOLD_TOTAL_UNSTABLE}",
        failedTotalAll: "${QUALITY_THRESHOLD_TOTAL_FAILURE}"
    }

    parallel qualityChecks

//      openTasks canComputeNew: false, canRunOnFailed: true, defaultEncoding: '', excludePattern: '', healthy: '', high: '', low: '', normal: '', pattern: '', shouldDetectModules: true, unHealthy: ''
//      warnings canComputeNew: false, canResolveRelativePaths: false, consoleParsers: [[parserName: 'Maven'], [parserName: 'Java Compiler (javac)']], defaultEncoding: '', excludePattern: '', healthy: '0', includePattern: '', messagesPattern: '', unHealthy: '100'
//      dry canComputeNew: false, defaultEncoding: '', healthy: '', pattern: 'duplicate-finder-result.xml', unHealthy: ''

    step([
      $class: 'AnalysisPublisher',
      canComputeNew: false,
      canRunOnFailed: false,
      defaultEncoding: '',
      dryActivated: false,
      openTasksActivated: false,
      warningsActivated: false,
      healthy: '0',
      unHealthy: '',
      unstableTotalHigh: "${QUALITY_THRESHOLD_HIGH_UNSTABLE}",
      failedTotalHigh: "${QUALITY_THRESHOLD_HIGH_FAILURE}",
      unstableTotalNormal: "${QUALITY_THRESHOLD_NORMAL_UNSTABLE}",
      failedTotalNormal: "${QUALITY_THRESHOLD_NORMAL_FAILURE}",
      unstableTotalLow: "${QUALITY_THRESHOLD_LOW_UNSTABLE}",
      failedTotalLow: "${QUALITY_THRESHOLD_LOW_FAILURE}",
      unstableTotalAll: "${QUALITY_THRESHOLD_TOTAL_UNSTABLE}",
      failedTotalAll: "${QUALITY_THRESHOLD_TOTAL_FAILURE}"
    ])
  }
}

def cleanupDockers() {
  script {
    sh '''
            CONTAINERS=$(docker ps -aq)
            if [ "x${CONTAINERS}" != "x" ]; then
              docker stop ${CONTAINERS}
              docker rm --force ${CONTAINERS}
            fi
            IMAGES=$(docker images ${JOB_NAME} -aq)
            if [ "x${IMAGES}" != "x" ]; then
              docker rmi --force ${IMAGES}
            fi
            IMAGES=$(docker images ${DOCKER_ECR_SERVER}/${JOB_NAME} -aq)
            if [ "x${IMAGES}" != "x" ]; then
              docker rmi --force ${IMAGES}
            fi
          '''
  }
}

def triggerIntegrationTest() {
  env.imageName = "${DOCKER_ECR_SERVER}/${IMAGE_NAME}"
  env.imageVersion = "${IMAGE_TAG}"

  echo "imageName is ${imageName}"
  echo "imageVersion is ${imageVersion}"

  echo "------------------- Get crumbIssuer of Remote Jenkins Server -------------------"

  def crumbRequestField = sh(
    returnStdout: true, script: '''
        curl -s ${INTEGRATION_URL}/crumbIssuer/api/json | jq -r .crumbRequestField
      '''
  )
  def crumbValue = sh(
    returnStdout: true, script: '''
        curl -s ${INTEGRATION_URL}/crumbIssuer/api/json | jq -r .crumb
      '''
  )

  env.CRUMB = "${crumbRequestField}:${crumbValue}"

  /*
  def crumbText = sh(script: "curl -s ${integrationTestServerURL}/crumbIssuer/api/xml", returnStdout: true)
  def crumbString = new XmlParser().parseText(crumbText)
  env.crumbValue = "${crumbString.crumb.text()}"
  env.crumbRequestField = "${crumbString.crumbRequestField.text()}"
  env.crumb = "${crumbRequestField}:${crumbValue}"

  echo "crumb is ${crumb}"

  crumbValue = sh(script: "echo ${crumbText} | jq -r .crumb", returnStdout: true)
  echo "crumbValue is ${crumbValue}"
  */
  /*
  env.CRUMB = sh(
    returnStdout: true, script: '''
      curl "${INTEGRATION_URL}/crumbIssuer/api/xml?xpath=concat(//crumbRequestField,\":\",//crumb)"
    '''
  )
  */

  echo "------------------- Build up Webhook -------------------"
  hook = registerWebhook()
  env.callBackUrl = hook.getURL()

  echo "------------------- Call Remote Jenkins Job -------------------"
  sh '''
      curl -X POST -H "${CRUMB}" -s -o /dev/null -w "%{http_code}" "${INTEGRATION_URL}/job/${INTEGRATION_JOB}/buildWithParameters?token=integrationTesting&callbackUrl=${callBackUrl}&servicename=${SERVICE_NAME}&imageName=${imageName}&imageVersion=${imageVersion}"
    '''

  echo "------------------- Wait For Response from Remote Jenkins Job -------------------"
  callbackResponse = waitForWebhook hook
  currentBuild.result = "${callbackResponse}"
}

def loadProperties() {
  def props = readProperties  file:"${PROPS_FILE}"
//  SCM_BRANCH = props['SCM_BRANCH']
  NOTIFICATION_SLACK_CHANNEL = props['NOTIFICATION_SLACK_CHANNEL']
  DOCKER_ECR_SERVER = props['DOCKER_ECR_SERVER']
  JENKINS_RUNNER_JOB_NAME = props['JENKINS_RUNNER_JOB_NAME']
  DOCKER_APP_PATH = props['DOCKER_APP_PATH']
  DOCKER_DOCKERFILE_PATH = props['DOCKER_DOCKERFILE_PATH']
  DOCKER_ECR_CREDENTIAL = props['DOCKER_ECR_CREDENTIAL']
  JENKINS_INTEGRATION_SERVER = props['JENKINS_INTEGRATION_SERVER']
  JENKINS_INTEGRATION_SERVER_PORT = props['JENKINS_INTEGRATION_SERVER_PORT']
  JENKINS_INTEGRATION_SERVER_PROTOCOL = props['JENKINS_INTEGRATION_SERVER_PROTOCOL']
  JENKINS_INTEGRATION_JOB_NAME = props['JENKINS_INTEGRATION_JOB_NAME']
  JACOCO_CLASS = props['JACOCO_CLASS_PATH']
  JACOCO_INCLUDE = props['JACOCO_INCLUDE']
  JACOCO_EXCLUDE = props['JACOCO_EXCLUDE']
  JACOCO_EXEC = props['JACOCO_EXEC_PATH']
  JACOCO_THREASHOLD = props['JACOCO_LINE_COVERAGE_PERCENTAGE']
  QUALITY_TEST_RESULT = props['QUALITY_TEST_RESULT']
  QUALITY_CHECKSTYLE_RESULT = props['QUALITY_CHECKSTYLE_RESULT']
  QUALITY_FINDBUGS_RESULT = props['QUALITY_FINDBUGS_RESULT']
  QUALITY_PMD_RESULT = props['QUALITY_PMD_RESULT']
  QUALITY_THRESHOLD_HIGH_UNSTABLE = props['QUALITY_THRESHOLD_HIGH_UNSTABLE']
  QUALITY_THRESHOLD_HIGH_FAILURE = props['QUALITY_THRESHOLD_HIGH_FAILURE']
  QUALITY_THRESHOLD_NORMAL_UNSTABLE = props['QUALITY_THRESHOLD_NORMAL_UNSTABLE']
  QUALITY_THRESHOLD_NORMAL_FAILURE = props['QUALITY_THRESHOLD_NORMAL_FAILURE']
  QUALITY_THRESHOLD_LOW_UNSTABLE = props['QUALITY_THRESHOLD_LOW_UNSTABLE']
  QUALITY_THRESHOLD_LOW_FAILURE = props['QUALITY_THRESHOLD_LOW_FAILURE']
  QUALITY_THRESHOLD_TOTAL_UNSTABLE = props['QUALITY_THRESHOLD_TOTAL_UNSTABLE']
  QUALITY_THRESHOLD_TOTAL_FAILURE = props['QUALITY_THRESHOLD_TOTAL_FAILURE']
}

def publishDocker() {
  def SHORT_COMMIT = sh(returnStdout: true, script: 'git rev-parse HEAD').trim().take(7)
  IMAGE_NAME = "${JOB_NAME}"

  def POM_FILE = readMavenPom file: 'pom.xml'
  IMAGE_VERSION = POM_FILE.version

  IMAGE_TAG = "${JOB_BASE_NAME}.${IMAGE_VERSION}.${BUILD_NUMBER}.${SHORT_COMMIT}"

  dir("${JENKINS_HOME}/workspace/${BUILD_RUNNER}/${JOB_NAME}/${DOCKER_APP_PATH}") {
    def DOCKER_IMAGE = docker.build("${IMAGE_NAME}:${IMAGE_TAG}", "-f ${DOCKERFILE_PATH} .")
    docker.withRegistry(
      "https://${DOCKER_ECR_SERVER}",
      "ecr:us-west-2:${DOCKER_ECR_CREDENTIAL}") {
      DOCKER_IMAGE.push()
    }
  }
}

def notifyResult() {
  script {
    def SLACK_CHANNEL = "${NOTIFICATION_SLACK_CHANNEL}"
    if (currentBuild.currentResult == 'ABORTED') {
      color = '#808080'
    }
    if (currentBuild.currentResult == 'FAILURE') {
      color = 'danger'
    }
    if (currentBuild.currentResult == 'SUCCESS') {
      color = 'good'
    }
    if (currentBuild.currentResult == 'UNSTABLE') {
      color = 'warning'
    }
    if ("x${SLACK_CHANNEL}" != "x") {
      slackSend(
        channel: "${SLACK_CHANNEL}",
        color: "${color}",
        message:
          "Build *${currentBuild.currentResult}* - ${JOB_NAME}:${SCM_BRANCH} - #${BUILD_NUMBER}\n" +
          "Build Result: ${BUILD_URL}"
      )
    }
  }
}

def notifyStart() {
  echo "=================== Send Notification For Build Start ==================="
  script {
    if ("x${SLACK_CHANNEL}" != "x") {
      slackSend(
        channel: "${SLACK_CHANNEL}",
        color: '#3498DB',
        message:
          "Build *STARTED* - ${JOB_NAME}:${SCM_BRANCH} - #${BUILD_NUMBER}\n" +
          "${BUILD_URL}"
      )
    }
  }
}

def notifyStash(String state) {
  step([$class: 'StashNotifier',
    commitSha1: '',
    credentialsId: '',
    disableInprogressNotification: false,
    ignoreUnverifiedSSLPeer: false,
    includeBuildNumberInKey: true,
    prependParentProjectKey: false,
    projectKey: '',
    stashServerBaseUrl: ''
  ])
}