# Azure Devops documentation
Here, you will be able to find everything related to concepts, setups, examples and information about how to implement CD for your projects and infrastructure using Azure Devops.

*`IMPORTANT:` If you have not read the generic [**devops**](../devops/README.md) section yet, we strongly suggest you to go there first in order to get yourself familliar with the generic devops concepts.*

# Infrastructure as code

### [Automating infrastructure deployments in the Cloud with Terraform and Azure Pipelines.](https://www.azuredevopslabs.com/labs/vstsextend/terraform/)
In this section, we will go over the basic concepts of how Azure pipelines and terraform can interact togheter in order to automate the provisionning of our infrastructure.

### Overview

![howto](https://www.azuredevopslabs.com/labs/vstsextend/terraform/images/Terraform-workflow.gif)

### Example

Let's say you have the terraform file below located `terraform/terraform.tf` in your project repository.

*terraform/terraform.tf*
```groovy
terraform {
    backend "azurerm" {}
}
provider "azurerm" {
    subscription_id = "74d6a1ea-aaaa-bbbb-cccc-28b098c3435f"
    tenant_id       = "11111111-1111-1111-1111-111111111111"
    skip_provider_registration = "true"
    features {}
}
...
```

Stage: Terraform Validate
This stage run the terraform validate command, and if there are any problems it will fail. As you can see the from YAML below, the stage dependsOn the runCheckov stage and we are also installing Terraform and running terraform init before the terraform validate command is executed;

One of the other things which will happen when this stage is executed happens as part of the terraform init task, as we are setting ensureBackend to true the task will check for the presence of the Azure Storage Account we wish to use to store our Terraform state file in, if it is not there then the task will helpfully create it for us.

Once your code has been validated we can move onto the next stage.

```yaml
  - stage: "validateTerraform"
    displayName: "Terraform - Validate"
    jobs:
      - job: "TerraformJobs"
        displayName: "Terraform > install, init and validate"
        continueOnError: false

        steps:
          - script: apt-get -y install terraform
            displayName: Install Terraform

          - script: terraform plan -var-file=plan.tfvars --out=plan.out
            displayName: Terraform plan

          - task: TerraformCLI@0
            inputs:
              command: "validate"
              environmentServiceName: "$(SUBSCRIPTION_NAME)"
            displayName: "Run > terraform validate"
```

Stage: Terraform Plan
This stage is where things get a little more interesting, eventually, as our environment does not persist across stages we need to install Terraform and run terraform init again.

Once that has been done we are running the terraform plan command, thanks to some of the features in the Terraform Azure DevOps extension by Charles Zipp  we are able to publish the results of running terraform plan to our pipeline run by setting the publishPlanResults option.

Before we look at the last tasks of this stage lets look at the code for the full stage;

```yaml
  - stage: "planTerraform"
    displayName: "Terraform - Plan"
    dependsOn:
      - "validateTerraform"
    jobs:
      - job: "TerraformJobs"
        displayName: "Terraform > install, init & plan"
        steps:
          - task: TerraformInstaller@0
            inputs:
              terraformVersion: "$(tf_version)"
            displayName: "Install > terraform"

          - task: TerraformCLI@0
            inputs:
              command: "init"
              backendType: "azurerm"
              backendServiceArm: "$(SUBSCRIPTION_NAME)"
              ensureBackend: true
              backendAzureRmResourceGroupName: "$(tf_environment)-$(tf_state_rg)"
              backendAzureRmResourceGroupLocation: "$(tz_state_location)"
              backendAzureRmStorageAccountName: "$(tf_state_sa_name)"
              backendAzureRmStorageAccountSku: "$(tf_state_sku)"
              backendAzureRmContainerName: $(tf_state_container_name)
              backendAzureRmKey: "$(tf_environment).terraform.tstate"
            displayName: "Run > terraform init"

          - task: TerraformCLI@0
            inputs:
              command: "plan"
              environmentServiceName: "$(SUBSCRIPTION_NAME)"
              publishPlanResults: "PlanResults"
              commandOptions: "-out=$(System.DefaultWorkingDirectory)/terraform.tfplan -detailed-exitcode"
            name: "plan"
            displayName: "Run > terraform plan"

          - task: TerraformCLI@0
            inputs:
              command: "show"
              environmentServiceName: "$(SUBSCRIPTION_NAME)"
              inputTargetPlanOrStateFilePath: "$(System.DefaultWorkingDirectory)/terraform.tfplan"
            displayName: "Run > terraform show"

          - bash: |
              if [ "$TERRAFORM_PLAN_HAS_CHANGES" = true ] && [ "$TERRAFORM_PLAN_HAS_DESTROY_CHANGES" = false ] ; then
                echo "##vso[task.setvariable variable=HAS_CHANGES_ONLY;isOutput=true]true"
                echo "##vso[task.logissue type=warning]Changes with no destroys detected, it is safe for the pipeline to proceed automatically"
                fi
              if [ "$TERRAFORM_PLAN_HAS_CHANGES" = true ] && [ "$TERRAFORM_PLAN_HAS_DESTROY_CHANGES" = true ] ; then
                echo "##vso[task.setvariable variable=HAS_DESTROY_CHANGES;isOutput=true]true"
                echo "##vso[task.logissue type=warning]Changes with Destroy detected, pipeline will require a manual approval to proceed"
                fi
              if [ "$TERRAFORM_PLAN_HAS_CHANGES" != true ] ; then
                echo "##vso[task.logissue type=warning]No changes detected, terraform apply will not run"
              fi              
            name: "setvar"
            displayName: "Vars > Set Variables for next stage"
```

### References
* [Azure DevOps Terraform Pipeline with Checkov & Approvals](https://www.russ.foo/2021/06/08/azure-devops-terraform-pipeline-with-checkov-approvals/)

# Docker images building & pushing.

## [Build and push Docker images to Azure Container Registry using Docker templates.](https://learn.microsoft.com/en-us/azure/devops/pipelines/ecosystems/containers/acr-template)

### Create the pipeline.

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

### How we build our own pipeline as code.

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

Azure container registry (ACR).

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

Amazon Elastic Container Registry (ECR).

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

# Additionals ressources.

[Official Azure pipeline tasks repository.](https://github.com/microsoft/azure-pipelines-tasks/tree/master/Tasks)