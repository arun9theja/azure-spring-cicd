# CI/CD Pipeline for Spring Boot Application using Azure Pipelines

This repository contains a CI/CD pipeline configuration for building and deploying a Spring Boot application using Azure Pipelines.

## Repository

Source Repository: [azure-spring-cicd](https://github.com/tushardashpute/azure-spring-cicd.git)

## Overview

This pipeline automates the build and deployment of a Spring Boot application by:

- Building a Docker image
- Pushing the image to Azure Container Registry (ACR)
- Using Azure Pipelines to execute CI/CD workflows

## Prerequisites

1. **Azure DevOps Account**: Set up an Azure DevOps account and create a project.
2. **Azure Container Registry (ACR)**: Ensure an ACR instance is available.
3. **Service Connection**: Configure a service connection in Azure DevOps for authentication with ACR.

## Steps

### 1. Create a Resource Group

A resource group in Azure is a container that holds related resources. To create one, use the Azure CLI:

```sh
az group create --name myResourceGroup --location eastus
```

### 2. Create an ACR Under the Same

Azure Container Registry (ACR) stores and manages container images. Create an ACR using:

```sh
az acr create --resource-group myResourceGroup --name azurespringci --sku Basic
```

### 3. Create a New Project in Azure DevOps

1. Navigate to [Azure DevOps](https://dev.azure.com/).
2. Click on **New Project**.
3. Provide a **Project Name**, set the visibility, and click **Create**.

### 4. Import the Source Repo Inside the Repos Section of the Project Created in Step 3

1. Navigate to **Repos** in your Azure DevOps project.
2. Click **Import a repository**.
3. Paste the repository URL: `https://github.com/tushardashpute/azure-spring-cicd.git`
4. Click **Import** to bring the source code into Azure DevOps.

### 5. Create a New Pipeline for CI Using azure-pipeline.yaml from the Repo

Build and push an image to Azure Container Registry.

1. Navigate to **Pipelines** in Azure DevOps.
2. Click on **New Pipeline**.
3. Choose **Azure Repos Git** and select your repository.
4. Select **Existing Azure Pipelines YAML file**.
5. Provide the path to `azure-pipelines.yaml` in your repository.
6. Click **Run** to create and start the pipeline.

### 6. Configure Service Connection to ACR

To allow Azure Pipelines to push images to Azure Container Registry, create a service connection:

1. Navigate to **Project Settings** in Azure DevOps.
2. Under **Pipelines**, click **Service connections**.
3. Click **New service connection** and choose **Docker Registry**.
4. Select **Azure Container Registry** and click **Next**.
5. Choose the ACR created in Step 2 (`azurespringci.azurecr.io`).
6. Authenticate using **Service principal (automatic)**.
7. Provide a name (e.g., `acr-service-connection`) and click **Save**.
8. Update the `dockerRegistryServiceConnection` variable in `azure-pipelines.yaml` with the new service connection name.

Build and push an image to Azure Container Registry.

1. Navigate to **Pipelines** in Azure DevOps.
2. Click on **New Pipeline**.
3. Choose **Azure Repos Git** and select your repository.
4. Select **Existing Azure Pipelines YAML file**.
5. Provide the path to `azure-pipelines.yaml` in your repository.
6. Click **Run** to create and start the pipeline.
7. Navigate to **Pipelines** in Azure DevOps.
8. Click on **New Pipeline**.
9. Choose **Azure Repos Git** and select your repository.
10. Select **Existing Azure Pipelines YAML file**.
11. Provide the path to `azure-pipelines.yaml` in your repository.
12. Click **Run** to create and start the pipeline.

## Pipeline Configuration

### Trigger

The pipeline is triggered on changes to the `develop` branch.

```yaml
trigger:
- develop
```

### Variables

The pipeline uses the following variables:

- **dockerRegistryServiceConnection**: Service connection ID for ACR
- **imageRepository**: Name of the Docker repository
- **containerRegistry**: ACR URL
- **dockerfilePath**: Path to the Dockerfile
- **tag**: Image tag (uses Build ID)
- **vmImageName**: Agent VM image

```yaml
variables:
  dockerRegistryServiceConnection: '***'
  imageRepository: 'devops'
  containerRegistry: 'azurespringci.azurecr.io'
  dockerfilePath: '$(Build.SourcesDirectory)/Dockerfile'
  tag: '$(Build.BuildId)'
  vmImageName: 'ubuntu-latest'
```

### Stages

#### Build Stage

This stage builds and pushes the Docker image to Azure Container Registry.

```yaml
stages:
- stage: Build
  displayName: Build and push stage
  jobs:
  - job: Build
    displayName: Build
    pool:
      vmImage: $(vmImageName)
    steps:
    - task: Docker@2
      displayName: Build and push an image to container registry
      inputs:
        command: buildAndPush
        repository: $(imageRepository)
        dockerfile: $(dockerfilePath)
        containerRegistry: $(dockerRegistryServiceConnection)
        tags: |
          $(tag)
```

## Deployment

1. Clone the repository:
   ```sh
   git clone https://github.com/tushardashpute/azure-spring-cicd.git
   cd azure-spring-cicd
   ```
2. Commit changes to the `develop` branch to trigger the pipeline.
3. Monitor the build in Azure DevOps Pipelines.
4. Retrieve the built image from ACR for deployment.

## Future Enhancements

- Add deployment stage to Kubernetes or Azure App Service.
- Implement automated tests within the CI pipeline.
- Integrate with monitoring and logging tools.
