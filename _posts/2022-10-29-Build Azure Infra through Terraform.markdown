---
layout: post
title: Build Azure Infra Through Terrafrom Script 
date: 2020-10-29 16:30:00 +0530
description: You’ll find this post in your `_posts` directory. Go ahead and edit it and re-build the site to see your changes. # Add post description (optional)
img: terraform.PNG # Add image post (optional)
fig-caption: # Add figcaption (optional)
tags: [Terraform, DevOps, Azure]
---
# AKS Architecture & Concepts

## Build the Infra with Terraform

We are first going to setup the infrastructure through Terraform code. Resources that we will deploy are Storage, Database and Azure Key Vault. First step would be to create **provider.tf** as below.

Here we are adding two providers, i.e. Azure DevOps and Azure Resource Manager. We are then going to provide the URL of our Azure DevOps organization and mention the personal access token.

```
terraform {

  required\_providers {

    azuredevops = {

      source  = "microsoft/azuredevops"

      version = "0.1.3"

    }

    azurerm = {

      source  = "hashicorp/azurerm"

      version = "3.14.0"

    }

  }

}


provider "azuredevops" {

  org\_service\_url       = "https://dev.azure.com/siddhartha2303"

  personal\_access\_token = "personal access token here"

}

provider "azurerm" {

  features {}

}
```

Personal access token on Azure DevOps can be created as below.

![Personal access token on Azure DevOps]({{site.baseurl}}/assets/img/Aspose.Words.46e0d902-7e6c-4f68-aa4e-889678ab0c6d.001.png)

Next step would be to create **variable.tf** file as blow. This contains all variables to be called inside **main.tf**

```
variable "rg\_name" {

  type        = string

  description = "This is resource group name"

}

variable "location" {

  type        = string

  description = "This is location of resource group"

}

variable "environment" {

  type = string

}

variable "kv\_name" {

  type = string

}

variable "strg\_name" {

  type = string

}

variable "strgContainer\_name" {

  type = string

}

variable "db\_name" {

  type = string

}

variable "dbsrv\_name" {

  type = string

}

variable "client\_secret" {

  type = string

}

variable "service\_endpoint\_id" {

  type = string

}
```

Below **terraform.tfvars** file contains the values for the variables that we defined above. Fill **client\_secret** and **service\_endpoint\_id**

```
strg\_name          = "tfstorageactdemo10"

strgContainer\_name = "tfstoragecontainer"

kv\_name            = "tfkvdemo-1000022"

environment        = "development"

location           = "eastus"

rg\_name            = "TfDemo01"

db\_name            = "tfdemodb006"

dbsrv\_name         = "tfdemodbsrv006"

client\_secret      = "place your client secret here" 

service\_endpoint\_id = "your ADO service endpoint ID"
```

Service endpoint ID can be found in Azure DevOps as below. If don’t exist then we need to create one. This is how Azure DevOps is integrated with Azure subscription. 

![Service endpoint ID]({site.baseurl}}/assets/img/Aspose.Words.46e0d902-7e6c-4f68-aa4e-889678ab0c6d.002.png) 

And for **client\_secret**, you need to create a Service Principal in Azure.

![]({site.baseurl}}/assets/img/Aspose.Words.46e0d902-7e6c-4f68-aa4e-889678ab0c6d.003.png)

We then need to start with main.tf Step by step approach would be as below.

* Create a resource group

```
resource "azurerm\_resource\_group" "rg" {

  name     = var.rg\_name

  location = var.location

  tags = {

    environment = var.environment

  }

}
```

* Get the details for Azure Dev Ops, current subscription and Service Principal

```
data "azuredevops\_project" "AKS-DEMO" {

  name = "AKS-DEMO"

}

data "azuread\_service\_principal" "tfServicepPrincipal" {

  display\_name = "tfServicepPrincipal"

}

data "azurerm\_subscription" "subscriptionID" {

}

data "azurerm\_client\_config" "current" {}
```

* Create Key Vault and assign Access Policy to Service Principal

```
resource "azurerm\_key\_vault" "kv1" {

  depends\_on                 = [azurerm\_resource\_group.rg, module.create\_storage]

  name                       = var.kv\_name

  location                   = var.location

  resource\_group\_name        = var.rg\_name

  tenant\_id                  = data.azurerm\_client\_config.current.tenant\_id

  soft\_delete\_retention\_days = 7

  purge\_protection\_enabled   = false

  sku\_name                   = "standard"

  access\_policy {

    tenant\_id = data.azurerm\_client\_config.current.tenant\_id

    object\_id = data.azurerm\_client\_config.current.object\_id

    //application\_id = data.azuread\_service\_principal.tfServicepPrincipal.application\_id

    key\_permissions = [

      "Get",

    ]

    secret\_permissions = [

      "Get", "Backup", "Delete", "List", "Purge", "Recover", "Restore", "Set",

    ]

    storage\_permissions = [

      "Get",

    ]

  }

  access\_policy {

    tenant\_id = data.azurerm\_client\_config.current.tenant\_id

    object\_id = data.azuread\_service\_principal.tfServicepPrincipal.object\_id

    key\_permissions = [

      "Get", "List"

    ]

    secret\_permissions = [

      "Get", "Backup", "Delete", "List", "Purge", "Recover", "Restore", "Set",

    ]

    storage\_permissions = [

      "Get",

    ]

  }

}
```

* Call the Storage module and provide the parameters

```
module "create\_storage" {

  source             = "../Modules/storage"

  rg\_name            = var.rg\_name

  strg\_name          = var.strg\_name

  strgContainer\_name = var.strgContainer\_name

  depends\_on         = [azurerm\_resource\_group.rg]

}
```

* Once these resources will be created Terraform will proceed to creating the Secrets in KeyVault. You can see in below code it’s dependent on creation of storage module **“depends\_on = [module.create\_storage]”** as storage keys will be available once it’s created.

```
resource "azurerm\_key\_vault\_secret" "client-id" {

  name         = "client-id"

  value        = data.azuread\_service\_principal.tfServicepPrincipal.application\_id

  key\_vault\_id = azurerm\_key\_vault.kv1.id

  depends\_on = [

    module.create\_storage

  ]

}

resource "azurerm\_key\_vault\_secret" "client-secret" {

  name         = "client-secret"

  value        = var.client\_secret

  key\_vault\_id = azurerm\_key\_vault.kv1.id

  depends\_on = [

    module.create\_storage

  ]

}

resource "azurerm\_key\_vault\_secret" "TenantID" {

  name         = "TenantID"

  value        = data.azurerm\_subscription.subscriptionID.tenant\_id

  key\_vault\_id = azurerm\_key\_vault.kv1.id

  depends\_on = [

    module.create\_storage

  ]

}

resource "azurerm\_key\_vault\_secret" "SubscriptionID" {

  name         = "SubscriptionID"

  value        = data.azurerm\_subscription.subscriptionID.subscription\_id

  key\_vault\_id = azurerm\_key\_vault.kv1.id

  depends\_on = [

    module.create\_storage

  ]

}

resource "azurerm\_key\_vault\_secret" "strgKey1" {

  name         = "strgKey1"

  value        = module.create\_storage.storage\_primary\_access\_key

  key\_vault\_id = azurerm\_key\_vault.kv1.id

  depends\_on = [

    module.create\_storage

  ]

}

resource "azurerm\_key\_vault\_secret" "strgKey2" {

  name         = "strgKey2"

  value        = module.create\_storage.storage\_secondary\_access\_key

  key\_vault\_id = azurerm\_key\_vault.kv1.id

  depends\_on = [

    module.create\_storage

]

}
```

* Next Terraform will proceed with SQL database creation. And Key Vault should be ready prior to this as Database is going to store it’s connection string into it.

```
module "create\_db" {

  source             = "../Modules/db"

  rg\_name            = var.rg\_name

  keyvault\_name      = var.kv\_name

  sql\_server\_name    = var.dbsrv\_name

  sql\_database\_name  = var.db\_name

  sql\_admin\_login    = var.dbsrv\_name

  sql\_admin\_password = "India@123"

  depends\_on         = [azurerm\_resource\_group.rg, azurerm\_key\_vault.kv1]

}
```

* Then it will configure Azure DevOps to link Azure Key Vault to get the secrets against each variables.

```
resource "azuredevops\_variable\_group" "azdevops-variable-group" {

  depends\_on = [

    azurerm\_key\_vault\_secret.client-id,

    azurerm\_key\_vault\_secret.client-secret,

    azurerm\_key\_vault\_secret.TenantID,

    azurerm\_key\_vault\_secret.SubscriptionID,

    azurerm\_key\_vault\_secret.strgKey1,

    azurerm\_key\_vault\_secret.strgKey2,

  ]

  project\_id   = data.azuredevops\_project.AKS-DEMO.project\_id

  name         = "azkeys"

  description  = "key vault keys"

  allow\_access = true

  key\_vault {

    name                = var.kv\_name

    service\_endpoint\_id = var.service\_endpoint\_id

  }

  variable {

    name = "client-secret"

  }

  variable {

    name = "client-id"

  }

  variable {

    name = "TenantID"

  }

  variable {

    name = "SubscriptionID"

  }

  variable {

    name = "strgKey1"

  }

  variable {

    name = "strgKey2"

  }

}
```

This will complete the infrastructure that we will consume as we proceed with Azure Kubernetes Services through DevOps CICD pipelines
