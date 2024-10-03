---
layout: post
title:  Passing Azure Library Values into Docker
categories: [Azure, DevOps]
---

I had a project recently where I created a private software package and hosted it on Azure Artifacts, a similar platform to Pypi or Github for distributing packages. There was a scenario where I needed to pass values from Azure Library into Docker.

Since my package was private, it could only be accessed by those with specific credentials- credentials which could authenticate that a user has correct access. One of the services requiring use of this package ran inside a continuously-running AWS ECS task. The ECS task ran from a docker image. In the Dockerfile used to create the image, we have code telling Docker to install some software packages. This is where I would run my `pip install` command to pull my package from Azure Artifacts.

Usually when you install a package from Pypi, you simply run `pip install <package name>`. If you want to specify a repository other than the default, you can use `pip install <package name> --extra-index-url http://xyz123.com`

From Azure Artifacts, you need to specificy exactly this, but add credentials to the beginning of the link: 

```bash
pip install <package name> --extra-index-url http://<user>:<token>@xyz123.com
```
<div style="display: flex;">
    <div style="flex: 30%; text-align: center;">
         <span style="font-size: 10px">pip install from a private repository</span>
    </div>
</div>


Remember, you can find the link to download your package by clicking the "Connect to Feed" option in Azure Artifacts. (For those unfamiliar, a Feed is a container for packages in Azure DevOps; it provides version and access control for project dependencies).

However, **it is bad practice** (think security flaw) to hardcode credentials directly into the software; it should be passed another way, and it made sense to use Azure's library feature to store the credentials. Library is a service that is used to store key-value pairs, and only those with proper permissions can access the items. These variables can be passed into any pipeline within the Azure ecosystem. This also makes it possible to work with Azure DevOps even if you want to run the code in another cloud environment like AWS; you would just need to have Azure push the pre-built image to ECR, Amazon's container repostory service, for instance. 

So, we are left with the option to load the credentials into the image at build-time. And don't worry about the token showing up in the pipeline log messages- the password is obfuscated if set as "secret": 

<br>

<div style="display: flex;">
    <div style="flex: 30%; text-align: center;">
        <img src="/images/azure-library-docker/azure-library-secure-token.png" alt="Secure Azure Library Value">
         <span style="font-size: 10px">Azure Library</span>
    </div>
</div>


But take note: if you are going to store the image in Azure's container registry and are only looking for a secure way of downloading packages, this workflow might isn't *totally* relevant and might be a little circuitous (but, still good to know and might be tangentially useful). You could simply use a Docker Registry Service Connection to to more directly download packages and run them inside Docker. However, there could be other reasons to pass key-values into Docker. So, Let's keep on keeping on. In this tutorial, we will simply print out the username stored in the library.

After creating the library group and variable, create your dockerfile. Then, upload it to your Azure repository. 

For our purposes, I am skipping any package creation as discussed above. No need to get complex.


```docker
FROM python:3.11-slim-bullseye

ARG username # here, we are telling Docker to expect an argument passed during the build process

ENV APP_ENV=${username} # here, we are passing the build argument to an environmental variable.
CMD echo ${APP_ENV} # We are going to print out the variable value to the command line.
```
<div style="display: flex;">
    <div style="flex: 30%; text-align: center;">
    <span style="font-size: 10px">DOCKERFILE</span>
    </div>
</div>

In Azure pipelines, we can create a task to build the docker image. There is a specific task to do this, called "Build an Image" in the traditional UI for building azure pipelines. In YAML, the task would be called `Docker@0`. You could also use a Bash command instead to accomplish the same thing:
```bash
docker build --build-arg username=$(username) -f $(Build.SourcesDirectory)/Dockerfile
```
<div style="display: flex;">
    <div style="flex: 30%; text-align: center;">
    <span style="font-size: 10px">Docker build command from the command line</span>
    </div>
</div>

You might need to change or add additional arguments, including to specify the location of the dockerfile. I put down the default location for this tutorial.

However, I won't be using the bash command directly in the pipeline, but it is good to know what is happening behind the scenes.


But whatever method you choose to use, you need to connect the pipeline the Azure Library group. In the traditional UI, this is where you can make the connection:

<br>

<div style="display: flex;">
    <div style="flex: 30%; text-align: center;">
        <img src="/images/azure-library-docker/variable-group.png " alt="Connecting to variable group">
        <span style="font-size: 10px">Connecting to Library Variable Group in traditional pipeline UI</span>
    </div>
</div>

<br>

In YAML, you would use a snippet like this:

```yaml
variables:
    - group: Variables-Dev  # this is where we connect the variable group to the YAML pipeline
```

<br>

In the traditional UI, we can  reference the build arguments. We specify the variable by using this format: `$(variable_name)`.

<br>

<div style="display: flex;">
    <div style="flex: 30%; text-align: center;">
        <img src="/images/azure-library-docker/BuildPipelineUI.png " alt="Build Pipeline UI">
        <span style="font-size: 10px">Building and pushing a Docker image in the traditional UI</span>
    </div>
</div>

<br>

In YAML, the whole thing would look like this (I've removed sensitive information):
```yaml
pool:
  vmImage: ubuntu-latest

variables:
    - group: Variables-Dev  # this is where we connect the variable group to the YAML pipeline

steps:
- task: Docker@0
  displayName: 'Build an image'
  inputs:
    azureSubscription: 'Azure subscription 1 (########################)'
    azureContainerRegistry: '{"loginServer":"##############.azurecr.io", "id" : "/subscriptions/################/resourceGroups/##########/providers/Microsoft.ContainerRegistry/registries/##########"}'
    buildArguments: 'username=$(username)'
    additionalImageTags: 'my_image'
- task: Docker@0
  displayName: 'Push an image'
  inputs:
    azureSubscription: 'Azure subscription 1 (###############)'
    azureContainerRegistry: '{"loginServer":"################.azurecr.io", "id" : "/subscriptions/##################/resourceGroups/#############/providers/Microsoft.ContainerRegistry/registries/#############"}'
    action: 'Push an image'
    additionalImageTags: 'my_image'

- task: Docker@2
  displayName: 'login to ACR'
  inputs:
    containerRegistry: testServiceConnection
    command: login

- script: 'docker run ################.azurecr.io/testproject:my_image'
  displayName: 'Command Line Script'
```
<div style="display: flex;">
    <div style="flex: 30%; text-align: center;">
        <span style="font-size: 10px">Doing the same thing in a more reproducible and versionable way</span>
    </div>
</div>

<br>



And you can see the username populating after running the dockerfile:


<div style="display: flex;">
    <div style="flex: 30%; text-align: center;">
        <img src="/images/azure-library-docker/final_output.png " alt="Final Output">
        <span style="font-size: 10px">Final output. See how we print the username</span>
    </div>
</div>