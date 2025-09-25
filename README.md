# Powershell-VM-creation-script
VM-Vnet,NSG,Disk attachment- Powershell script

# Variables
$resourceGroup = "resgrp02"
$location = "centralus"
$vmName = "MyLinuxVM"
$subnetName = "MySubnet"
$vnetName = "MyVNet"
$publicIpName = "MyPublicIP"
$nicName = "MyNIC"
$securityGroupName = "MyNSG"

# Create Resource Group
New-AzResourceGroup -Name $resourceGroup -Location $location

# Create VNet + Subnet
$subnetConfig = New-AzVirtualNetworkSubnetConfig -Name $subnetName -AddressPrefix "10.0.0.0/24"
$vnet = New-AzVirtualNetwork -Name $vnetName -ResourceGroupName $resourceGroup -Location $location `
    -AddressPrefix "10.0.0.0/16" -Subnet $subnetConfig

# Create Public IP
$publicIp = New-AzPublicIpAddress -Name $publicIpName -ResourceGroupName $resourceGroup `
    -Location $location -AllocationMethod Static

# Create NSG with SSH Rule
$nsgRuleSSH = New-AzNetworkSecurityRuleConfig -Name "Allow-SSH" -Protocol "Tcp" `
    -Direction Inbound -Priority 1000 -SourceAddressPrefix * -SourcePortRange * `
    -DestinationAddressPrefix * -DestinationPortRange 22 -Access Allow

$nsg = New-AzNetworkSecurityGroup -ResourceGroupName $resourceGroup -Location $location `
    -Name $securityGroupName -SecurityRules $nsgRuleSSH

# Create NIC
$nic = New-AzNetworkInterface -Name $nicName -ResourceGroupName $resourceGroup `
    -Location $location -SubnetId $vnet.Subnets[0].Id -PublicIpAddressId $publicIp.Id -NetworkSecurityGroupId $nsg.Id

# Credentials for VM
$cred = Get-Credential -Message "Enter username and password for the VM."

# VM Config
$vmConfig = New-AzVMConfig -VMName $vmName -VMSize "Standard_B1s"

# Add OS Disk (Linux Ubuntu 20.04)
$vmConfig = Set-AzVMOperatingSystem -VM $vmConfig -Linux -ComputerName $vmName -Credential $cred
$vmConfig = Set-AzVMSourceImage -VM $vmConfig -PublisherName "Canonical" -Offer "0001-com-ubuntu-server-focal" `
    -Skus "20_04-lts-gen2" -Version "latest"

# Attach NIC
$vmConfig = Add-AzVMNetworkInterface -VM $vmConfig -Id $nic.Id

# Add OS Disk explicitly
$vmConfig = Set-AzVMOSDisk -VM $vmConfig -Name "${vmName}-OSDisk" -CreateOption FromImage -Caching ReadWrite

# Create VM
New-AzVM -ResourceGroupName $resourceGroup -Location $location -VM $vmConfig
