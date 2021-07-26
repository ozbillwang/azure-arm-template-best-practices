# Azure ARM Template Best Practices 🌐

[Azure Resource manager (ARM) template](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/overview) best practices for Mac/Linux users

I am not fan of Microsoft **Powershell**, so this document is for Mac/Linux users.

<!-- START doctoc generated TOC please keep comment here to allow auto update -->
<!-- DON'T EDIT THIS SECTION, INSTEAD RE-RUN doctoc TO UPDATE -->
**Table of Contents**  *generated with [DocToc](https://github.com/thlorenz/doctoc)*

- [Official ARM template best practices](#official-arm-template-best-practices)
- [ARM template is not immutable, not IaC.](#arm-template-is-not-immutable-not-iac)
- [resourceId](#resourceid)
- [export template](#export-template)
- [Use popular IDE editers](#use-popular-ide-editers)
- [Azure templates (PREVIEW)](#azure-templates-preview)
- [Azure Blueprint](#azure-blueprint)
- [Azure command line - az cli](#azure-command-line---az-cli)
  - [ARM template Dry-run with az cli](#arm-template-dry-run-with-az-cli)
- [Azure resource group](#azure-resource-group)
- [Contributing](#contributing)

<!-- END doctoc generated TOC please keep comment here to allow auto update -->


### Official ARM template best practices

Go through Azure official ARM template best practices first

https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/best-practices

### ARM template is not immutable, not IaC.

This is important concept for any thing else. Azure ARM template is not **immutable**, they are not [IaC (infrastructure as code)](https://en.wikipedia.org/wiki/Infrastructure_as_code) at all.

I worked on [Hashicopy Terraform](https://www.terraform.io/) and [AWS Cloudformation template](https://aws.amazon.com/cloudformation/) a lot. So when I worked on deploying Azure resources with codes this year, I realized this is a disaster design by someone in Azure, who doesn't have any concepts on how infrastructure as code and Immutable are important for developer or DevOps.

The strange designs I found until now:

+ ARM template has no state file or similar function, so it doesn't care what resources you already have, it just overrides the setting. So that means, 
  + If the resource is not exist, it will create that resource with the setting you put in ARM template. 
  + If the resource is exist, it just overrides the setting, no care what's the original setting on it
  + If you clean some resources and apply this template again, ARM template doesn't delete the cleaned resources. 
  + If you accidentally apply a totally different template, **but with same template name**, it would not stop you, just create or update the resources in template.
+ There is no way to reference resources from another ARM template's output, even it has **output** in its syntax.
+ There is no way to reference resources from another resource in same template, **this is really weird design**. You have to use the way I discuss below, by **resourceId**
+ Azure API is slow, sometime takes about 15 minutes or longer to update some configurations.
  + For example, set a Virtual Network Gateway, or connection from virtual hub to expressroute circuit, etc.
  + For example, update a resource's tags, still take long time. Because it overrides the whole setting, not just update tags on that resources (So does it mean Azure API only allows REST API's **POST** and **PUT**, not **PATCH**???)
+ some resources need generate uniq key, such as service key, such as ExpressRoute Circuit. The service key is an uuid, and you can't define it yourself. Because of that, you can't manage ExpressRoute Circuit with ARM templates for its full setting. If you dry-run the template, it always prompts there is a change about the service key.
+ the setting in your resources are drifting when the ARM template is still applying. Sometime, this will surprise you. For example, when you apply an update on connection between Virtual Hub/WAN to ExpressRoute circuit, the connection is disappeared in the middle of ARM template updating, it will finally appear again, if your template has no issue.

### resourceId

[resourceId](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/template-functions-resource#resourceid) in ARM template is the key concept you need understand. 

```
# resourceId([subscription_id], [resource_group_name], [resource_type], resource #1, resource #2, and so on if required)
"[resourceId('xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx', 'otherResourceGroup', 'Microsoft.Storage/storageAccounts','examplestorage')]"

# a real sample
"[resourceId(parameters('vnet_rg'),'Microsoft.Network/virtualNetworks/subnets', parameters('vnet_name'), 'AzureBastionSubnet')]"
```

In above sample, 

+ If you create/update the resource in same subscription, you don't need provide the subscription id in it. 
+ If you create/update the resource in same resource group ( `az <sub_command> --resource-group xxx` ), you don't need provide the resource group in it. 
+ For example, Search the resource type via Google with key word "Azure ARM template subnet", you get this page : https://docs.microsoft.com/en-us/azure/templates/microsoft.network/virtualnetworks/subnets?tabs=json , so the resource type for subnet is **Microsoft.Network/virtualNetworks/subnets**
+ If resource type have **n** sessions, then in resourceId(), you need provide **n-1** parameters after resource type.

### export template

Azure has feature to export exist resources into ARM template, it makes us easily to prepare the ARM template, but remember, it has a lot of hardcodes, in most case, you can't directly use it if you want to apply it to other subscriptions. 

you need adjust the templates, some are:

+ resource location

```
  "parameters": {
    "location": {
      "defaultValue": "[resourceGroup().location]",
      "type": "string"
    }
  },
  ...
    "resources": [
    {
      "type": "Microsoft.Network/networkSecurityGroups",
      "apiVersion": "2020-11-01",
      "name": "[variables('nsg_name')]",
      "location": "[parameters('location')]",
      "properties": {
        "securityRules": []
      }
    },
    ...
   ]
   
```

+ resource name - normally I replace with parameters or variables
+ 

### Use popular IDE editers

Use popular IDE editers, such as VS Code, can save you huge time to deal with the template json file

ref: https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/quickstart-create-templates-use-visual-studio-code?tabs=CLI

### Azure Templates (PREVIEW)

[Azure Templates](https://azure.microsoft.com/en-us/resources/videos/azure-templates-in-the-new-portal/) is new Azure managed service, still in PREVIEW status currently. At first read from its name, I think it would be matched to AWS Cloudformation template, but it is not.

it is closer to AWS Service Catalogs

+ It is ARM template
+ It supports to deploy ARM template to different subscriptions

But it has 

+ No version control, so you can't update the templates with incremental versions.
+ No records or logs to trace who used this template to deploy the resources internally. 
+ Still no State file or similar functions to record the deployed resources
+ Still no function to detect the configuration drift

So when you consider this service, be ready for the inconvenience.

### Azure Blueprint

[Azure Blueprint](https://docs.microsoft.com/en-us/azure/governance/blueprints/overview) is another type of ARM template

+ It supports versioning.
+ It supports to manage blueprint via management groups (I prefer this way) or subscriptions.
+ One blueprint can have multiple Artifacts

But 
+ **Resource groups in Blueprint is totally different concept to Azure Resource groups.** don't mix them.
+ It doesn't support **latest** version yet
+ Artifacts in Blueprint is ARM template, but Blueprint is not ARM template, so you have to manually combine the Artifact ARM template into blueprint if you do that via az cli

### Azure command line - az cli

I am Mac user, anything can be managed by command or SDK, I would like to deploy ARM template via az cli. So [install azure cli](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) is what you need

```
# for mac user
$ brew update && brew install azure-cli

$ az login

$ az account show

# if you want to switch to other scriptions
$ az account set -s <subscription_name>
```
Put this alias your default Shell profile

```
alias azi='az interactive'
```
So you can [run AZ CLI with interactive mode](https://docs.microsoft.com/en-us/cli/azure/interactive-azure-cli) to avoid to rememeber the sub-commands and options.

#### ARM template Dry-run with az cli

az cli support dry-run option ( **what-if** ) when you deploy ARM template, smartly use it.

Here I give you code for reference on how easily manage the ARM template by az cli

- create resource group if not exist
- have option to dry-run or apply the ARM template.

```
$ cat deploy.sh

#!/usr/bin/env bash

usage()
{
    echo "usage: $0 [resource_group_name] [location] [template_name] [template_file_name] [parameter_file_name] (plan|apply)"
}

if [ "$#" -eq "0" ]; then
  echo "missing parameters, exit ..."
  usage
  exit 1
fi

resourceGroup="$1"
location="$2"
name="$3"
templateFile="$4"
parameterFile="$5"
dryRun="${6:-plan}"

case ${dryRun} in
  plan)
    echo "ARM Dry run"
    AZCLI="az deployment group what-if"
    ;;
  apply)
    echo "ARM Deployment"
    AZCLI="az deployment group create"
    ;;
  *)
    echo "wrong option, exit ..."
    usage
    exit 1
esac


resourceGroupStatus=$(az group list --query "[?name=='${resourceGroup}']" --output tsv)

if [ -z "${resourceGroupStatus}" ]; then
  az group create --name ${resourceGroup} --location ${location}
fi

${AZCLI} --resource-group "${resourceGroup}" \
  --name "${name}" --template-file "${templateFile}" --parameters @${parameterFile}
```
### Azure resource group

Azure resource group is the important difference if compare with AWS.

+ Recommend to create ARM template in a new resource group always. One template in one resource group.
+ Recommand to use template name as resource name, so you can easily match them.
+ You can easily clean them by delete resource group, more then delete them one by one

### Contributing

* Update [README.md](README.md)
* install [doctoc](https://github.com/thlorenz/doctoc)

```
sudo npm install -g doctoc
```

* update README

```
doctoc --github README.md
```
* commit the update and raise pull request for reviewing.
