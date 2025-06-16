Project Title: End-to-End CI/CD Automation Using Terraform, Jenkins, Docker, and Kubernetes

Overview

This project demonstrates complete automation of infrastructure provisioning and CI/CD deployment pipeline using Terraform, Jenkins, Docker, and Kubernetes on AWS. It provisions three EC2 Ubuntu servers (1 master, 2 slaves), installs Kubernetes, deploys a sample web app from GitHub, builds a Docker image, pushes it to Docker Hub, and deploys it to Kubernetes.

Tools Used

Terraform

Jenkins

Docker

Kubernetes

AWS EC2 (Ubuntu 22.04)

GitHub

Docker Hub

Step-by-Step Implementation

1. Infrastructure Provisioning using Terraform

Installed Terraform on Ubuntu

wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform

Terraform Configuration (main.tf)
            Project Title: End-to-End CI/CD Automation Using Terraform, Jenkins, Docker, and Kubernetes

Overview

This project demonstrates complete automation of infrastructure provisioning and CI/CD deployment pipeline using Terraform, Jenkins, Docker, and Kubernetes on AWS. It provisions three EC2 Ubuntu servers (1 master, 2 slaves), installs Kubernetes, deploys a sample web app from GitHub, builds a Docker image, pushes it to Docker Hub, and deploys it to Kubernetes.

Tools Used

Terraform

Jenkins

Docker

Kubernetes

AWS EC2 (Ubuntu 22.04)

GitHub

Docker Hub

Step-by-Step Implementation

1. Infrastructure Provisioning using Terraform

Installed Terraform on Ubuntu

wget -O - https://apt.releases.hashicorp.com/gpg | sudo gpg --dearmor -o /usr/share/keyrings/hashicorp-archive-keyring.gpg
echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list
sudo apt update && sudo apt install terraform

Terraform Configuration (main.tf)

terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
    }
  }
}

provider "aws" {
  region     = "ap-south-1"
  access_key = "<Your Access Key>"
  secret_key = "<Your Secret Key>"
}

resource "aws_instance" "example" {
  ami           = "ami-0f5ee92e2d63afc18"
  count         = 2
  instance_type = "t2.medium"
  key_name      = "ubuntu"
  tags = {
    Name = "kub-slave"
  }
}

resource "aws_instance" "main" {
  ami           = "ami-0f5ee92e2d63afc18"
  count         = 1
  instance_type = "t2.medium"
  key_name      = "ubuntu"
  tags = {
    Name = "kub1-master"
  }
}

Terraform Commands

terraform init
terraform plan
terraform apply

2. Installing Docker & Kubernetes (install.sh on All Nodes)

sudo apt update -y
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker

sudo apt-get install -y apt-transport-https ca-certificates curl
directory=/etc/apt/keyrings
sudo mkdir -p $directory
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o $directory/kubernetes-archive-keyring.gpg
echo "deb [signed-by=$directory/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update && sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

3. Kubernetes Cluster Initialization

On Master Node

sudo kubeadm init

Copy the join token generated and run on both slave nodes.

Post Init on Master

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

Apply CNI (Weave)

kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
kubectl get nodes

4. Install Jenkins on Master Node (jenkins.sh)

sudo apt install openjdk-17-jdk -y
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian/jenkins.io-2023.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins -y

Start Jenkins, unlock and install suggested plugins.

5. Jenkins Node Configuration

Added slave node in Jenkins named kub-master

SSH credentials: used Ubuntu key (.pem)

Remote directory: /home/ubuntu/jenkins

Host Key Verification Strategy: Non verifying

6. Jenkins Pipeline Setup

Global Credentials

DockerHub credentials added with ID used in environment.

Jenkinsfile

pipeline {
    agent none
    environment {
        DOCKERHUB_CREDENTIALS = credentials('e814f99d-8cc0-425d-840e-0c10c489f570')
    }
    stages {
        stage('Hello') {
            agent {
                label 'Kub-master'
            }
            steps {
                echo 'Hello World'
            }
        }
        stage('Git Checkout') {
            agent {
                label 'Kub-master'
            }
            steps {
                git 'https://github.com/intellipaat2/website.git'
            }
        }
        stage('Docker Build & Push') {
            agent {
                label 'Kub-master'
            }
            steps {
                sh 'sudo docker build /home/ubuntu/jenkins/workspace/pipeline -t ajithpethanan/demo1'
                sh 'sudo echo $DOCKERHUB_CREDENTIALS_PSW | sudo docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
                sh 'sudo docker push ajithpethanan/demo1'
            }
        }
        stage('Kubernetes Deployment') {
            agent {
                label 'Kub-master'
            }
            steps {
                sh 'sudo kubectl create -f deploy.yml'
                sh 'sudo kubectl create -f svc.yml'
            }
        }
    }
}

7. Kubernetes YAML Files

deploy.yml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: ajithpethanan/demo1
        ports:
        - containerPort: 80

svc.yml

apiVersion: v1
kind: Service
metadata:
  name: my-nginx
spec:
  type: NodePort
  ports:
    - targetPort: 80
      port: 80
      nodePort: 30008
  selector:
    app: nginx

8. Web Application

index.html

<html>
<head>
<title> Intellipaat </title>
</head>
<body style="background-image:url('images/github3.jpg'); background-size: 100%">
<h2 ALIGN=CENTER>Hello world!!!!!!!!!!!!!</h2>
</body>
</html>

Dockerfile
 

Terraform Commands

             terraform init
             terraform plan
             terraform apply

2. Installing Docker & Kubernetes (install.sh on All Nodes)

sudo apt update -y
sudo apt install docker.io -y
sudo systemctl start docker
sudo systemctl enable docker

sudo apt-get install -y apt-transport-https ca-certificates curl
directory=/etc/apt/keyrings
sudo mkdir -p $directory
curl -fsSL https://packages.cloud.google.com/apt/doc/apt-key.gpg | sudo gpg --dearmor -o $directory/kubernetes-archive-keyring.gpg
echo "deb [signed-by=$directory/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt update && sudo apt install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl

3. Kubernetes Cluster Initialization

On Master Node

sudo kubeadm init

Copy the join token generated and run on both slave nodes.

Post Init on Master

mkdir -p $HOME/.kube
sudo cp -i /etc/kubernetes/admin.conf $HOME/.kube/config
sudo chown $(id -u):$(id -g) $HOME/.kube/config

Apply CNI (Weave)

kubectl apply -f https://github.com/weaveworks/weave/releases/download/v2.8.1/weave-daemonset-k8s.yaml
kubectl get nodes

4. Install Jenkins on Master Node (jenkins.sh)

sudo apt install openjdk-17-jdk -y
sudo wget -O /etc/apt/keyrings/jenkins-keyring.asc https://pkg.jenkins.io/debian/jenkins.io-2023.key
echo "deb [signed-by=/etc/apt/keyrings/jenkins-keyring.asc] https://pkg.jenkins.io/debian binary/" | sudo tee /etc/apt/sources.list.d/jenkins.list > /dev/null
sudo apt-get update
sudo apt-get install jenkins -y

Start Jenkins, unlock and install suggested plugins.

5. Jenkins Node Configuration

Added slave node in Jenkins named kub-master

SSH credentials: used Ubuntu key (.pem)

Remote directory: /home/ubuntu/jenkins

Host Key Verification Strategy: Non verifying

6. Jenkins Pipeline Setup

Global Credentials

DockerHub credentials added with ID used in environment.

Jenkinsfile

pipeline {
    agent none
    environment {
        DOCKERHUB_CREDENTIALS = credentials('e814f99d-8cc0-425d-840e-0c10c489f570')
    }
    stages {
        stage('Hello') {
            agent {
                label 'Kub-master'
            }
            steps {
                echo 'Hello World'
            }
        }
        stage('Git Checkout') {
            agent {
                label 'Kub-master'
            }
            steps {
                git 'https://github.com/intellipaat2/website.git'
            }
        }
        stage('Docker Build & Push') {
            agent {
                label 'Kub-master'
            }
            steps {
                sh 'sudo docker build /home/ubuntu/jenkins/workspace/pipeline -t ajithpethanan/demo1'
                sh 'sudo echo $DOCKERHUB_CREDENTIALS_PSW | sudo docker login -u $DOCKERHUB_CREDENTIALS_USR --password-stdin'
                sh 'sudo docker push ajithpethanan/demo1'
            }
        }
        stage('Kubernetes Deployment') {
            agent {
                label 'Kub-master'
            }
            steps {
                sh 'sudo kubectl create -f deploy.yml'
                sh 'sudo kubectl create -f svc.yml'
            }
        }
    }
}

7. Kubernetes YAML Files

deploy.yml

apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
  labels:
    app: nginx
spec:
  replicas: 2
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
      - name: nginx
        image: ajithpethanan/demo1
        ports:
        - containerPort: 80

svc.yml

apiVersion: v1
kind: Service
metadata:
  name: my-nginx
spec:
  type: NodePort
  ports:
    - targetPort: 80
      port: 80
      nodePort: 30008
  selector:
    app: nginx

8. Web Application

index.html

<html>
<head>
<title> Intellipaat </title>
</head>
<body style="background-image:url('images/github3.jpg'); background-size: 100%">
<h2 ALIGN=CENTER>Hello world!!!!!!!!!!!!!</h2>
</body>
</html>

Dockerfile
