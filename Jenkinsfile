pipeline {
  agent any
  environment {
    AWS_REGION = 'us-east-1'
    ECR_REPO = '487383300987.dkr.ecr.us-east-1.amazonaws.com/car-mod'
    DOCKER_IMAGE = "car-mod:latest"
    AWS_CREDENTIALS = credentials('aws-credentials')
    NODE_CREDENTIALS = credentials('node')
    EC2_USER = 'ubuntu'
    EC2_HOST = 'EC2 - Pub IP'
  }
  stages {
    stage('Checkout Code') {
      steps {
        // For private repo with credentials:
        git branch: 'jenkins', credentialsId: 'Jenkins-prod', url: 'repo URL'
      }
    }
    stage('Set Up Virtual Environment') {
      steps {
        sh '''
          python3 -m venv venv
          . venv/bin/activate
          pip install --upgrade pip
          pip install -r src/requirements.txt
        '''
      }
    }
    stage('Run Tests') {
      steps {
        sh '''
          . venv/bin/activate
          pytest
        '''
      }
    }
    stage('Build Docker Image') {
      steps {
        sh 'docker build -t car-mod:latest .'
      }
    }
    stage('Scan Docker Image') {
      steps {
        sh '''
          trivy image --severity HIGH,CRITICAL $DOCKER_IMAGE
          trivy image --exit-code 1 --severity CRITICAL --ignore-unfixed $DOCKER_IMAGE
        '''
      }
    }
    stage('Push to ECR') {
      steps {
        sh '''
          aws ecr get-login-password --region $AWS_REGION | docker login --username AWS --password-stdin $ECR_REPO
          docker tag $DOCKER_IMAGE $ECR_REPO:latest
          docker push $ECR_REPO:latest
        '''
      }
    }
    stage('Deploy to EC2 Node') {
      steps {
          sh '''
            ssh -o StrictHostKeyChecking=no -T -i $NODE_CREDENTIALS $SSH_KEY $EC2_USER@$EC2_HOST << EOF
            aws ecr get-login-password --region us-east-1 | docker login --username AWS --password-stdin 487383300987.dkr.ecr.us-east-1.amazonaws.com
            docker pull $ECR_REPO:latest
            # Check if container exists
            if docker ps -a --format '{{.Names}}' | grep -q "^car-mod-app$"; then
              echo "Removing existing container..."
              docker rm -f car-mod-app
            fi
    
            # Run the container
            docker run -d --name car-mod-app -p 9000:9000 $ECR_REPO:latest
          '''
        }
    }
  }
}