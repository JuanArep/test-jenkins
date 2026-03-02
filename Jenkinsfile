pipeline {
  agent any

  environment {
    SONAR_PROJECT_KEY = "my-spring-ci"
    SONAR_HOST_URL    = "http://localhost:9000"
  }

  triggers {
    pollSCM('H/2 * * * *')
  }

  stages {
    stage('Build & Unit Test') {
      steps {
        sh 'mvn -B clean test'
      }
      post {
        always {
          junit testResults: 'target/surefire-reports/*.xml', allowEmptyResults: false
        }
      }
    }

    stage('SonarQube Scan') {
      steps {
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

          echo "Starting app from $JAR on port 8081..."
          nohup java -jar "$JAR" --server.port=8081 > app.log 2>&1 &
          APP_PID=$!
          echo "App PID: $APP_PID"

          for i in $(seq 1 30); do
            if curl -fsS http://localhost:8081/api/hello | grep -q "hello"; then
              echo "Smoke test passed"
              break
            fi
            echo "Waiting for app... attempt $i/30"
            sleep 2
          done

          curl -fsS http://localhost:8081/api/hello | grep -q "hello"

          kill "$APP_PID" || true
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