# Ed-Fi on Azure
The Azure CLI commands below will deploy the Ed-Fi Platform on Azure. This installer uses the Azure CLI in place of infrastructure as code solutions such as Terraform because the purpose of the installer is to explain each step along the way. Take your time and read through each section to ensure you understand each component of the Ed-Fi platform on Azure.

* Ed-Fi API and ODS Suite 3 v5.3
* Ed-Fi Admin App v2.3.2
* TPDM Core v1.1.1

![Azure](/img/architecture.png)

## Costs
| Component             | Azure product | Configuration                                   | Yearly cost            |
| --------------------- | -------------------- | ----------------------------------------------- | ---------------------- |
| Ed-Fi ODS             | Azure Database for PostgreSQL            | PostgreSQL 11,<br>2 vCPUs,<br>4 GB of memory,<br> Flexible Server,<br>Burstable      | $720 / year              |
| Ed-Fi API             | Azure App Services                       | 2 vCPU,<br>3.5 GB of memory                                                         | $300 / year |
| Ed-Fi Admin App       | Azure App Services                       | 2 vCPU,<br>3.5 GB of memory                                                         | $300 / year |

## Prerequisites
There are several ways to run through this installer. You can use the Bash environment in [Azure Cloud Shell](https://docs.microsoft.com/en-us/azure/cloud-shell/quickstart). For more information, see [Azure Cloud Shell Quickstart - Bash](https://docs.microsoft.com/en-us/azure/cloud-shell/quickstart). If you prefer to run CLI reference commands locally, you can [install](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) the Azure CLI.

If you are on MacOS, the Azure CLI can be installed via [Homebrew](https://brew.sh).
```sh
brew update && brew install azure-cli;
```

# Table of Contents  
* [Environment variables](#environment-variables)
* [Ed-Fi ODS](#ed-fi-ods)


## Environment variables
Throughout the installer, you will be creating resources in Azure. Those resources are given unique ids which you will store in various environment variables to be used in future commands.

Run the commands below to create an environment variable that is set to your Azure subscription ID as well as authenticate and configure the Azure CLI.

```sh
export AZ_SUBSCRIPTION_ID=XXXXXXXXXX;

az login;
az account set --subscription $AZ_SUBSCRIPTION_ID;
az configure --defaults location=centralus;
```

## Ed-Fi ODS
The Ed-Fi Alliance supports PostgreSQL and SQL Server for the Ed-Fi ODS. This installer uses PostgreSQL via a fully managed PostgreSQL database as a service offering, *Azure Database for PostgreSQL - Flexible Server*. Using a managed service allows us to look to Azure to handle things like security and software updates, automatic backups, and more.

Update the value below for your specific district and run the command to create a `AZ_PG_SERVER_NAME` environment variable.
```sh
export AZ_PG_SERVER_NAME="edfi-ods-grand-bend";
```

Running the command below to create your PostgreSQL instance will also create a bunch of other things in the background including a resource group, virtual network, subnet, and private dns zone. After the command runs and the PostgreSQL instance has been created, the CLI will output the password for the *postgres* user.
```sh
az postgres flexible-server create \
    --location centralus \
    --resource-group analytics \
    --name $AZ_PG_SERVER_NAME \
    --database-name EdFi_Admin \
    --vnet virtual-network \
    --subnet sql-instances \
    --admin-user postgres \
    --sku-name Standard_B2s \
    --tier Burstable \
    --storage-size 32 \
    --version 11 \
    --high-availability Disabled;
```

Store the password for your `postgres` user as an environment variable.
```sh
export DB_PASS=XXXXXXXXXX;
```

Let's define a few of the resources you just created.

**Resource group:** A resource group is a container that holds related resources for an Azure solution. This sits under your Azure subscription. By running the command above, you now have a resource group, `analytics`, that will be comprised of all your Ed-Fi related resources.

**Virtual network:** An Azure Virtual Network (VNet) enables resources to securly communicate with each other and the internet. A VNet is similar to a traditional network that would exist on-premises, but this one is virtual.

**Subnet:** A subnet is a range of IP addresses in the virtual network.

**Private DNS:** A Domain Name System (DNS) is responsible for translating (or resolving) a service name to an IP address. The command above created a private DNS zone to allow resources on your VNet to talk to each other.

Run the command below to enable pgcrypto, a module that provides cryptographic functions for PostgreSQL, and is used by Ed-Fi.
```sh
az postgres flexible-server parameter set \
    --resource-group analytics  \
    --server-name $AZ_PG_SERVER_NAME \
    --name azure.extensions \
    --value PGCRYPTO;
```

When you created your PostgreSQL instance, you created a database, `EdFi_Admin`. We are now going to create the databases listed below.

* EdFi_Security
* EdFi_Ods_2021
* EdFi_Ods_2022
* EdFi_Ods_2023

```sh
az postgres flexible-server db create \
    --resource-group analytics \
    --server-name $AZ_PG_SERVER_NAME \
    --database-name EdFi_Security;
```
```sh
az postgres flexible-server db create \
    --resource-group analytics \
    --server-name $AZ_PG_SERVER_NAME \
    --database-name EdFi_Ods_2023;
```
```sh
az postgres flexible-server db create \
    --resource-group analytics \
    --server-name $AZ_PG_SERVER_NAME \
    --database-name EdFi_Ods_2022;
```
```sh
az postgres flexible-server db create \
    --resource-group analytics \
    --server-name $AZ_PG_SERVER_NAME \
    --database-name EdFi_Ods_2021;
```

### Ubuntu jump server
The PostgreSQL instance created above is a private instance and cannot be directly accessed externally. It does not have a public IP. In this section you will create an Ubuntu VM that is able to connect to the PostgreSQL instance. You will then SSH into the VM to import the Ed-Fi database templates. You are also able to create a SSH tunnel to forward ports appropriate should you want to connect a SQL IDE on your local machine to the PostgreSQL instance.

This VM is not included in the pricing above because we will shutdown the VM after the import has completed.

Run the command below to create a subnet that will be used by the VM.
```sh
# virtual network subnet for vms
az network vnet subnet create \
    --name vm-subnet \
    --vnet-name virtual-network \
    --resource-group analytics \
    --address-prefixes 10.0.1.0/24;
```

Update the value for `VM_PASS` and run the command below. This env variable will be used when setting the `azureuser` password on your VM.
```sh
export VM_PASS=XXXXXXXXXX;
```
```sh
az vm create \
  --resource-group analytics \
  --location centralus \
  --name ubuntu \
  --image UbuntuLTS \
  --accept-term \
  --vnet-name virtual-network \
  --subnet vm-subnet \
  --admin-username azureuser \
  --admin-password $VM_PASS \
  --public-ip-sku Standard \
  --size Standard_B1ls;
```

Now that your VM has been created, you will SSH into it and run various `psql` import commands. After the command above ran, `publicIpAddress` was printed out to the console. Use that IP address in the command below.
```sh
ssh azureuser@XXX.XXX.XXX.XXX; # DEV TODO: update ip to vms newly created public ip
```

Run the command below to install necessary dependencies.
```sh
sudo apt install postgresql-client-common postgresql-client zip;
```

Run the command below to clone this git repo into the VM.
```sh
git clone https://github.com/K12-Analytics-Engineering/edfi-on-azure.git;
```

Change into the `edfi-on-azure` directory.
```sh
cd edfi-on-azure;
```

Download the necesasry database backups.
```sh
bash init.sh;
```

Run the command below to seed your various databases. Replace `<POSTGRESPASSWORD>` with your PostgreSQL password. Replace `<POSTGRESQL FULL SERVER NAME>` with your server name. This will be `AZ_PG_SERVER_NAME` along with *.postgres.database.azure.com* (ie. `edfi-ods-grand-bend.postgres.database.azure.com`)
```sh
bash import-ods-data.sh <POSTGRESPASSWORD> <POSTGRESQL FULL SERVER NAME>
```

Run the command below to shutdown your VM.
```sh
sudo shutdown;
```


### Ed-Fi API
Before you deploy the Ed-Fi API on Azure App Services, you will first build and a push a Docker image to Azure Container Registry.

Run the command below to create a `LEA_NAME` variable that will be used throughout the next section. **Do not* use any hyphens, underscores, or special characters.
```sh
export LEA_NAME="grandbend"; # DEV TODO: update name
```

Run the command below to create a Key Vault. Azure Key Vault provies us with a secure place to store sensitive information such as passwords. Information stored in Key Vault is entrypted at rest and in transit.
```sh
az keyvault create \
    --name "vault-${LEA_NAME}" \
    --resource-group analytics;
```

Run the command below to store your `postgres` user password as a secret in Key Vault.
```sh
az keyvault secret set \
    --name db-pass \
    --vault-name "vault-${LEA_NAME}" \
    --value $DB_PASS;
```

Run the command below to create an Azure Container Registry. Container Registry is a managed registry service for storing our container images.
```sh
az acr create \
    --name $LEA_NAME \
    --resource-group analytics \
    --sku Basic \
    --admin-enabled true;
```

Run the command below to build an image of the Ed-Fi API and push it to your container registry.
```sh
az acr build --registry $LEA_NAME --image api edfi-api/.;
```

Run the command below to create a subnet that your Ed-Fi API will run within.
```sh
az network vnet subnet create \
    --name app-services-subnet \
    --vnet-name virtual-network \
    --resource-group analytics \
    --address-prefixes 10.0.2.0/24 \
    --delegations Microsoft.Web/serverFarms;
```

Run the command below to create your Azure App Service plan. App Service is a fully managed platform for building web applications.
```sh
az appservice plan create \
    --name edfi \
    --resource-group analytics \
     --is-linux \
     --sku B2;
```

Run the command below to create your App Service service.
```sh
az webapp create \
    --resource-group analytics \
    --plan edfi \
    --name "edfi-api-${LEA_NAME}" \
    --deployment-container-image-name "${LEA_NAME}.azurecr.io/api:latest" \
    --vnet virtual-network \
    --subnet app-services-subnet \
    --https-only true \
    --assign-identity '[system]';
```

Run the command below to find the unique id of the newly created App Service.
```sh
az webapp identity show \
    --name "edfi-api-${LEA_NAME}" \
    --resource-group analytics \
    --query principalId;
```

Enter the output from above for the object-id below. Do not include the surrounding quotes.
```sh
az keyvault set-policy \
    --name "vault-${LEA_NAME}" \
    --object-id "XXXXXXXXX" \
    --secret-permissions get list;
```

Run the command below to find the id of your `db-pass` secret.
```sh
az keyvault secret show \
    --name db-pass \
    --vault-name "vault-${LEA_NAME}" \
    --query id;
```

Replace `XXXXXXX` with the output of the command above. Do not include the surrounding quotes. This creates an environment variable, `DB_HOST`, in your App Service that points to your Azure Key Vault secret.
```sh
az webapp config appsettings set \
    --resource-group analytics \
    --name "edfi-api-${LEA_NAME}" \
    --settings DB_PASS="@Microsoft.KeyVault(SecretUri=XXXXXXX)";
```

Run the command below. This creates an environment variable, `DB_HOST`, that contains the value of your PostgreSQL instance.
```sh
az webapp config appsettings set \
    --resource-group analytics \
    --name "edfi-api-${LEA_NAME}" \
    --settings DB_HOST="${AZ_PG_SERVER_NAME}.postgres.database.azure.com";
```

Run the command below to restart your App Service allowing the newly created environment variables to take effect.
```sh
az webapp restart \
    --name "edfi-api-${LEA_NAME}" \
    --resource-group analytics;
```

Your Ed-Fi API has been deployed! Head to App Services in your Azure portal to locate your URL.


### Ed-Fi Admin App
Before you deploy the Ed-Fi Admin App on Azure App Services, you will first build and a push a Docker image to Azure Container Registry.

Run the command below to build an image of the Ed-Fi Admin App and push it to your container registry.
```sh
az acr build --registry $LEA_NAME --image adminapp edfi-admin-app/.;
```

Run the command below to create your App Service service.
```sh
az webapp create \
    --resource-group analytics \
    --plan edfi \
    --name "edfi-admin-app-${LEA_NAME}" \
    --deployment-container-image-name "${LEA_NAME}.azurecr.io/adminapp:latest" \
    --vnet virtual-network \
    --subnet app-services-subnet \
    --https-only true \
    --assign-identity '[system]';
```

Run the command below to find the unique id of the newly created App Service.
```sh
az webapp identity show \
    --name "edfi-admin-app-${LEA_NAME}" \
    --resource-group analytics \
    --query principalId;
```

Enter the output from above for the object-id below. Do not include the surrounding quotes.
```sh
az keyvault set-policy \
    --name "vault-${LEA_NAME}" \
    --object-id "XXXXXXXXX" \
    --secret-permissions get list;
```

Run the command below to find the id of your `db-pass` secret.
```sh
az keyvault secret show \
    --name db-pass \
    --vault-name "vault-${LEA_NAME}" \
    --query id;
```

Replace `XXXXXXX` with the output of the command above. Do not include the surrounding quotes. This creates an environment variable, `DB_HOST`, in your App Service that points to your Azure Key Vault secret.
```sh
az webapp config appsettings set \
    --resource-group analytics \
    --name "edfi-admin-app-${LEA_NAME}" \
    --settings DB_PASS="@Microsoft.KeyVault(SecretUri=XXXXXXX)";
```

Run the command below. This creates an environment variable, `DB_HOST`, that contains the value of your PostgreSQL instance.
```sh
az webapp config appsettings set \
    --resource-group analytics \
    --name "edfi-admin-app-${LEA_NAME}" \
    --settings DB_HOST="${AZ_PG_SERVER_NAME}.postgres.database.azure.com";
```

Run the command below. This creates an environment variable, `ENCRYPTION_KEY`, that will be used by the Admin App.
```sh
az webapp config appsettings set \
    --resource-group analytics \
    --name "edfi-admin-app-${LEA_NAME}" \
    --settings ENCRYPTION_KEY=$(/usr/bin/openssl rand -base64 32);
```

Run the command below. This creates an environment variable, `API_URL`
```sh
az webapp config appsettings set \
    --resource-group analytics \
    --name "edfi-admin-app-${LEA_NAME}" \
    --settings API_URL="https://edfi-api-${LEA_NAME}.azurewebsites.net";
```

Run the command below to restart your App Service allowing the newly created environment variables to take effect.
```sh
az webapp restart \
    --name "edfi-admin-app-${LEA_NAME}" \
    --resource-group analytics;
```

Your Admin App has been deployed!
