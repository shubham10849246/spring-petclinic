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

    
    PERF_PORT    = "8080"
    PERF_BASEURL = "http://localhost:${PERF_PORT}"
    JMETER_PLAN  = "perf/petclinic.jmx"
    PERF_OUTDIR  = "performance-results"

  }

  stages {

    stage('Checkout') {
      steps {
	deleteDir()
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
      options { timeout(time: 15, unit: 'MINUTES') }
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
          sh '''#!/usr/bin/env bash
            set -euxo pipefail
            mvn -B test
          '''
        }
      }
      post {
        always {
          junit allowEmptyResults: true, testResults: 'target/surefire-reports/*.xml'
          archiveArtifacts artifacts: 'target/surefire-reports/**', allowEmptyArchive: true
        }
      }
    }

    
    
    stage('Functional Tests (Soft Gate)') {
      options { timeout(time: 30, unit: 'MINUTES') }
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
          sh '''#!/usr/bin/env bash
            set -euxo pipefail
            # Skip unit tests to avoid rerun during verify lifecycle
            mvn -B verify -Pfunctional-tests -DskipUnitTests=true
          '''
        }
      }
      post {
        always {
          junit allowEmptyResults: true, testResults: 'target/failsafe-reports/*.xml'
          archiveArtifacts artifacts: 'target/failsafe-reports/**', allowEmptyArchive: true
        }
      }
    }

    stage('Functional Tests - Postgres (Soft Gate)') {
  options { timeout(time: 30, unit: 'MINUTES') }
  steps {
    catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
      sh '''#!/usr/bin/env bash
      set -euxo pipefail

      # Quick docker sanity check
      docker ps

      mvn -B verify -Ppostgres-tests -DskipUnitTests=true
      '''
    }
  }
  post {
    always {
      junit allowEmptyResults: true, testResults: 'target/failsafe-reports/*.xml'
      archiveArtifacts artifacts: 'target/failsafe-reports/**', allowEmptyArchive: true
    }
  }
}
    
    stage('Performance Tests (Soft Gate)') {
      options { timeout(time: 40, unit: 'MINUTES') }
      steps {
        catchError(buildResult: 'SUCCESS', stageResult: 'FAILURE') {
          sh '''#!/usr/bin/env bash
            set -euxo pipefail

            if [ ! -f "${JMETER_PLAN}" ]; then
              echo "❌ JMeter plan not found: ${JMETER_PLAN}"
              echo "👉 Place your test plan at ${JMETER_PLAN} or update JMETER_PLAN in Jenkinsfile"
              exit 1
            fi

            
            # Find JMeter binary
            if command -v jmeter >/dev/null 2>&1; then
              JMETER_BIN="jmeter"
            elif [ -x "/opt/jmeter/bin/jmeter" ]; then
              JMETER_BIN="/opt/jmeter/bin/jmeter"
            else
              echo "❌ JMeter not found. Install and expose 'jmeter' in PATH or use /opt/jmeter/bin/jmeter"
              exit 1
            fi

            # Start app from built jar on fixed port
            JAR_FILE=$(ls -1 target/*.jar | head -n 1)
            echo "Using jar: ${JAR_FILE}"

            rm -f app-perf.log app.pid || true

            nohup java -jar "${JAR_FILE}" --server.port=${PERF_PORT} > app-perf.log 2>&1 &
            echo $! > app.pid

            echo "Waiting for health: ${PERF_BASEURL}/actuator/health"
            for i in $(seq 1 60); do
              if curl -fsS "${PERF_BASEURL}/actuator/health" >/dev/null 2>&1; then
                echo "✅ App is UP"
                
break
              fi
              sleep 2
            done

            if ! curl -fsS "${PERF_BASEURL}/actuator/health" >/dev/null 2>&1; then
              echo "❌ App did not become ready; last logs:"
              tail -n 200 app-perf.log || true
              exit 1
            fi

            # Run JMeter
            rm -rf "${PERF_OUTDIR}" || true
            mkdir -p "${PERF_OUTDIR}/report"

            ${JMETER_BIN} -n \
              -t "${JMETER_PLAN}" \
              -l "${PERF_OUTDIR}/results.jtl" \
              -e -o "${PERF_OUTDIR}/report" \
              -JbaseUrl="${PERF_BASEURL}"

            echo "✅ JMeter finished"
          '''
        }
      }
      
      post {
        always {
          archiveArtifacts artifacts: 'app-perf.log,performance-results/**', allowEmptyArchive: true
        }
      }
    }
    
    stage('SonarQube Scan') {
  steps {
    withSonarQubeEnv('sonarqube') {
      sh '''#!/usr/bin/env bash
      set -e
      mvn -B sonar:sonar \
        -Dsonar.projectKey=spring-petclinic \
        -Dsonar.projectName="Spring Petclinic" \
        -Dsonar.coverage.jacoco.xmlReportPaths=target/site/jacoco/jacoco.xml
      '''
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
always {
    sh '''
      sudo chown -R jenkins:jenkins $WORKSPACE || true
      rm -rf $WORKSPACE/target || true
    '''
    clean
}
    success {
      echo "✅ CI/CD SUCCESS: Image pushed to ECR (deploy stage skipped/commented)"
    }
    failure {
      echo "❌ Pipeline failed – check logs above"
    }
  }

} // ✅ closes pipeline
