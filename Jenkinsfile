pipeline{
  agent {
    label "jenkins-agent"
  }
    tools{
      jdk 'java21'
      git 'git'
    }
    stages{
      stage("cleaning workspace"){
        steps{
          cleanWs()
        }
      }
      stage("checkout code"){
        steps{
          git branch: 'main', credentialsId: 'github', url: 'git@github.com:Rutvikgalale/Register-app.git'
        }
      }
    }
}
