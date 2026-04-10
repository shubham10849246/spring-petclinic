pipeline {
  agent none

  tools {
    maven 'Maven3'
    jdk   'JDK17'
  }

  environment {
    APP_NAME     = 'spring-petclinic'
    AWS_REGION   = 'ap-south-1'
    AWS_ACCOUNT  = '713332525966'
    ECR_REPO     = 'petclinic'
    ECR_REGISTRY = "${AWS_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com"

    SONAR_PROJECT_KEY = 'petclinic'
    SONARQUBE_SERVER  = 'sonarqube'

    GIT_SHORT = "${env.GIT_COMMIT?.take(7) ?: 'nogit'}"
    IMAGE_TAG = "${env.BUILD_NUMBER}-${GIT_SHORT}"
    IMAGE_URI = "${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}"
  }

  options {
    timestamps()
    disableConcurrentBuilds()
    timeout(time: 45, unit: 'MINUTES')
    buildDiscarder(logRotator(numToKeepStr: '30'))
  }

  stages {

    stage('Checkout') {
      agent { label 'slave1' }
      steps { checkout scm }
    }

    stage('Build + Unit Tests') {
      agent { label 'slave1' }
      steps {
        sh 'mvn -B -U clean test'
      }
      post {
        always {
          junit 'target/surefire-reports/*.xml'
          archiveArtifacts artifacts: 'target/*.jar', fingerprint: true, onlyIfSuccessful: false
        }
      }
    }

    stage('SonarQube Scan') {
      agent { label 'slave1' }
      steps {
        withSonarQubeEnv(env.SONARQUBE_SERVER) {
          sh """
            mvn -B sonar:sonar \
              -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
              -Dsonar.projectName=${APP_NAME}
          """
        }
      }
    }

    stage('Quality Gate') {
      agent { label 'slave1' }
      steps {
        timeout(time: 2, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }

    stage('Docker Build') {
      agent { label 'slave1' }
      steps {
        sh '''
          docker build -t ${IMAGE_URI} .
          docker images | head -n 20
        '''
      }
    }

    stage('ECR Login & Push') {
      agent { label 'slave1' }
      steps {
        sh '''
          aws ecr get-login-password --region ${AWS_REGION} | \
            docker login --username AWS --password-stdin ${ECR_REGISTRY}

          aws ecr describe-repositories --repository-names ${ECR_REPO} --region ${AWS_REGION} >/dev/null 2>&1 || \
          aws ecr create-repository --repository-name ${ECR_REPO} --region ${AWS_REGION}

          docker push ${IMAGE_URI}
        '''
      }
    }

    stage('Update GitOps Repo (Image Tag)') {
      agent { label 'slave2' }
      steps {
        // RECOMMENDATION: clone gitops repo, update deployment manifest image to ${IMAGE_URI}, commit & push
        echo "Update GitOps manifests to use ${IMAGE_URI} and push to Git."
      }
    }

    stage('Post-Deploy Smoke Test') {
      agent { label 'slave2' }
      steps {
        // RECOMMENDATION: curl service endpoint or actuator health after Argo sync
        echo "Run smoke checks (HTTP 200 /actuator/health)."
      }
    }
  }

  post {
    success { echo "✅ Image pushed: ${IMAGE_URI}" }
    always  { sh 'docker image prune -f || true' }
  }
}
