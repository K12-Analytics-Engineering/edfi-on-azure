# Ed-Fi on Azure
The Azure CLI commands below will deploy the following Ed-Fi components on Azure.

* Ed-Fi API and ODS Suite 3 v5.3
* Ed-Fi Admin App v2.3.2
* TPDM Core v1.1.1

![Azure](/img/architecture.png)

## Costs
| Component             | Azure product | Configuration                                   | Yearly cost            |
| --------------------- | -------------------- | ----------------------------------------------- | ---------------------- |
| Ed-Fi ODS             | Azure Database for PostgreSQL            | PostgreSQL 11,<br>2 vCPUs,<br>4 GB of memory,<br> Flexible Server,<br>Burstable      | $720 / year              |
| Ed-Fi API             | Azure App Services            | --    | -- |
| Ed-Fi Admin App       | Azure App Services            | --     | -- |

## Prerequisites
Use the Bash environment in [Azure Cloud Shell](https://docs.microsoft.com/en-us/azure/cloud-shell/quickstart). For more information, see [Azure Cloud Shell Quickstart - Bash](https://docs.microsoft.com/en-us/azure/cloud-shell/quickstart). If you prefer to run CLI reference commands locally, [install](https://docs.microsoft.com/en-us/cli/azure/install-azure-cli) the Azure CLI

```sh
# MacOS with brew
brew update && brew install azure-cli;
```

## Deploy
```sh
az login;
az account set --subscription "XXXXXXXXXX"; # DEV TODO: update azure subscription id
az configure --defaults location=centralus;
```

### Azure Database for PostgreSQL - Flexible Server
Azure Database for PostgreSQL - Flexible Server is a fully managed PostgreSQL database as a service offering. The command below will also create a resource group, virtual network, subnet, and private dns zone. After the command run and the PostgreSQL instance has been created, the CLI will output the password for the *postgres* user.
```sh
# create sql instance
az postgres flexible-server create \
    --location centralus \
    --resource-group analytics \
    --name edfi-ods \
    --database-name EdFi_Admin \
    --vnet virtual-network \
    --subnet sql-instances \
    --admin-user postgres \
    --sku-name Standard_B2s \
    --tier Burstable \
    --storage-size 32 \
    --version 11 \
    --high-availability Disabled;

# enable pgcrypto extension
az postgres flexible-server parameter set \
    --resource-group analytics  \
    --server-name edfi-ods \
    --name azure.extensions \
    --value PGCRYPTO;

# create databases
az postgres flexible-server db create \
    --resource-group analytics \
    --server-name edfi-ods \
    --database-name EdFi_Security;

az postgres flexible-server db create \
    --resource-group analytics \
    --server-name edfi-ods \
    --database-name EdFi_Ods_2023;

az postgres flexible-server db create \
    --resource-group analytics \
    --server-name edfi-ods \
    --database-name EdFi_Ods_2022;

az postgres flexible-server db create \
    --resource-group analytics \
    --server-name edfi-ods \
    --database-name EdFi_Ods_2021;
```

### Ubuntu jump server
The PostgreSQL instance created above is a private instance and thus cannot be accessed externally. It does not have a public IP. In this section you will create an Ubuntu VM that is able to connect to the PostgreSQL instance. This will allow you to import the Ed-Fi database templates. This VM is not included in the pricing above because we will shutdown the VM after the import has completed.

```sh
# virtual network subnet for vms
az network vnet subnet create \
    --name vm-subnet \
    --vnet-name virtual-network \
    --resource-group analytics \
    --address-prefixes 10.0.1.0/24;

# create ubuntu vm
# DEV TODO: update admin password
az vm create \
  --resource-group analytics \
  --location centralus \
  --name ubuntu \
  --image UbuntuLTS \
  --accept-term \
  --vnet-name virtual-network \
  --subnet vm-subnet \
  --admin-username azureuser \
  --admin-password a1z3u2r@ePa55 \
  --public-ip-sku Standard \
  --size Standard_B1ls;
```

Now that your VM has been created, you will SSH into it and run various `psql` import commands.

```sh
# connect to vm
ssh azureuser@XXX.XXX.XXX.XXX; # DEV TODO: update ip to vms newly created public ip

# install dependencies
sudo apt install postgresql-client-common postgresql-client zip;

# clone this git repo to vm
git clone https://github.com/K12-Analytics-Engineering/edfi-on-azure.git;

# download and import ed-fi db templates
cd edfi-azure;
bash init.sh
bash import-ods-data.sh <POSTGRESPASSWORD> # DEV TODO: replace with postgres password

# shutdown vm
sudo shutdown;
```


### Ed-Fi API
Before you deploy the Ed-Fi API on Azure App Services, you will first build and a push a Docker image to Azure Container Registry.
```sh
# create container registry
az acr create \
    --name edfialliance \
    --resource-group analytics \
    --sku Basic \
    --admin-enabled true;

az acr build --registry edfialliance --image api edfi-api/.;

# app services
az network vnet subnet create \
    --name app-services-subnet \
    --vnet-name virtual-network \
    --resource-group analytics \
    --address-prefixes 10.0.2.0/24 \
    --delegations Microsoft.Web/serverFarms;

az appservice plan create \
    --name edfi \
    --resource-group analytics \
     --is-linux \
     --sku P1V2;

az webapp create \
    --resource-group analytics \
    --plan edfi \
    --name edfi-api \
    --deployment-container-image-name edfialliance.azurecr.io/api:latest \
    --vnet virtual-network \
    --subnet app-services-subnet;

az webapp config appsettings set \
    --resource-group analytics \
    --name edfi-api \
    --settings DB_PASS="XXXXXXXXX"; # DEV TODO: replace with postgres password

az webapp restart \
    --name edfi-api \
    --resource-group analytics;
```


### Ed-Fi Admin App
Before you deploy the Ed-Fi Admin App on Azure App Services, you will first build and a push a Docker image to Azure Container Registry.
```sh
az acr build --registry edfialliance --image adminapp edfi-admin-app/.;

az webapp create \
    --resource-group analytics \
    --plan edfi \
    --name edfi-admin-app \
    --deployment-container-image-name edfialliance.azurecr.io/adminapp:latest \
    --vnet virtual-network \
    --subnet app-services-subnet;

az webapp config appsettings set \
    --resource-group analytics \
    --name edfi-admin-app \
    --settings DB_PASS="XXXXXXXXX"; # DEV TODO: replace with postgres password

az webapp config appsettings set \
    --resource-group analytics \
    --name edfi-admin-app \
    --settings ENCRYPTION_KEY=$(/usr/bin/openssl rand -base64 32);

az webapp config appsettings set \
    --resource-group analytics \
    --name edfi-admin-app \
    --settings API_URL="https://edfi-api.azurewebsites.net"; # DEV TODO: replace with edfi api url (ie. https://edfi-api.azurewebsites.net)

az webapp restart \
    --name edfi-admin-app \
    --resource-group analytics;
```
