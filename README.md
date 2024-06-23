

# End-to-End Containerization and Deployment of Node.js Application

## Overview

This document outlines the steps taken to containerize a Node.js application and deploy it using Kubernetes. The process includes setting up a CI/CD pipeline with Jenkins, configuring AWS ECR for image storage, and deploying the application with Kubernetes StatefulSets, Services, and Ingress.

## 1. Cloning the Repository

The application is based on a Node.js project from the following GitHub repository:
[Node Passport Login](https://github.com/bradtraversy/node_passport_login).

## 2. Dockerizing the Application

### Dockerfile

A Dockerfile was created to containerize the Node.js application:

```Dockerfile
FROM node:alpine

WORKDIR /app

COPY package*.json ./

RUN npm install

COPY . .

EXPOSE 5000

CMD ["npm", "start"]
```

## 3. Setting Up Kubernetes Cluster

Two EC2 instances were provisioned for the Kubernetes cluster, with one serving as the master node and the other as the worker node. The following steps were used to initialize the cluster:

### On Master Node

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
sudo kubeadm init --pod-network-cidr=192.168.0.0/16
```

### On Worker Node

```bash
sudo apt-get update
sudo apt-get install -y apt-transport-https ca-certificates curl
sudo curl -fsSLo /usr/share/keyrings/kubernetes-archive-keyring.gpg https://packages.cloud.google.com/apt/doc/apt-key.gpg
echo "deb [signed-by=/usr/share/keyrings/kubernetes-archive-keyring.gpg] https://apt.kubernetes.io/ kubernetes-xenial main" | sudo tee /etc/apt/sources.list.d/kubernetes.list
sudo apt-get update
sudo apt-get install -y kubelet kubeadm kubectl
sudo apt-mark hold kubelet kubeadm kubectl
```

Joined the worker node to the master using the token generated from master node using the below command
## 4. Kubernetes Manifests
```
kubeadm token create --print-join-command
```
### MongoDB StatefulSet and Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: mongodb
  namespace: testing
spec:
  ports:
    - port: 27017
      targetPort: 27017
  clusterIP: None
  selector:
    app: mongodb
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: mongodb
  namespace: testing
spec:
  serviceName: "mongodb"
  replicas: 1
  selector:
    matchLabels:
      app: mongodb
  template:
    metadata:
      labels:
        app: mongodb
    spec:
      containers:
        - name: mongodb
          image: mongo:latest
          ports:
            - containerPort: 27017
          volumeMounts:
            - name: mongo-storage
              mountPath: /data/db
  volumeClaimTemplates:
    - metadata:
        name: mongo-storage
      spec:
        accessModes: ["ReadWriteOnce"]
        resources:
          requests:
            storage: 1Gi
        storageClassName: standard
```

### Node.js Application StatefulSet and Service

```yaml
apiVersion: v1
kind: Service
metadata:
  name: node-app
  namespace: testing
spec:
  type: NodePort
  ports:
    - port: 5000
      targetPort: 5000
  selector:
    app: node-app
---
apiVersion: apps/v1
kind: StatefulSet
metadata:
  name: node-app
  namespace: testing
spec:
  serviceName: "node-app"
  replicas: 3
  selector:
    matchLabels:
      app: node-app
  template:
    metadata:
      labels:
        app: node-app
    spec:
      containers:
        - name: node-app
          image: 107534347551.dkr.ecr.ap-south-1.amazonaws.com/node-app:v1.${BUILD_ID}
          ports:
            - containerPort: 5000
          env:
            - name: PORT
              value: "5000"
            - name: MONGODB_URI
              value: "mongodb://mongodb:27017/sample-db"
          imagePullSecrets:
            - name: ecr-secret
```

### Persistent Volume

```yaml
apiVersion: v1
kind: PersistentVolume
metadata:
  name: mongo-pv
spec:
  capacity:
    storage: 2Gi
  accessModes:
    - ReadWriteOnce
  persistentVolumeReclaimPolicy: Retain
  storageClassName: standard
  hostPath:
    path: /mnt/data
```

### Ingress

```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: node-app-ingress
  namespace: testing
  annotations:
    nginx.ingress.kubernetes.io/rewrite-target: /
spec:
  rules:
    - host: node-app.ashwaq.xyz
      http:
        paths:
          - path: /
            pathType: Prefix
            backend:
              service:
                name: node-app
                port:
                  number: 5000
```

## 5. Deploying NGINX Ingress Controller

```bash
kubectl apply -f https://raw.githubusercontent.com/kubernetes/ingress-nginx/main/deploy/static/provider/baremetal/deploy.yaml
```

## 6. Jenkins Pipeline for CI/CD

A Jenkins pipeline script was used to automate the build and deployment process:

```groovy
pipeline {
    agent any

    environment {
        AWS_REGION = 'ap-south-1'
        ECR_REPO_NAME = 'node-app'
        IMAGE_TAG = "v1.${BUILD_NUMBER}"
        ECR_URI = "107534347551.dkr.ecr.${AWS_REGION}.amazonaws.com/${ECR_REPO_NAME}:${IMAGE_TAG}"
        KUBE_NAMESPACE = 'testing'
        KUBE_CONFIG_CREDENTIALS = 'kubeconfig-creds'
    }

    stages {
        stage('Checkout') {
            steps {
                git 'https://github.com/ashwaq06/End-to-End-Containerization-and-Deployment'
            }
        }

        stage('Build Docker Image') {
            steps {
                script {
                    sh "docker build -t ${ECR_REPO_NAME}:${IMAGE_TAG} ."
                }
            }
        }

        stage('Login to AWS ECR') {
            steps {
                script {
                    sh "aws ecr get-login-password --region ${AWS_REGION} | docker login --username AWS --password-stdin ${ECR_URI}"
                }
            }
        }

        stage('Push Docker Image to ECR') {
            steps {
                script {
                    sh "docker tag ${ECR_REPO_NAME}:${IMAGE_TAG} ${ECR_URI}"
                    sh "docker push ${ECR_URI}"
                }
            }
        }

        stage('Deploy to Kubernetes') {
            steps {
                script {
                    withCredentials([file(credentialsId: "${KUBE_CONFIG_CREDENTIALS}",  variable: 'KUBECONFIG')]) {
                        sh 'envsubst < beast-k8s/statefulset.yaml | kubectl apply -n ${KUBE_NAMESPACE} -f -'
                        sh "kubectl apply -f beast-k8s/pv.yml -n ${KUBE_NAMESPACE}"
                    }
                }
            }
        }

        stage('Cleanup') {
            steps {
                script {
                    sh "docker rmi -f ${ECR_REPO_NAME}:${IMAGE_TAG}"
                }
            }
        }
    }

    post {
        always {
            deleteDir()
        }
    }
}
```

---
## Screenshots of the K8s cluster, Jenkins and DNS Records:
![Screenshot 2024-06-23 192335](https://github.com/ashwaq06/End-to-End-Containerization-and-Deployment/assets/80192952/4e83dc0d-58e7-4f89-a0bf-e73439215023)

![Screenshot 2024-06-23 194338](https://github.com/ashwaq06/End-to-End-Containerization-and-Deployment/assets/80192952/0ba1984a-fc95-4077-a592-131563731700)

![Screenshot 2024-06-23 201656](https://github.com/ashwaq06/End-to-End-Containerization-and-Deployment/assets/80192952/6e87dd40-5ea0-42b5-970c-4e8d8753df33)


## Troubleshooting

### Common Issues Encountered

1. **CrashLoopBackOff**: Pods going into `CrashLoopBackOff` due to various issues, such as missing environment variables or connectivity issues.
2. **DNS Resolution Errors**: Errors like `getaddrinfo EAI_AGAIN mongodb` indicating DNS resolution problems within the cluster.



---

This documentation provides a detailed walkthrough of setting up a CI/CD pipeline for a Node.js application using Jenkins, Docker, and Kubernetes. It includes steps for containerization, cluster setup, deployment manifests, and Jenkins pipeline configuration.

