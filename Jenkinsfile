pipeline {
  agent { label 'slave' }

  options {
    timestamps()
    buildDiscarder(logRotator(numToKeepStr: '10'))
  }

  environment {
    // Public ECR
    ECR_REGISTRY = "public.ecr.aws/e0f4k4s5"
    ECR_REPO     = "public.ecr.aws/e0f4k4s5/petclinic"
    IMAGE_NAME   = "petclinic"
    IMAGE_TAG    = "${BUILD_NUMBER}"

    // Kubernetes
    //KUBECONFIG_PATH = "/home/jenkins/.kube/config"
    //K8S_NAMESPACE   = "petclinic"
    //EKS_CONTEXT     = "arn:aws:eks:ap-south-1:013461378686:cluster/petclinic-eks"
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
