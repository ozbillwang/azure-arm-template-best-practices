# Azure ARM Template Best Practices

Azure ARM template best practices for Mac/Linux users

### ARM template is not immutable, not IaC.

This is important concept for any thing else. Azure ARM template is not immutable, they are not [IaC (infrastructure as code)](https://en.wikipedia.org/wiki/Infrastructure_as_code) at all.

I worked on Hashicopy Terraform and AWS Cloudformation template a lot. So when I worked on deploying Azure resource with codes this year, I realized this is a disaster design by someone in Azure, who doesn't have any concepts how Iac and Immutable is important for developer or DevOps.

The strange design I found until now:

+ ARM template has no state file, so it doesn't care what resources you already have, it just overrides the setting. So that means, 
- If the resource is not exist, it will create that resource with the setting you put in ARM template. 
- If the resource is exist, it just overrides the setting, no care what's the original setting on it

+ If you clean some resources and apply this template again, ARM template doesn't delete the cleaned resources. 
+ Its API is slow, sometime takes about 15 minutes to update some configurations. For example, a Virtual Network Gateway. For example, update a resource's tags, still take that long time, because it overrides the whole setting, not just update tags on that resources.
+ the setting in your resources are drafting when the ARM template is still applying. Sometime, this will surprise you. For example, when you apply an update on connection between Virtual Hub/WAN to ExpressRoute circuit. the connection is disappeared in the middle of ARM template updating.  


### Azure templates (PREVIEW)

[ARM templates] is new Azure managed service, still in PREVIEW status currently. At first read from its name, I think it would be matched to AWS Cloudformation template, but it is not. 

it is more close to AWS Service Catalogs

+ It is ARM template
+ It supports to deploy template to different subscriptions

But it has 

+ No version control, so you can't update the templates with incremental versions.
+ No records or logs to trace who used this template to deploy the resources internally. 
+ Still no State file or similar functions to record the deployed resources
+ No function to detect the configuration draft

### Azure Blueprint



