# Spring PetClinic Sample Application [![Build Status](https://github.com/spring-projects/spring-petclinic/actions/workflows/maven-build.yml/badge.svg)](https://github.com/spring-projects/spring-petclinic/actions/workflows/maven-build.yml)

This repo is cloned from the classic Spring PetClinic sample that can be found [here](https://github.com/spring-projects/spring-petclinic).

The purpose of this demo is to demonstrate how to use Azure Container Registry (ACR) to build a Docker image from a Spring Boot application without having Docker desktop installed in the developer box. This Docker image can be deployed in any Docker compatible hosting environment, such as Azure App Service, Azure Functions, Azure Kubernetes Services or Azure Container Apps. This image can be used in other environments out of Azure, for instance in a local environment or any other cloud provider.
For demo purposes it will be deployed in Azure Container Apps.

It can be deployed by executing the commands in [azure.azcli](azure.azcli) script file.
```bash
LOCATION=[SET YOUR PREFERED LOCATION HERE]
RESOURCE_GROUP=[NAME YOUR RESOURCE GROUP]
ACR_NAME=[NAME YOUR AZURE CONTAINER REGISTRY]
CONTAINERAPPS_ENVIRONMENT=[NAME YOUR CONTAINER APPS ENVIRONMENT]
CONTAINERAPPS_NAME=[NAME YOUR CONTAINER APP]
# create a resource group to hold resources for the demo
az group create --location $LOCATION --name $RESOURCE_GROUP
# create an Azure Container Registry (ACR) to hold the images for the demo
az acr create --resource-group $RESOURCE_GROUP --name $ACR_NAME --sku Standard --location $LOCATION

# register container apps extension
az extension add --name containerapp --upgrade
# register Microsoft.App namespace provider
az provider register --namespace Microsoft.App
# create an azure container app environment
az containerapp env create \
    --name $CONTAINERAPPS_ENVIRONMENT \
    --resource-group $RESOURCE_GROUP \
    --location $LOCATION

# Create a user managed identity and assign AcrPull role on the ACR. By creating an user managed identity it is possible to assign AcrPull role before starting the deployment
USER_IDENTITY=$(az identity create -g $RESOURCE_GROUP -n $CONTAINERAPPS_NAME --location $LOCATION --query clientId -o tsv)
ACR_RESOURCEID=$(az acr show --name $ACR_NAME --resource-group $RESOURCE_GROUP --query "id" --output tsv)
az role assignment create \
    --assignee "$USER_IDENTITY" --role AcrPull --scope "$ACR_RESOURCEID"

# Build the application and hence build the docker image in the ACR
mvn package -PbuildAcr -DRESOURCE_GROUP=$RESOURCE_GROUP -DACR_NAME=$ACR_NAME

# Create the container app
az containerapp create \
    --name ${CONTAINERAPPS_NAME} \
    --resource-group $RESOURCE_GROUP \
    --environment $CONTAINERAPPS_ENVIRONMENT \
    --container-name spring-petclinic-container \
    --user-assigned ${CONTAINERAPPS_NAME} \
    --registry-server $ACR_NAME.azurecr.io \
    --image $ACR_NAME.azurecr.io/spring-petclinic:2.7.0-SNAPSHOT \
    --ingress external \
    --target-port 8080 \
    --cpu 1 \
    --memory 2 

```