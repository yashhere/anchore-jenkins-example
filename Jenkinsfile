def app
def dockerfile
def anchorefile
def repotag
def path
def identity
def directory

pipeline {
  agent any

  // using the Timestamper plugin we can add timestamps to the console log
  options 
   {
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
    
    stage ('Artifactory configuration') {
            steps {
                rtMavenDeployer (
                    id: "MAVEN_DEPLOYER",
                    serverId: "${SERVER_ID}",
                    releaseRepo: "Jenkins-integration",
                    snapshotRepo: "Jenkins-snapshot"
                )

                rtMavenResolver (
                    id: "MAVEN_RESOLVER",
                    serverId: "${SERVER_ID}",
                    releaseRepo: "Jenkins-integration",
                    snapshotRepo: "Jenkins-snapshot"
                )
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
        

        rtPublishBuildInfo (
            serverId: "${SERVER_ID}"
        )
            
      }
    }

    stage('Analyze') {
      steps {
        writeFile file: anchorefile, text: "docker.io" + "/" + repotag + " " + dockerfile
        anchore name: anchorefile,
          engineurl: "${ANCHORE_ENGINE}",
          engineCredentialsId: 'anchore-credentials',
          bailOnFail: false,
          annotations: [
            [key: 'added-by', value: 'jenkins']
          ]
      }
    }
  }
  post {
    always {
      script {
        echo "${env.JENKINS_HOME}"
        echo "${env.WORKSPACE}"
        directory = "${env.JENKINS_HOME}" + "/jobs/" + "${JOB_NAME}" + "/builds/" + "${BUILD_NUMBER}" + "/archive/AnchoreReport.CI-CD_" + "${BUILD_NUMBER}"
        dir(directory) {
          findFiles(glob: '*.*')
          sh "ls -la"
        }
        withAWS(region: 'ap-south-1', credentials: 'aws-s3') {
          identity = awsIdentity(); //Log AWS credentials
          // Upload files from working directory 'dist' in your project workspace
          s3Upload(bucket: "artifacts-for-grafeas", path: "${JOB_NAME}" + "-" + "${BUILD_NUMBER}", includePathPattern: '*.json', workingDir: directory);
        }
      }
    }
  }
}