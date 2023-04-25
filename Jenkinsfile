def COLOR_MAP = [
    'SUCCESS': 'good', 
    'FAILURE': 'danger',
]
pipeline {
  agent any
  environment {
    WORKSPACE = "${env.WORKSPACE}"
    //NEXUS_USER = "$NEXUS_CREDS_USR"
    //NEXUS_PASSWORD = "$nexus-credentials"
  }
  tools {
    maven 'localMaven'
    jdk 'localJdk'
  }
  stages {
    stage('Build') {
      steps {
        sh 'mvn clean package'
      }
      post {
        success {
          echo ' now Archiving '
          archiveArtifacts artifacts: '**/*.war'
        }
      }
    }
    stage('Unit Test'){
        steps {
            sh 'mvn test'
        }
    }
    stage('Integration Test'){
        steps {
          sh 'mvn verify -DskipUnitTests'
        }
    }
    stage ('Checkstyle Code Analysis'){
        steps {
            sh 'mvn checkstyle:checkstyle'
        }
        post {
            success {
                echo 'Generated Analysis Result'
            }
        }
    }
    stage('SonarQube scanning') {
        steps {
            withSonarQubeEnv('SonarQube') { 
                withCredentials([string(credentialsId: 'SonarQube-Token', variable: 'SONAR_TOKEN')]) {
                sh """
                mvn sonar:sonar \
                -Dsonar.projectKey=jjtech-cicd-pipeline \
                -Dsonar.host.url=http://172.31.83.105:9000 \
                -Dsonar.login=$SONAR_TOKEN
                """
                }
            }
        }
    }
    stage('Quality Gate') {
        steps {
          timeout(time : 1, unit : 'HOURS'){
          waitForQualityGate abortPipeline: true
          }
       }
    }
    // stage('Upload artifact to Nexus') {
    //     steps {
    //         withCredentials([usernamePassword(credentialsId: 'nexus-credentials', passwordVariable: 'PASSWORD', usernameVariable: 'USER_NAME')]) {
    //         sh "sed -i \"s/.*<username><\\/username>/<username>$USER_NAME<\\/username>/g\" ${WORKSPACE}/nexus-setup/settings.xml"
    //         sh "sed -i \"s/.*<password><\\/password>/<password>$PASSWORD<\\/password>/g\" ${WORKSPACE}/nexus-setup/settings.xml"
    //         sh 'cp ${WORKSPACE}/nexus-setup/settings.xml /var/lib/jenkins/.m2'
    //         sh 'mvn clean deploy -DskipTests'
    //         }
    //     }
    // }
    stage('Upload to Artifactory') {
      steps {
        sh "mvn clean deploy -DskipTests"
      }
    }
    stage('Deploy to DEV') {
      environment {
        HOSTS = "dev"
      }
      steps {
        sh "ansible-playbook ${WORKSPACE}/deploy.yaml --extra-vars \"hosts=$HOSTS workspace_path=$WORKSPACE\""
      }
     }
    // stage('Approval for stage') {
    //   steps {
    //     input('Do you want to proceed?')
    //   }
    // }
    stage('Deploy to Stage') {
      environment {
        HOSTS = "stage" // Make sure to update to "stage"
      }
      steps {
        sh "ansible-playbook ${WORKSPACE}/deploy.yaml --extra-vars \"hosts=$HOSTS workspace_path=$WORKSPACE\""
      }
    }
    stage('Approval') {
      steps {
        input('Do you want to proceed?')
      }
    }
    stage('Deploy to PROD') {
      environment {
        HOSTS = "prod"
      }
      steps {
        sh "ansible-playbook ${WORKSPACE}/deploy.yaml --extra-vars \"hosts=$HOSTS workspace_path=$WORKSPACE\""
      }
    }
  }
  post {
    always {
        echo 'Slack Notifications.'
        slackSend channel: '#cicd-project-alerts', //update and provide your channel name
        color: COLOR_MAP[currentBuild.currentResult],
        message: "*${currentBuild.currentResult}:* Job ${env.JOB_NAME} build ${env.BUILD_NUMBER} \n More info at: ${env.BUILD_URL}"
    }
  }
}

//slackSend channel: '#mbandi-cloudformation-cicd', message: "Please find the pipeline status of the following ${env.JOB_NAME ${env.BUILD_NUMBER} ${env.BUILD_URL}"
