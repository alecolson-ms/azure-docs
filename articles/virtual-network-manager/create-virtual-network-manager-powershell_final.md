---
title: 'Quickstart: Create a mesh network topology with Azure Virtual Network Manager via Azure Powershell'
description: Use this quickstart to learn how to create a mesh network topology with Virtual Network Manager using the Azure Powershell.
author: mbender-ms
ms.author: mbender
ms.service: virtual-network-manager
ms.topic: quickstart
ms.date: 08/24/2022
ms.custom: mode-api, devx-track-azurecli 
ms.devlang: azurepowershell
---

# Quickstart: Create a mesh network topology with Azure Virtual Network Manager via the Azure CLI

Get started with Azure Virtual Network Manager by using the Azure CLI to manage connectivity for all your virtual networks.

In this quickstart, you'll deploy three virtual networks and use Azure Virtual Network Manager to create a mesh network topology. Then you'll verify if the connectivity configuration got applied.

> [!IMPORTANT]
> Azure Virtual Network Manager is currently in public preview.
> This preview version is provided without a service level agreement, and it's not recommended for production workloads. Certain features might not be supported or might have constrained capabilities.
> For more information, see [Supplemental Terms of Use for Microsoft Azure Previews](https://azure.microsoft.com/support/legal/preview-supplemental-terms/).

## Prerequisites

* An Azure account with an active subscription. [Create an account for free](https://azure.microsoft.com/free/?WT.mc_id=A261C142F).
* During preview, the `4.20.0-preview` version of `Az.Network` is required to access the required cmdlets.
* If you're running PowerShell locally, you also need to run `Connect-AzAccount` to create a connection with Azure.

> [!IMPORTANT]
> Perform this quickstart using Powershell locally, not through Azure Cloud Shell. The version of `Az.Network` in Azure Cloud Shell does not currently support the Azure Virtual Network Manager cmdlets.

## Install Azure PowerShell module

Install the latest *Az.Network* Azure PowerShell module using this command:

```azurepowershell-interactive
 Install-Module -Name Az.Network -RequiredVersion 4.20.0-preview -AllowPrerelease
```

##  Select your Azure subscription

Select the subscription where network manager will be deployed.

```azurecli-interactive
SetAzContext --subscription "<subscription id>"
```

## Create a resource group 

Before you can deploy Azure Virtual Network Manager, you have to create a resource group to host a network manager with New-AzResourceGroup. This example creates a resource group named **myAVNMResourceGroup** in the **westus** location:

```azurecli-interactive
$location = "West US"
$rg = @{
    Name = 'myAVNMResourceGroup'
    Location = $location
}
New-AzResourceGroup @rg
```

## Create a Virtual Network Manager

Define the scope and access type this Network Manager instance will have. Create the scope with New-AzNetworkManagerScope. Replace the value  *<subscription_id>* with the subscription you want Virtual Network Manager to manage virtual networks for. For management groups, replace *<mgName\>* with the management group to manage.

```azurecli-interactive
Import-Module -Name Az.Network -RequiredVersion "4.20.0"

[System.Collections.Generic.List[string]]$subGroup = @()  
$subGroup.Add("/subscriptions/<subscription_id>")
[System.Collections.Generic.List[string]]$mgGroup = @()  
$mgGroup.Add("/providers/Microsoft.Management/managementGroups/<mgName>")

[System.Collections.Generic.List[String]]$access = @()  
$access.Add("Connectivity");  
$access.Add("SecurityAdmin"); 

$scope = New-AzNetworkManagerScope -Subscription $subGroup -ManagementGroup $mgGroup
```

Create the Virtual Network Manager with New-AzNetworkManager.

```azurecli-interactive
$avnm = @{
    Name = 'myAVNM'
    ResourceGroupName = $rg.Name
    NetworkManagerScope = $scope
    NetworkManagerScopeAccess = $access
    Location = $location
}
$networkmanager = New-AzNetworkManager @avnm
```

## Create a network group

Virtual Network Manager applies configurations to groups of VNets by placing them in **Network Groups.** Create a network group with New-AzNetworkManagerGroup.

```azurepowershell-interactive
$ng = @{
    Name = 'myNetworkGroup'
    ResourceGroupName = $rg.Name
    NetworkManagerName = $networkManager.Name
    MemberType = 'Microsoft.Network/VirtualNetworks'
}
$networkgroup = New-AzNetworkManagerGroup @ng
```
## Create virtual networks

Create five virtual networks with New-AzVirtualNetwork. This example creates virtual networks named **VNetA**, **VNetB**,**VNetC** and **VNetD** in the **West US** location. Each virtual network will have a tag of **networkType** used for dynamic membership. If you already have virtual networks you want create a mesh network with, you can skip to the next section.

```azurepowershell-interactive
$vnetA = @{
    Name = 'VNetA'
    ResourceGroupName = 'myAVNMResourceGroup'
    Location = $location
    AddressPrefix = '10.0.0.0/16'  
    Tag = @{
	   NetworkType = "Prod"
    }	  
}
$virtualNetworkA = New-AzVirtualNetwork @vnetA

$vnetB = @{
    Name = 'VNetB'
    ResourceGroupName = 'myAVNMResourceGroup'
    Location = $location
    AddressPrefix = '10.1.0.0/16'  
    Tag = @{
	   NetworkType = "Prod"
    }	  
}
$virtualNetworkB = New-AzVirtualNetwork @vnetB

$vnetC = @{
    Name = 'VNetC'
    ResourceGroupName = 'myAVNMResourceGroup'
    Location = $location
    AddressPrefix = '10.2.0.0/16'  
    Tag = @{
	   NetworkType = "Prod"
    }	  
}
$virtualNetworkC = New-AzVirtualNetwork @vnetC

$vnetD = @{
    Name = 'VNetD'
    ResourceGroupName = 'myAVNMResourceGroup'
    Location = $location
    AddressPrefix = '10.3.0.0/16'  
    Tag = @{
	   NetworkType = "Test"
    }	  
}
$virtualNetworkD = New-AzVirtualNetwork @vnetD

$vnetE = @{
    Name = 'VNetE'
    ResourceGroupName = 'myAVNMResourceGroup'
    Location = $location
    AddressPrefix = '10.4.0.0/16'  
    Tag = @{
	   NetworkType = "Test"
    }	  
}
$virtualNetworkE = New-AzVirtualNetwork @vnetE
```

### Add a subnet to each virtual network

To complete the configuration of the virtual networks add a /24 subnet to each one. Create a subnet configuration named **default** with Add-AzVirutalNetworkSubnetConfig:

```azurepowershell-interactive
$subnetA = @{
    Name = 'default'
    VirtualNetwork = $virtualNetworkA
    AddressPrefix = '10.0.0.0/24'
}
$subnetConfigA = Add-AzVirtualNetworkSubnetConfig @subnetA
$virtualnetworkA | Set-AzVirtualNetwork

$subnetB = @{
    Name = 'default'
    VirtualNetwork = $virtualNetworkB
    AddressPrefix = '10.1.0.0/24'
}
$subnetConfigC = Add-AzVirtualNetworkSubnetConfig @subnetB
$virtualnetworkB | Set-AzVirtualNetwork

$subnetC = @{
    Name = 'default'
    VirtualNetwork = $virtualNetworkC
    AddressPrefix = '10.2.0.0/24'
}
$subnetConfigC = Add-AzVirtualNetworkSubnetConfig @subnetC
$virtualnetworkC | Set-AzVirtualNetwork
```

## Define membership for a mesh configuration

Azure Virtual Network manager allows you two methods for adding membership to a network group. Static membership involves manually adding virtual networks, and dynamic membership involves using Azure Policy to dynamically add virtual networks based on conditions. Choose the option you wish to complete for your mesh configuration membership:

### Static membership option

Using **static membership**, you'll manually add 3 VNets for your Mesh configuration to your Network Group with [az network manager group static-member create](/cli/azure/network/manager/group/static-member#az-network-manager-group-static-member-create). Replace <subscription_id> with the subscription these VNets were created under.

```azurecli
az network manager group static-member create \
    --name "VNetA" \
    --network-group "myNetworkGroup" \
    --network-manager "myAVNM" \
    --resource-group "myAVNMResourceGroup" \
    --resource-id "/subscriptions/<subscription_id>/resourceGroups/myAVNMResourceGroup/providers/Microsoft.Network/virtualnetworks/VNetA"
```

```azurecli
az network manager group static-member create \
    --name "VNetB" \
    --network-group "myNetworkGroup" \
    --network-manager "myAVNM" \
    --resource-group "myAVNMResourceGroup" \
    --resource-id "/subscriptions/<subscription_id>/resourceGroups/myAVNMResourceGroup/providers/Microsoft.Network/virtualnetworks/VNetB"
```

```azurecli
az network manager group static-member create \
    --name "VNetC" \
    --network-group "myNetworkGroup" \
    --network-manager "myAVNM" \
    --resource-group "myAVNMResourceGroup" \
    --resource-id "/subscriptions/<subscription_id>/resourceGroups/myAVNMResourceGroup/providers/Microsoft.Network/virtualnetworks/VNetC"
```
### Dynamic membership option

Using [Azure Policy](concept-azure-policy-integration.md), you'll dynamically add the three VNets with a tag **networkType** value of *Prod* to the Network Group. These are the three virtual networks to become part of the mesh configuration.

> [!NOTE] 
> Policies can be applied to a subscription or management group, and must always be defined *at or above* the level they're created. Only virtual networks within a policy scope are added to a Network Group.

### Create a Policy definition
Create a Policy definition with [az policy definition create](/cli/azure/policy/definition#az-policy-definition-create) for virtual networks tagged as **Prod**. Replace *<subscription_id>* with the subscription you want to apply this policy to. If you want to apply it to a management group, replace `--subscription <subscription_id>` with `--management-group <mgName>`

```azurecli
az policy definition create \
    --name "ProdVNets" \
    --description "Choose Prod virtual networks only" \
    --rules "{\"if\":{\"allOf\":[{\"field\":\"Name\",\"contains\":\"VNet\"},{\"field\":\"tags['NetworkType']\",\"equals\":\"Prod\"}]},\"then\":{\"effect\":\"addToNetworkGroup\",\"details\":{\"networkGroupId\":\"/subscriptions/<subscription_id>/resourceGroups/myAVNMResourceGroup/providers/Microsoft.Network/networkManagers/myAVNM/networkGroups/myNetworkGroup\"}}}" \
    --subscription <subscription_id> \
    --mode "Microsoft.Network.Data"

```
### Apply a Policy definition

Once a policy is defined, it must also be applied with [az policy assignment create](/cli/azure/policy/assignment#az-policy-assignment-create). Replace *<subscription_id>* with the subscription you want to apply this policy to. If you want to apply it to a management group, replace `--scope "/subscriptions/<subscription_id>"` with `--scope "/providers/Microsoft.Management/managementGroups/<mgName>`, and replace *<mgName\>* with your management group.

```azurecli


az policy assignment create \
    --name "ProdVNets" \
    --description "Take only virtual networks tagged NetworkType:Prod" \
    --scope "/subscriptions/<subscription_id>" \
    --policy "/subscriptions/<subscription_id>/providers/Microsoft.Authorization/policyDefinitions/ProdVNets"
```

## Create a configuration

Now that the Network Group is created, and has the correct VNets, create a mesh network topology configuration with [az network manager connect-config create](/cli/azure/network/manager/connect-config#az-network-manager-connect-config-create). Replace <subscription_id> with your subscription.

```azurecli
az network manager connect-config create \
    --configuration-name "connectivityconfig" \
    --description "Production Mesh Connectivity Config Example" \
    --applies-to-groups network-group-id="/subscriptions/<subscription_id>/resourceGroups/myAVNMResourceGroup/providers/Microsoft.Network/networkManagers/myAVNM/networkGroups/myNetworkGroup" \
    --connectivity-topology "Mesh" \
    --network-manager-name "myAVNM" \
    --resource-group "myAVNMResourceGroup"
```
## Commit deployment

For the configuration to take effect, commit the configuration to the target regions with [az network manager post-commit](/cli/azure/network/manager#az-network-manager-post-commit):

```azurecli
#Currently broken - can only do via portal
az network manager post-commit \
    --network-manager-name "myAVNM" \
    --commit-type "Connectivity" \
    --configuration-ids "/subscriptions/<subscription_id>/resourceGroups/myANVMResourceGroup/providers/Microsoft.Network/networkManagers/myAVNM/connectivityConfigurations/connectivityconfig" \
    --target-locations "westus" \
    --resource-group "myAVNMResourceGroup"
```
## Verify configuration
Virtual Networks will display configurations applied to them with [az network manager list-effective-connectivity-config](/cli/azure/network/manager#az-network-manager-list-effective-connectivity-config):

```azurecli
az network manager list-effective-connectivity-config \
    --resource-group "myAVNMResourceGroup" \
    --virtual-network-name "VNetA"

az network manager list-effective-connectivity-config \
    --resource-group "myAVNMResourceGroup" \
    --virtual-network-name "VNetB"


az network manager list-effective-connectivity-config \
    --resource-group "myAVNMResourceGroup" \
    --virtual-network-name "VNetC"

az network manager list-effective-connectivity-config \
    --resource-group "myAVNMResourceGroup" \
    --virtual-network-name "VNetD"
```
For the virtual networks that are part of the connectivity configuration, you'll see an output similar to this:

```json
{
  "skipToken": "",
  "value": [
    {
      "appliesToGroups": [
        {
          "groupConnectivity": "None",
          "isGlobal": "False",
          "networkGroupId": "/subscriptions/<subscription_id>/resourceGroups/myAVNMResourceGroup/providers/Microsoft.Network/networkManagers/myAVNM/networkGroups/myNetworkGroup",
          "useHubGateway": "False"
        }
      ],
      "configurationGroups": [
        {
          "description": "Network Group for Production virtual networks",
          "id": "/subscriptions/<subscription_id>/resourceGroups/myAVNMResourceGroup/providers/Microsoft.Network/networkManagers/myAVNM/networkGroups/myNetworkGroup",
          "provisioningState": "Succeeded",
          "resourceGroup": "myAVNMResourceGroup"
        }
      ],
      "connectivityTopology": "Mesh",
      "deleteExistingPeering": "False",
      "description": "Production Mesh Connectivity Config Example",
      "hubs": [],
      "id": "/subscriptions/<subscription_id>/resourceGroups/myAVNMResourceGroup/providers/Microsoft.Network/networkManagers/myAVNM/connectivityConfigurations/connectivityconfig",
      "isGlobal": "False",
      "provisioningState": "Succeeded",
      "resourceGroup": "myAVNMResourceGroup"
    }
  ]
}
```
For virtual networks not part of the network group like **VNetD**, you'll see an output similar to this:

```json
az network manager list-effective-connectivity-config     --resource-group "myAVNMResourceGroup"     --virtual-network-name "VNetD-test"
{
  "skipToken": "",
  "value": []
}
```
## Clean up resources

If you no longer need the Azure Virtual Network Manager, you'll need to make sure all of following are true before you can delete the resource:

* There are no deployments of configurations to any region.
* All configurations have been deleted.
* All network groups have been deleted.

1. Remove the connectivity deployment by committing no configurations with [az network manager post-commit](/cli/azure/network/manager#az-network-manager-post-commit):

    ```azurecli
    az network manager post-commit \
        --network-manager-name "myAVNM" \
        --commit-type "Connectivity" \
        --target-locations "westus" \
        --resource-group "myAVNMResourceGroup"
    ```

1. Remove the connectivity configuration with [az network manager connect-config delete](/cli/azure/network/manager/connect-config#az-network-manager-connect-config-delete):

    ```azurecli
    az network manager connect-config delete \
        --configuration-name "connectivityconfig" \
        --name "myAVNM" \
        --resource-group "myAVNMResourceGroup"
    ```

1. Remove the network group with [az network manager group delete](/cli/azure/network/manager/group#az-network-manager-group-delete):

    ```azurecli
    az network manager group delete \
        --name "myNetworkGroup" \
        --network-manager-name "myAVNM" \
        --resource-group "myAVNMResourceGroup"
    ```

1. Delete the network manager instance with [az network manager delete](/cli/azure/network/manager#az-network-manager-delete):

    ```azurecli
    az network manager delete \
        --name "myAVNM" \
        --resource-group "myAVNMResourceGroup"
    ```

1. If you no longer need the resource created, delete the resource group with [az group delete](/cli/azure/group#az-group-delete):

    ```azurecli
    az group delete \
        --name "myAVNMResourceGroup"
    ```

## Next steps

After you've created the Azure Virtual Network Manager, continue on to learn how to block network traffic by using the security admin configuration:

> [!div class="nextstepaction"]
[Block network traffic with security admin rules](how-to-block-network-traffic-portal.md)
[Create a secured hub and spoke network](tutorial-create-secured-hub-and-spoke.md)
