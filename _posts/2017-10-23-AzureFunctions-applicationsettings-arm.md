---
layout: post
title: Azure Functions - Manage Application Settings via ARM
author: Mattias Lögdberg
tags: [Azure Functions, Integration]
categories: [Azure Functions]
image: 
description: 
permalink: /azurefunctions/arm-manage-application-settings
---

One of the most common task when working with deployments is the need to handle is application settings. This is where we are adding environment variables to the current Azure Function App, the environment variables often has diffrent values in diffrent environmnets (Dev/Test/Prod).
So we need to manage them to make sure that they have the correct values.

[![](/assets/uploads/2017/10/azure_functions_application_settings.png)](/assets/uploads/2017/10/azure_functions_application_settings.png)


These can be managed manually since they are rarely changed but it requires dicipline and good release documentation and sometimes quite advanced procedures to get the information, some of these cases are i.e. getting the *URL* of a *Logic App*, a value from *Key Vault* or we might just walue **quality** and **reliable** deployments.

I prefer to manage these settings via ARM templates since it gives me the possiblity to get values from Key Vault or automate the process to get values from other Azure resources and at the same time gives me control and robustness so I know that the settings are set and that the values are correct.
A note to this, make sure that operations persons are aware that settings are been set from ARM template deployment since if they do changes directly in the Azure functions Application Settings tab they will be overwriten at the next deployment and that might cause problems.


Let's look in to the ARM template, I've just copied this from the Azure Portal (automation option when creating a new Azure Function).
```
{
  "parameters": {
    "name": {
      "type": "string"
    },
    "storageName": {
      "type": "string"
    },
    "location": {
      "type": "string"
    }
  },
  "resources": [
    {
      "name": "[parameters('name')]",
      "type": "Microsoft.Web/sites",
      "dependsOn": [
        "[resourceId('Microsoft.Storage/storageAccounts', parameters('storageName'))]",
        "[resourceId('microsoft.insights/components/', parameters('name'))]"
      ],
      "properties": {
        "siteConfig": {
          "appSettings": [
            {
              "name": "AzureWebJobsDashboard",
              "value": "[concat('DefaultEndpointsProtocol=https;AccountName=',parameters('storageName'),';AccountKey=',listKeys(resourceId('Microsoft.Storage/storageAccounts', parameters('storageName')), '2015-05-01-preview').key1)]"
            },
            {
              "name": "AzureWebJobsStorage",
              "value": "[concat('DefaultEndpointsProtocol=https;AccountName=',parameters('storageName'),';AccountKey=',listKeys(resourceId('Microsoft.Storage/storageAccounts', parameters('storageName')), '2015-05-01-preview').key1)]"
            },
            {
              "name": "FUNCTIONS_EXTENSION_VERSION",
              "value": "~1"
            },
            {
              "name": "WEBSITE_CONTENTAZUREFILECONNECTIONSTRING",
              "value": "[concat('DefaultEndpointsProtocol=https;AccountName=',parameters('storageName'),';AccountKey=',listKeys(resourceId('Microsoft.Storage/storageAccounts', parameters('storageName')), '2015-05-01-preview').key1)]"
            },
            {
              "name": "WEBSITE_CONTENTSHARE",
              "value": "[concat(toLower(parameters('name')), 'a217')]"
            },
            {
              "name": "WEBSITE_NODE_DEFAULT_VERSION",
              "value": "6.5.0"
            },
            {
              "name": "APPINSIGHTS_INSTRUMENTATIONKEY",
              "value": "[reference(resourceId('microsoft.insights/components/', parameters('name')), '2015-05-01').InstrumentationKey]"
            }
          ]
        },
        "name": "[parameters('name')]",
        "clientAffinityEnabled": false
      },
      "apiVersion": "2016-03-01",
      "location": "[parameters('location')]",
      "kind": "functionapp"
    },
    {
      "apiVersion": "2015-05-01-preview",
      "type": "Microsoft.Storage/storageAccounts",
      "name": "[parameters('storageName')]",
      "location": "[parameters('location')]",
      "properties": {
        "accountType": "Standard_LRS"
      }
    },
    {
      "apiVersion": "2015-05-01",
      "name": "[parameters('name')]",
      "type": "microsoft.insights/components",
      "location": "West Europe",
      "tags": {
        "[concat('hidden-link:', resourceGroup().id, '/providers/Microsoft.Web/sites/', parameters('name'))]": "Resource"
      },
      "properties": {
        "ApplicationId": "[parameters('name')]",
        "Request_Source": "IbizaWebAppExtensionCreate"
      }
    }
  ],
  "$schema": "http://schema.management.azure.com/schemas/2014-04-01-preview/deploymentTemplate.json#",
  "contentVersion": "1.0.0.0"
}
```

The interesting section is the **appSettings** under the **siteConfig** shown bellow.

Here is the standard properties that are used when deploying a Azure Function and here we can add new Application Settings. (note that ARM functions are used to get keys to the storage account)
```
"properties": {
        "siteConfig": {
          "appSettings": [
            {
              "name": "AzureWebJobsDashboard",
              "value": "[concat('DefaultEndpointsProtocol=https;AccountName=',parameters('storageName'),';AccountKey=',listKeys(resourceId('Microsoft.Storage/storageAccounts', parameters('storageName')), '2015-05-01-preview').key1)]"
            },
            {
              "name": "AzureWebJobsStorage",
              "value": "[concat('DefaultEndpointsProtocol=https;AccountName=',parameters('storageName'),';AccountKey=',listKeys(resourceId('Microsoft.Storage/storageAccounts', parameters('storageName')), '2015-05-01-preview').key1)]"
            },
            {
              "name": "FUNCTIONS_EXTENSION_VERSION",
              "value": "~1"
            },
            {
              "name": "WEBSITE_CONTENTAZUREFILECONNECTIONSTRING",
              "value": "[concat('DefaultEndpointsProtocol=https;AccountName=',parameters('storageName'),';AccountKey=',listKeys(resourceId('Microsoft.Storage/storageAccounts', parameters('storageName')), '2015-05-01-preview').key1)]"
            },
            {
              "name": "WEBSITE_CONTENTSHARE",
              "value": "[concat(toLower(parameters('name')), 'a217')]"
            },
            {
              "name": "WEBSITE_NODE_DEFAULT_VERSION",
              "value": "6.5.0"
            },
            {
              "name": "APPINSIGHTS_INSTRUMENTATIONKEY",
              "value": "[reference(resourceId('microsoft.insights/components/', parameters('name')), '2015-05-01').InstrumentationKey]"
            }
          ]
        },
```

Adding a new property to the list is easy and adding a new key **customkey** looks like this, and we also add a new ARM parameter so we can change the value between environments

```
    {
      "name": "APPINSIGHTS_INSTRUMENTATIONKEY",
      "value": "[reference(resourceId('microsoft.insights/components/', parameters('name')), '2015-05-01').InstrumentationKey]"
    },
    {
      "name": "customkey",
      "value": "[parameters('customkeyparam')]"
    }
  ]
},
```

Added parameter:

```
"location": {
  "type": "string"
},
"customkeyparam" :{
  "type": "string"
}
```

And now we can use this with a parameter file, I copied the paramter file the same way and now we can update this parameter file and adding the new parameter:

```
{
  "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "name": {
      "value": "ibizmalotest"
    },
    "storageName": {
      "value": "ibizmalotest912c"
    },
    "location": {
      "value": "West Europe"
    }
    "customkeyparam" :{
      "value": "MyCustomValueSetViaARM"
    }
  }
}
```

After doing a Resource Group Deployment, we can go in and see the new setting in the Application Setting on the Azure Function App and start using it.

[![](/assets/uploads/2017/10/azure_functions_application_settings_customkey_value.png)](/assets/uploads/2017/10/azure_functions_application_settings_customkey_value.png)


**Summary:**

I like this approach since we can make sure that parameters are set and prepare before deployment without needing to change values in the Function App, I also love the ARM functions that we can use during deployments to automate processes or just get values from *Azure Key Vaults* to make it easier to manage secrets (note that secrets are in plain text in application settings for the users who has access).

The ARM functions can really speed up and automate the deployment processes even further, here we can as shown aboive get keys to storage account or get the URL of a Logic App during deployment time, wich is a complex/time consuming task and when using ARM functions there is also a garantee that the Logic App is deployed and ready to be used.

Creating a good ARM template will make it easier to move between environments and it will add quality to your deployments.

**But remember** to make sure that changes that are made directly to the *Function App* Application Settings are reflected to the parameter files and arm templates aswell to prevent changes to be overwriten.

Sample getting the URL of a Locic App (assumes added parameters for *Resource Group* and *Logic App* name and the trigger name). (template file)
```
    {
      "name": "APPINSIGHTS_INSTRUMENTATIONKEY",
      "value": "[reference(resourceId('microsoft.insights/components/', parameters('name')), '2015-05-01').InstrumentationKey]"
    },
    {
      "name": "LogicAppUrl",
      "value": "[listCallbackUrl(resourceId(parameters('logicApp_resourcegroup'),'Microsoft.Logic/workflows/triggers', parameters('logicApp_name'),parameters('logicApp_trigger')), providers('Microsoft.Logic', 'workflows').apiVersions[0]).value]"
    }
  ]
},
``` 

Sample *Azure Key Vault* get a specific Value from Key Vault. (parameter file)
```
{
  "$schema": "https://schema.management.azure.com/schemas/2015-01-01/deploymentParameters.json#",
  "contentVersion": "1.0.0.0",
  "parameters": {
    "name": {
      "value": "ibizmalotest"
    },
    "storageName": {
      "value": "ibizmalotest912c"
    },
    "location": {
      "value": "West Europe"
    }
    "customkeyparam" :{
      "reference": {
        "keyVault": {
          "id": "/subscriptions/fake-a4af-4bc9-a733-f88f0eaa4296/resourceGroups/testenvironment/providers/Microsoft.KeyVault/vaults/ibizmalotest"
        },
        "secretName": "customkeysecretname"
      }
    }
  }
}
```

[![](/assets/uploads/2017/10/azure_functions_key_vault_customsecretname.png)](/assets/uploads/2017/10/azure_functions_key_vault_customsecretname.png)

