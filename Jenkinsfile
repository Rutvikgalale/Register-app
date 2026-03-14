pipeline{
  agent {
    label "jenkins-agent"
  }
    tools{
      jdk 'java21'
      maven 'maven3'
    }
    environment{
      app_name = "Register-app"
      docker_user = "rutvikg"
      docker_pass = "docker"
      image_name = "${docker_user}/${app_name}"
      image_tag = "${image_name}-${build_number}"
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
              script{
                def scannerHome = tool 'sonar' //// 'sonar' must match the name in  manage jenkins -> tool configuraion
                sh """
                ${scannerHome}/bin/sonar-scanner \
                -Dsonar.projectKey=Register-app \
                -Dsonar.sources=. \
                -Dsonar.java.binaries=server/target/classes,webapp/target/webapp/WEB-INF/classes \
                -Dsonar.host.url=http://172.31.47.102:9000 \
                -Dsonar.login=$sonar_token
                """
              }
            }
          }
        }
      }
      stage("code quality gate"){
        steps{
          timeout(time: 5, unit: 'MINUTES') { // Jenkins will wait up to 5 minutes for SonarQube to send back the analysis
            waitForQualityGate abortPipeline: false
          }
        }
      }
      stage("docker build & push"){
        steps{
          script{
            withCredentials([usernamePassword(credentialsId: "docker", username: "docker_user", password: "docker_pass")]){
              sh """
              echo $DOCKER_PASS | docker login -u $DOCKER_USER --password-stdin
              docker build -t "${docker_user}/${app_name}:${$build_number}" .
              docker push "${docker_user}/${app_name}:${build_number}"
    }
}
