# azure-apigateway-workshop
azure-apigateway-workshop https://docs.microsoft.com/en-gb/learn/modules/load-balance-web-traffic-with-application-gateway/3-exercise-create-web-sites

## Resource Group and Network
```shell

# create the resource group
az group create --name az-apigateway --location eastus

# create vnet and a subnet 
az network vnet create \
  --resource-group az-apigateway \
  --name vnet-vehicleAppVnet \
  --address-prefix 10.0.0.0/16 \
  --subnet-name subnet-webServerSubnet \
  --subnet-prefix 10.0.1.0/24
```

## Virtual Machines
```powershell

az vm list-sizes --location eastus --output table

az vm create \
  --resource-group az-apigateway \
  --name vm-webServer1 \
  --image UbuntuLTS \
  --admin-username azureuser \
  --generate-ssh-keys \
  --vnet-name vnet-vehicleAppVnet \
  --subnet subnet-webServerSubnet \
  --public-ip-address "" \
  --nsg "" \
  --custom-data module-files/scripts/vmconfig.sh \
  --size Standard_A1 \
  --no-wait
 
az vm create \
  --resource-group az-apigateway \
  --name vm-webServer2 \
  --image UbuntuLTS \
  --admin-username azureuser \
  --generate-ssh-keys \
  --vnet-name vnet-vehicleAppVnet \
  --subnet subnet-webServerSubnet \
  --public-ip-address "" \
  --nsg "" \
  --custom-data module-files/scripts/vmconfig.sh \
  --size Standard_A1 \
  --no-wait

az vm list \
  --resource-group az-apigateway \
  --show-details \
  --output table
```

## APP Service
```azurecli

APPSERVICE="licenserenewal$RANDOM"

# create a service plan to host the web app
az appservice plan create \
    --resource-group az-apigateway \
    --name vehicleAppServicePlan \
    --sku F1 \
    --location eastus

# deploy the web app
az webapp create \
    --resource-group az-apigateway \
    --name $APPSERVICE \
    --plan vehicleAppServicePlan \
    --deployment-source-url https://github.com/MicrosoftDocs/mslearn-load-balance-web-traffic-with-application-gateway \
    --deployment-source-branch appService
```

## Azure API Gateway
```azurecli

# create a subnet inside the solution vnet
az network vnet subnet create \
  --resource-group az-apigateway \
  --vnet-name vnet-vehicleAppVnet  \
  --name subnet-appGatewaySubnet \
  --address-prefixes 10.0.0.0/24

# create a public ip for the api-gateway
az network public-ip create \
  --resource-group az-apigateway \
  --name appGatewayPublicIp \
  --sku Standard \
  --dns-name vehicleapp-az-apigateway

az network application-gateway create \
--resource-group az-apigateway \
--name vehicleAppGateway \
--capacity 2 \
--vnet-name vnet-vehicleAppVnet \
--subnet subnet-appGatewaySubnet \
--public-ip-address appGatewayPublicIp \
--http-settings-protocol Http \
--http-settings-port 8080 \
--frontend-port 8080 \
--location eastus \
--sku WAF_v2 \
--no-wait


WEBSERVER1IP="$(az vm list-ip-addresses \
  --resource-group az-apigateway \
  --name vm-webServer1 \
  --query [0].virtualMachine.network.privateIpAddresses[0] \
  --output tsv)"

WEBSERVER2IP="$(az vm list-ip-addresses \
  --resource-group az-apigateway \
  --name vm-webserver2 \
  --query [0].virtualMachine.network.privateIpAddresses[0] \
  --output tsv)"

# add the VM backend apps
az network application-gateway address-pool create \
  --gateway-name vehicleAppGateway \
  --resource-group az-apigateway \
  --name vmPool \
  --servers $WEBSERVER1IP $WEBSERVER2IP

# add the web app backend
az network application-gateway address-pool create \
    --resource-group az-apigateway \
    --gateway-name vehicleAppGateway \
    --name appServicePool \
    --servers $APPSERVICE.azurewebsites.net

# add port 80 for the entry point
az network application-gateway frontend-port create \
    --resource-group az-apigateway \
    --gateway-name vehicleAppGateway \
    --name port80 \
    --port 80

az network application-gateway http-listener create \
    --resource-group az-apigateway \
    --name vehicleListener \
    --frontend-port port80 \
    --gateway-name vehicleAppGateway
```

## Health Probe
```azurecli

az network application-gateway probe create \
    --resource-group az-apigateway \
    --gateway-name vehicleAppGateway \
    --name customProbe \
    --path / \
    --interval 15 \
    --threshold 3 \
    --timeout 10 \
    --protocol Http \
    --host-name-from-http-settings true

az network application-gateway http-settings update \
    --resource-group az-apigateway \
    --gateway-name vehicleAppGateway \
    --name appGatewayBackendHttpSettings \
    --host-name-from-backend-pool true \
    --port 80 \
    --probe customProbe

az network application-gateway url-path-map create \
    --resource-group az-apigateway \
    --gateway-name vehicleAppGateway \
    --name urlPathMap \
    --paths /VehicleRegistration/* \
    --http-settings appGatewayBackendHttpSettings \
    --address-pool vmPool

az network application-gateway url-path-map rule create \
    --resource-group az-apigateway \
    --gateway-name vehicleAppGateway \
    --name appServiceUrlPathMap \
    --paths /LicenseRenewal/* \
    --http-settings appGatewayBackendHttpSettings \
    --address-pool appServicePool \
    --path-map-name urlPathMap

az network application-gateway rule create \
    --resource-group az-apigateway \
    --gateway-name vehicleAppGateway \
    --name appServiceRule \
    --http-listener vehicleListener \
    --rule-type PathBasedRouting \
    --address-pool appServicePool \
    --url-path-map urlPathMap

az network application-gateway rule delete \
    --resource-group az-apigateway \
    --gateway-name vehicleAppGateway \
    --name rule1
```

# Test Setup
```azurecli

echo http://$(az network public-ip show \
  --resource-group az-apigateway \
  --name appGatewayPublicIp \
  --query dnsSettings.fqdn \
  --output tsv)

http://vehicleapp-az-apigateway.eastus.cloudapp.azure.com/LicenseRenewal/create

http://vehicleapp-az-apigateway.eastus.cloudapp.azure.com/
```