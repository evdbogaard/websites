---
title: "Bicep vs ARM templates"
date: 2021-05-30T10:00:00+02:00
draft: false
toc: false
images:
tags:
  - azure
  - arm
  - bicep
---

For a while now Bicep v0.3 is out. Prior to this version it was already a great tool to use, but since v0.3 it is fully capable to do the same as an ARM template. So, what is it and why would you want to use it?

Infrastructure as code solutions allow you to declare how your infrastructure should look in a file (e.g json). When setup the way you want it makes it easier to add new resources and make sure they have the same settings as others. Especially for resources you only add once a year it can be easy for a developer to forget the exact settings he used the last time. Instead of looking them up they are already documented in the file and only adding the name to an array makes sure it gets created correctly. This also lowers the changes of human error, which is always nice.

# What is Bicep?
Bicep is a Domain Specific Language (DSL) that allows declarative deployment of Azure resources. Everything you can do with an ARM template you can also do with Bicep and as soon as a new resource is added to Azure it is immediately supported by Bicep.

# Azure Resource Manager (ARM) templates
As Bicep is closely linked to ARM templates you can't really talk about one, without mentioning the other. They both allow you to do the exact same thing, however when you start working with ARM templates you quickly notice it has some drawbacks. Let's start by comparing some ARM and Bicep templates with each other.

```
{
    "$schema": "http://schema.management.azure.com/schemas/2015-01-01/deploymentTemplate.json#",
    "contentVersion": "1.0.0.0",
    "parameters": {
        "subscriptionId": {
            "type": "string",
            "defaultValue": "[subscription().id]"
        },
        "name": {
            "type": "string",
            "defaultValue": "demo-bicep-webapp"
        },
        "location": {
            "type": "string",
            "defaultValue": "[resourceGroup().location]"
        },
        "serverFarmResourceGroup": {
            "type": "string",
            "defaultValue": "[resourceGroup().name]"
        },
        "sku": {
            "type": "string",
            "defaultValue": "Free"
        },
        "skuCode": {
            "type": "string",
            "defaultValue": "F1"
        }
    },
    "resources": [
        {
            "apiVersion": "2018-11-01",
            "name": "[parameters('name')]",
            "type": "Microsoft.Web/sites",
            "location": "[parameters('location')]",
            "dependsOn": [
                "[concat('Microsoft.Web/serverfarms/', parameters('name'))]"
            ],
            "properties": {
                "name": "[parameters('name')]",
                "siteConfig": {
                    "metadata": [
                        {
                            "name": "CURRENT_STACK",
                            "value": "dotnetcore"
                        }
                    ]
                },
                "serverFarmId": "[concat('/subscriptions/', parameters('subscriptionId'),'/resourcegroups/', parameters('serverFarmResourceGroup'), '/providers/Microsoft.Web/serverfarms/', parameters('name'))]"
            }
        },
        {
            "apiVersion": "2018-11-01",
            "name": "[parameters('name')]",
            "type": "Microsoft.Web/serverfarms",
            "location": "[parameters('location')]",
            "properties": {
                "name": "[parameters('name')]"
            },
            "sku": {
                "Tier": "[parameters('sku')]",
                "Name": "[parameters('skuCode')]"
            }
        }
    ]
}
```
Above you see a small ARM template that creates an App Service Plan and a Web App. As you can see there is a lot of boilerplate code to get everything working. Each value for a parameter needs to be on a new line. You have to figure out yourself which resources depend on each other and add them to the correct `dependsOn` block. And you need a pretty elaborate `concat` to get the Web App with the `serverFarmId` of the App Service Plan you create in the same template.

Below you see the same template, but written in Bicep. First thing you could notice is that it's only half the size of the ARM one. Paramters are more compact as the type, name, and default value can all be written in a single line. Secondly Bicep is smart enough to figure out if resources depend on each other. We use `appServicePlan` inside the resource of `webApp`, this allows it to know it first needs to deploy `appServicePlan` and automatically adds the `dependsOn` part when it gets converted from Bicep to an ARM template.  

```
param name string = 'demo-bicep-webapp'
param location string = resourceGroup().location
param sku string = 'Free'
param skuCode string = 'F1'

resource webApp 'Microsoft.Web/sites@2018-11-01' = {
  name: name
  location: location
  properties: {
    name: name
    siteConfig: {
      metadata: [
        {
          name: 'CURRENT_STACK'
          value: 'dotnetcore'
        }
      ]
    }
    serverFarmId: appServicePlan.id
  }
}

resource appServicePlan 'Microsoft.Web/serverfarms@2018-11-01' = {
  name: name
  location: location
  properties: {
    name: name
  }
  sku: {
    Tier: sku
    Name: skuCode
  }
}
```

# Modules
With the small examples used above it is easy to see what is going on. However, when you're deploying your entire infrastructure this way a single file quickly becomes too big.
ARM has linked templates to deal with this problem. A separate file you can link towards which holds a part of your deployment. Unfortunately linked templates cannot be a local file. The file must be accessible through an url. This can be a link to a storage account or a public repository like GitHub.

To use it add a deployment to your resources array with the link to the template and a link to the parameters file if available. Below is a small example of how this looks, taken from Microsoft docs.
```
"resources": [
  {
    "type": "Microsoft.Resources/deployments",
    "apiVersion": "2020-10-01",
    "name": "linkedTemplate",
    "properties": {
      "mode": "Incremental",
      "templateLink": {
        "uri": "https://mystorageaccount.blob.core.windows.net/AzureTemplates/newStorageAccount.json",
        "contentVersion": "1.0.0.0"
      },
      "parametersLink": {
        "uri": "https://mystorageaccount.blob.core.windows.net/AzureTemplates/newStorageAccount.parameters.json",
        "contentVersion": "1.0.0.0"
      }
    }
  }
]
```

With Bicep they introduced modules. When converted to ARM template a module translated to a nested template. A nested template is similar to a linked template with the difference that instead of an external link, the template is written in the same file. This generates a big ARM template that isn't friendly to read, but as you work from Bicep you don't need to make changes to the ARM template directly.

Each file saved with a bicep extension can be used as a module. From another bicep file use the module keyword and supply the name and params it needs. The name is only for the deployment and must be unique in the deployment itself. The params are the same params the file you point to uses. Modules also support a scope which allows you to deploy the module in a different resource group.
```
module webapp './webapp.bicep' = {
  name: 'webappDeploy'
  params: {
    name: 'demo-bicep-webapp'
  }
}
```

# CLI commands
With the introduction of Bicep v0.3 it has been integrated with the Azure CLI. Before you had to install Bicep yourself, but now if you have Azure CLI installed you can do all the same commands starting with `az bicep`.

The command you probably use the most is `az bicep build`. It takes a single `--file` as input argument and converts it to an ARM template with the json extension. By default it has the same name as the input file, but you can choose to give it a different name with the `--output` argument. Prior to v0.3 it was possible to build multiple files at once, but since the integration with Azure CLI it seems only one file can be build at a time. If someone knows of a way to have multiple files build at once I'd be happy to learn that.

Besides building a Bicep file there is also a command to deploy your file to a resource group or subscription. This command previously only accepted an ARM template, but now it can handle Bicep files directly. In the background it converts them first to an ARM template and continues the deployment with that template.
The commands to do this are in the `az deployment` group. You can choose either `sub` or `group` depending if you want to deploy to an subscription or resource group. `create` is used to start a new deployment. Furthermore there are arguments to point to the bicep file, which can be a local file or an URL to a file, and add a parameters file if necessary. The total command could look like this: `az deployment group create -g demo-rg --template-file webapp.bicep --parameters webapp.parameters.dev.json`

There is another argument you can pass along with the above command. It is called `--confirm-with-what-if` or simply `-c`. This argument checks what you current deployment would change in your resource group or subscription and shows a list of changes it will make. You then need to confirm that you want those changes before it applies them.
![image](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/2rrxbzhq55ibneady8f4.png)

# Decompile an ARM template
As Bicep is new compared to ARM templates there is the possibility that you already have multiple templates. To make the step towards Bicep easier they added a command to convert an ARM template to Bicep. The command for this is `az bicep decompile`. It takes a json file as input and tries it best to make it in Bicep. If we take the ARM template from the beginning it converts to this:

```
param subscriptionId string = subscription().id
param name string = 'demo-bicep-webapp'
param location string = resourceGroup().location
param serverFarmResourceGroup string = resourceGroup().name
param sku string = 'Free'
param skuCode string = 'F1'

resource name_resource 'Microsoft.Web/sites@2018-11-01' = {
  name: name
  location: location
  properties: {
    name: name
    siteConfig: {
      metadata: [
        {
          name: 'CURRENT_STACK'
          value: 'dotnetcore'
        }
      ]
    }
    serverFarmId: '/subscriptions/${subscriptionId}/resourcegroups/${serverFarmResourceGroup}/providers/Microsoft.Web/serverfarms/${name}'
  }
  dependsOn: [
    Microsoft_Web_serverfarms_name
  ]
}

resource Microsoft_Web_serverfarms_name 'Microsoft.Web/serverfarms@2018-11-01' = {
  name: name
  location: location
  properties: {
    name: name
  }
  sku: {
    Tier: sku
    Name: skuCode
  }
}
```

As you can see it's not perfect as the `serverFarmId` can be simplified and that would eliminate the need of the `dependsOn`, but it is a great start if you have multiple ARM templates and want to move to Bicep.

There is more Bicep can do. As stated earlier everything that is possible in an ARM template is also possible in Bicep, with the exception of some know limitations (https://github.com/Azure/bicep#known-limitations). They are actively working on Bicep so the list gets shorter over time. The modules, less boilerplate code, and cli integration are already enough for me to choose Bicep over ARM template, but be sure to check it out yourself to see what more it supports.

# References
- https://github.com/Azure/bicep