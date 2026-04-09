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
    
    SONAR_PROJECT_KEY  = 'petclinic'
    SONARQUBE_SERVER   = 'sonarqube'  // this must match the name in Jenkins Sonar config

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

    
    stage('SonarQube Scan') {
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
      steps {
        timeout(time: 10, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
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
    sh '''
      echo "Logging into ECR..."
      aws ecr get-login-password --region ${AWS_REGION} | \
        docker login --username AWS --password-stdin ${ECR_REGISTRY}

      echo "Ensuring ECR repo exists..."
      aws ecr describe-repositories \
        --repository-names ${ECR_REPO} \
        --region ${AWS_REGION} \
        >/dev/null 2>&1 || \
      aws ecr create-repository \
        --repository-name ${ECR_REPO} \
        --region ${AWS_REGION}

      echo "Pushing images..."
      docker push ${IMAGE_URI}
      docker push ${IMAGE_LATEST}
    '''
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
