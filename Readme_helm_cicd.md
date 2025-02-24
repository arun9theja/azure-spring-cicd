# **CI/CD Pipeline for AKS Deployment using Helm & Azure DevOps** 🚀  

This repository contains an **Azure DevOps pipeline** that automates:  
✅ **Building & Pushing Docker Image** to **Azure Container Registry (ACR)**  
✅ **Deploying the Application to AKS** using **Helm**  
✅ **Rolling Back in Case of Deployment Failure**  

---

## **🔹 Prerequisites**  

Before running the pipeline, ensure you have the following:  
1️⃣ **Azure Kubernetes Service (AKS)** cluster set up.  
2️⃣ **Azure Container Registry (ACR)** for storing Docker images.  
3️⃣ **Helm installed** in your local environment for debugging (optional).  

---

## **🔹 Creating Required Azure DevOps Service Connections**  

### **1️⃣ Create Service Connection for ACR**
1. Go to **Azure DevOps** → **Project Settings** → **Service connections**  
2. Click **New service connection** → Select **Docker Registry**  
3. Choose **Azure Container Registry** and select your **ACR**  
4. Provide a name, e.g., `azurespringci`, and save it.  

### **2️⃣ Create Service Connection for AKS**
1. Go to **Azure DevOps** → **Project Settings** → **Service connections**  
2. Click **New service connection** → Select **Kubernetes**  
3. Choose **Azure Subscription** and select your **AKS cluster**  
4. Provide a name, e.g., `myspringboot`, and save it.  

---

## **🔹 Azure DevOps Pipeline Setup**  

### **1️⃣ Configure Azure Pipeline**  
1. Navigate to **Azure DevOps** → **Pipelines** → **New Pipeline**  
2. Select **GitHub (YAML)** and choose this repository  
3. Select **Existing YAML File** and choose `azure-pipelines.yaml`  
4. Save and **Run the pipeline** 🎯  

---

## **🔹 Pipeline Overview**  

### **📌 Build Stage**
✅ **Build & Push Docker Image** to **Azure Container Registry (ACR)**  
✅ Uses the `Docker@2` task  

### **📌 Deploy Stage**
✅ **Creates `imagePullSecret`** for AKS  
✅ **Deploys App to AKS** using Helm  
✅ **Rollback on Failure** to the last stable release  

---

## **🔹 `azure-pipelines.yaml`**  

```yaml
trigger:
  - develop

resources:
  - repo: self

variables:
  ACR_NAME: 'azurespringci.azurecr.io'
  ACR_CONNECTION: 'azurespringci'
  IMAGE_NAME: 'devops'
  IMAGE_TAG: '$(Build.BuildId)'
  KUBE_NAMESPACE: 'default'
  AKS_RESOURCE_GROUP: 'myResourceGroup'
  AKS_CLUSTER_NAME: 'argo-aks'
  HELM_RELEASE_NAME: 'devops-app'
  DOCKERFILE_PATH: '**/Dockerfile'
  IMAGE_PULL_SECRET: 'azure-auth'
  K8S_SERVICE_CONNECTION: 'myspringboot'

stages:
  - stage: Build
    displayName: 🔨 Build and Push Docker Image
    jobs:
      - job: BuildAndPush
        displayName: 🚀 Build and Push Image
        pool:
          vmImage: ubuntu-latest
        steps:
          - task: Docker@2
            displayName: 🏗️ Build & Push Docker Image
            inputs:
              command: buildAndPush
              repository: $(IMAGE_NAME)
              dockerfile: $(DOCKERFILE_PATH)
              containerRegistry: $(ACR_CONNECTION)
              tags: |
                $(IMAGE_TAG)

  - stage: Deploy
    displayName: 🚀 Deploy to AKS
    jobs:
      - job: DeployToAKS
        displayName: 🏗️ Deploy Helm Chart
        pool:
          vmImage: 'ubuntu-latest'

        steps:
          - checkout: self

          - task: HelmInstaller@1
            displayName: 📥 Install Helm
            inputs:
              helmVersion: 'latest'

          - task: KubernetesManifest@0
            displayName: 🔑 Create ImagePullSecret
            inputs:
              action: createSecret
              secretName: $(IMAGE_PULL_SECRET)
              dockerRegistryEndpoint: $(ACR_CONNECTION)
              kubernetesServiceConnection: $(K8S_SERVICE_CONNECTION)

          - task: HelmDeploy@0
            displayName: 🚀 Deploy Helm Chart to AKS
            inputs:
              connectionType: 'Kubernetes Service Connection'
              kubernetesServiceConnection: $(K8S_SERVICE_CONNECTION)
              namespace: $(KUBE_NAMESPACE)
              command: 'upgrade'
              chartType: 'FilePath'
              chartPath: 'helm/devops-chart'
              releaseName: $(HELM_RELEASE_NAME)
              overrideValues: |
                image.repository=$(ACR_NAME)/$(IMAGE_NAME)
                image.tag=$(IMAGE_TAG)
                imagePullSecrets[0].name=$(IMAGE_PULL_SECRET)
              install: true
            name: helmDeploy

          - script: |
              echo "🚨 Deployment failed! Rolling back to the last stable release..."
              helm rollback $(HELM_RELEASE_NAME) --namespace $(KUBE_NAMESPACE)
            displayName: 🔄 Rollback on Failure
            condition: failed()
```

---

## **🔹 Rollback Strategy**  

| Scenario | Outcome |
|----------|---------|
| **Deployment Succeeds** ✅ | App runs as expected |
| **Deployment Fails** ❌ | Rolls back to the last stable version |

The rollback ensures zero downtime and stability! 🚀  

---

## **🔹 Helm Chart Structure**  

Ensure you have a **Helm chart** located at `helm/devops-chart/`. The structure should look like this:  

```
helm/
 ├── devops-chart/
 │   ├── Chart.yaml
 │   ├── values.yaml
 │   ├── templates/
 │   │   ├── deployment.yaml
 │   │   ├── service.yaml
 │   │   ├── ingress.yaml
 │   │   ├── secrets.yaml
```

### **Sample `values.yaml`**
```yaml
image:
  repository: azurespringci.azurecr.io/devops
  tag: latest
imagePullSecrets:
  - name: azure-auth

replicaCount: 2

service:
  type: ClusterIP
  port: 8080
```

---

## **🔹 Verify Deployment**  

After the pipeline runs successfully, verify the deployment in **AKS**:  

```sh
kubectl get pods -n default
kubectl get deployments -n default
kubectl get services -n default
```

To check logs:  
```sh
kubectl logs -f <pod-name> -n default
```

---

## **🎯 Conclusion**  

🎉 You now have a **fully automated CI/CD pipeline** for **Azure Kubernetes Service (AKS)** using **Azure DevOps & Helm**. This setup:  
✅ **Builds & Pushes Docker images**  
✅ **Deploys the app using Helm**  
✅ **Automatically rolls back on failure**  

