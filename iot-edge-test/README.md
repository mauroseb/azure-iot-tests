# Azure IoT Edge Test

## Description
The ARM template ```edgeDeploy.json``` deploys a VM to host IoT Edge services based on the official Azure IoT Edge docs but wiith some updates and modifications to be used with student subscriptions.
  - Basic SKU IPs were [retired](https://azure.microsoft.com/en-gb/updates?id=upgrade-to-standard-sku-public-ip-addresses-in-azure-by-30-september-2025-basic-sku-will-be-retired) in 30 September 2025 so they were replaced with Standard SKU IPs.
  - The VM size needs to be smaller than the default to be able to use it with Azure for Students Subscription (using Standard_b1s)

## Prerequisites

- There is a working IoT Hub to bind to
- There is a device identity already created in the IoT Hub to use for the edge device

## Usage

```console
$ export LOCATION=switzerlandnorth

$ export RG_NAME=iot-edge-rg

$ export HUB_NAME=iot-hub-1

$ export DEV_NAME=iot-edge-dev-1

$ az deployment group create \
--name "iot_edge_1" \
--resource-group $RG_NAME \
--template-url "https://raw.githubusercontent.com/mauroseb/azure-iot-tests/refs/heads/main/iot-edge-test/edgeDeploy.json" \
--parameters vmSize='Standard_B1ms' \
--parameters dnsLabelPrefix='iot-edge-dev-1' \
--parameters adminUsername='azureuser' \
--parameters deviceConnectionString=$(az iot hub device-identity connection-string show --device-id $DEV_NAME --hub-name $HUB_NAME -o tsv) \
--parameters authenticationType='password' \
--parameters adminPasswordOrKey="VerySecretPassword123"
```

