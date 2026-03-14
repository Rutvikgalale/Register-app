pipeline{
  agent {
    label "jenkins-agent"
  }
    tools{
      jdk 'java21'
      maven 'maven3'
    }
    stages{
      stage("cleaning workspace"){
        steps{
          cleanWs()
        }
      }
      stage("checkout code"){
        steps{
          git branch: 'main', credentialsId: 'github', url: 'https://github.com/Rutvikgalale/Register-app.git'
        }
      }
      stage("code build"){
        steps{
          sh 'mvn clean package'
        }
      }
      stage("code test"){
        steps{
          sh 'mvn test'
        }
      }
      stage("code quality analysis"){
        steps{
          withSonarQubeEnv('sonar'){ // 'sonar' is the name you configured in Manage Jenkins → System
            withCredentials([string(credentialsId: 'sonar-token', variable: 'sonar_token')]) {
              sh 'sonar-scanner \
                -Dsonar.projectKey=Register-app \
                -Dsonar.sources=src \
                -Dsonar.host.url=http://172.31.47.102:9000 \
                -Dsonar.login=$sonar_token'
          }
        }
      }
    }
}
