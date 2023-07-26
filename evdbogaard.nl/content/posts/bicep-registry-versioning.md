---
title: "Using versioning with Bicep Registry"
date: 2023-02-15T10:00:00+02:00
draft: false
toc: false
images:
tags:
  - infrastructureascode
  - bicep
  - azure
---

One of the great things about Bicep is that it allows you to split it up in smaller modules that can be easily referenced from another Bicep file. This increases readability of your files and also allows for easier reuse of these modules. When you want to reference the same module in different repositories there are a couple of ways to do this. One of them is by using a Bicep Registry. For this you can use [Azure Container Registry](https://azure.microsoft.com/en-us/products/container-registry) which next to container images also accepts Bicep files. To later on reference them in a Bicep file you can use the following url `br:evdbregistry.azurecr.io/bicep/modules/mymodule:v1`.
The first part is url of the specific registry and the module path. You end with a tag which points to a specific version of that module inside the registry.

At the company I work we started using a Bicep Registry as well and wanted to have versioning for all files that are put inside the registry. To do this we setup a new repository and pipeline to handle this. However, there were two main problems we needed to tackle.

These issues were:
- Which tags to use when pushing to the registry?
- How to push only the files that changed into the registry, instead of everything all the time?

## Which tag to pick?
When you push a bicep module towards a registry you need to supply a tag. An easy way for tags is to keep track of which version it is (i.e. v1, v2, etc.). The information on which version a specific file is can be stored inside the bicep module itself with the help of the `metadata` keyword.

When a bicep file is build into an ARM Json file some extra data is added in the metadata section. This contains version of bicep used and a hash. You can add to this section by using the `metadata` keyword as follows: `metadata version = 'v1'`.
When you build your bicep file you can see the metadata is added in the json file.

```
{
  "$schema": "https://schema.management.azure.com/schemas/2019-04-01/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0",
  "metadata": {
    "_generator": {
      "name": "bicep",
      "version": "0.14.6.61914",
      "templateHash": "7937327168356229929"
    },
    "version": "v1"
  },
  "resources": []
}
```

## Finding changed files
We the version info we could in theory write every file all the time to the container registry, but it is a big waste to do this for files that didn't even change.
To determine which files have changed I use the following git command: `git diff --name-only --diff-filter=d HEAD^ HEAD`

This gives the differences between the current commit (HEAD) and it's parent (HEAD^). Because of `--name-only` the information we get back is stripped down to only file names that have changed. The `diff-filter` is added to exclude deleted files as we don't want to write those to a registry as they no longer exist.

**note:** Azure DevOps Pipelines by default use a shallow fetch as a way of optimization. For this command to work you need a depth of 2. To edit this go to your pipeline -> edit -> trigger -> YAML -> Get sources -> Shallow fetch

![Shallow fetch](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/m9kifqa8mailuc5ao0u1.png)

## Putting everything in YAML
The pipeline consists out of two stages. The first stage will find all changed files, build them, and put them into an artifact. The second stage will get the artifact and push the files one by one to the registry.
Each stage has its own PowerShell script to do this.

### Build stage
```
$changedFiles = git diff --name-only --diff-filter=d HEAD^ HEAD
$artifactsDirectory = "$(Build.ArtifactStagingDirectory)"
foreach($fileName in $changedFiles)
{
  Write-Host "Found $fileName"
  if (!$fileName.EndsWith(".bicep"))
  {
    continue
  }

  $file = Get-ChildItem $fileName
  az bicep build -f $fileName --outfile "${artifactsDirectory}/$($file.BaseName).json"
}
```

First we get all files that have been changed since the last commit. From this list we need to filter out anything that isn't a bicep file. There are other files in the repository (like the yml file itself) that don't need to be pushed to the registry. Finally we use `az bicep build` to generate a json file. This is a good check to see if the bicep file itself is valid, and in the next stage we need to read the json file to get the version information out of it.
It doesn't matter that we converted the bicep files to json already in this stage. The `az bicep publish` accepts both json and bicep files.
All build files are put in together in the artifact staging directory and are published as an artifact in the final step of this stage.

### Publish stage
```
$files = Get-ChildItem $(Pipeline.Workspace) -Filter "*.json" -R

foreach ($file in $files)
  {
    $fileBaseName = $file.BaseName
    $jsonFile = Get-Content $file.FullName | Out-String | ConvertFrom-Json
    $version = $jsonFile.metadata.version

    Write-Host "Pushing to registry ${fileBaseName}:${version}"
    az bicep publish --file $file.FullName --target "br:evdbregistry.azurecr.io/bicep/modules/${fileBaseName}:${version}"
}
```

In this stage the artifact is downloaded and we load in all the files one by one. As they are json files we can easily convert them to json and pick the version information from the metadata block. Finally we use `az bicep publish` and give the necessary parameters to push the script inside the registry.

Full yaml file:
```
trigger:
  branches:
    include:
      - main

pool:
  vmImage: 'ubuntu-latest'

stages:
  - stage: build
    jobs:
      - job: generate_artifact
        displayName: "Generate artifacts"
        steps:
        - task: PowerShell@2
          displayName: Copy changed files
          inputs:
            targetType: 'inline'
            script: |
              $changedFiles = git diff --name-only --diff-filter=d HEAD^ HEAD
              $artifactsDirectory = "$(Build.ArtifactStagingDirectory)"
              foreach($fileName in $changedFiles)
              {
                Write-Host "Found $fileName"
                if (!$fileName.EndsWith(".bicep"))
                {
                  continue
                }

                $file = Get-ChildItem $fileName
                az bicep build -f $fileName --outfile "${artifactsDirectory}/$($file.BaseName).json"
              }
        
        - publish: $(Build.ArtifactStagingDirectory)
          artifact: $(Build.BuildNumber)
  
  - stage: publish
    displayName: "Publish to container registry"
    dependsOn: build
    condition: succeeded()
    jobs:
      - deployment: publish_to_registry
        displayName: Publish to registry
        environment: 'BicepEnv'
        strategy:
          runOnce:
            deploy:
              steps:
              - task: AzureCLI@2
                displayName: 'Publish to registry'
                inputs:
                  azureSubscription: 'Bicep ARM'
                  scriptType: 'pscore'
                  scriptLocation: 'inlineScript'
                  inlineScript: |
                    $files = Get-ChildItem $(Pipeline.Workspace) -Filter "*.json" -R

                    foreach ($file in $files)
                    {
                      $fileBaseName = $file.BaseName
                      $jsonFile = Get-Content $file.FullName | Out-String | ConvertFrom-Json
                      $version = $jsonFile.metadata.version

                      Write-Host "Pushing to registry ${fileBaseName}:${version}"
                      az bicep publish --file $file.FullName --target "br:evdbregistry.azurecr.io/bicep/modules/${fileBaseName}:${version}"
                    }
```

## Automatic version number
What I don't like about above solution is that it still requires manual update of version numbers. If you're not careful and update a file, without updating the version number it will just overwrite the version that is stored inside the registry.

If you want an automatic version number you can use the pipeline variable `$(Build.BuildNumber)`. This will generate a string that looks like `20230101.1`. This guarantees that the tag is always unique as each build has a new number.

The downside of this approach is that there is no clear way to know which tags exists without looking into the registry itself. So it's up to yourself to see what you like best, fully automatic tags or more predictable version numbers that need to be updated manually.