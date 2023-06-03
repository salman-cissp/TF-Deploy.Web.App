# Deploy ASP .NET Core Web App using Terraform via GitOps
<br>
Deploy an ASP .Net Core app to Azure using Terraform and deploying it automatically using the GitOps workflow. An opensource tool NubesGen is used to automatically create the terraform scripts and gitops workflow<br><br>
Each time you an env-* branch is created in Git, a new environment will be created.<br>
That environment is an Azure resource group, containing all the resources configured with Terraform. When that environment is created, and each time 'git push' is executed to that branch, two things happen:<br><br>

The GitHub Action will apply the current Terraform configuration, so that your Azure resource group is synchronized with the configuration store in Git.<br><br>
The GitHub Action will then package and deploy the code stored in the Git branch, so that code runs on the infrastructure that was configured in the previous step.<br>

![TF-DeployWebApp](https://github.com/salman-cissp/TF-Deploy.Web.App/assets/134168108/78a358bb-f7ba-4635-9c2f-f85cc7661bba)

- Load the app in VS Code<br><br>
![image](https://github.com/salman-cissp/TF-Deploy.Web.App/assets/134168108/6d1b08e7-c45f-4ad1-8528-fee83fa2a7a1)

- Create github repo, and push the app to the repo<br><br>
```
git init
git add .  
git commit -m "first commit"
git remote add origin https://github.com/irtiash/aspnetcore.git
git branch -M main
git push -u origin main
```
- Set up GitOps with NubesGen by running the script.<br><br>
This script sets up a GitOps workflow for an Azure infrastructure using Terraform. It creates an Azure Resource Group, a Storage Account, and a Blob Container to store Terraform's remote state. It also creates a Virtual Network and secures the Storage Account in it. Additionally, it creates a service principal and sets up two GitHub secrets for authentication purposes.

```
RESOURCE_GROUP_NAME=rg-terraform-001
# The location of the resource group. For example `eastus`.
LOCATION=eastus
# The storage account (inside the resource group) used by Terraform to store its remote state.
TF_STORAGE_ACCOUNT=st$RANDOM$RANDOM$RANDOM$RANDOM
# The container name (inside the storage account) used by Terraform to store its remote state.
CONTAINER_NAME=tfstate
#####
# Execute the following commands to set up GitOps.
#####
# Create a new Azure Resource Group
az group create --name $RESOURCE_GROUP_NAME --location $LOCATION
# Create the Storage Account
az storage account create --resource-group $RESOURCE_GROUP_NAME --name $TF_STORAGE_ACCOUNT --sku Standard_LRS --allow-blob-public-access false --encryption-services blob
# Get the Storage Account key
ACCOUNT_KEY=$(az storage account keys list --resource-group $RESOURCE_GROUP_NAME --account-name $TF_STORAGE_ACCOUNT --query '[0].value' -o tsv)
# Create a Blob Container in the Storage Account
az storage container create --name $CONTAINER_NAME --account-name $TF_STORAGE_ACCOUNT --account-key $ACCOUNT_KEY
# Create a Virtual Network
VNET=vnet-$TF_STORAGE_ACCOUNT
SUBNET=snet-$TF_STORAGE_ACCOUNT
az network vnet create --resource-group $RESOURCE_GROUP_NAME --name $VNET --subnet-name $SUBNET
az network vnet subnet update --resource-group $RESOURCE_GROUP_NAME --name $SUBNET --vnet-name $VNET --service-endpoints "Microsoft.Storage"
# Secure the storage account in the Virtual Network
az storage account network-rule add  --resource-group $RESOURCE_GROUP_NAME --account-name $TF_STORAGE_ACCOUNT --vnet-name $VNET --subnet $SUBNET
az storage account update  --resource-group $RESOURCE_GROUP_NAME --name $TF_STORAGE_ACCOUNT --default-action Deny --bypass None
# Get the subscription ID
SUBSCRIPTION_ID=$(az account show --query id --output tsv --only-show-errors)
# Create a service principal
SERVICE_PRINCIPAL=$(az ad sp create-for-rbac --role="Contributor" --scopes="/subscriptions/$SUBSCRIPTION_ID" --sdk-auth --only-show-errors)
# Get the current GitHub remote repository
github.com/irtiash/aspnetcore=$(git config --get remote.origin.url)
# Set the two GitHub secrets
gh secret set AZURE_CREDENTIALS -b "$SERVICE_PRINCIPAL" -R github.com/irtiash/aspnetcore && gh secret set TF_STORAGE_ACCOUNT -b "$TF_STORAGE_ACCOUNT" -R github.com/irtiash/aspnetcore
```
<br><br>
- Check that the resources have created in Azure<br><br>
![image](https://github.com/salman-cissp/TF-Deploy.Web.App/assets/134168108/bb584044-c989-48ec-8249-2e9b7b874733)

- Generate a NubesGen configuration. It adds a terraform and github workflow folder<br><br>
` curl "https://nubesgen.com/demo.tgz?runtime=dotnet&application=app_service.standard&gitops=true" | tar -xzvf - `<br>

![image](https://github.com/salman-cissp/TF-Deploy.Web.App/assets/134168108/aee200f5-8f94-453a-b9be-61cafc23040a)


- Create a new branch called 'env-dev', and push the code to it:<br><br>
```
git checkout -b env-dev
git add .
git commit -m 'Configure GitOps with NubesGen'
git push --set-upstream origin env-dev
```
- Go to the GitHub project, and check that the GitHub Action is running. <br><br>
![image](https://github.com/salman-cissp/TF-Deploy.Web.App/assets/134168108/6a3c5a92-f723-4e3d-974b-9352860306f9)

- Check the resources in Azure<br><br>
![image](https://github.com/salman-cissp/TF-Deploy.Web.App/assets/134168108/be63903f-ced5-4385-b99d-a0b59e5f634a)

- Check the deployed app URL<br><br>
![image](https://github.com/salman-cissp/TF-Deploy.Web.App/assets/134168108/d6fbd740-ef93-4727-bd61-387443f37dff)

- Check if the app has been deployed<br><br>
![image](https://github.com/salman-cissp/TF-Deploy.Web.App/assets/134168108/5b73de2b-d05e-439b-9d29-11989e502475)

## Resources
- [Using .NET with NubesGen](https://docs.nubesgen.com/runtimes/dot-net/)


