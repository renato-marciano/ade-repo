# Running through Quick starts with AZ CLI

[DEV CENTER QUICK STARTS](https://learn.microsoft.com/en-us/azure/deployment-environments/quickstart-create-and-configure-devcenter)

## Prereq

- az cli
- bash

## QUICK START 1

```bash
export RG=rg-ade-renato
export LOCATION=eastus
export KV=kv-ade-renato
export DEVC=devc-renato
export GITHUB_TOKEN= <repo access>
SUB_NAME= <subscription name>
export SUBID=$(az account show -n "${SUB_NAME}" --query id -o tsv)


az config set extension.use_dynamic_install=yes_without_prompt
az extension add --name devcenter
az login --use-device-code
az group create -n $RG -l $LOCATION
az configure --defaults group=$RG
az configure --defaults location=eastus
az devcenter admin --help

# available regions
# 'australiaeast,canadacentral,westeurope,japaneast,uksouth,eastus,eastus2,southcentralus,westus3'

az devcenter admin devcenter create -n $DEVC
az keyvault create -n $KV
az keyvault secret set --vault-name $KV --name GHPAT --value $GITHUB_TOKEN 
az devcenter admin devcenter update -n $DEVC --identity-type SystemAssigned
OID=$(az ad sp list --display-name $DEVC --query [].id -o tsv) 
az keyvault set-policy -n $KV --secret-permissions get --object-id $OID

mkdir ade-repo
cd ade-repo
git init

# Install gh cli
type -p curl >/dev/null || (sudo apt update && sudo apt install curl -y)
curl -fsSL https://cli.github.com/packages/githubcli-archive-keyring.gpg | sudo dd of=/usr/share/keyrings/githubcli-archive-keyring.gpg \
&& sudo chmod go+r /usr/share/keyrings/githubcli-archive-keyring.gpg \
&& echo "deb [arch=$(dpkg --print-architecture) signed-by=/usr/share/keyrings/githubcli-archive-keyring.gpg] https://cli.github.com/packages stable main" | sudo tee /etc/apt/sources.list.d/github-cli.list > /dev/null \
&& sudo apt update \
&& sudo apt install gh -y

# Create Repo
gh repo create ade-repo --private --source=. --remote=upstream
mkdir templates
cd templates
touch .gitkeep
git add . 
git commit -m "Added templ dir"
git push --set-upstream upstream main

export SECRETID=$(az keyvault secret show --vault-name $KV --name GHPAT --query id -o tsv)
echo
export CAT=cat-renato
export REPO= <repo url>

az devcenter admin catalog create --git-hub path="/templates" branch="main" secret-identifier=$SECRETID uri=$REPO -n $CAT -d $DEVC
az devcenter admin environment-type create -d $DEVC -n dev 
```

## QUICK START 2

```bash

az role assignment create --assignee $OID \
--role "Owner" \
--scope "/subscriptions/$SUBID"

DEVCID=$(az devcenter admin devcenter show -n $DEVC --query id -o tsv)
# Location needs to be same as devcenter 
az devcenter admin project create -n project-renato \
    --description "My first project." \
    --dev-center-id $DEVCID \

# Required for next for next command
az configure --defaults group=

# Add environment type for a project
ROID=$(az role definition list -n "Owner" --scope /subscriptions/$SUBID --query [].name -o tsv)

az devcenter admin project-environment-type create -n dev \
    --project project-renato \
    --identity-type "SystemAssigned" \
    --roles "{\"${ROID}\":{}}" \
    --deployment-target-id "/subscriptions/${SUBID}" \
    --status Enabled

az configure --defaults group=$RG

export MYOID=$(az ad signed-in-user show --query id -o tsv)

az role assignment create --assignee $MYOID \
--role "Deployment Environments User" \
--scope "/subscriptions/$SUBID"
```


## Quick start 3

First create azuredeploy.json and manifest.yaml. See examples in this repo.

```bash
# Sync catalog
az devcenter admin catalog sync --name "cat-renato" -d $DEVC

# First create parameters.json under /workspace to define params for ARM. Do not check in if it has secrets
# Warning: Output shows secrets. How to not do that?
az devcenter dev environment create --dev-center-name $DEVC \
    --project-name project-renato --environment-name dev --environment-type dev \
    --catalog-item-name "CatalogItem" --catalog-name cat-renato --param @/workspace/parameters.json
```

## Clean up

```bash
# This deletes all resources created from the environment
az devcenter dev environment delete --dev-center-name $DEVC --project-name project-renato --environment-name dev -y

az devcenter admin project delete -n project-renato -y 
az devcenter admin devcenter delete -n $DEVC -y 
```

## Extra Commands

```bash
# Extra commands - May not need?
MYOID=$(az ad signed-in-user show --query id -o tsv)
az role assignment create --assignee $MYOID \
--role "DevCenter Project Admin" \
--scope "/subscriptions/$SUBID"

az role assignment create --assignee $MYOID \
--role "DevCenter Dev Box User" \
--scope "/subscriptions/$SUBID"
```