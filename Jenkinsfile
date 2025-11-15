// Jenkinsfile - build image, push to registry, and deploy to remote Ubuntu server via SSH
pipeline {
  agent any

  environment {
    // Change these to match your setup in Jenkins > Credentials
    DOCKER_REGISTRY = "docker.io"                    // or your registry host
    DOCKER_REPO     = "vikas0101/web" // e.g. myuser/staticsite
    SSH_TARGET      = "ubuntu@13.232.188.41"         // user@EC2_PUBLIC_IP
    SSH_KEY_CRED    = "ec2-ssh-key"                  // Jenkins SSH private key credential id
    DOCKER_CRED_ID  = "docker-hub-creds"             // username/password or token credential id
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
    }

    stage('Prepare image') {
      steps {
        script {
          // image tag using short commit hash and timestamp
          SHORT_SHA = sh(script: "git rev-parse --short HEAD", returnStdout: true).trim()
          IMAGE_TAG = "${env.DOCKER_REPO}:${SHORT_SHA}"
          IMAGE_LATEST = "${env.DOCKER_REPO}:latest"
        }
      }
    }

    stage('Build Docker image') {
      steps {
        // Create a simple Dockerfile if repository doesn't have one
        script {
          if (!fileExists('Dockerfile')) {
            writeFile file: 'Dockerfile', text: """
            FROM nginx:alpine
            COPY . /usr/share/nginx/html
            """
          }
        }

        // build image
        sh "docker build -t ${IMAGE_TAG} -t ${IMAGE_LATEST} ."
      }
    }

    stage('Login & Push image') {
      steps {
        withCredentials([usernamePassword(credentialsId: "${DOCKER_CRED_ID}", usernameVariable: 'REG_USER', passwordVariable: 'REG_PSW')]) {
          sh """
            echo "$REG_PSW" | docker login ${DOCKER_REGISTRY} -u "$REG_USER" --password-stdin
            docker push ${IMAGE_TAG}
            docker push ${IMAGE_LATEST}
            docker logout ${DOCKER_REGISTRY} || true
          """
        }
      }
    }

    stage('Deploy to EC2') {
      steps {
        // We'll use SSH agent plugin (Jenkins) to SSH to server and run docker commands
        sshagent (credentials: [env.SSH_KEY_CRED]) {
          sh """
            ssh -o StrictHostKeyChecking=no ${SSH_TARGET} << 'EOF'
              # create docker dir and pull image
              docker pull ${IMAGE_TAG} || docker pull ${IMAGE_LATEST}

              # stop & remove existing container (if exists)
              if docker ps -a --format '{{.Names}}' | grep -q '^static-site\$'; then
                docker stop static-site || true
                docker rm static-site || true
              fi

              # run new container (map port 80 on host)
              docker run -d --name static-site -p 80:80 --restart=always ${IMAGE_TAG} || docker run -d --name static-site -p 80:80 --restart=always ${IMAGE_LATEST}

              # optional: prune unused images
              docker image prune -f
            EOF
          """
        }
      }
    }
  }

  post {
    success {
      echo "Deployed ${IMAGE_TAG} to ${SSH_TARGET}"
    }
    failure {
      echo "Build or deploy failed"
    }
  }
}
