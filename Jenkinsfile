def app
def dockerfile
def anchorefile
def repotag
def path

pipeline {
  agent any

  environment {
      DOCKER_REPOSITORY = "https://github.com/yashhere/anchore-jenkins-example"
      DOCKER_REGISTRY_URL = "https://index.docker.io/v2"
      DOCKER_REGISTRY_HOST = "docker.io"
  }

  stages {
    stage('Clone') {
      steps {
        checkout scm
        script {
          path = sh returnStdout: true, script: "pwd"
          path = path.trim()
          dockerfile = path + "/Dockerfile"
          anchorefile = path + "/anchore_images"
        }
      }
    }

    stage('Build') {
      // Build the image and push it to a staging repository
      steps {
        script {
          repotag = "${env.DOCKER_REPOSITORY}" + ":${BUILD_NUMBER}"
          docker.withRegistry("${env.DOCKER_REGISTRY_URL}", 'docker-credentials') {
            app = docker.build(repotag)
            app.push()
          }
        }
      }
    }

    stage('Parallel') {
      steps {
        parallel(
          "Test": {
            script {
              app.inside {
                  sh 'echo "Dummy - tests passed"'
              }
            }
          },
          "Analyze": {
            writeFile file: anchorefile, text: "${env.DOCKER_REGISTRY_HOST}" + "/" + repotag + " " + dockerfile
            anchore name: anchorefile, engineurl: "${ANCHORE_ENGINE}", engineCredentialsId: 'anchore-credentials', annotations: [[key: 'added-by', value: 'jenkins']]
          }
        )
      }
    }
  }
}