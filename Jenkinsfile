pipeline {
  agent { label 'slave' }

  options {
    timestamps()

    buildDiscarder(logRotator(numToKeepStr: '10'))
  }

  environment {
    // Public ECR
    ECR_REGISTRY = "public.ecr.aws/q3i5i7u5"
    ECR_REPO     = "public.ecr.aws/q3i5i7u5/spring-petclinic"
    IMAGE_NAME   = "petclinic"
    IMAGE_TAG    = "${BUILD_NUMBER}"

    // Kubernetes (commented for now)
    // KUBECONFIG_PATH = "/home/jenkins/.kube/config"
    // K8S_NAMESPACE   = "petclinic"
    // EKS_CONTEXT     = "arn:aws:eks:ap-south-1:013461378686:cluster/petclinic-eks"
  }

  stages {

    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Build (Maven)') {
      steps {
        sh '''
          java -version
          mvn -v
          mvn clean package -DskipTests
        '''
      }
      post {
        success {
          archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
        }
      }
    }


    stage('Unit Tests (Soft Gate)') {
      options { timeout(time: 10, unit: 'MINUTES') }
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
          sh '''
            mvn -B test
          '''
        }
      }
      post {
        always {
          junit allowEmptyResults: true, testResults: 'target/surefire-reports/*.xml'
        }
      }
    }

stage('Functional Tests (Soft Gate)') {
  options { timeout(time: 25, unit: 'MINUTES') }
  steps {
    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
      sh '''
        # Start app in background (example: Spring Boot)
        nohup mvn -B spring-boot:run > app.log 2>&1 &
        APP_PID=$!

        # Wait until app is up (adjust port/endpoint)
        for i in {1..30}; do
          curl -s http://localhost:8080/actuator/health && break
          sleep 2
        done

        # Run integration tests
        mvn -B verify -Pfunctional-tests

        # Stop app
        kill $APP_PID || true
      '''
    }
  }
  post {
    always {
      junit allowEmptyResults: true, testResults: 'target/failsafe-reports/*.xml'
      archiveArtifacts artifacts: 'app.log,target/failsafe-reports/**', allowEmptyArchive: true
    }
  }
}

    stage('Performance Tests (Soft Gate)') {
  options { timeout(time: 30, unit: 'MINUTES') }
  steps {
    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
      sh '''
        rm -rf performance-results
        mkdir -p performance-results/report

        # Ensure JMeter is on PATH OR use /opt/jmeter/bin/jmeter
        jmeter -n \
          -t perf/petclinic.jmx \
          -l performance-results/results.jtl \
          -e -o performance-results/report

        echo "JMeter run completed. Reports generated in performance-results/report"
      '''
    }
  }
  post {
    always {
      archiveArtifacts artifacts: 'performance-results/**', allowEmptyArchive: true
    }
  }
}
    stage('Docker Build') {
      steps {
        sh '''
          docker --version
          docker build -t ${IMAGE_NAME}:${IMAGE_TAG} .
        '''
      }
    }

    stage('Push to Public ECR') {
      steps {
        sh '''
          aws --version
          aws sts get-caller-identity

          aws ecr-public get-login-password --region us-east-1 \
          | docker login --username AWS --password-stdin ${ECR_REGISTRY}

          docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${ECR_REPO}:${IMAGE_TAG}
          docker push ${ECR_REPO}:${IMAGE_TAG}

          docker tag ${IMAGE_NAME}:${IMAGE_TAG} ${ECR_REPO}:latest
          docker push ${ECR_REPO}:latest
        '''
      }
    }

    // Deploy stages can be added back later when kubeconfig is ready
    // stage('Deploy to EKS') { ... }
    // stage('Verify Deployment') { ... }

  } // ✅ closes stages

  post {
    success {
      echo "✅ CI/CD SUCCESS: Image pushed to ECR (deploy stage skipped/commented)"
    }
    failure {
      echo "❌ Pipeline failed – check logs above"
    }
  }

} // ✅ closes pipeline
