# CI/CD Pipeline for Spring Boot Application using Azure Pipelines

This repository contains a CI/CD pipeline configuration for building and deploying a Spring Boot application using Azure Pipelines.

## Repository

Source Repository: [azure-spring-cicd](https://github.com/tushardashpute/azure-spring-cicd.git)

## Overview

This pipeline automates the build and deployment of a Spring Boot application by:
- Building a Docker image
- Pushing the image to Azure Container Registry (ACR)
- Using Azure Pipelines to execute CI/CD workflows

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
  dockerRegistryServiceConnection: 'xxxxxxxxxxxxxxxxxxxxxxxxxxxx'
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

## Prerequisites

1. **Azure DevOps Account**: Set up an Azure DevOps account and create a project.
2. **Azure Container Registry (ACR)**: Ensure an ACR instance is available.
3. **Service Connection**: Configure a service connection in Azure DevOps for authentication with ACR.
4. **Agent Pool**: Use a self-hosted or Microsoft-hosted agent for pipeline execution.

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

