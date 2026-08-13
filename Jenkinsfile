pipeline {
  agent {
    docker {
      image 'node:20-alpine'
      args '-u 105:109 -v /var/run/docker.sock:/var/run/docker.sock'
    }
  }
  options {
    skipDefaultCheckout(true)
  }
  stages {
    stage('Checkout') {
      steps {
        cleanWs()
        sh '''
          echo "Starting build process..."
          rm -rf node-app/node_modules || true
        '''
      }
    }
    stage('Build and Test') {
      steps {
        sh '''
          cd node-app
          if [ -f package-lock.json ]; then
            npm ci
          else
            npm install --no-audit --no-fund
          fi
          npm test
        '''
      }
    }
    
    stage('SonarQube Analysis') {
      steps {
        withSonarQubeEnv('sonarqube') {
          sh '''
            apk add --no-cache openjdk17-jre-headless
            export JAVA_HOME=/usr/lib/jvm/java-17-openjdk
            export PATH=$JAVA_HOME/bin:$PATH

            cd node-app
            npx sonar-scanner \
              -Dsonar.projectKey=node-express-app \
              -Dsonar.projectName="Node Express App" \
              -Dsonar.sources=. \
              -Dsonar.exclusions=node_modules/**,coverage/** \
              -Dsonar.host.url=$SONAR_HOST_URL
          '''
        }
      }
    }

    stage('Build and Push Docker Image') {
      environment {
        DOCKER_IMAGE = "gopikakt2005/ultimate-cicd:${BUILD_NUMBER}"
      }
      steps {
        script {
            sh 'docker build -t ${DOCKER_IMAGE} node-app'
            def dockerImage = docker.image("${DOCKER_IMAGE}")
            docker.withRegistry('https://index.docker.io/v1/', "docker-cred") {
                dockerImage.push()
                dockerImage.push("latest")
            }
        }
      }
    }

    stage('Update Deployment File') {
      environment {
        GIT_REPO_NAME = "java-maven-sonar-argocd"
        GIT_USER_NAME = "gopikakt2005"
      }
      steps {
        withCredentials([
            usernamePassword(
                credentialsId: 'git-hub',
                usernameVariable: 'GITHUB_USERNAME',
                passwordVariable: 'GITHUB_TOKEN'
            )
        ]) {
            sh '''
                rm -rf repo-temp
                git clone https://${GITHUB_USERNAME}:${GITHUB_TOKEN}@github.com/${GIT_USER_NAME}/${GIT_REPO_NAME}.git repo-temp
                cd repo-temp
                
                git config user.email "gopikakt2005@gmail.com"
                git config user.name "${GIT_USER_NAME}"

                sed -i "s|image: .*|image: gopikakt2005/ultimate-cicd:${BUILD_NUMBER}|g" node-app-manifests/deployment.yml

                git add node-app-manifests/deployment.yml
                git commit -m "Update static site image tag to ${BUILD_NUMBER} [skip ci]" || echo "No changes to commit"
                git push origin main
            '''
        }
      }
    }
  }
}