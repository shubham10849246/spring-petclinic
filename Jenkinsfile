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

    stage('Container Image Scan (Trivy)') {
  agent { label 'slave1' }
  steps {
    sh '''
      mkdir -p reports/security
      trivy image \
        --severity HIGH,CRITICAL \
        --exit-code 1 \
        --format json \
        -o reports/security/trivy-report.json \
        ${IMAGE_URI}
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

    stage('ArgoCD Sync Validation') {
  agent { label 'slave2' }
  steps {
    sh '''
      APP_STATUS=$(argocd app get spring-petclinic -o json)

      HEALTH=$(echo "$APP_STATUS" | jq -r '.status.health.status')
      SYNC=$(echo "$APP_STATUS" | jq -r '.status.sync.status')

      echo "Health: $HEALTH"
      echo "Sync: $SYNC"

      if [ "$HEALTH" != "Healthy" ] || [ "$SYNC" != "Synced" ]; then
        echo "❌ ArgoCD validation failed"
        exit 1
      fi
    '''
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

    stage('Metrics & Evidence') {
  agent { label 'slave1' }
  steps {
    archiveArtifacts artifacts: 'reports/**/*', fingerprint: true
    junit allowEmptyResults: true, testResults: '''
      target/surefire-reports/*.xml,
      target/failsafe-reports/*.xml
    '''
  }
}
  }

  
  post {
  success {
    echo "✅ Image pushed: ${env.IMAGE_URI}"
  }
  always {
    node('slave1') {
      sh '''
        docker system prune -af --volumes || true
      '''
    }
  }
}
}

