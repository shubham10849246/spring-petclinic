pipeline {
  agent any

  tools {
    // Configure these names in: Manage Jenkins -> Global Tool Configuration
    maven 'Maven3'
    jdk   'JDK17'
  }

  environment {

    JAVA_HOME = "/usr/lib/jvm/java-17-openjdk-amd64"
    PATH = "${JAVA_HOME}/bin:${env.PATH}
    // ----- APP SETTINGS -----
    APP_NAME     = 'spring-petclinic'
    DOCKERFILE   = 'Dockerfile'
    DOCKER_CTX   = '.'

    // ----- AWS/ECR SETTINGS -----
    AWS_REGION   = 'ap-south-1'
    AWS_ACCOUNT  = '713332525966'
    ECR_REPO     = 'petclinic'  // must exist in ECR
    ECR_REGISTRY = "${AWS_ACCOUNT}.dkr.ecr.${AWS_REGION}.amazonaws.com"

    // ----- SONAR SETTINGS -----
    SONAR_PROJECT_KEY = 'petclinic'
    // SonarQube server name configured in Jenkins
    SONARQUBE_SERVER   = 'sonarqube'

    // Tagging
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
        sh 'echo "Commit: ${GIT_COMMIT}"'
      }
    }

    stage('Maven Build (compile/package)') {
      steps {
        sh 'mvn -B -U clean package'
      }
      post {
        always {
          archiveArtifacts artifacts: 'target/*.jar', fingerprint: true, onlyIfSuccessful: false
        }
      }
    }

    stage('Unit Testing') {
      steps {
        sh 'mvn -B test'
      }
      post {
        always {
          junit 'target/surefire-reports/*.xml'
        }
      }
    }

    stage('Functional Testing') {
      steps {
        // Typical options:
        // 1) mvn verify with integration test profile
        // 2) run Postman/Newman tests against a started app
        // Below is a simple Maven IT approach
        sh 'mvn -B verify -Pfunctional-tests'
      }
      post {
        always {
          junit 'target/failsafe-reports/*.xml'
        }
      }
    }

    stage('Performance Testing') {
      steps {
        // Example: JMeter non-GUI run (requires jmeter installed on agent)
        // If you don't have jmeter, I can give Docker-based JMeter stage.
        sh '''
          mkdir -p perf/results perf/report || true
          if command -v jmeter >/dev/null 2>&1; then
            jmeter -n -t perf/testplan.jmx -l perf/results/results.jtl
          else
            echo "JMETER not installed. Skipping perf test (install jmeter or use Docker)."
            exit 0
          fi
        '''
      }
      post {
        always {
          archiveArtifacts artifacts: 'perf/results/**/*', onlyIfSuccessful: false
        }
      }
    }

    stage('SonarQube Scan') {
      steps {
        withSonarQubeEnv("${SONARQUBE_SERVER}") {
          sh """
            mvn -B sonar:sonar \
              -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
              -Dsonar.projectName=${APP_NAME}
          """
        }
      }
    }

    stage('Quality Gate (Optional but recommended)') {
      steps {
        // Requires: SonarQube webhook configured in Jenkins
        timeout(time: 10, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }

    stage('Docker Build') {
      steps {
        sh """
          docker build -f ${DOCKERFILE} -t ${IMAGE_URI} ${DOCKER_CTX}
          docker tag ${IMAGE_URI} ${IMAGE_LATEST}
          docker images | head -n 20
        """
      }
    }

    stage('ECR Login & Push') {
      steps {
        withCredentials([[
          $class: 'AmazonWebServicesCredentialsBinding',
          credentialsId: 'aws-jenkins-creds'   // <-- change to your Jenkins credential ID
        ]]) {
          sh """
            aws --version
            aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_REGISTRY}

            # Ensure repo exists (optional)
            aws ecr describe-repositories --repository-names ${ECR_REPO} --region ${AWS_REGION} >/dev/null 2>&1 || \
              aws ecr create-repository --repository-name ${ECR_REPO} --region ${AWS_REGION}

            docker push ${IMAGE_URI}
            docker push ${IMAGE_LATEST}
          """
        }
      }
    }
  }

  post {
    success {
      echo "✅ SUCCESS: Image pushed to ECR -> ${IMAGE_URI}"
    }
    failure {
      echo "❌ FAILED: Check console logs for stage that failed."
    }
    always {
      // Cleanup to save disk on agent
      sh 'docker system prune -af || true'
    }
  }
}
