---
title: 'Quickstart: Create a mesh network topology with Azure Virtual Network Manager via Azure Powershell'
description: Use this quickstart to learn how to create a mesh network topology with Virtual Network Manager using the Azure Powershell.
author: mbender-ms
ms.author: mbender
ms.service: virtual-network-manager
ms.topic: quickstart
ms.date: 08/24/2022
ms.custom: mode-api, devx-track-azurepowershell
ms.devlang: azurepowershell
---

# Quickstart: Create a mesh network topology with Azure Virtual Network Manager via the Azure Powershell

Get started with Azure Virtual Network Manager by using the Azure Powershell to manage connectivity for all your virtual networks.

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

```azurepowershell-interactive
SetAzContext --subscription "<subscription id>"
```

## Create a resource group 

Before you can deploy Azure Virtual Network Manager, you have to create a resource group to host a network manager with New-AzResourceGroup. This example creates a resource group named **myAVNMResourceGroup** in the **westus** location:

```azurepowershell-interactive
$location = "West US"
$rg = @{
    Name = 'myAVNMResourceGroup'
    Location = $location
}
New-AzResourceGroup @rg
```

## Create a Virtual Network Manager

Define the scope and access type this Network Manager instance will have. Create the scope with New-AzNetworkManagerScope. Replace the value  *<subscription_id>* with the subscription you want Virtual Network Manager to manage virtual networks for. For management groups, replace *<mgName\>* with the management group to manage.

```azurepowershell-interactive
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

```azurepowershell-interactive
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
$subnetConfigB = Add-AzVirtualNetworkSubnetConfig @subnetB
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

```azurepowershell-interactive
$smA = @{
    Name = Get-UniqueString $virtualNetworkA.Id
    ResourceGroupName = $rg.Name
    NetworkGroupName = $networkGroup.Name
    NetworkManagerName = $networkManager.Name
    ResourceId = $virtualNetworkA.Id
}
$staticmemberA = New-AzNetworkManagerStaticMember @smA

$smB = @{
    Name = Get-UniqueString $virtualNetworkB.Id
    ResourceGroupName = $rg.Name
    NetworkGroupName = $networkGroup.Name
    NetworkManagerName = $networkManager.Name
    ResourceId = $virtualNetworkB.Id
}
$staticmemberB = New-AzNetworkManagerStaticMember @smB

$smC = @{
    Name = Get-UniqueString $virtualNetworkC.Id
    ResourceGroupName = $rg.Name
    NetworkGroupName = $networkGroup.Name
    NetworkManagerName = $networkManager.Name
    ResourceId = $virtualNetworkC.Id
}
$staticmemberC = New-AzNetworkManagerStaticMember @smC
```
### Dynamic membership option

Using [Azure Policy](concept-azure-policy-integration.md), you'll dynamically add the three VNets with a tag **networkType** value of *Prod* to the Network Group. These are the three virtual networks to become part of the mesh configuration.

> [!NOTE] 
> Policies can be applied to a subscription or management group, and must always be defined *at or above* the level they're created. Only virtual networks within a policy scope are added to a Network Group.

### Create a Policy definition
Create a Policy definition with New-AzPolicyDefinition for virtual networks tagged as **Prod**. Replace *<subscription_id>* with the subscription you want to apply this policy to. If you want to apply it to a management group, replace `Subscription = <subscription_id>` with `ManagementGroup = <mgName>`

Define the conditional statement and store it in a variable.
> [!NOTE]
> It is recommended to scope all of your conditionals to only scan for type `Microsoft.Network/virtualNetwork` for efficiency.

```azurecli
$conditionalMembership = '{ 
    "allof":[
        { 
        "field": "type", 
        "equals": "Microsoft.Network/virtualNetwork" 
        }
        { 
        "field": "name", 
        "contains": "VNet" 
        } 
	{
	"field": "tags[''NetworkType'']",
	"equals": "Prod"
	}
    ] 
}' 
```
Create the policy definition.

```azurecli
$defn = @{
    Name = 'ProdVNets'
    Mode = 'Microsoft.Network.Data'
    Policy = $conditionalMembership
    Subscription = <subscription>
}
    
$policyDefinition = New-AzPolicyDefinition $defn  
```
### Apply a Policy definition

Once a policy is defined, it must also be applied with New-AzPolicyAssignment. Replace *<subscription_id>* with the subscription you want to apply this policy to. If you want to apply it to a management group, replace `Scope = "/subscriptions/<subscription_id>"` with `Scope = "/providers/Microsoft.Management/managementGroups/<mgName>`, and replace *<mgName\>* with your management group.

```azurepowershell-interactive
$assgn = @{
Name = 'ProdVNets'
Description = 'Take only virtual networks tagged NetworkType:Prod'
PolicyDefinition  = $policyDefinition
Scope = "/providers/Microsoft.Management/managementGroups/{mgName}"
}

$policyAssignment = New-AzPolicyAssignment $assgn
```

## Create a configuration

Now that the Network Group is created, and has the correct VNets, create a mesh network topology configuration with [az network manager connect-config create](/cli/azure/network/manager/connect-config#az-network-manager-connect-config-create). Replace <subscription_id> with your subscription.

Create a connectivity group item to add a network group to with New-AzNetworkManagerConnectivityGroupItem.

```azurepowershell-interactive
$gi = @{
NetworkGroupId = $networkgroup.Id
}
$groupItem = New-AzNetworkManagerConnectivityGroupItem @gi
```

Create a configuration group and add the group item from the previous step.

```azurepowershell-interactive
$configGroup = @()
$configGroup.Add($groupItem)
```
Create the connectivity configuration with New-AzNetworkManagerConnectivityConfiguration.

```azurepowershell-interactive
$config = @{
Name = 'connectivityconfig'
ResourceGroupName = $rg.Name
NetworkManagerName = $networkManager.Name
ConnectivityTopology = 'Mesh'
AppliesToGroup = $configGroup
}
$connectivityconfig = New-AzNetworkManagerConnectivityConfiguration @config
```   
## Commit deployment

For the configuration to take effect, commit the configuration to the target regions with Deploy-AzNetworkManagerCommit.

```azurepowershell-interactive
[System.Collections.Generic.List[string]]$configIds = @()  
$configIds.add($connectivityconfig.id) 
[System.Collections.Generic.List[string]]$target = @()   
$target.Add("westus")     

$deployment = @{
    Name = $networkManager.Name
    ResourceGroupName = $rg.Name
    ConfigurationId = $configIds
    TargetLocation = $target
    CommitType = 'Connectivity'
}
Deploy-AzNetworkManagerCommit @deployment 
```

## Verify configuration
Virtual Networks will display configurations applied to them with [az network manager list-effective-connectivity-config](/cli/azure/network/manager#az-network-manager-list-effective-connectivity-config):

```azurecli
Get-AzNetworkManagerEffectiveConnectivityConfigurationList -ResourceGroupName $rg.Name -VirtualNetworkName "VNetA"
Get-AzNetworkManagerEffectiveConnectivityConfigurationList -ResourceGroupName $rg.Name -VirtualNetworkName "VNetB"
Get-AzNetworkManagerEffectiveConnectivityConfigurationList -ResourceGroupName $rg.Name -VirtualNetworkName "VNetC"
Get-AzNetworkManagerEffectiveConnectivityConfigurationList -ResourceGroupName $rg.Name -VirtualNetworkName "VNetD"
Get-AzNetworkManagerEffectiveConnectivityConfigurationList -ResourceGroupName $rg.Name -VirtualNetworkName "VNetE"
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

1. Remove the connectivity deployment by deploying an empty configuration with Deploy-AzNetworkManagerCommit.

```azurepowershell-interactive
[System.Collections.Generic.List[string]]$configIds = @()
[System.Collections.Generic.List[string]]$target = @()   
$target.Add("westus")     
$removedeployment = @{
Name = 'myAVNM'
ResourceGroupName = 'myAVNMResourceGroup'
ConfigurationId = $configIds
Target = $target
CommitType = 'Connectivity'
}
Deploy-AzNetworkManagerCommit @removedeployment
```

1. Remove the connectivity configuration with Remove-AzNetworkManagerConnectivityConfiguration

```azurepowershell-interactive

Remove-AzNetworkManagerConnectivityConfiguration @connectivityconfig.Id   

```
2. Remove the policy resources with Remove-AzPolicy.

```azurepowershell-interactive

Remove-AzPolicyAssignment $policyAssignment.Id
Remove-AzPolicyAssignment $policyDefinition.Id

```

3. Remove the network group with Remove-AzNetworkManagerGroup.

```azurepowershell-interactive
Remove-AzNetworkManagerGroup $networkGroup.Id
```

4. Delete the network manager instance with Remove-AzNetworkManager.

```azurepowershell-interactive
Remove-AzNetworkManager $networkManager.Id
```

```azurepowershell-interactive
RemoveAzVirtualNetwork -Name VNetA -ResourceGroupName $rg.Name
RemoveAzVirtualNetwork -Name VNetB -ResourceGroupName $rg.Name
RemoveAzVirtualNetwork -Name VNetC -ResourceGroupName $rg.Name
RemoveAzVirtualNetwork -Name VNetD -ResourceGroupName $rg.Name
RemoveAzVirtualNetwork -Name VNetE -ResourceGroupName $rg.Name
```

6. If you no longer need any resources under the group the network manager belongs to, delete the resource group with [Remove-AzResourceGroup](/powershell/module/az.resources/remove-azresourcegroup).

```azurepowershell-interactive
Remove-AzResourceGroup -Name $rg.Name
```

## Next steps

After you've created the Azure Virtual Network Manager, continue on to learn how to block network traffic by using the security admin configuration:

> [!div class="nextstepaction"]
[Block network traffic with security admin rules](how-to-block-network-traffic-portal.md)
[Create a secured hub and spoke network](tutorial-create-secured-hub-and-spoke.md)
