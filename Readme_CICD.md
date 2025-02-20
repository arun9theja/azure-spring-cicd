### **📌 CI/CD Pipeline with Azure DevOps, Azure Container Registry (ACR), and Azure Kubernetes Service (AKS)**  

This repository contains an end-to-end **CI/CD pipeline** using **Azure DevOps** to build, push, and deploy a **Spring Boot** application in a **Kubernetes cluster (AKS)**.  

---

## **📖 Table of Contents**
- [📌 Overview](#-overview)  
- [🛠️ Prerequisites](#️-prerequisites)  
- [🔧 CI Pipeline (Build & Push Docker Image)](#-ci-pipeline-build--push-docker-image)  
- [🚀 CD Pipeline (Deploy to AKS)](#-cd-pipeline-deploy-to-aks)  
- [📂 Kubernetes Deployment Files](#-kubernetes-deployment-files)  
- [✅ How to Run the Pipeline](#-how-to-run-the-pipeline)  
- [📝 Additional Notes](#-additional-notes)  

---

## **📌 Overview**  

This pipeline is designed to:  
1. **CI Pipeline (Continuous Integration)**:  
   - Compile and package the **Spring Boot** app using **Maven**  
   - Run **SonarQube** for static code analysis  
   - Build a **Docker image**  
   - Push the image to **Azure Container Registry (ACR)**  

2. **CD Pipeline (Continuous Deployment)**:  
   - Pull the latest Docker image from ACR  
   - Deploy it to **Azure Kubernetes Service (AKS)**  
   - Update the running application with the new image  

---

## **🛠️ Prerequisites**  

Ensure you have the following set up before running the pipeline:  

✅ **Azure DevOps Project** with CI/CD pipelines  
✅ **Azure Container Registry (ACR)**  
✅ **Azure Kubernetes Service (AKS) cluster**  
✅ **SonarQube instance** for static code analysis  
✅ **Service Connections in Azure DevOps**  
   - **Azure Service Connection** to manage AKS  
   - **Docker Service Connection** to authenticate with ACR  
   - **Kubernetes Service Connection** for deployment  

---

## **🔧 CI Pipeline (Build & Push Docker Image)**  

### **Pipeline YAML - `azure-pipelines-ci.yaml`**
```yaml
trigger:
- develop

resources:
- repo: self

variables:
  dockerRegistryServiceConnection: 'e192d461-843a-4bbb-8c24-b09a40ab3678'
  imageRepository: 'devops'
  containerRegistry: 'azurespringci.azurecr.io'
  dockerfilePath: '$(Build.SourcesDirectory)/Dockerfile'
  tag: '$(Build.BuildId)'
  SONAR_HOST_URL: 'http://40.76.249.199:9000'
  ACR_NAME: 'azurespringci.azurecr.io'
  BUILD_VERSION: '$(Build.BuildId)'
  SONAR_TOKEN: 'sqa_2ac25b2741c599d954b65c73e457d40dc1584755'
  vmImageName: 'ubuntu-latest'

steps:
  - checkout: self

  - task: JavaToolInstaller@0
    inputs:
      versionSpec: '21'
      jdkSourceOption: PreInstalled
      jdkArchitectureOption: x64

  - script: |
      sudo apt-get update
      sudo apt-get install -y unzip
      curl -o sonar-scanner-cli.zip https://binaries.sonarsource.com/Distribution/sonar-scanner-cli/sonar-scanner-cli-4.6.2.2472-linux.zip
      unzip sonar-scanner-cli.zip
      sudo mv sonar-scanner-4.6.2.2472-linux /opt/sonar-scanner
      export PATH=$PATH:/opt/sonar-scanner/bin
    displayName: 'Install SonarQube Scanner'

  - script: |
      mvn clean package -DskipTests
      export PROJECT_KEY=$(mvn help:evaluate -Dexpression=project.artifactId -q -DforceStdout)
      echo "sonar.projectKey=$PROJECT_KEY" > sonar-project.properties
      echo "sonar.sources=src" >> sonar-project.properties
      echo "sonar.host.url=$(SONAR_HOST_URL)" >> sonar-project.properties
      echo "sonar.login=$(SONAR_TOKEN)" >> sonar-project.properties
      echo "sonar.java.binaries=./target/classes" >> sonar-project.properties
    displayName: 'Generate SonarQube Properties File'

  - script: |
      /opt/sonar-scanner/bin/sonar-scanner -Dproject.settings=sonar-project.properties
    displayName: 'Run SonarQube Scan'

  - task: Docker@2
    displayName: 'Login to ACR'
    inputs:
      command: login
      containerRegistry: $(dockerRegistryServiceConnection)

  - task: Docker@2
    displayName: 'Build & Push Docker Image'
    inputs:
      command: buildAndPush
      repository: $(imageRepository)
      dockerfile: $(dockerfilePath)
      containerRegistry: $(dockerRegistryServiceConnection)
      tags: |
        $(tag)
```

---

## **🚀 CD Pipeline (Deploy to AKS)**  

### **Pipeline YAML - `azure-pipelines-cd.yaml`**
```yaml
trigger:
  - develop

resources:
  - repo: self

variables:
  ACR_NAME: 'azurespringci.azurecr.io'
  IMAGE_NAME: 'devops'
  KUBE_NAMESPACE: 'default'
  AKS_RESOURCE_GROUP: 'myResourceGroup'
  AKS_CLUSTER_NAME: 'myAKSCluster'
  IMAGE_TAG: '$(Build.BuildId)'

stages:
  - stage: Deploy
    jobs:
      - job: DeployToAKS
        pool:
          vmImage: 'ubuntu-latest'

        steps:
          - checkout: self

          - task: AzureCLI@2
            displayName: 'Login to Azure'
            inputs:
              azureSubscription: 'AzureServiceConnection'
              scriptType: 'bash'
              scriptLocation: 'inlineScript'
              inlineScript: |
                az aks get-credentials --resource-group $(AKS_RESOURCE_GROUP) --name $(AKS_CLUSTER_NAME)
                kubectl config view

          - task: Kubernetes@1
            displayName: 'Deploy to AKS'
            inputs:
              connectionType: 'Kubernetes Service Connection'
              kubernetesServiceConnection: 'AKS-ServiceConnection'
              namespace: $(KUBE_NAMESPACE)
              command: 'apply'
              useConfigurationFile: true
              configuration: 'k8s/deployment.yaml'

          - script: |
              kubectl set image deployment/devops-deployment devops-container=$(ACR_NAME)/$(IMAGE_NAME):$(IMAGE_TAG) -n $(KUBE_NAMESPACE)
              kubectl rollout status deployment/devops-deployment -n $(KUBE_NAMESPACE)
            displayName: 'Update Deployment with New Image'
```

---

## **📂 Kubernetes Deployment Files**  

### **Deployment YAML - `k8s/deployment.yaml`**
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: devops-deployment
  namespace: default
spec:
  replicas: 2
  selector:
    matchLabels:
      app: devops
  template:
    metadata:
      labels:
        app: devops
    spec:
      containers:
        - name: devops-container
          image: azurespringci.azurecr.io/devops:latest
          ports:
            - containerPort: 8080
---
apiVersion: v1
kind: Service
metadata:
  name: devops-service
  namespace: default
spec:
  selector:
    app: devops
  ports:
    - protocol: TCP
      port: 80
      targetPort: 8080
  type: LoadBalancer
```

---

## **✅ How to Run the Pipeline**  
1. **Commit the YAML files to your repository**  
2. **Set up service connections in Azure DevOps**  
3. **Run the CI/CD pipelines from Azure DevOps**  
4. **Application will be deployed to AKS** 🎉  

---
