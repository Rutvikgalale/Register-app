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
          cleanws()
        }
      }
      stage("checkout code"){
        steps{
          git branch: 'main', credentialsId: 'github', url: 'git@github.com:Rutvikgalale/Register-app.git'
        }
      }
    }
}
