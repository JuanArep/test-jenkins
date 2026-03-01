pipeline {
  agent any

  environment {
    // SonarQube project key (must be unique in SonarQube)
    SONAR_PROJECT_KEY = "my-spring-ci"
    SONAR_HOST_URL    = "http://localhost:9000"
  }

  triggers {
    // For local WSL Jenkins, GitHub webhook is often not reachable.
    // Poll SCM every 2 minutes for now. You can switch to webhooks later.
    pollSCM('H/2 * * * *')
  }

  stages {
    stage('Checkout') {
      steps {
        checkout scm
      }
      post {
        success { echo "Checkout completed" }
      }
    }

    stage('Build & Unit Test') {
      steps {
        sh 'mvn -B clean test'
      }
      post {
        success { echo "Tests passed" }
      }
    }

    stage('SonarQube Scan') {
      steps {
        // Uses Jenkins-configured SonarQube server context if plugin is installed.
        // If withSonarQubeEnv is unavailable, we’ll fall back to explicit args.
        script {
          try {
            withSonarQubeEnv('sonarqube') {
              sh """
                mvn -B sonar:sonar \
                  -Dsonar.projectKey=${SONAR_PROJECT_KEY}
              """
            }
          } catch (Exception e) {
            echo "withSonarQubeEnv not available; using explicit host/token env vars"
            sh """
              mvn -B sonar:sonar \
                -Dsonar.projectKey=${SONAR_PROJECT_KEY} \
                -Dsonar.host.url=${SONAR_HOST_URL}
            """
          }
        }
      }
    }

    stage('Quality Gate') {
      steps {
        // Requires SonarQube webhook to Jenkins for best behavior.
        // If not configured, this may time out. It's still useful to learn the concept.
        timeout(time: 5, unit: 'MINUTES') {
          waitForQualityGate abortPipeline: true
        }
      }
    }

    stage('Package') {
      steps {
        sh 'mvn -B clean package -DskipTests'
        sh 'ls -lah target/*.jar'
      }
    }

    stage('Archive Artifact') {
      steps {
        archiveArtifacts artifacts: 'target/*.jar', fingerprint: true
      }
    }

    stage('Smoke Test') {
      steps {
        sh '''
          set -e
          JAR=$(ls -1 target/*.jar | head -n 1)

          # Start app in background on 8081 to avoid collisions
          nohup java -jar "$JAR" --server.port=8081 > app.log 2>&1 &
          APP_PID=$!

          # Wait briefly then hit endpoint
          sleep 5
          curl -fsS http://localhost:8081/api/hello | grep -q "hello"

          # Cleanup
          kill $APP_PID
          echo "Smoke test passed"
        '''
      }
      post {
        always {
          archiveArtifacts artifacts: 'app.log', allowEmptyArchive: true
        }
      }
    }
  }
}
