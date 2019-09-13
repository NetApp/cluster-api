pipeline {
  agent {
    kubernetes {
      label 'cluster-api'
      defaultContainer 'jnlp'
      yamlFile 'JenkinsPod.yaml'
    }
  }

  environment {
    DOCKER_REGISTRY = 'gcr.io'
    ORG        = 'stackpoint-public'
    APP_NAME   = 'cluster-api'
    REPOSITORY = "${ORG}/${APP_NAME}"
    GO111MODULE = 'on'
    GOPATH = "${WORKSPACE}/go"
  }

  stages {

    stage('checkout'){
      steps {
        container('builder-base') {
          dir("${GOPATH}/src/sigs.k8s.io/cluster-api") {
            checkout scm
          }
        }
      }
    }

    stage('build') {
      steps {
        container('builder-base') {
          script {
            image = docker.build("${REPOSITORY}")
          }
        }
      }
    }

    stage('publish: dev') {
      when {
        anyOf {
            branch 'PR-*'
            branch 'netapp-*'
        }
      }
      environment {
        GIT_COMMIT_SHORT = sh(
                script: "printf \$(git rev-parse --short ${GIT_COMMIT})",
                returnStdout: true
        ).trim()
      }
      steps {
        container('builder-base') {
          script {
            docker.withRegistry("https://${DOCKER_REGISTRY}", "gcr:${ORG}") {
              image.push("netapp-dev-${GIT_COMMIT_SHORT}")
            }
          }
        }
      }
    }

    stage('publish: netapp') {
      when {
        branch 'netapp'
      }
      environment {
        GIT_COMMIT_SHORT = sh(
                script: "printf \$(git rev-parse --short ${GIT_COMMIT})",
                returnStdout: true
        ).trim()
      }
      steps {
        container('builder-base') {
          script {
            docker.withRegistry("https://${DOCKER_REGISTRY}", "gcr:${ORG}") {
              image.push("netapp-${GIT_COMMIT_SHORT}")
              image.push("netapp")
            }
          }
        }
      }
    }

  }
}