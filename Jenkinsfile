pipeline {
  agent any

  environment {
    // AWS credentials and configuration
    AWS_ACCESS_KEY_ID     = credentials('aws-access-key-id')
    AWS_SECRET_ACCESS_KEY = credentials('aws-secret-access-key')
    AWS_DEFAULT_REGION    = 'us-east-1' // Change to your region
    ECR_REPOSITORY        = 'my-ecr-repo' // Your ECR repository name
    EKS_CLUSTER_NAME      = 'my-eks-cluster' // Your EKS cluster name
    DOCKER_IMAGE_TAG      = 'latest'

    // Nexus configuration
    NEXUS_URL             = 'http://your-nexus-server:8081' // Nexus server URL
    NEXUS_REPOSITORY      = 'maven-releases' // Nexus repository name
    NEXUS_CREDENTIALS_ID  = 'nexus-credentials' // Jenkins credentials ID for Nexus
  }

  stages {
    // Stage 1: Build the application
    stage('Build') {
      steps {
        sh 'mvn clean package'
      }
    }

    // Stage 2: Run Checkmarx Security Scan
    stage('Checkmarx Security Scan') {
      steps {
        withCredentials([usernamePassword(credentialsId: 'checkmarx-creds', usernameVariable: 'CX_USER', passwordVariable: 'CX_PASS')]) {
          sh '''
            ./checkmarx-cli scan \
              --project-name "my-project" \
              --source-dir . \
              --username $CX_USER \
              --password $CX_PASS
          '''
        }
      }
    }

    // Stage 3: Run SonarQube Analysis
    stage('SonarQube Analysis') {
      steps {
        withSonarQubeEnv('SonarQube-Server') {
          sh 'mvn sonar:sonar'
        }
      }
    }

    // Stage 4: Upload Artifact to Nexus
    stage('Upload to Nexus') {
      steps {
        nexusArtifactUploader(
          nexusVersion: 'nexus3',
          protocol: 'http',
          nexusUrl: "${NEXUS_URL}",
          groupId: 'com.example',
          version: '1.0.0',
          repository: "${NEXUS_REPOSITORY}",
          credentialsId: "${NEXUS_CREDENTIALS_ID}",
          artifacts: [
            [artifactId: 'my-app',
             classifier: '',
             file: 'target/my-app.jar',
             type: 'jar']
          ]
        )
      }
    }

    // Stage 5: Build and Push Docker Image to AWS ECR
    stage('Build and Push Docker Image') {
      steps {
        script {
          // Authenticate Docker to AWS ECR
          sh '''
            aws ecr get-login-password --region ${AWS_DEFAULT_REGION} | \
            docker login --username AWS --password-stdin ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com
          '''

          // Build Docker image
          sh "docker build -t ${ECR_REPOSITORY}:${DOCKER_IMAGE_TAG} ."

          // Tag Docker image
          sh "docker tag ${ECR_REPOSITORY}:${DOCKER_IMAGE_TAG} ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${ECR_REPOSITORY}:${DOCKER_IMAGE_TAG}"

          // Push Docker image to ECR
          sh "docker push ${AWS_ACCOUNT_ID}.dkr.ecr.${AWS_DEFAULT_REGION}.amazonaws.com/${ECR_REPOSITORY}:${DOCKER_IMAGE_TAG}"
        }
      }
    }

    // Stage 6: Deploy to AWS EKS
    stage('Deploy to EKS') {
      steps {
        script {
          // Update kubeconfig to access the EKS cluster
          sh "aws eks update-kubeconfig --name ${EKS_CLUSTER_NAME} --region ${AWS_DEFAULT_REGION}"

          // Deploy to Kubernetes using kubectl
          sh "kubectl apply -f deployment.yaml"
        }
      }
    }
  }

  // Post-build actions (optional)
  post {
    success {
      echo 'Pipeline completed successfully!'
    }
    failure {
      echo 'Pipeline failed. Check the logs for details.'
    }
  }
}
