# Azure IoT Edge Test

## Description

The conten of this folder aims to create a testing environment for Azure IoT Edge, by deploying a new IoT Hub instance, a VM that will work as the new IoT Edge, and Will deploy modules to emulate a temperature sensor on it.

The ARM template ```edgeDeploy.json``` deploys a VM to host IoT Edge services. It is based on the official Azure IoT Edge documentation but with some updates and modifications to be used with student subscriptions:

  - Only a few low usage regions are available

  - Basic SKU IPs were [retired](https://azure.microsoft.com/en-gb/updates?id=upgrade-to-standard-sku-public-ip-addresses-in-azure-by-30-september-2025-basic-sku-will-be-retired) in 30 September 2025 so they were replaced with Standard SKU IPs.

  - The VM size needs to be smaller than the default to be able to use it with Azure for Students Subscription (using Standard_b1s)


## Prerequisites


- You have installed azure CLI and you are logged in
    ```$ az login```

-  the IoT extensions for it:

    ```$ az extension add --name azure-iot```

- There is a valid subscription associated with your account
    ```console
       $ az account list --output table
       $ az account set -s <Subscription-ID>
    ```

## Procedure

1. Export required variables

```console
$ export LOCATION=switzerlandnorth

$ export RG_NAME=mauro-unir-rg

$ export HUB_NAME=iot-hub-1

$ export DEV_NAME=iot-edge-dev-1
```

2. Create a resource group

```
$ az group create --name $RG_NAME --location $LOCATION
```

3. Create an IoT Hub instance that will act as the master of the IoT Edge device

```
$ az iot hub create --resource-group $RG_NAME --name $HUB_NAME --sku F1 --partition-count 2 --location $LOCATION
```

4. Create device identity to be used by the Edge VM

```
$ az iot hub device-identity create --device-id $DEV_NAME $EDGE_ENABLED --hub-name $HUB_NAME
```

5. Get Connectino String

```
$ az iot hub device-identity connection-string show --device-id $DEV_NAME --hub-name $HUB_NAME
```

6. Create new VM to be used as Edge device

```
$ az deployment group create \
--name "iot_edge_deployment_1" \
--resource-group $RG_NAME \
--template-uri "https://raw.githubusercontent.com/mauroseb/azure-iot-tests/refs/heads/main/iot-edge-test/edgeDeploy.json" \
--parameters vmSize='Standard_B1ms' \
--parameters dnsLabelPrefix=$DEV_NAME \
--parameters adminUsername='azureuser' \
--parameters deviceConnectionString=$(az iot hub device-identity connection-string show --device-id $DEV_NAME --hub-name $HUB_NAME -o tsv) \
--parameters authenticationType='password' \
--parameters adminPasswordOrKey="VerySecretPassword123"
```

**- Alternatively create the Edge VM with SSH Pubkey access -**

 Generate SSH Key pair

```
$ ssh-keygen -m PEM -t rsa -b 4096 -q -f ~/.ssh/iotedge-vm-key -N ""

$ az deployment group create \
--name "iot_edge_deployment_1" \
--resource-group $RG_NAME \
--template-uri "https://raw.githubusercontent.com/mauroseb/azure-iot-tests/refs/heads/main/iot-edge-test/edgeDeploy.json" \
--parameters vmSize='Standard_B1ms' \
--parameters dnsLabelPrefix=$DEV_NAME' \
--parameters adminUsername='azureuser' \
--parameters deviceConnectionString=$(az iot hub device-identity connection-string show --device-id $DEV_NAME --hub-name $HUB_NAME -o tsv) \
--parameters authenticationType='sshPublicKey' \
--parameters adminPasswordOrKey="$(< ~/.ssh/iotedge-vm-key.pub)"
```

7. Login into the new device

```console
$ ssh azureuser@<DEVICE_FQDN>
```

For example:
```
$ ssh azureuser@iot-edge-01.switzerlandnorth.cloudapp.azure.com
```



8. Investigate available commands in the edge device

- Check if the device comlpies with Azure's set of requirements
```
$ sudo iotedge check

Configuration checks (aziot-identity-service)
---------------------------------------------
√ keyd configuration is well-formed - OK
...
```
- List running modules (containers)

```
$ iotedge list
NAME             STATUS           DESCRIPTION      Config
edgeAgent        running          Up 5 minutes     mcr.microsoft.com/azureiotedge-agent:1.5
azureuser@vm-ldo6w2zlqhy6k:~$ iotedge system
```
- And again but from docker CLI
```
 sudo docker ps -a
CONTAINER ID   IMAGE                                      COMMAND                  CREATED         STATUS         PORTS     NAMES
4fb766e12a2d   mcr.microsoft.com/azureiotedge-agent:1.5   "/bin/sh -c 'exec /a…"   6 minutes ago   Up 6 minutes             edgeAgent
```

- Check configuration set at deployment
```
$ sudo cat /etc/aziot/config.toml
[provisioning]
source = "manual"
connection_string = "<REDACTED>"

[agent]
name = "edgeAgent"
type = "docker"

[agent.config]
image = "mcr.microsoft.com/azureiotedge-agent:1.5"

[connect]
workload_uri = "unix:///var/run/iotedge/workload.sock"
management_uri = "unix:///var/run/iotedge/mgmt.sock"

[listen]
workload_uri = "fd://aziot-edged.workload.socket"
management_uri = "fd://aziot-edged.mgmt.socket"

[moby_runtime]
uri = "unix:///var/run/docker.sock"
network = "azure-iot-edge"
```

- Check the running system processes
```
$ iotedge system status
System services:
    aziot-edged             Running
    aziot-identityd         Running
    aziot-keyd              Running
    aziot-certd             Running
    aziot-tpmd              Ready
```

9. Install a new module from Microsoft Registry

    - URL of the catalog for IoT  : https://mcr.microsoft.com/catalog?cat=IoT%20Edge%20Modules&alphaSort=asc&alphaSortKey=Name

    - URL of the simulated temperature sensor module to add: mcr.microsoft.com/azureiotedge-simulated-temperature-sensor


10. Check the module status

```
$ iotedge list
NAME             STATUS           DESCRIPTION      Config
edgeAgent        running          Up 5 minutes     mcr.microsoft.com/azureiotedge-agent:1.5
```

11. Delete everything when finished
```
$ az group delete --name $RG_NAME --location $LOCATION
```

