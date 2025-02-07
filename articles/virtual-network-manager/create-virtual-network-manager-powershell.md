---
title: 'Quickstart: Create a mesh network with Azure Virtual Network Manager using Azure PowerShell'
description: Use this quickstart to learn how to create a mesh network with Virtual Network Manager using Azure PowerShell.
author: mbender-ms
ms.author: mbender
ms.service: virtual-network-manager
ms.topic: quickstart
ms.date: 08/9/2022
ms.custom: template-quickstart, ignite-fall-2021, mode-api
---

# Quickstart: Create a mesh network with Azure Virtual Network Manager using Azure PowerShell

Get started with Azure Virtual Network Manager by using the Azure Powershell to manage connectivity for all your virtual networks.

In this quickstart, you'll create groups of three virtual networks (manually, and dynamically) and use Azure Virtual Network Manager to apply a mesh network topology on them. Then, you'll verify if the connectivity configuration got applied.

> [!IMPORTANT]
> Azure Virtual Network Manager is currently in public preview.
> This preview version is provided without a service level agreement, and it's not recommended for production workloads. Certain features might not be supported or might have constrained capabilities.
> For more information, see [Supplemental Terms of Use for Microsoft Azure Previews](https://azure.microsoft.com/support/legal/preview-supplemental-terms/).

## Prerequisites

* An Azure account with an active subscription. [Create an account for free](https://azure.microsoft.com/free/?WT.mc_id=A261C142F).
* During preview, the `4.15.1-preview` version of `Az.Network` is required to access the required cmdlets.
* If you're running PowerShell locally, you also need to run `Connect-AzAccount` to create a connection with Azure.

> [!IMPORTANT]
> Perform this quickstart using Powershell locally, not through Azure Cloud Shell. The version of `Az.Network` in Azure Cloud Shell does not currently support the Azure Virtual Network Manager cmdlets.

## Install Azure PowerShell module

Install the latest *Az.Network* Azure PowerShell module using this command:

```azurepowershell-interactive
 Install-Module -Name Az.Network -RequiredVersion 4.15.1-preview -AllowPrerelease
```

##  Set context to your subscription.

A network manager must exist under one subscription. Resources **managed** by that network manager, such as VNets, can exist across several other subscriptions and tenants.
You'll switch between **one manager subscription,** and **one target subscription.** (These can be the same, if you so choose).

Switch to the subscription owning your network manager.

```azurecli-interactive
SetAzContext --subscription "{manager subscription ID}"
```

## Create a resource group

Before you can create an Azure Virtual Network Manager, you have to create a resource group to host the Network Manager. Create a resource group with [New-AzResourceGroup](/powershell/module/az.Resources/New-azResourceGroup). This example creates a resource group named **managerAVNMResourceGroup** in the **WestUS** location.

```azurepowershell-interactive

$location = "West US"
$rg = @{
    Name = 'managerAVNMResourceGroup'
    Location = $location
}
New-AzResourceGroup @rg

```

## Create a Virtual Network Manager

Define the scope and access type this Network Manager instance will have. Virtual Network Managers can manage **management groups** (which can contain other management groups, and many subscriptions under them)) **subscriptions**, or a combination of both. Replace {mgName} and {subId} with any management groups or subscriptions you want to manage virtual networks under.

```azurepowershell-interactive

Import-Module -Name Az.Network -RequiredVersion "4.15.1"

[System.Collections.Generic.List[string]]$subGroup = @()  
$subGroup.Add("/subscriptions/{subId}")
[System.Collections.Generic.List[string]]$mgGroup = @()  
$mgGroup.Add("/providers/Microsoft.Management/managementGroups/{mgName}")

[System.Collections.Generic.List[String]]$access = @()  
$access.Add("Connectivity");  
$access.Add("SecurityAdmin"); 

$scope = New-AzNetworkManagerScope -Subscription $subGroup -ManagementGroup $mgGroup

```

1. Create the Virtual Network Manager with New-AzNetworkManager. This example creates an Azure Virtual Network Manager named **myAVNM** in the West US location.
    
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

## Create a Network Group

Virtual Network Manager applies configurations to groups of VNets by placing them in **Network Groups.** Create a Network Group with New-AzNetworkManagerGroup.

```azurepowershell-interactive
$ng = @{
    Name = 'myNetworkGroup'
    ResourceGroupName = $rg.Name
    NetworkManagerName = $networkManager.Name
    MemberType = 'Microsoft.Network/VirtualNetworks'
}
$networkgroup = New-AzNetworkManagerGroup @ng
```
    
## Create Virtual Networks to be managed by your Network Manager.

Switch to your managed subscription. This subscription should be under the network manager scopes you defined above.

```azurecli-interactive
SetAzContext --subscription "{target subscription ID}"
```

Resources under these managed subscriptions also need to exist in resource groups under those subscriptions. Create a resource group with [az group create](/cli/azure/group#az-group-create). This example creates a resource group named **targetAVNMResourceGroup** in the **westus** location:

```azurecli-interactive
New-AzResourceGroup --name targetAVNMResourceGroup --location "West US"
```

Create three virtual networks with [New-AzVirtualNetwork](/powershell/module/az.network/new-azvirtualnetwork). This example creates virtual networks named **VNetA**, **VNetB** and **VNetC** in the **West US** location, given the arbitrary tag **Color=Red** or **Color=Blue** to later demonstrate policy. If you already have virtual networks you want create a mesh network with, you can skip to the next section.

```azurepowershell-interactive
$vnetA = @{
    Name = 'VNetA'
    ResourceGroupName = 'myAVNMResourceGroup'
    Location = $location
    AddressPrefix = '10.0.0.0/16'  
    Tag = @{
	   Color = "Red"
    }	  
}

$virtualNetworkA = New-AzVirtualNetwork @vnetA

$vnetB = @{
    Name = 'VNetB'
    ResourceGroupName = 'myAVNMResourceGroup'
    Location = $location
    AddressPrefix = '10.1.0.0/16'
    Tag = @{
	   Color = "Red"
    }	  
}
$virtualNetworkB = New-AzVirtualNetwork @vnetB

$vnetC = @{
    Name = 'VNetC'
    ResourceGroupName = 'myAVNMResourceGroup'
    Location = $location
    AddressPrefix = '10.2.0.0/16'
    Tag = @{
	   Color = "Red"
    }		  
}
$virtualNetworkC = New-AzVirtualNetwork @vnetC
```

Optionally, create virtual networks called **Unscoped_VNet** and **VNt** to see how Policy can dynamically include/exclude VNets managed by your Virtual Network Manager.

```
$Unscoped_VNet = @{
    Name = 'Unscoped_VNet'
    ResourceGroupName = 'myAVNMResourceGroup'
    Location = $location
    AddressPrefix = '10.3.0.0/16'
    Tag = @{
	   Color = "Blue"
    }		  
}
$virtualNetworkUnscoped = New-AzVirtualNetwork @Unscoped_VNet

$VNt = @{
    Name = 'VNt'
    ResourceGroupName = 'myAVNMResourceGroup'
    Location = $location
    AddressPrefix = '10.4.0.0/16'
    Tag = @{
	   Color = "Red"
    }	  
}
$virtualNetworkUnscoped = New-AzVirtualNetwork @Unscoped_VNet
```

### Add a subnet to each virtual network

To complete the configuration of the virtual networks, add a /24 subnet to each one. Create a subnet configuration named **default** with [Add-AzVirtualNetworkSubnetConfig](/powershell/module/az.network/add-azvirtualnetworksubnetconfig).

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
        
## Option 1 (Static Membership): Manually add the 3 VNets for your Mesh configuration to the Network Group.

Using **static membership**, you'll directly add 3 VNets to your Network Group.

Switch back to the subscription owning your Network Manager.

```azurepowershell-interactive
SetAzContext --subscription "{manager subscription ID}"
```
    
Add these static members to the Network Group. 
> [!NOTE]
> Static members must have a network group scoped unique name. It's recommended to use a consistent hash of the virtual network ID. Below is an approach using the ARM Templates uniqueString() implementation.
   
```azurepowershell-interactive
function Get-UniqueString ([string]$id, $length=13)
{
$Array = (new-object System.Security.Cryptography.SHA512Managed).ComputeHash($id.ToCharArray())
-join ($hashArray[1..$length] | ForEach-Object { [char]($_ % 26 + [byte][char]'a') })
}
```
       
```azurepowershell-interactive
$smA = @{
    Name = Get-UniqueString $virtualNetworkA.Id
    ResourceGroupName = $rg.Name
    NetworkGroupName = $networkGroup.Name
    NetworkManagerName = $networkManager.Name
    ResourceId = $virtualNetworkA.Id
}
$statimemberA = New-AzNetworkManagerStaticMember @sm
```
        
```azurepowershell-interactive
$smB = @{
    Name = Get-UniqueString $virtualNetworkB.Id
    ResourceGroupName = $rg.Name
    NetworkGroupName = $networkGroup.Name
    NetworkManagerName = $networkManager.Name
    ResourceId = $virtualNetworkB.Id
}
$statimemberB = New-AzNetworkManagerStaticMember @sm
```
    
```azurepowershell-interactive
$smC = @{
    Name = Get-UniqueString $virtualNetworkC.Id
    ResourceGroupName = $rg.Name
    NetworkGroupName = $networkGroup.Name
    NetworkManagerName = $networkManager.Name
    ResourceId = $virtualNetworkC.Id
}
$statimemberC = New-AzNetworkManagerStaticMember @sm
```
    
## Option 2 (Dynamic Membership): Using Azure Policy, dynamically add the 3 VNets for your Mesh configuration to the Network Group.

Using Dynamic Membership through Azure Policy, you'll create a policy to find and automatically add the right VNets to your network group.

Switch to the subscription owning your network manager.

```azurepowershell-interactive
SetAzContext --subscription "{manager subscription ID}"
```

With Azure Policy, create a policy to accept only VNets with the tag "Color:Red" and "VNet" in the name.  This policy will accept VNetA, VNetB and VNetC, but reject the Unscoped_VNet and VNt. 

Policies can be applied to a subscription or management group, and must always be defined _at or above_ the level they're created. Only virtual networks within a policy scope are added to a Network Group.

1. Define the conditional statement and store it in a variable.
> [!NOTE]
> It is recommended to scope all of your conditionals to only scan for type `Microsoft.Network/virtualNetwork` for efficiency.

```azurepowershell-interactive
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
	"field": "tags[''Color'']",
	"equals": "Red"
	}
    ] 
}' 
```
        
Create the Azure Policy definition using the conditional statement defined in the last step using New-AzPolicyDefinition. Replace {mgName} with the management group you want to apply this policy to. If you want to apply it to a subscription, replace the `ManagementGroup = {mgName}` parameter with `Subscription = {subId}`

> [!IMPORTANT]
> Policy resources must have a scope unique name. It is recommended to use a consistent hash of the network group. Below is an approach using the ARM Templates uniqueString() implementation.
   
```azurepowershell-interactive
    function Get-UniqueString ([string]$id, $length=13)
    {
    $hashArray = (new-object System.Security.Cryptography.SHA512Managed).ComputeHash($id.ToCharArray())
    -join ($hashArray[1..$length] | ForEach-Object { [char]($_ % 26 + [byte][char]'a') })
    }
```

```azurepowershell-interactive
$defn = @{
    Name = Get-UniqueString $networkgroup.Id
    Mode = 'Microsoft.Network.Data'
    Policy = $conditionalMembership
    ManagementGroup = {mgName}
}
    
$policyDefinition = New-AzPolicyDefinition $defn 
```

Once a policy is defined, it must also be applied. Replace {mgName} with the management group you want to apply this policy to. If you want to apply it to a subscription, replace the `Scope = "/providers/Microsoft.Management/managementGroups/{mgName}` parameter with `Scope = "/subscriptions/{subId}"`.

```azurepowershell-interactive
$assgn = @{
Name = Get-UniqueString $networkgroup.Id
PolicyDefinition  = $policyDefinition
Scope = "/providers/Microsoft.Management/managementGroups/{mgName}"
}

$policyAssignment = New-AzPolicyAssignment $assgn
```
        
## Create a configuration

Now that the Network Group is created, and has the correct VNets, configurations can be applied to this group (and all VNets under it).

1. Create a connectivity group item to add a network group to with New-AzNetworkManagerConnectivityGroupItem.

```azurepowershell-interactive
$gi = @{
NetworkGroupId = $networkgroup.Id
}
$groupItem = New-AzNetworkManagerConnectivityGroupItem @gi
```
    
1. Create a configuration group and add the group item from the previous step.

```azurepowershell-interactive
[System.Collections.Generic.List[Microsoft.Azure.Commands.Network.Models.PSNetworkManagerConnectivityGroupItem]]$configGroup = @()
$configGroup.Add($groupItem)
```
    
1. Create the connectivity configuration with New-AzNetworkManagerConnectivityConfiguration.

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

Commit the configuration to the target regions with Deploy-AzNetworkManagerCommit. This will trigger your configuration to begin taking effect.

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

## Verify that the configurations have been deployed as intended.

Switch back to the subscription owning your Virtual Networks.

```azurepowershell-interactive
SetAzContext --subscription "{target subscription ID}"
```

Verify these configurations have been deployed.

```
Get-AzNetworkManagerEffectiveConnectivityConfigurationList -ResourceGroupName "targetAVNMResourceGroup" -VirtualNetworkName "VNetA"
Get-AzNetworkManagerEffectiveConnectivityConfigurationList -ResourceGroupName "targetAVNMResourceGroup" -VirtualNetworkName "VNetB"
Get-AzNetworkManagerEffectiveConnectivityConfigurationList -ResourceGroupName "targetAVNMResourceGroup" -VirtualNetworkName "VNetC"
```

Optionally, if you created **Unscoped_VNet** and **VNt** earlier, you can check that configurations have _not_ been applied to them accordingly:
```
Get-AzNetworkManagerEffectiveConnectivityConfigurationList -ResourceGroupName "targetAVNMResourceGroup" -VirtualNetworkName "Unscoped_VNet"
Get-AzNetworkManagerEffectiveConnectivityConfigurationList -ResourceGroupName "targetAVNMResourceGroup" -VirtualNetworkName "VNt"
```

## Clean up resources

If you no longer need the Azure Virtual Network Manager, you'll need to make sure all of following is true before you can delete the resource:

* There are no deployments of configurations to any region.
* All configurations have been deleted.
* All network groups have been deleted.

Switch back to the subscription owning your Network Manager.

```azurepowershell-interactive
SetAzContext --subscription "{manager subscription ID}"
```

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

5. If you no longer need any resources under the group the network manager belongs to, delete the resource group with [Remove-AzResourceGroup](/powershell/module/az.resources/remove-azresourcegroup).

```azurepowershell-interactive
Remove-AzResourceGroup -Name 'managerAVNMResourceGroup'
```
Switch back to the subscription owning your Virtual Networks.

```azurepowershell-interactive
SetAzContext --subscription "{target subscription ID}"
```

1. Delete the sample VNets for testing with 

```azurepowershell-interactive
RemoveAzVirtualNetwork -Name VNetA -ResourceGroupName 'targetAVNMResourceGroup'
RemoveAzVirtualNetwork -Name VNetB -ResourceGroupName 'targetAVNMResourceGroup'
RemoveAzVirtualNetwork -Name VNetC -ResourceGroupName 'targetAVNMResourceGroup'
RemoveAzVirtualNetwork -Name Unscoped_VNet -ResourceGroupName 'targetAVNMResourceGroup'
RemoveAzVirtualNetwork -Name VNt -ResourceGroupName 'targetAVNMResourceGroup'
```
2. If you no longer need any resources under the group the virtual networks belongs to, delete the resource group with [Remove-AzResourceGroup](/powershell/module/az.resources/remove-azresourcegroup).

```azurepowershell-interactive
Remove-AzResourceGroup -Name 'targetAVNMResourceGroup'
```

## Next steps

> [!div class="nextstepaction"]
> Learn how to [Block network traffic with security admin rules](how-to-block-network-traffic-powershell.md)
