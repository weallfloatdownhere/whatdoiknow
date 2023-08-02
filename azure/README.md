# Azure Devops documentation
Here, you will be able to find everything related to concepts, setups, examples and information about how to implement CI for your projects using Azure Devops.

## [Build and push Docker images to Azure Container Registry using Docker templates.](https://learn.microsoft.com/en-us/azure/devops/pipelines/ecosystems/containers/acr-template)

<font size=4>Create the pipeline.</font>

1. Sign in to your Azure DevOps organization and navigate to your project.

2. Select Pipelines, and then select New Pipeline to create a new pipeline.

    ![create](https://learn.microsoft.com/en-us/azure/devops/pipelines/ecosystems/media/new-pipeline.png?view=azure-devops)

3. Select **GitHub YAML**, and then select **Authorize Azure Pipelines** to provide the appropriate permissions to access your repository.

4. You might be asked to sign in to GitHub. If so, enter your GitHub credentials, and then select your repository from the list of repositories.

5. From the **Configure** tab, select the **Docker - Build and push an image to Azure Container Registry** task.

    ![conf](https://learn.microsoft.com/en-us/azure/devops/pipelines/ecosystems/media/docker-task.png?view=azure-devops)

6. Select your Azure Subscription, and then select Continue.

7. Select your Container registry from the dropdown menu, and then provide an Image Name to your container image.

8. Select Validate and configure when you are done.

    ![conf](https://learn.microsoft.com/en-us/azure/devops/pipelines/ecosystems/media/docker-container-registry.png?view=azure-devops)

    As Azure Pipelines creates your pipeline, it will:

    * Create a Docker registry service connection to enable your pipeline to push images to your container registry. 
    
    * Generate an azure-pipelines.yml file, which defines your pipeline.

9. Review your pipeline YAML, and then select Save and run when you are ready.

    ![review](https://learn.microsoft.com/en-us/azure/devops/pipelines/ecosystems/media/review-your-pipeline.png?view=azure-devops)

10. Add a Commit message, and then select Save and run to commit your changes and run your pipeline.

11. As your pipeline runs, select the build job to watch your pipeline in action.

    ![commit](https://learn.microsoft.com/en-us/azure/devops/pipelines/ecosystems/media/jobs-build.png?view=azure-devops)

<font size=4>How we build our own pipeline as code.</font>

The pipeline that we just created in the previous section was generated from the Docker container template YAML. The build stage uses the Docker task Docker@2 to build and push your Docker image to the container registry.

`.azure/azure-pipelines.yml`
This file is our main pipeline and only triggers on the master branch.
It uses the templates/build.yml file as a template and will use the stages defined in that file.

```yaml
# .azure/azure-pipelines.yml
trigger:  
  branches:
    include:
    - master
    - releases/*
    exclude:
    - releases/old*

stages:
# these act as includes, you could also write those stage directly in this file instead.
- template: .azure/templates/build.yml
- template: .azure/templates/push.yml
```

## Azure pipeline steps templates

### Build docker image from Dockerfile.

```yaml
# .azure/templates/build.yml
- stage: Build
  displayName: Build a docker image
  jobs:  
  - job: Build
    displayName: Build job
    pool:
      vmImage: $(vmImageName)
    steps:
    - task: Docker@2
      displayName: Building the image
      inputs:
        command: build
        dockerfile: '$(Build.SourcesDirectory)/Dockerfile'
        tags: |
          $(tag)
```

### Push docker image in various container registry services.

<font size=3>Azure container registry (ACR).</font>

```yaml
# .azure/templates/push-acr.yml
- stage: Push
  displayName: Push to ACR
  jobs:  
  - job: Push
    displayName: Push job
    pool:
      vmImage: ubuntu-latest
    steps:
    - task: Docker@2
      displayName: Push image to ACR.
      inputs:
        command: push
        repository: $(imageRepository)
        dockerfile: $(dockerfilePath)
        containerRegistry: $(dockerRegistryServiceConnection)
        tags: |
          $(tag)
```

<font size=3>Amazon Elastic Container Registry (ECR).</font>

```yaml
# .azure/templates/push-ecr.yml
- stage: Push
  displayName: Push to ECR
  jobs:  
  - job: Push
    displayName: Push job
    pool:
      vmImage: ubuntu-latest
    steps:
    - task: ECRPushImage@1
      inputs:
        awsCredentials: 'AWS_ECR'
        regionName: $(AWS_REGION)
        imageSource: 'imagename'
        sourceImageName: $(DOCKER_REPOSITORY_NAME)
        sourceImageTag: $(Build.BuildId)
        pushTag: latest
        repositoryName: $(DOCKER_REPOSITORY_NAME)
```