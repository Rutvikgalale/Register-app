pipeline{
  agent {
    label "jenkins-agent"
  }
    tools{
      jdk 'java21'
      maven 'maven3'
    }
    environment{
      app_name = "register-app"
      docker_user = "rutvikg"
      image_name = "${docker_user}/${app_name}"
      image_tag = "${BUILD_NUMBER}"
      jenkins_api_token = credentials("jenkins_api_token")
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
      stage("code quality gate") {
        steps {
          timeout(time: 5, unit: 'MINUTES') {
            script {
              def qg = waitForQualityGate abortPipeline: false
              echo "SonarQube Quality Gate status: ${qg.status}"
            }
          }
        }
      }

      stage("docker build & push"){
        steps{
          script{
            withCredentials([usernamePassword(credentialsId: "docker", usernameVariable: "docker_user", passwordVariable: "docker_pass")]){
              sh """
              echo $docker_pass | docker login -u $docker_user --password-stdin
              docker build -t "${image_name}:${image_tag}" .
              docker push "${image_name}:${image_tag}"
              """
            }
          }
        }
      }

      stage("trivy scanning"){
        steps{
          script{
            sh ('docker run --rm -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image ${image_name}:${image_tag} --no-progress --scanners vuln  --exit-code 0 --severity HIGH,CRITICAL --format table')
          }
        }
      }

      stage("trigger cd pipeline"){
        steps{
          script{
            sh "curl -v -k --user admin:${jenkins_api_token} -X POST -H 'cache-control: no-cache' -H 'content-type: application/x-www-form-urlencoded' --data 'image_tag=${image_tag}' 'http://ec2-13-201-62-90.ap-south-1.compute.amazonaws.com:8080/job/register-app-cd/buildWithParameters?token=cd-token'"
          }
        }
      }
      stage("update deployment tags"){
        steps{
          sh """
            sed -i "s|image:.*|image: ${image_name}:${image_tag}|g" /home/ubuntu/workspace/register-app/manifests/deployment.yaml
            cat /home/ubuntu/workspace/register-app/manifests/deployment.yaml
            git add /home/ubuntu/workspace/register-app/manifests/deployment.yaml
          """
        }
      }

      stage("pushing deployment file to Git"){
        steps{
          sh """
          git config --global user.name "Rutvikgalale"
          git config --global user.email "rutvikgalale16@gmail.com"
          git add /home/ubuntu/workspace/register-app/manifests/deployment.yaml
          git commit -m "deployment manifest updated"
          """
          withCredentials([usernamePassword(credentialsId: 'github', usernameVariable: 'GIT_USER', passwordVariable: 'GIT_PASS')]) {
            sh """
              git push https://${GIT_USER}:${GIT_PASS}@github.com/Rutvikgalale/Register-app main
            """
          }
        }
      }
    }
    post{
      always{
        sh """
          trivy clean --all || true
          docker system prune -af || true
        """
      }
    }
}
