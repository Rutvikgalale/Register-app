install & configure sonar:
sudo apt update
sudo apt upgrade -y

installation of postgres:
sudo apt install postgresql postgresql-contrib -y

psql --version

sudo systemctl enable PostgreSQL
sudo systemctl status PostgreSQL

psql -U postgres -h localhost -p 5432


after installation of postgres postgres user has been created
sudo passwd postgres
creation of database for sonar:
sudo su - postgres
psql
create user sonar
alter user sonar with encrypted password 'sonar';
create database sonarqube owner sonar;
grant all privileges on database sonarqube to sonar;

database for sonar has been created

sudo apt install openjdk-17-jdk
update-alternatives --config java

Linux kernel tunning:
sudo vi /etc/security/limits.conf
sonarqube - nofile 65536
sonarqube - nproc 4096

for memory:
sudo vi /etc/sysctl.conf
vm.max_map_count=262144


sonar installation:
wget https://binaries.sonarsource.com/Distribution/sonarqube/sonarqube-26.3.0.120487.zip

sudo apt install unzip -y
unzip sonarqube-26.3.0.120487.zip

sonar user creation
sudo groupadd sonar
sudo useradd -d /home/ubuntu/sonar -g sonar -s /bin/bash sonar
chown -R sonar:sonar SonarQube

/home/ubuntu/sonarqube/conf
vi sonar.properties
cat sonar.properties 
sonar.jdbc.username=sonar
sonar.jdbc.password=sonar
sonar.jdbc.url=jdbc:postgresql://private_ip:5432/sonarqube
sonar.web.port=9000
ubuntu@sonar:~/sonarqube/conf$ 


sudo vi /etc/systemd/system/sonar.service


[Unit]
Description=SonarQube service
After=syslog.target network.target

[Service]
Type=forking

ExecStart=/home/ubuntu/sonarqube/bin/linux-x86-64/sonar.sh start
ExecStop=/home/ubuntu/sonarqube/bin/linux-x86-64/sonar.sh stop

User=sonar
Group=sonar
Restart=always
LimitNOFILE=65536
LimitNPROC=4096

[Install]
WantedBy=multi-user.target



sudo vi /etc/postgresql/16/main/postgresql.conf

listen_addresses = '*'


sudo vi /etc/postgresql/16/main/pg_hba.conf

host    sonarqube    sonar    private_ip/32    md5
private ip


http://65.1.135.202:9000
admin/Rutvikgalale@96k

integrate SonarQube with Jenkins:
manage Jenkins -> sonar gates, SonarQube scanner, sonar quality gates install
manage Jenkins -> tools -> name: sonar -> install automatically

http://65.1.135.202:9000
icon -> security -> token-name: sonar-token -> global analysis token -> generate -> sqa_cf1a84dba5cf81af6262ca1640263be46da9f34f

manage Jenkins -> global -> add credentials -> secret text -> secret -> sqa_cf1a84dba5cf81af6262ca1640263be46da9f34f -> ID -> ID: sonar-token
manage Jenkins -> system -> SonarQube installations -> name: sonar -> URL: http://172.31.47.102:9000 -> Server authentication token -> sonar-token -> jenkins-sonar-token



http://172.31.47.102	:9000
projects -> Create a local project -> Project display name: Register-app -> Project key: Register-app -> branch: main
 
need installation of sonarscanner on Jenkins agent as well its manual way
other way:
Manage Jenkins → Global Tool Configuration -> SonarQube scanner -> install automatically -> name: sonar 
in jenkinsfile
script{
                def scannerHome = tool 'sonar' //// 'sonar' must match the name in  manage jenkins -> tool configuraion
                sh "${scannerHome}/bin/sonar-scanner \



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
    }
}

we can use another approach:
stage("code quality analysis") {
  steps {
    withSonarQubeEnv('sonar') {
      withCredentials([string(credentialsId: 'sonar-token', variable: 'SONAR_TOKEN')]) {
        sh "mvn sonar:sonar -Dsonar.projectKey=Register-app -Dsonar.host.url=$SONAR_HOST_URL -Dsonar.token=$SONAR_TOKEN"
      }
    }
  }
}



configuration of SonarQube webhook:

The webhook is the “bridge” between SonarQube and Jenkins. It ensures Jenkins knows when the analysis is done.
http://65.1.135.202:9000
Administration -> configuration -> webhooks -> create -> name: sonar-jenkins-webhook -> URL: http://172.31.46.118:8080/sonarqube-webhook -> create

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
    }
}

If the Quality Gate fails (status = ERROR), Jenkins aborts the pipeline immediately and marks the build as FAILED.
If the Quality Gate passes, Jenkins continues normally.
Jenkins will not fail pipeline in case of false


build and push docker images using pipeline:

need to install plugins related docker:
manage Jenkins -> plugins -> docker, docker commons, docker pipeline, docker api, docker build step, cloudbees -> install
need to configure dockerhub credentials:
dockerhub -> icon -> account settings -> PAT -> generate new token -> description: Register-app -> permission: Read, write, delete -> generate -> dckr_pat_lD6tF-apXDE20xZT7rJC-7hyOB8
manage Jenkins -> credentials -> global -> add credentials -> username with password: Username: rutvikg , password: dckr_pat_lD6tF-apXDE20xZT7rJC-7hyOB8, ID: docker, 

cat Dockerfile
FROM tomcat:latest
RUN cp -R  /usr/local/tomcat/webapps.dist/*  /usr/local/tomcat/webapps
COPY /webapp/target/*.war /usr/local/tomcat/webapps



cat Jenkinsfile
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
      image_tag = "${image_name}:${BUILD_NUMBER}"
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
            withCredentials([usernamePassword(credentialsId: "docker", usernameVariable: "docker_user", passwordVariable: "docker_pass")]){
              sh """
              echo $docker_pass | docker login -u $docker_user --password-stdin
              docker build -t "${docker_user}/${app_name}:${BUILD_NUMBER}" .
              docker push "${docker_user}/${app_name}:${BUILD_NUMBER}"
              """
            }
          }
        }
      }
    }
}




now will integrate Trivy to scan image:

 stage("trivy scanning"){
        steps{
          script{
            sh ('docker run -v /var/run/docker.sock:/var/run/docker.sock aquasec/trivy image rutvikg/register-app:${BUILD_NUMBER} --no-progress --scanners vuln  --exit-code 0 --severity HIGH,CRITICAL --format table')
          }
        }
      }



for cleaning artifacts:

stage("cleaning artifacts"){
        steps{
          script{
            sh "docker rmi ${image_name}:${BUILD_NUMBER}"
          }
        }
      }




setup of bootstrap server for eksctl & setup of kubernetes using eksctl:
Bootstrap server : 
Ensures every compute resource (node or pod) knows how to connect to the control plane securely and consistently.


new VM:
EKS-bootstrap-server
sudo apt update -y
sudo apt upgrade -y
sudo hostnamectl set-hostname EKS-bootstrap-server

installation of awscli
installation of kubectl
installation of eksctl

IAM -> roles -> create role -> aws service -> ec2 -> adminaccess -> eksctl_role -> create
EKS-bootstrap-server -> actions -> security -> modify IAM role -> eksctl_role

bootstrap server :
creation of cluster:

eksctl create cluster --name mycluster --region ap-south-1 --node-type t3.small

kubectl get nodes
NAME                                            STATUS   ROLES    AGE     VERSION
ip-192-168-21-248.ap-south-1.compute.internal   Ready    <none>   3m15s   v1.34.4-eks-f69f56f
ip-192-168-33-152.ap-south-1.compute.internal   Ready    <none>   3m16s   v1.34.4-eks-f69f56f
ubuntu@EKS-bootstrap-server:~$ 

 if you don’t specify --nodes, eksctl will by default create 2 worker nodes in the nodegroup.


argocd installation on eks cluster and add EKS cluster to argocd:
Argo CD is a GitOps continuous delivery tool 
Installing Argo CD on your EKS cluster means you deploy Argo CD itself as a set of pods inside the cluster.
Once installed, Argo CD manages application deployments by syncing Kubernetes manifests from Git repositories into your cluster.
Argo CD can manage multiple Kubernetes clusters.
By default, it manages the cluster where it’s installed. But you can “add” other clusters so Argo CD can deploy workloads to them too.
Adding an EKS cluster means you register its kubeconfig/credentials with Argo CD, so Argo CD knows how to talk to that cluster’s API server.



cd /home/ubuntu/argocd
vi namespace.yaml
apiVersion: v1
kind: Naemspace
metadata:
  name: argocd

kubectl create -f namespace.yaml

lets create and apply manifests files to argocd
kubectl apply -n argocd -f https://raw.githubusercontent.com/argoproj/argo-cd/stable/manifests/install.yaml

kubectl get pods -n argocd
NAME                                               READY   STATUS    RESTARTS   AGE
argocd-application-controller-0                    1/1     Running   0          88s
argocd-applicationset-controller-974d64569-v2nv2   1/1     Running   0          89s
argocd-dex-server-66fc67645-cvtkv                  1/1     Running   0          89s
argocd-notifications-controller-5474d4cbb6-r4dt9   1/1     Running   0          89s
argocd-redis-6888c8c66f-jd8b5                      1/1     Running   0          88s
argocd-repo-server-6c4975f4ff-l9tvn                1/1     Running   0          88s
argocd-server-5f7ff864d5-gppx8                     1/1     Running   0          88s
ubuntu@EKS-bootstrap-server:~/argocd$ 


to interact with apiserver we need to deploy argocd cli

wget https://github.com/argoproj/argo-cd/releases/latest/download/argocd-linux-amd64 -O argocd
chmod +x argocd
sudo mv argocd /usr/local/bin/
argocd version


now we need to expose argocd server to outside:
we have 2 options:
1. using nodeport
2. using loadbalancer

1. using node port:

ubuntu@EKS-bootstrap-server:~/argocd$ kubectl get svc -n argocd
NAME                                      TYPE        CLUSTER-IP       EXTERNAL-IP   PORT(S)                      AGE
argocd-applicationset-controller          ClusterIP   10.100.222.184   <none>        7000/TCP,8080/TCP            7m56s
argocd-dex-server                         ClusterIP   10.100.93.76     <none>        5556/TCP,5557/TCP,5558/TCP   7m56s
argocd-metrics                            ClusterIP   10.100.176.5     <none>        8082/TCP                     7m56s
argocd-notifications-controller-metrics   ClusterIP   10.100.193.64    <none>        9001/TCP                     7m56s
argocd-redis                              ClusterIP   10.100.34.17     <none>        6379/TCP                     7m56s
argocd-repo-server                        ClusterIP   10.100.164.164   <none>        8081/TCP,8084/TCP            7m56s
argocd-server                             ClusterIP   10.100.149.38    <none>        80/TCP,443/TCP               7m56s
argocd-server-metrics                     ClusterIP   10.100.189.186   <none>        8083/TCP                     7m56s
ubuntu@EKS-bootstrap-server:~/argocd$ 

we need to change svc from clusterIP to nodeport for argocd server:

kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "NodePort"}}'

ubuntu@EKS-bootstrap-server:~/argocd$ kubectl get svc argocd-server -n argocd
NAME            TYPE       CLUSTER-IP      EXTERNAL-IP   PORT(S)                      AGE
argocd-server   NodePort   10.100.149.38   <none>        80:31755/TCP,443:31186/TCP   10m
ubuntu@EKS-bootstrap-server:~/argocd$ 


Argo CD only serves the UI over HTTPS, not plain HTTP.

kubectl port-forward svc/argocd-server -n argocd 8086:443 --address=0.0.0.0 &

https://13.232.53.141:8086
using nodeport we can directly access like this 


this using nodeport/clusterIP & port-forwarding
kubectl port-forward svc/argocd-server -n argocd 8086:443 --address=0.0.0.0 &

https://13.232.53.141:8086


now will use Loadbalancer:

kubectl patch svc argocd-server -n argocd -p '{"spec": {"type": "LoadBalancer"}}'

ubuntu@EKS-bootstrap-server:~/argocd$ kubectl get svc argocd-server -n argocd
NAME            TYPE           CLUSTER-IP      EXTERNAL-IP                                                              PORT(S)                      AGE
argocd-server   LoadBalancer   10.100.118.44   af9537f86aa0340d1999a93fdf0fad62-86758261.ap-south-1.elb.amazonaws.com   80:30135/TCP,443:32413/TCP   29m
ubuntu@EKS-bootstrap-server:~/argocd$ 

af9537f86aa0340d1999a93fdf0fad62-86758261.ap-south-1.elb.amazonaws.com: aws load balancer endpoint

able to access application on af9537f86aa0340d1999a93fdf0fad62-86758261.ap-south-1.elb.amazonaws.com

kubectl get secret -n argocd argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d && echo

admin/h-kfyxyq9FGZS67k

user-info:
update password: Rutvik@96k

we have added argocd / installed / deployed on EKS cluster
now need to add EKS cluster manifests to argocd

from cli we need to login to argocd:

argocd login af9537f86aa0340d1999a93fdf0fad62-86758261.ap-south-1.elb.amazonaws.com
username: admin
password: Rutvik@96k

dashboard : settings -> clusters -> default cluster(we have only 1 cluster)
ubuntu@EKS-bootstrap-server:~/argocd$ argocd cluster list
SERVER                          NAME        VERSION  STATUS   MESSAGE                                                  PROJECT
https://kubernetes.default.svc  in-cluster           Unknown  Cluster has no applications and is not being monitored.  
ubuntu@EKS-bootstrap-server:~/argocd$ 


now we need to extract name of eks cluster to give command to add eks cluster  to argocd

ubuntu@EKS-bootstrap-server:~/argocd$ kubectl config get-contexts
CURRENT   NAME                                              CLUSTER                          AUTHINFO                                          NAMESPACE
*         iam-root-account@mycluster.ap-south-1.eksctl.io   mycluster.ap-south-1.eksctl.io   iam-root-account@mycluster.ap-south-1.eksctl.io   
ubuntu@EKS-bootstrap-server:~/argocd$ 

argocd cluster add iam-root-account@mycluster.ap-south-1.eksctl.io --name mycluster

added EKS cluster mycluster to argocd

now on dashboard inside settings able to see 2 cluster 1. default one & 2. mycluster

ubuntu@EKS-bootstrap-server:~/argocd$ argocd cluster list
SERVER                                                                     NAME        VERSION  STATUS   MESSAGE                                                  PROJECT
https://684ED1058191BC4AF4E7080A32A81AA7.yl4.ap-south-1.eks.amazonaws.com  mycluster            Unknown  Cluster has no applications and is not being monitored.  
https://kubernetes.default.svc                                             in-cluster           Unknown  Cluster has no applications and is not being monitored.  
ubuntu@EKS-bootstrap-server:~/argocd$ 


configure argocd to deploy pods on EKS & automate argocd deployment job using GitOps GitHub repo:

dashboard : settings -> connect repo -> via http/https -> project: default -> URL: https://github.com/Rutvikgalale/Register-app.git
-> connect

on main machine i.e Rutvik
cd /home/ubuntu/Rutvik/Register-app
cd manifests/
 cat deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: Register-app-deployment
  labels:
    app: Register-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: Register-app
  template:
    metadata:
      labels:
        app: Register-app
    spec:
      containers:
        - name: Register-app
          image: rutvikg/
          imagePullPolicy: Always


 cat service.yaml
apiVersion: v1
kind: Service
metadata:
  name: Register-app-Service
  labels:
    app: Register-app
spec:
  type: LoadBalancer
  selector:
    app: Register-app
  ports:
    - Port: 8080
      targetPort: 8080


on dashboard:
applications: new app -> name: register-app -> project-name: default -> policy: automatic -> prune & self heal -> repo: select -> branch: main -> 
path: manifests 

destination: eks cluster select -> namespace: default -> create

application has been created on cluster

 kubectl get pods
NAME                                       READY   STATUS    RESTARTS   AGE
register-app-deployment-5d5bfd7f49-54sc4   1/1     Running   0          143m
register-app-deployment-5d5bfd7f49-6zfwf   1/1     Running   0          143m
register-app-deployment-5d5bfd7f49-dgc5d   1/1     Running   0          143m
ubuntu@EKS-bootstrap-server:~$


kubectl get svc
NAME                   TYPE           CLUSTER-IP      EXTERNAL-IP                                                                PORT(S)          AGE
kubernetes             ClusterIP      10.100.0.1      <none>                                                                     443/TCP          5h25m
register-app-service   LoadBalancer   10.100.15.245   a6d7f5e0552524b5b9c31ae9494cda5d-1146591922.ap-south-1.elb.amazonaws.com   8080:32628/TCP   139m
ubuntu@EKS-bootstrap-server:~$


this is deployed application on new cluster and created those things on new cluster
http://a6d7f5e0552524b5b9c31ae9494cda5d-1146591922.ap-south-1.elb.amazonaws.com:8080/webapp/
application is accessible



now we need automate this deployment process:
will configure cd pipeline in Jenkins:

http://13.201.62.90:8080/
register-app -> configure -> this project is parameterized -> string -> name: image_tag -> trigger build remotely -> Authentication Token: cd-token
-> pipeline script from SCM -> git -> repo -> crede -> GitHub -> main(branch) -> save


manage Jenkins -> admin -> security -> add new token -> jenkins_api_token -> 113dd4b0638c6c002db57566174509ef61
manage Jenkins -> credentials -> add credentials -> secret text -> id: jenkins_api_token




apiVersion: apps/v1
kind: Deployment
metadata:
  name: register-app
  labels:
    app: register-app
spec:
  replicas: 3
  selector:
    matchLabels:
      app: register-app
  template:
    metadata:
      labels:
        app: register-app
    spec:
      containers:
        - name: register-app
          image: rutvikg/register-app:59
          imagePullPolicy: Always
          ports:
            - containerPort: 8080



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





