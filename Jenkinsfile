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
  }

  options {
    timestamps()
    disableConcurrentBuilds()
  }
  
  parameters {
    booleanParam(name: 'RUN_SONAR', defaultValue: true, description: 'Run SonarQube scan + Quality Gate')
    booleanParam(name: 'RUN_IT', defaultValue: false, description: 'Run Integration Tests (Testcontainers) - requires Docker')
  }


  stages {

    
    stage('Checkout') {
      agent { label 'slave1' }
      steps {
        cleanWs()
        checkout scm

        script {
          // Compute tags AFTER checkout so env.GIT_COMMIT exists
          def shortGit = sh(script: "git rev-parse --short=7 HEAD", returnStdout: true).trim()
          env.GIT_SHORT = shortGit
          env.IMAGE_TAG = "${env.BUILD_NUMBER}-${shortGit}"
          env.IMAGE_URI = "${env.ECR_REGISTRY}/${env.ECR_REPO}:${env.IMAGE_TAG}"
          echo "Computed IMAGE_URI = ${env.IMAGE_URI}"
        }
      }
    }
    
    stage('Pre-flight (Tools)') {
      agent { label 'slave1' }
      steps {
        sh '''
          set -e
          echo "=== JAVA ==="
          java -version
          echo "=== MAVEN ==="
          mvn -version
          echo "=== DOCKER (if installed) ==="
          docker version || true
          echo "=== AWS CLI (if installed) ==="
          aws --version || true
        '''
      }
    }
    
    stage('Build + Unit Tests (skip IT)') {
      agent { label 'slave1' }
      steps {
        wrap([$class: 'AnsiColorBuildWrapper', colorMapName: 'xterm']) {
        sh '''
          set -e
          mvn -B -U clean verify -DskipITs=true
        '''
	}
      }
      post {
        always {
          junit allowEmptyResults: true, testResults: 'target/surefire-reports/*.xml'
          archiveArtifacts artifacts: 'target/*.jar,target/site/jacoco/**', fingerprint: true, onlyIfSuccessful: false
        }
      }
    }
	
    stage('Integration Tests (Testcontainers)') {
      when { expression { return params.RUN_IT } }
      agent { label 'slave1' }
      steps {
        sh '''
          set -e
          echo "Checking Docker availability for Testcontainers..."
          docker info >/dev/null 2>&1          
          mvn -B -U verify \
            -DskipITs=false \
            -Dspring.profiles.active=default \
            -DskipPostgresITs=true
        '''
      }
      post {
        always {
          junit allowEmptyResults: true, testResults: 'target/failsafe-reports/*.xml'
        }
      }
    }

    
    stage('SonarQube Scan') {
      when { expression { return params.RUN_SONAR } }
      agent { label 'slave1' }
      steps {
        withSonarQubeEnv(env.SONARQUBE_SERVER) {
          sh """
            mvn -B sonar:sonar \
              -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
              -Dsonar.projectName=${APP_NAME} \
              -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
          """
        }
      }
    }

    stage('Quality Gate') {
      when { expression { return params.RUN_SONAR } }
      agent { label 'slave1' }
      steps {
        timeout(time: 3, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }
    
    
    stage('Docker Build') {
      agent { label 'slave1' }
      steps {
        sh '''
          set -e
          docker build --pull -t ${IMAGE_URI} .
          docker images | head -n 20
        '''
      }
    }

    stage('ECR Login & Push') {
      agent { label 'slave1' }
      steps {
        sh '''
          set -e

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
        echo "Update GitOps manifests to use ${IMAGE_URI} and push to Git."
        // Example (uncomment & configure your Git credentials + repo):
        // sh '''
        //   git clone https://github.com/shubham10849246/petclinic-gitops.git gitops
        //   cd gitops
        //   sed -i "s|image: .*|image: ${IMAGE_URI}|g" k8s/deployment.yaml
        //   git add k8s/deployment.yaml
        //   git commit -m "Update image to ${IMAGE_URI}"
        //   git push
        // '''
      }
    }

    stage('Post-Deploy Smoke Test') {
      agent { label 'slave2' }
      steps {
        echo "Run smoke checks (HTTP 200 /actuator/health)."
        // Example:
        // sh 'curl -f http://<your-service-url>/actuator/health'
      }
    }
  }

  
  post {
  success {
    echo "✅ Image pushed: ${env.IMAGE_URI}"
  }
  always {
    node('slave1') {
      sh 'docker image prune -f || true'
    }
  }
}
}

