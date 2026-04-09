pipeline {
  agent { label 'slave1' }

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

    GIT_SHORT = "${env.GIT_COMMIT?.take(7) ?: 'nogit'}"
    IMAGE_TAG = "${env.BUILD_NUMBER}-${GIT_SHORT}"
    IMAGE_URI = "${ECR_REGISTRY}/${ECR_REPO}:${IMAGE_TAG}"
    IMAGE_LATEST = "${ECR_REGISTRY}/${ECR_REPO}:latest"
  }

  options {
    timestamps()
    disableConcurrentBuilds()
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Tool Check (on Agent)') {
      steps {
        sh '''
          echo "NODE: $(hostname)"
          echo "JAVA_HOME=$JAVA_HOME"
          java -version
          mvn -version
          docker --version
          aws --version || true
        '''
      }
    }

    stage('Maven Build (skip tests)') {
      steps {
        sh 'mvn -B -U clean package -Dmaven.test.skip=true'
      }
      post {
        always {
          archiveArtifacts artifacts: 'target/*.jar', fingerprint: true, onlyIfSuccessful: false
        }
      }
    }

    stage('Docker Permission Check') {
  steps {
    sh '''
      echo "USER=$(whoami)"
      echo "GROUPS=$(id)"
      ls -l /var/run/docker.sock
      docker ps
    '''
  }
}


    stage('Docker Build') {
  steps {
    sh '''
      docker build -t ${IMAGE_URI} .
      docker tag ${IMAGE_URI} ${IMAGE_LATEST}
      docker images | head -n 20
    '''
  }
}


    stage('ECR Login & Push') {
      steps {
        withCredentials([[
          $class: 'AmazonWebServicesCredentialsBinding',
          credentialsId: 'aws-jenkins-creds'
        ]]) {
          sh '''
            aws ecr get-login-password --region ${AWS_REGION} | \
              docker login --username AWS --password-stdin ${ECR_REGISTRY}

            aws ecr describe-repositories --repository-names ${ECR_REPO} --region ${AWS_REGION} >/dev/null 2>&1 || \
              aws ecr create-repository --repository-name ${ECR_REPO} --region ${AWS_REGION}

            docker push ${IMAGE_URI}
            docker push ${IMAGE_LATEST}
          '''
        }
      }
    }
  }

  post {
    always {
      sh 'docker system prune -af || true'
    }
    success {
      echo "✅ Image pushed: ${IMAGE_URI}"
    }
  }
}
