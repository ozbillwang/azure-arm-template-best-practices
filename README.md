# Azure ARM Template Best Practices

Azure ARM template best practices for Mac/Linux users

### ARM template is not immutable, not IaC.

This is important concept for any thing else. Azure ARM template is not **immutable**, they are not [IaC (infrastructure as code)](https://en.wikipedia.org/wiki/Infrastructure_as_code) at all.

I worked on Hashicopy Terraform and AWS Cloudformation template a lot. So when I worked on deploying Azure resource with codes this year, I realized this is a disaster design by someone in Azure, who doesn't have any concepts how Iac and Immutable is important for developer or DevOps.

The strange design I found until now:

+ ARM template has no state file, so it doesn't care what resources you already have, it just overrides the setting. So that means, 
- If the resource is not exist, it will create that resource with the setting you put in ARM template. 
- If the resource is exist, it just overrides the setting, no care what's the original setting on it

+ If you clean some resources and apply this template again, ARM template doesn't delete the cleaned resources. 
+ Its API is slow, sometime takes about 15 minutes to update some configurations. For example, a Virtual Network Gateway. For example, update a resource's tags, still take that long time, because it overrides the whole setting, not just update tags on that resources.
+ the setting in your resources are drafting when the ARM template is still applying. Sometime, this will surprise you. For example, when you apply an update on connection between Virtual Hub/WAN to ExpressRoute circuit. the connection is disappeared in the middle of ARM template updating.  


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

### Azure cli

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




