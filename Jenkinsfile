def app
def dockerfile
def anchorefile
def repotag
def path

pipeline {
  agent any

  // using the Timestamper plugin we can add timestamps to the console log
  options {
    timestamps()
  }

  environment {
    DOCKER_REPOSITORY = "yaagarwa/anchore-jenkins-example"

    //Use Pipeline Utility Steps plugin to read information from pom.xml into env variables
    IMAGE = readMavenPom().getArtifactId()
    VERSION = readMavenPom().getVersion()
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
      agent {
        docker {
          /*
           * Reuse the workspace on the agent defined at top-level of Pipeline but run inside a container.
           * In this case we are running a container with maven so we don't have to install specific versions
           * of maven directly on the agent
           */
          reuseNode true
          image 'maven:3.6.3-jdk-8'
        }
      }
      steps {
        // using the Pipeline Maven plugin we can set maven configuration settings, publish test results, and annotate the Jenkins console
        withMaven(options: [junitPublisher(ignoreAttachments: false)]) {
          sh 'mvn clean package'
        }
      }
      post {
        success {
          // we only worry about archiving the jar file if the build steps are successful
          archiveArtifacts(artifacts: '**/target/*.jar', allowEmptyArchive: true)
        }
      }
      // Build the image and push it to a staging repository
      // steps {
      //   script {
      //     repotag = "${env.DOCKER_REPOSITORY}" + ":${BUILD_NUMBER}"
      //     docker.withRegistry("${env.DOCKER_REGISTRY_URL}", 'docker-credentials') {
      //       app = docker.build(repotag)
      //       app.push()
      //     }
      //   }
      // }
    }

    stage('Build and Publish Image') {
      steps {
        /*
         * Multiline strings can be used for larger scripts. It is also possible to put scripts in your shared library
         * and load them with 'libaryResource'
         */
        script {
          repotag = "${env.DOCKER_REPOSITORY}" + ":${BUILD_NUMBER}"
          docker.withRegistry("", 'docker-credentials') {
            app = docker.build(repotag)
            app.push()
          }
        }
      }
    }

    stage('Parallel') {
      steps {
        parallel(
          "Integration Test": {
            steps {
              sh 'mvn test'
            }
          },
          // stage('Sonar Scan') {
          //   agent {
          //     docker {
          //       // we can use the same image and workspace as we did previously
          //       reuseNode true
          //       image 'maven:3.5.0-jdk-8'
          //     }
          //   }
          //   environment {
          //     //use 'sonar' credentials scoped only to this stage
          //     SONAR = credentials('sonar-credentials')
          //   }
          //   steps {
          //     sh 'mvn sonar:sonar -Dsonar.login=$SONAR_PSW'
          //   }
          // },
          "Analyze": {
            writeFile file: anchorefile,
            text: "${env.DOCKER_REGISTRY_HOST}" + "/" + repotag + " " + dockerfile
            anchore name: anchorefile,
            engineurl: "${ANCHORE_ENGINE}",
            engineCredentialsId: 'anchore-credentials',
            annotations: [
              [key: 'added-by', value: 'jenkins']
            ]
          }
        )
      }
    }
  }
}