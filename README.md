# Azure ARM Template Best Practices

[Azure ARM template](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/overview) best practices for Mac/Linux users

I am not fan of Microsoft **Powershell**, so this document is for Mac/Linux users.

### Official ARM template best practices

Go through Azure official ARM template best practices first

https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/best-practices

### ARM template is not immutable, not IaC.

This is important concept for any thing else. Azure ARM template is not **immutable**, they are not [IaC (infrastructure as code)](https://en.wikipedia.org/wiki/Infrastructure_as_code) at all.

I worked on Hashicopy Terraform and AWS Cloudformation template a lot. So when I worked on deploying Azure resources with codes this year, I realized this is a disaster design by someone in Azure, who doesn't have any concepts on how Iac and Immutable are important for developer or DevOps.

The strange designs I found until now:

+ ARM template has no state file or similar function, so it doesn't care what resources you already have, it just overrides the setting. So that means, 
  + If the resource is not exist, it will create that resource with the setting you put in ARM template. 
  + If the resource is exist, it just overrides the setting, no care what's the original setting on it
  + If you clean some resources and apply this template again, ARM template doesn't delete the cleaned resources. 
  + If you accidentally apply a totally different template, **but with same template name**, it would not stop you, just create or update the resources in template.
  + There is no way to reference the resource from output in this ARM template by another ARM template.
+ Its API is slow, sometime takes about 15 minutes to update some configurations. For example, a Virtual Network Gateway. For example, update a resource's tags, still take that long time, because it overrides the whole setting, not just update tags on that resources.
+ the setting in your resources are drafting when the ARM template is still applying. Sometime, this will surprise you. For example, when you apply an update on connection between Virtual Hub/WAN to ExpressRoute circuit. the connection is disappeared in the middle of ARM template updating.  

### resourceId

[resourceId](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/template-functions-resource#resourceid) in ARM template is the key concept you need understand. 

```
# resourceId([subscription_id], [resource_group_name], [resource_type], resource #1, resource #2, and so on if required)
"[resourceId('xxxxxxxx-xxxx-xxxx-xxxx-xxxxxxxxxxxx', 'otherResourceGroup', 'Microsoft.Storage/storageAccounts','examplestorage')]"

# a real sample
"[resourceId(parameters('vnet_rg'),'Microsoft.Network/virtualNetworks/subnets', parameters('vnet_name'), 'AzureBastionSubnet')]"
```

In above sample, 

+ If you create/update the resource in same subscription, you don't need provide the subscription in it. 
+ If you create/update the resource in same resource group ( `az <sub_command> --resource-group xxx` ), you don't need provide the resource group in it. 
+ Search the resource type via Google with key word "Azure ARM template subnet", you get this page : https://docs.microsoft.com/en-us/azure/templates/microsoft.network/virtualnetworks/subnets?tabs=json , so the resource type for subnet is **Microsoft.Network/virtualNetworks/subnets**
+ If resource type have n sessions, then in resourceId(), you need provide n-1 parameters.

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

### Azure templates (PREVIEW)

[ARM templates](https://docs.microsoft.com/en-us/azure/azure-resource-manager/templates/overview) is new Azure managed service, still in PREVIEW status currently. At first read from its name, I think it would be matched to AWS Cloudformation template, but it is not. 

it is closer to AWS Service Catalogs

+ It is ARM template
+ It supports to deploy ARM template to different subscriptions

But it has 

+ No version control, so you can't update the templates with incremental versions.
+ No records or logs to trace who used this template to deploy the resources internally. 
+ Still no State file or similar functions to record the deployed resources
+ No function to detect the configuration draft

So when you consider this service, be ready for the inconvenience.

### Azure Blueprint

[Azure Blueprint](https://docs.microsoft.com/en-us/azure/governance/blueprints/overview) is another type of ARM template

+ it supports versioning.
+ it supports to manage blueprint via management groups (I prefer this way) or subscriptions.
+ One blueprint can have multiple Artifacts

But 
+ **Resource groups in Blueprint is totally different concept to Azure Resource groups.** don't mix them.
+ it doesn't support **latest** version yet
+ Artifacts in Blueprint is ARM template, but Blueprint is not ARM template, so you have to manually combine the Artifact ARM template into blueprint if you do that via az cli

### Azure command line - az cli

I am Mac user, anything can be managed by command or SDK, I would like to do that. So [install azure cli](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) is what you need

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




