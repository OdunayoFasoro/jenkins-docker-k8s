# jenkins-docker-k8s
Building a multi tier architecture by intergrating jenkims docker  and kubernetes 
#install and configure docker on Ubuntu

+sudo apt update
+sudo apt install docker.io -y
+sudo systemctl start docker


----


# run script as root user
sudo -i 
# jenkins-install.sh
curl -fsSL https://pkg.jenkins.io/debian-stable/jenkins.io.key | sudo tee \
  /usr/share/keyrings/jenkins-keyring.asc > /dev/null
echo deb [signed-by=/usr/share/keyrings/jenkins-keyring.asc] \
  https://pkg.jenkins.io/debian-stable binary/ | sudo tee \
  /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update -y
sudo apt-get install jenkins -y 
sudo systemctl enable jenkins 
sudo systemctl start jenkins 

---

Alternative Jenkins Installation 

# Jenkins Installation And Setup In AWS EC2 ubuntu Instance.
# Installation of Java
sudo apt update   # Update the repositories
sudo apt install openjdk-11-jdk
java -version
# Install Jenkins
sudo wget -O /etc/yum.repos.d/jenkins.repo \
    https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io.key
sudo dnf upgrade
# Add required dependencies for the jenkins package
sudo dnf install chkconfig java-devel
sudo dnf install jenkins
# Start Jenkins
sudo systemctl daemon-reload  # To Register the Jenkins service 
sudo systemctl start jenkins
systemctl status jenkins


---

usermod -aG docker jenkins
sudo systemctl restart docker.service
sudo echo "jenkins ALL=(ALL) NOPASSWD:ALL" | sudo tee /etc/sudoers.d/jenkins

sudo chown jenkins /var/run/docker.sock


2a) install AWSCLI using the apt package manager

sudo apt update -y
sudo apt-get install python3-pip -y
sudo pip3 install awscli --upgrade --user
sudo apt install awscli -y 
aws --version

or 2b) install AWSCLI using the script below
sudo apt update -y
sudo apt install unzip wget -y
sudo curl https://s3.amazonaws.com/aws-cli/awscli-bundle.zip -o awscli-bundle.zip
sudo apt install unzip python -y
sudo unzip awscli-bundle.zip
sudo ./awscli-bundle/install -i /usr/local/aws -b /usr/local/bin/aws

#install AWS CLI

sudo apt update -y
sudo apt-get install python3-pip -y
sudo pip3 install awscli --upgrade --user
sudo apt-get install awscli
aws --version


3) Install kops software on an ubuntu instance by running the commands below:
=============================================================================

sudo apt install wget -y
sudo wget https://github.com/kubernetes/kops/releases/download/v1.22.0/kops-linux-amd64
sudo chmod +x kops-linux-amd64
sudo mv kops-linux-amd64 /usr/local/bin/kops

4) Install kubectl kubernetes client if it is not already installed
==========================================================================
 sudo curl -LO https://storage.googleapis.com/kubernetes-release/release/$(curl -s https://storage.googleapis.com/kubernetes-release/release/stable.txt)/bin/linux/amd64/kubectl
 sudo chmod +x ./kubectl
 sudo mv ./kubectl /usr/local/bin/kubectl

5) Create an IAM role from AWS Console or CLI with the below Policies.
======================================================================
AmazonEC2FullAccess 
AmazonS3FullAccess
IAMFullAccess 
AmazonVPCFullAccess

Then Attach IAM role to ubuntu server from Console Select KOPS Server --> Actions --> Instance Settings --> Attach/Replace IAM Role --> Select the role which You Created. --> Save.

6) create an S3 bucket
======================

Execute the commands below in your KOPS control Server. use unique s3 bucket name. If you get bucket name exists error.
aws s3 mb s3://ayomide30bucket.local
aws s3 ls # to verify

6b) create an S3 bucket
========================
Expose environment variable:
# Add env variables in bashrc

vi .bashrc
# Give Unique Name And S3 Bucket which you created.
export NAME=ayomide30project.k8s.local
export KOPS_STATE_STORE=s3://ayomide30bucket.local

source .bashrc  


7) Create sshkeys before creating cluster
===========================================
    ssh-keygen

8) Create kubernetes cluster definitions on S3 bucket
======================================================

kops create cluster --zones us-east-1a --networking weave --master-size t2.medium --master-count 1 --node-size t2.medium --node-count=2 ${NAME}
# copy the sshkey into your cluster to be able to access your kubernetes node from the kops server
kops create secret --name ${NAME} sshpublickey admin -i ~/.ssh/id_rsa.pub


9) Initialise your kops kubernetes cluster by running the command below
=======================================================================
#validate cluster
kops update cluster ${NAME} --yes


10a) Validate your cluster(KOPS will take some time to create cluster ,Execute below command after 3 or 4 mins)

kops validate cluster

Suggestions:
validate cluster: kops validate cluster --wait 10m
list nodes: kubectl get nodes --show-labels
ssh to the master: ssh -i ~/.ssh/id_rsa ubuntu@api.class.k8s.local
the ubuntu user is specific to Ubuntu. If not using Ubuntu please use the appropriate user based on your OS.
read about installing addons at: https://kops.sigs.k8s.io/operations/addons.


10b - Export the kubeconfig file to manage your kubernetes cluster from a remote server. For this demo, Our remote server shall be our kops server
 kops export kubecfg $NAME --admin


11a) To list nodes and pod to ensure that you can make calls to the kubernetes apiSAerver and run workloads
  kubectl get nodes 


11b) Alternative you can ssh into your kubernetes master server using the command below and manage your cluster from the master

ssh -i ~/.ssh/id_rsa ubuntu@ipAddress
ssh -i ~/.ssh/id_rsa ubuntu@18.222.139.125
ssh -i ~/.ssh/id_rsa ubuntu@172.20.58.124
11b. Alternative, Enable PasswordAuthentication in the master server and assign passwd
sudo sed -i "/^[^#]*PasswordAuthentication[[:space:]]no/c\PasswordAuthentication yes" /etc/ssh/sshd_config
sudo service sshd restart
sudo passwd ubuntu
11c) To list nodes
  kubectl get nodes 

12) To Delete Cluster
=====================

kops delete cluster --name=${NAME} --state=${KOPS_STATE_STORE} --yes

====================================================================================================

13 # IF you want to SSH to Kubernetes Master or Nodes Created by KOPS. You can SSH From KOPS_Server

sh -i ~/.ssh/id_rsa ubuntu@ipAddress ssh -i ~/.ssh/id_rsa ubuntu@3.90.203.23

``

Error: Validation failed: unexpected error during validation: error listing nodes: Unauthorized
===============================================================================================
#Run this code

kops create secret --name ${NAME} sshpublickey admin -i ~/.ssh/id_rsa.pub
kops update cluster ${NAME} --yes --admin



Pipeline Script
===============

node {

  def mavenHome = tool name: 'maven3.9.0'
  
  stage('SCM Clone') {
    git credentialsId: 'GitHubCredentials', url: 'https://github.com/Isaacwade/spring-boot-docker.git'
  }

  stage('MavenBuild') {
    sh "${mavenHome}/bin/mvn clean package"
  }

  stage('Quality Report') {
    //sh "${mavenHome}/bin/mvn sonar:sonar"
  }

  stage('NexusUpload') {
    //sh "${mavenHome}/bin/mvn deploy"
  }

  stage('BuildDockerImage') {
    sh "docker build -t isaacoluwade/spring-boot-mongo:v2 ."
  }

  stage('DockerPush') {

    withCredentials([string(credentialsId: 'DockerHubCredentials', variable: 'DockerHubCredentials')]) {
        sh "docker login -u isaacoluwade -p ${DockerHubCredentials}"
}    


    sh "docker push isaacoluwade/spring-boot-mongo:v2"
  }
  
  stage('RemoveDockerImages'){

    sh 'docker rmi $(docker images -q)'
  }
 
  stage('deployToKubenetes'){

     sh "kubectl apply -f springapp.yml "
  }
}
