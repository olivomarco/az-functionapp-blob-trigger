# Azure Function App for Python (Blob Trigger)

This repo contains a small Azure Function App written in Python that is triggered by a new blob in a storage account. The function app is deployed to Azure using the Azure CLI.

## How to use

Run following bash script:

```bash
RAND=$(( $RANDOM % 1000 + 1 ))
PREFIX=myprefix
PLAN="consumption"

az group create --name $PREFIX-rg --location westeurope
az storage account create --name sa$PREFIX$RAND --resource-group $PREFIX-rg --location westeurope --sku Standard_LRS --kind StorageV2
az storage container create --name "exports" --account-name sa$PREFIX$RAND --resource-group $PREFIX-rg

CONN_STRING=$(az storage account show-connection-string --name sa$PREFIX$RAND -g $PREFIX-rg --query 'connectionString' | jq -r)

az storage account create --name func$PREFIX$RAND --resource-group $PREFIX-rg --location westeurope --sku Standard_LRS --kind StorageV2

if [ "$PLAN" = "consumption" ]; then
    az functionapp create --name func-exp-$PREFIX$RAND -g $PREFIX-rg --storage-account func$PREFIX$RAND --consumption-plan-location westeurope --runtime python --runtime-version 3.9 --os-type linux
else
    az functionapp plan create --name $PREFIX-plan --resource-group $PREFIX-rg --location westeurope --sku $PLAN --is-linux
    az functionapp create --name func-exp-$PREFIX$RAND -g $PREFIX-rg --storage-account func$PREFIX$RAND --plan $PREFIX-plan --runtime python --runtime-version 3.9 --os-type linux
fi

az functionapp config appsettings set --name func-exp-$PREFIX$RAND -g $PREFIX-rg --settings exportstorage=$CONN_STRING

zip -r func.zip .
az functionapp deployment source config-zip --name func-exp-$PREFIX$RAND -g $PREFIX-rg --src ./func.zip
```

## How to test

Upload [sample.csv](/sample.csv) in the exports container of the first storage account created (the one named with the pattern `saXXXXXX`). The function app will be triggered and the file will be processed.

Please note that if you upload a file with a different extension or to a different container, the function will not be triggered.
