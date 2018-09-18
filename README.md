# azureworkshopsecurity

## 0. Environment preparation
### Create multiple RGs
    for i in {1..30}; 
    do
    az group create --name malworkshoprg$i --location eastus
    done

## 1. Building the basic environment

#### Create the VNET and Subnet1
    az network vnet create \
    --name malworkshoprg1vnet \
    --resource-group malworkshoprg1 \
    --location eastus \
    --address-prefix 192.168.0.0/16 \
    --subnet-name subnet1 \
    --subnet-prefix 192.168.1.0/24

### Create the Subnet2
    az network vnet subnet create \
    --address-prefix 192.168.2.0/24 \
    --name Subnet2 \
    --resource-group malworkshoprg1 \
    --vnet-name malworkshoprg1vnet

### Checking out your new VNET
    az network vnet show \
    -g malworkshoprg1 \
    -n malworkshoprg1vnet \
    --query '{Name:name,Where:location,Group:resourceGroup,Status:provisioningState,SubnetCount:subnets | length(@)}' \
    -o table

### Create the NSG for Subnet1
    az network nsg create \
    --resource-group malworkshoprg1 \
    --name malworkshoprg1nsg \
    --location eastus

### Create the NSG for Subnet2
    az network nsg create \
    --resource-group malworkshoprg1 \
    --name malworkshoprg1nsg2 \
    --location eastus

### Checking the default rules from NSG
    az network nsg show \
    -g malworkshoprg1 \
    -n malworkshoprg1nsg \
    --query 'defaultSecurityRules[].{Access:access,Desc:description,DestPortRange:destinationPortRange,Direction:direction,Priority:priority}' \
    -o table

### Adding RDP access for Subnet1 (JumpBox Server)
    az network nsg rule create \
    --resource-group malworkshoprg1 \
    --nsg-name malworkshoprg1nsg \
    --name rdp-rule \
    --access Allow \
    --protocol Tcp \
    --direction Inbound \
    --priority 100 \
    --source-address-prefix Internet \
    --source-port-range "*" \
    --destination-address-prefix "*" \
    --destination-port-range 3389

### Adding SSH access for Subnet2 (Web Servers)
    az network nsg rule create \
    --resource-group malworkshoprg1 \
    --nsg-name malworkshoprg1nsg2 \
    --name ssh-rule \
    --access Allow \
    --protocol Tcp \
    --direction Inbound \
    --priority 101 \
    --source-address-prefix Internet \
    --source-port-range "*" \
    --destination-address-prefix "*" \
    --destination-port-range 22

### Addint HTTP (80) access for Subnet2 (Web Servers)
    az network nsg rule create \
    --resource-group malworkshoprg1 \
    --nsg-name malworkshoprg1nsg2 \
    --name web-rule \
    --access Allow \
    --protocol Tcp \
    --direction Inbound \
    --priority 200 \
    --source-address-prefix Internet \
    --source-port-range "*" \
    --destination-address-prefix "*" \
    --destination-port-range 80

### Checking the UPDATED rules from NSG for Subnet2 (Web Servers)
    at Azure Portal

### Bind NSG1 (malworkshoprg1nsg1) with the Subnet1
    az network vnet subnet update \
    --vnet-name malworkshoprg1vnet \
    --name subnet1 \
    --resource-group malworkshoprg1 \
    --network-security-group malworkshoprg1nsg

### Bind NSG2 (malworkshoprg1nsg2) with the Subnet2
    az network vnet subnet update \
    --vnet-name malworkshoprg1vnet \
    --name Subnet2 \
    --resource-group malworkshoprg1 \
    --network-security-group malworkshoprg1nsg2


### Create Windows VM (JumpBox Server)
### Before creating the VM ...
### List Windows image VMs available (optional)
    az vm image list --offer Windows --all --output table

### List VM sizes (we'll need to choose one)
    az vm list-sizes --location eastus --output table

### Create the public ip for the VM
    az network public-ip create --resource-group malworkshoprg1 \
    --name malworkshoprg1pip --allocation-method dynamic --idle-timeout 4

### Create the NIC for the VM (NIC, VM, Public IP are different resources)
    az network nic create \
    -n malworkshoprg1nicvmwin \
    -g malworkshoprg1 \
    --subnet subnet1 \
    --public-ip-address malworkshoprg1pip \
    --vnet-name malworkshoprg1vnet

### Finally, Let's create the VM (WARNING: VM name cannot have more than 15 characters)
    az vm create \
    --resource-group malworkshoprg1 \
    --name malwsrg1vmwin \
    --image win2016datacenter \
    --size Standard_B1s \
    --nics malworkshoprg1nicvmwin  \
    --data-disk-sizes-gb 128 \
    --admin-username malworkshoprg1user \
    --admin-password Go010101you! \
    --authentication-type password

### Create Linux VMs in HA - Availability Set (Web Servers)
#### Create the Availability Set
    az vm availability-set create \
        --resource-group malworkshoprg1 \
        --name malworkshoprg1as \
        --platform-fault-domain-count 2 \
        --platform-update-domain-count 2
    
### Create 2 VMS Linux

### Create the public ip and nic for each VM
### VM Linux1
    az network public-ip create --resource-group malworkshoprg1 \
    --name vmlinuxpip1 --allocation-method dynamic --idle-timeout 4

    az network nic create \
        -n vmlinuxnic1 \
        -g malworkshoprg1 \
        --subnet Subnet2 \
        --network-security-group malworkshoprg1nsg2 \
        --public-ip-address vmlinuxpip1 \
        --vnet-name malworkshoprg1vnet

### VM Linux2
    az network public-ip create --resource-group malworkshoprg1 \
    --name vmlinuxpip2 --allocation-method dynamic --idle-timeout 4

    az network nic create \
        -n vmlinuxnic2 \
        -g malworkshoprg1 \
        --subnet Subnet2 \
        --network-security-group malworkshoprg1nsg2 \
        --public-ip-address vmlinuxpip2 \
        --vnet-name malworkshoprg1vnet
        
 ### Create VM Linux1 and Linux2 (inside the same AS)
    az vm create \
    --resource-group malworkshoprg1 \
    --os-disk-name osdisk1 \
    --name vmlinux1 \
    --availability-set malworkshoprg1as \
    --nics vmlinuxnic1 \
    --storage-sku standard_lrs \
    --size Standard_DS1_v2  \
    --image UbuntuLTS \
    --admin-username malworkshoprg1user \
    --admin-password Go010101you! \
    --authentication-type password
    
    az vm create \
    --resource-group malworkshoprg1 \
    --os-disk-name osdisk2 \
    --name vmlinux2 \
    --availability-set malworkshoprg1as \
    --nics vmlinuxnic2 \
    --storage-sku standard_lrs \
    --size Standard_DS1_v2  \
    --image UbuntuLTS \
    --admin-username malworkshoprg1user \
    --admin-password Go010101you! \
    --authentication-type password

### Install Apache on both Linux VMs (access through SSH on Bash from Azure Portal)
    ssh malworkshoprg"x"user@publicip
    sudo apt update
    sudo apt install apache2
    (execute twice the procedure above, once in each VM)

### Create Storage Account (our blob storage - not the same as stgforbash)
    at portal

### Create MS-SQL DB (PaaS)
    at portal

### Create a Web App PaaS service
    at portal

## 2. STARTING WITH SECURITY

### Encrypting VM disk (for Linux Server)
### Creating key vault and key
    az provider register -n Microsoft.KeyVault

    az keyvault create \
        --name malworkshoprg1kv2 \
        --resource-group malworkshoprg1 \
        --location eastus \
        --enabled-for-disk-encryption True

    az keyvault key create --vault-name malworkshoprg1kv2 --name keyrg1 --protection software
    
### Create an Azure Active Directory service principal
    az ad sp create-for-rbac

### result after command run (THAT'S ONLY AN EXAMPLE):
      "appId": "1376e4ce-e817-4074-ba12-d9651851b1a2",
      "displayName": "azure-cli-2018-09-17-19-56-50",
      "name": "http://azure-cli-2018-09-17-19-56-50",
      "password": "ba1c9b2c-222d-47db-9325-fd8a51b7a549",
      "tenant": "cadb8fc3-740e-4f2d-b0d5-473b447179ba"
      
### continuing ... (creating the variables)
    read sp_id sp_password <<< $(az ad sp create-for-rbac --query [appId,password] -o tsv)

### Setting permission on key vault for service principal
    az keyvault set-policy --name malworkshoprg1kv2 --spn $sp_id \
      --key-permissions wrapKey \
      --secret-permissions set
      
### Encrypting the Virtual Machine (bringing all together)
    To encrypt the virtual disks, you bring together all the previous components:
    Specify the Azure Active Directory service principal and password.
    Specify the Key Vault to store the metadata for your encrypted disks.
    Specify the cryptographic keys to use for the actual encryption and decryption.
    Specify whether you want to encrypt the OS disk, the data disks, or all.
    
    az vm encryption enable \
    --resource-group malworkshoprg1 \
    --name malwslinuxvm \
    --aad-client-id $sp_id \
    --aad-client-secret $sp_password \
    --disk-encryption-keyvault malworkshoprg1kv2 \
    --key-encryption-key keyrg1 \
    --volume-type all
    
### monitor the process of encrypting
    az vm encryption show --resource-group malworkshoprg1 --name malwslinuxvm
    
### when the status appearing as "pending" , you can reboot the VM
    az vm restart --resource-group malworkshoprg1 --name malwslinuxvm
    
### confirm that your disk is encrypted
    az vm encryption show --resource-group malworkshoprg1 --name malwslinuxvm
    
