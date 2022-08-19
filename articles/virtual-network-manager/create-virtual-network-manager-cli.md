---
title: 'Quickstart: Create groups of virtual networks, and apply mesh network topology with Azure Virtual Network Manager via the Azure CLI'
description: Use this quickstart to learn how create groups of virtual networks, and create a mesh network topology with Virtual Network Manager using the Azure CLI.
author: mbender-ms
ms.author: mbender
ms.service: virtual-network-manager
ms.topic: quickstart
ms.date: 11/16/2021
ms.custom: mode-api, devx-track-azurecli 
ms.devlang: azurecli
---

# Quickstart: Create a mesh network topology with Azure Virtual Network Manager via the Azure CLI

Get started with Azure Virtual Network Manager by using the Azure CLI to manage connectivity for all your virtual networks.

In this quickstart, you'll create groups of three virtual networks (manually, and dynamically) and use Azure Virtual Network Manager to apply a mesh network topology on them. Then, you'll verify if the connectivity configuration got applied.

> [!IMPORTANT]
> Azure Virtual Network Manager is currently in public preview.
> This preview version is provided without a service level agreement, and it's not recommended for production workloads. Certain features might not be supported or might have constrained capabilities.
> For more information, see [Supplemental Terms of Use for Microsoft Azure Previews](https://azure.microsoft.com/support/legal/preview-supplemental-terms/).

## Prerequisites

* An Azure account with an active subscription. [Create an account for free](https://azure.microsoft.com/free/?WT.mc_id=A261C142F).
* Make sure you have the [latest Azure CLI](/cli/azure/install-azure-cli) or you can use Azure Cloud Shell in the portal.
* Run `az extension add -n virtual-network-manager` to add the Azure Virtual Network Manager extension.

##  Sign in to your Azure account and select your subscription

To begin your configuration, sign in to your Azure account. If you use the Cloud Shell "Try It", you're signed in automatically. Use the following examples to help you connect:

```azurecli-interactive
az login
```

A network manager must exist under one subscription. 
Resources **managed** by that network manager, however, such as VNets, can exist across several other subscriptions and tenants.
You'll switch between **one manager subscription,** and **one target subscription.** (These can be the same, if you so choose).

Switch to the subscription owning your network manager.

```azurecli-interactive
az account set \
    --subscription "{manager subscription ID}"
```

## Create a resource group 

Before you can create a Virtual Network Manager, you have to create a resource group to host it. Create a resource group with [az group create](/cli/azure/group#az-group-create). This example creates a resource group named **managerAVNMResourceGroup** in the **westus** location:

```azurecli-interactive
az group create \
    --name "managerAVNMResourceGroup" \
    --location "westus"
```

## Create a Virtual Network Manager

Define the scope and access type this Network Manager instance will have. Create the scope by using [az network manager create](/cli/azure/network/manager#az-network-manager-create). 

Virtual Network Managers can manage **mangement groups** (with management groups and subscriptions under them), or **subscriptions**. Replace {mgName} and {subId} with any management groups or subscriptions you want to manage virtual networks under.

```azurecli-interactive
az network manager create \
    --location "westus" \
    --name "myAVNM" \
    --resource-group "myAVNMResourceGroup" \
    --scope-accesses "Connectivity" "SecurityAdmin" \
    --network-manager-scopes management-groups="/Microsoft.Management/{mgName}" subscriptions="/subscriptions/{subId}"
```

## Create a Network Group

Virtual Network Manager applies configurations to groups of VNets by placing them in **Network Groups.** Create a network group using **static membership** with [az network manager group create](/cli/azure/network/manager/group#az-network-manager-group-create).

```azurecli-interactive
az network manager group create \
    --name "myNetworkGroup" \
    --network-manager-name "myAVNM" \
    --resource-group "myAVNMResourceGroup"
```

## Create Virtual Networks managed by your Network Manager.

Switch to the subscription you want to create VNets under. This subscription should be under the network manager scopes you defined above.

```azurecli-interactive
az account set \
    --subscription "{target subscription ID}"
```

Resources under these managed subscriptions also need to exist in resource groups under those subscriptions. Create a resource group with [az group create](/cli/azure/group#az-group-create). This example creates a resource group named **targetAVNMResourceGroup** in the **westus** location:

```azurecli-interactive
az group create \
    --name "targetAVNMResourceGroup" \
    --location "westus"
```

Create three virtual networks with [az network vnet create](/cli/azure/network/vnet#az-network-vnet-create). This example creates virtual networks named **VNetA**, **VNetB**, and **VNetC** in the **eastus** location, given the arbitrary tag **Color=Red** or **Color=Blue** to later demonstrate policy. If you already have virtual networks you want create a mesh network with, you can skip to the next section.

```azurecli-interactive
az network vnet create \
    --name "VNetA" \
    --resource-group "targetAVNMResourceGroup" \
    --address-prefix "10.0.0.0/16"
    --tags "Color=Red"

az network vnet create \
    --name "VNetB" \
    --resource-group "targetAVNMResourceGroup" \
    --address-prefix "10.1.0.0/16"
    --tags "Color=Red"

az network vnet create \
    --name "VNetC" \
    --resource-group "targetAVNMResourceGroup" \
    --address-prefix "10.2.0.0/16"
    --tags "Color=Red"
```

Optionally, create virtual networks called **Unscoped_VNet** and **VNt** to see how Policy can dynamically include/exclude VNets managed by your Virtual Network Manager.

```azurecli-interactive
az network vnet create \
    --name "Unscoped_VNet" \
    --resource-group "targetAVNMResourceGroup" \
    --address-prefix "10.3.0.0/16"
    --tags "Color=Blue"

az network vnet create \
    --name "VNt" \
    --resource-group "targetAVNMResourceGroup" \
    --address-prefix "10.4.0.0/16"
    --tags "Color=Red"
```

### Add a subnet to each virtual network

To complete the configuration of the virtual networks add a /24 subnet to each one. Create a subnet configuration named **default** with [az network vnet subnet create](/cli/azure/network/vnet/subnet#az-network-vnet-subnet-create):

```azurecli-interactive 
az network vnet subnet create \
    --name "default" \
    --resource-group "myAVNMResourceGroup" \
    --vnet-name "VNetA" \
    --address-prefix "10.0.0.0/24"

az network vnet subnet create \
    --name "default" \
    --resource-group "myAVNMResourceGroup" \
    --vnet-name "VNetB" \
    --address-prefix "10.1.0.0/24"

az network vnet subnet create \
    --name "default" \
    --resource-group "myAVNMResourceGroup" \
    --vnet-name "VNetC" \
    --address-prefix "10.2.0.0/24"
```


## Option 1 (Static Membership): Manually add the 3 VNets for your Mesh configuration to the Network Group.

Using **static membership**, you'll directly add 3 VNets to your Network Group.

Ensure you're using the subscription that owns your Network Manager.

```azurecli-interactive
az account set \
    --subscription "{manager subscription ID}"
```

Add these static members to the Network Group. Replace {target subscription id} with the subscription these VNets were created under.

```azurecli-interactive
az network manager group static-member create \
    --name "VNetA" \
    --network-group "myNetworkGroup" \
    --network-manager "myAVNM" \
    --resource-group "targetAVNMResourceGroup" \
    --resource-id "/subscriptions/{target subscription id}/resourceGroups/targetAVNMResourceGroup/providers/Microsoft.Network/virtualnetworks/VNetA"
```

```azurecli-interactive
az network manager group static-member create \
    --name "VNetB" \
    --network-group "myNetworkGroup" \
    --network-manager "myAVNM" \
    --resource-group "targetAVNMResourceGroup" \
    --resource-id "/subscriptions/{target subscription id}/resourceGroups/targetAVNMResourceGroup/providers/Microsoft.Network/virtualnetworks/VNetB"
```

```azurecli-interactive
az network manager group static-member create \
    --name "VNetC" \
    --network-group "myNetworkGroup" \
    --network-manager "myAVNM" \
    --resource-group "targetAVNMResourceGroup" \
    --resource-id "/subscriptions/{target subscription id}/resourceGroups/targetAVNMResourceGroup/providers/Microsoft.Network/virtualnetworks/VNetC"
```

## Option 2 (Dynamic Membership): Using Azure Policy, dynamically add the 3 VNets for your Mesh configuration to the Network Group.

Using Dynamic Membership through Azure Policy, you'll create a policy to find and automatically add the right VNets to your network group.

Switch to the subscription owning your network manager.

```azurecli-interactive
az account set \
    --subscription "{manager subscription ID}"
```

With Azure Policy, create a policy to accept only VNets with the tag "Color:Red" and "VNet" in the name.  This policy will accept VNetA, VNetB and VNetC, but reject the Unscoped_VNet and VNt. 

Policies can be applied to a subscription or management group, and must always be defined _at or above_ the level they're created. Only virtual networks within a policy scope are added to a Network Group.

Create a Policy definition with [az policy definition create](/cli/azure/policy/definition#az-policy-definition-create). Replace {mg} with the management group you want to apply this policy to. If you want to apply it to a subscription, replace the `--management-group {mgName}` parameter with `--subscription {subId}`.

```azurecli-interactive
az policy definition create \
    --name "takeRedVNets" \
    --description "Take only virtual networks with VNet in the name and the tag Color:Red" \
    --rules ""{\"if\":{\"allOf\":[{\"field\":\"Name\",\"contains\":\"VNet\"},{\"field\":\"tags['Color']\",\"equals\":\"Red\"}]},\"then\":{\"effect\":\"addToNetworkGroup\",\"details\":{\"networkGroupId\":\"%networkGroupId%\"}}}"" \
    --management-group "{mgName}" \
    --mode "Microsoft.Network.Data"
```

Once a policy is defined, it must also be applied. Replace {mg} with the management group you want to apply this policy to. If you want to apply it to a subscription, replace the `--management-group "/providers/Microsoft.Management/managementGroups/{mgName}` parameter with `--subscription "/subscriptions/{subId}"`.

```azurecli-interactive
az policy assignment create \
    --name "takeRedVNets" \
    --description "Take only virtual networks with VNet in the name and the tag Color:Red" \
    --rules ""{\"if\":{\"allOf\":[{\"field\":\"Name\",\"contains\":\"VNet\"},{\"field\":\"tags['Color']\",\"equals\":\"Red\"}]},\"then\":{\"effect\":\"addToNetworkGroup\",\"details\":{\"networkGroupId\":\"%networkGroupId%\"}}}"" \
    --management-group "/providers/Microsoft.Management/managementGroups/{mg}" \
    --mode "Microsoft.Network.Data"
```

## Create a configuration

Now that the Network Group is created, and has the correct VNets, configurations can be applied to this group (and all VNets under it).

Create a mesh network topology configuration with [az network manager connect-config create](/cli/azure/network/manager/connect-config#az-network-manager-connect-config-create). Replace {manager_subscription_id} with the subscription that owns your network manager.

```azurecli-interactive
az network manager connect-config create \
    --configuration-name "connectivityconfig" \
    --description "CLI Mesh Connectivity Config Example" \
    --applies-to-groups network-group-id="/subscriptions/{manager_subscription_id}/resourceGroups/targetAVNMResourceGroup/providers/Microsoft.Network/networkManagers/myAVNM/networkGroups/myNetworkGroup" \
    --connectivity-topology "Mesh" \
    --delete-existing-peering true \
    --network-manager-name "myAVNM" \
    --resource-group "myAVNMResourceGroup"
```

## Commit deployment

Commit the configuration to the target regions with with [az network manager post-commit](/cli/azure/network/manager#az-network-manager-post-commit). This will trigger your configuration to begin taking effect.

```azurecli-interactive
az network manager post-commit \
    --network-manager-name "myAVNM" \
    --commit-type "Connectivity" \
    --configuration-ids "/subscriptions/{subscriptionId}/resourceGroups/myANVMResourceGroup/providers/Microsoft.Network/networkManagers/myAVNM/connectivityConfigurations/connectivityconfig" \
    --target-locations "westus" \
    --resource-group "myAVNMResourceGroup"
```

## Verify that the configurations have been deployed as intended.

Only the subscription owning the Virtual Networks can verify that configurations have been applied to them. Switch back to that subscription.

```azurecli-interactive
az account set \
    --subscription "<target subscription ID>"
```

Virtual Networks will display configurations applied to them with [az network manager list-effective-connectivity-config](/cli/azure/network/manager#az-network-manager-list-effective-connectivity-config):

```azurecli-interactive
az network manager list-effective-connectivity-config \
    -g "targetAVNMResourceGroup"
    --virtual-network-name "VNetA"

az network manager list-effective-connectivity-config \
    -g "targetAVNMResourceGroup"
    --virtual-network-name "VNetB"


az network manager list-effective-connectivity-config \
    -g "targetAVNMResourceGroup"
    --virtual-network-name "VNetC"
```

Optionally, if you created **Unscoped_VNet** and **VNt** earlier, you can check that configurations have _not_ been applied to them accordingly:

```
az network manager list-effective-connectivity-config \
    -g "targetAVNMResourceGroup"
    --virtual-network-name "Unscoped_VNet"

az network manager list-effective-connectivity-config \
    -g "targetAVNMResourceGroup"
    --virtual-network-name "VNt"
```

## Clean up resources

If you no longer need the Azure Virtual Network Manager, you'll need to make sure all of following are true before you can delete the resource:

* There are no deployments of configurations to any region.
* All configurations have been deleted.
* All network groups have been deleted.

Ensure you're using the subscription that owns your network manager.

```azurecli-interactive
az account set \
    --subscription "<manager subscription ID>"
```

1. Remove the connectivity deployment by committing no configurations with [az network manager post-commit](/cli/azure/network/manager#az-network-manager-post-commit):

    ```azurecli-interactive
    az network manager post-commit \
        --network-manager-name "myAVNM" \
        --commit-type "Connectivity" \
        --target-locations "westus" \
        --resource-group "managerAVNMResourceGroup"
    ```

1. Remove the connectivity configuration with [az network manager connect-config delete](/cli/azure/network/manager/connect-config#az-network-manager-connect-config-delete):

    ```azurecli-interactive
    az network manager connect-config delete \
        --configuration-name "connectivityconfig" \
        --name "myAVNM" \
        --resource-group "managerAVNMResourceGroup"
    ```

1. Remove the network group with [az network manager group delete](/cli/azure/network/manager/group#az-network-manager-group-delete):

    ```azurecli-interactive
    az network manager group delete \
        --name "myNetworkGroup" \
        --network-manager-name "myAVNM" \
        --resource-group "managerAVNMResourceGroup"
    ```

1. Delete the network manager instance with [az network manager delete](/cli/azure/network/manager#az-network-manager-delete):

    ```azurecli-interactive
    az network manager delete \
        --name "myAVNM" \
        --resource-group "managerAVNMResourceGroup"
    ```

1. If you no longer need any resources under the group the network manager belongs to, delete the resource group with [az group delete](/cli/azure/group#az-group-delete):

    ```azurecli-interactive
    az group delete \
        --name "managerAVNMResourceGroup"
    ```
    
Ensure you're using the subscription that owns your virtual networks.

```azurecli-interactive
az account set \
    --subscription "<target subscription ID>"
```
    
1. Delete the sample VNets for testing with [az network vnet delete](/cli/azure/network/vnet#az-network-vnet-delete):

```azurecli-interactive
az network vnet delete \
    --name "VNetA" \
    --resource-group "targetAVNMResourceGroup"

az network vnet delete \
    --name "VNetB" \
    --resource-group "targetAVNMResourceGroup"

az network vnet delete \
    --name "VNetC" \
    --resource-group "targetAVNMResourceGroup"

az network vnet delete \
    --name "Unscoped_VNet" \
    --resource-group "targetAVNMResourceGroup"

az network vnet delete \
    --name "VNt" \
    --resource-group "targetAVNMResourceGroup"
```

1. If you no longer need any resources under the resource group the virtual networks belongs to, delete it with [az group delete](/cli/azure/group#az-group-delete):

    ```azurecli-interactive
    az group delete \
        --name "targetAVNMResourceGroup"
    ```

## Next steps

After you've created the Azure Virtual Network Manager, continue on to learn how to block network traffic by using the security admin configuration:

> [!div class="nextstepaction"]
> [Block network traffic with security admin rules](how-to-block-network-traffic-portal.md)
