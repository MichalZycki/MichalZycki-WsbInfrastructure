# WsbInfrastructure

# Set up variables
$subscriptionId = "xxxxxxxxxxxxxxxxx"  
$wsbid = "xxxxxxxxxxx"  
$resourceGroupName = "wsbrg$wsbid"  
$location = "northeurope"  
$vmName = "sqlvm$wsbid"  
$adminUsername = "sqladmin"  
$adminPassword = "P@ssword123$wsbid"  
$vnetName = "wsbvnet$wsbid"  
$subnetName = "wsbsubnet$wsbid"  
$publicIpName = "wsbpublicIP$wsbid"  
$nsgName = "wsbnsg$wsbid"  
$nicName = "wsbnic$wsbid"  
$vmSize = "Standard_D8as_v4" #Standard_D8as_v4 / Standard_D2s_v3

# Set the subscription context
Set-AzContext -SubscriptionId $subscriptionId

# Create the resource group
New-AzResourceGroup -Name $resourceGroupName -Location $location

# Create a virtual network
$vnet = New-AzVirtualNetwork -ResourceGroupName $resourceGroupName -Location $location -Name $vnetName -AddressPrefix "10.0.0.0/16"

# Create a subnet
$subnet = Add-AzVirtualNetworkSubnetConfig -Name $subnetName -AddressPrefix "10.0.0.0/24" -VirtualNetwork $vnet
$vnet | Set-AzVirtualNetwork

# Retrieve the subnet ID after setting the virtual network
$subnetId = $vnet.Subnets[0].Id

# Retrieve the updated virtual network to ensure changes are committed
$vnet = Get-AzVirtualNetwork -ResourceGroupName $resourceGroupName -Name $vnetName

# Retrieve the subnet ID (ensure it's now available)
$subnetId = ($vnet.Subnets | Where-Object { $_.Name -eq $subnetName }).Id

# Create a public IP address with Static allocation (RDP will use this IP)
$publicIp = New-AzPublicIpAddress -Name $publicIpName -ResourceGroupName $resourceGroupName -Location $location -AllocationMethod Static

# Create a Network Security Group (NSG)
$nsg = New-AzNetworkSecurityGroup -ResourceGroupName $resourceGroupName -Location $location -Name $nsgName

# Add an NSG rule to allow inbound RDP (Port 3389) from any IP
$nsgRuleRDP = New-AzNetworkSecurityRuleConfig -Name "Allow-RDP" `
    -Description "Allow RDP" -Access Allow -Protocol Tcp -Direction Inbound `
    -Priority 1000 -SourceAddressPrefix "*" -SourcePortRange "*" `
    -DestinationAddressPrefix "*" -DestinationPortRange 3389

# Add the rule to the NSG
$nsg.SecurityRules.Add($nsgRuleRDP)

# Update the NSG with the new rule
Set-AzNetworkSecurityGroup -NetworkSecurityGroup $nsg

# Create a NIC (Network Interface) and associate it with the subnet and public IP
$nic = New-AzNetworkInterface -ResourceGroupName $resourceGroupName -Location $location -Name $nicName `
    -SubnetId $subnetId -PublicIpAddressId $publicIp.Id -NetworkSecurityGroupId $nsg.Id

# Define the virtual machine configuration
$vmConfig = New-AzVMConfig -VMName $vmName -VMSize $vmSize

# Set credentials for the SQL VM
$securePassword = ConvertTo-SecureString $adminPassword -AsPlainText -Force
$cred = New-Object System.Management.Automation.PSCredential ($adminUsername, $securePassword)

# Set the operating system
$vmConfig = Set-AzVMOperatingSystem -VM $vmConfig -Windows -ComputerName $vmName -Credential $cred -ProvisionVMAgent -EnableAutoUpdate

# Set the network interface
$vmConfig = Add-AzVMNetworkInterface -VM $vmConfig -Id $nic.Id

# Set the SQL image (choose the SQL VM image from the Azure marketplace)
$vmConfig = Set-AzVMSourceImage -VM $vmConfig -PublisherName "MicrosoftSQLServer" -Offer "sql2022-ws2022" -Skus "sqldev-gen2" -Version "latest"

# Turn off Boot Diagnostics (disable boot diagnostics)
$vmConfig = Set-AzVMBootDiagnostic -VM $vmConfig -Enable $false

# Create the VM
New-AzVM -ResourceGroupName $resourceGroupName -Location $location -VM $vmConfig

# Configure the VM as a SQL Server VM
$sqlVm = New-AzSqlVM -ResourceGroupName $resourceGroupName -Name $vmName -Location $location -LicenseType "PAYG" -SQLManagementType "Full"
