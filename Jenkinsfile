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

    stage('gazelle'){
      steps {
        container('bazel') {
          dir("${GOPATH}/src/sigs.k8s.io/cluster-api") {
            sh('./hack/update-bazel.sh')
          }
        }
      }
    }

    stage('verify'){
      parallel {
        stage('verify-boilerplate'){
          steps {
            container('builder-base') {
              dir("${GOPATH}/src/sigs.k8s.io/cluster-api") {
                sh("./hack/verify-boilerplate.sh")
              }
            }
          }
        }
        stage('verify_clientset'){
          steps {
            container('golang') {
              dir("${GOPATH}/src/sigs.k8s.io/cluster-api") {
                sh("./hack/verify_clientset.sh")
              }
            }
          }
        }
        stage('verify-bazel'){
          steps {
            container('bazel') {
              dir("${GOPATH}/src/sigs.k8s.io/cluster-api") {
                sh(script: "./hack/verify-bazel.sh", returnStatus: true)
              }
            }
          }
        }
      }
    }

    stage('generate'){
      stages {
        stage('go'){
          steps {
            container('golang') {
              dir("${GOPATH}/src/sigs.k8s.io/cluster-api") {
                sh('go generate ./pkg/... ./cmd/...')
              }
            }
          }
        }
       stage('gazelle'){
            steps {
              container('bazel') {
                dir("${GOPATH}/src/sigs.k8s.io/cluster-api") {
                    sh('./hack/update-bazel.sh')
                }
              }
            }
          }
      }
    }

    stage('manifests'){
      steps {
        container('golang') {
          dir("${GOPATH}/src/sigs.k8s.io/cluster-api") {
            sh('go run vendor/sigs.k8s.io/controller-tools/cmd/controller-gen/main.go all')
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