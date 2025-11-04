# Moving Blobs Between Private Azure Storage Accounts Across Regions
The complete process of securely moving blobs from a __private Azure Storage Account in East Asia__ to another __private Azure Storage Account in UAE North__, using __Azure Storage Explorer__ from a __VM in East Asia.__


# Resources and services used
* Azure subscription
* Two resource groups in East Asia and UAE North.
* Two storage accounts:Source: Private, in East Asia
                       Destination: Private, in UAE North
* A VM in East Asia with Azure Bastion access
* Azure Storage Explorer installed on the VM
* Two Private endpoints
* One Private DNS Zone
* Vnet Peering

## Create Azure Resources
__Create Virtual Networks and bastian host__
East Asia VNet: 10.0.0.0/16
In the vnet creation settings, select __Enable Azure Bastion__.
In Azure Bastion, eneter the name as __bastion__.
In Azure Bastion public IP address, Select __Create a public IP address__.
Enter __public-ip-bastion__ in Name.
Select OK.
In the IP Addresses tab, select default subnet and change name to __subnet-1__.
IPv4 address range:	Leave the default of 10.0.0.0/16.
Starting address:	Leave the default of 10.0.0.0.
Size:	Leave the default of /24 (256 addresses).
Select Review + create.

Do the same again to create another vnet in UAE North with a bastian.
UAE North VNet: Use a non overlapping address space, e.g., 10.1.0.0/16

__Create Storage Accounts and Private Endpoints__
Create both storage accounts in the two resource groups and two locations respectively and with public access disabled. 
Performance:	Leave the default Standard.
Redundancy:	Select Locally-redundant storage (LRS).
In the networking section, disable public access.
Add private endpoint for the blob subresource in their respective VNets.
Name:	Enter private-endpoint.
Network Interface Name:	Leave the default of private-endpoint-nic.
Region:	Select East Asia.

Repeat the same to create a SA in UAE North.

__Create Private DNS Zones__
Create an Azure Private DNS Zone for each storage account type (privatelink.blob.core.windows.net) 

Within the Private DNS Zone, create an "A" record that maps the storage account's FQDN (e.g., yourstorageaccount.blob.core.windows.net) to the private IP address of its corresponding Private Endpoint.

<img width="932" height="756" alt="image" src="https://github.com/user-attachments/assets/16bd3cc0-6407-44ac-bb9d-5ad33a38fc90" />


Next link Private DNS Zones to VNets:
In Private DNS Zone, navigate to "Virtual network links" under "Settings."
Add a link to the VNet where the Private Endpoint for that storage account is located.
Crucially, add a link to the peered VNet in the other region as well. This allows resources in the peered VNet to resolve the private DNS name of the storage account.

<img width="952" height="757" alt="image" src="https://github.com/user-attachments/assets/3e78df4d-3e21-4b3a-8662-404dbfbe2b3e" />

__Add blob containers__
In each SA, create a new container each and select __Private (no anonymous access)__ under Public access level.

__Create Virtual Machine in East Asia__
Deploy a VM in the East Asia VNet.
Use Azure Bastion to connect securely.

Virtual machine name:	Enter vm-1.
Region:	Select East Asia.
Availability options:	Select No infrastructure redundancy required.
Security type:	Leave the default of Standard.
Image:	Select Windows Server 2022 Datacenter - x64 Gen2.
VM architecture:	Leave the default of x64.
Size:	Select a size (B Series).
Authentication type:	Select Password.
Username:	Enter azureuser.
Password:	Enter a password.	
Public inbound ports	Select None.

In Networking section assign vnet1 and subnet1.
Virtual network:	Select vnet-1.
Subnet:	Select subnet-1 (10.0.0.0/24).
Public IP:	Select None.
NIC network security group:	Select Advanced.
Configure network security group:	Select Create new.
                Enter nsg-1 for the name.
                Leave the rest at the defaults and select OK.
                
__Storage access key__
The connection string is required for the storage explorer to connect to the two storage accounts. 
Go to each storage account. Under settings> access keys> copy the connection strings.

<img width="1122" height="661" alt="image" src="https://github.com/user-attachments/assets/b77fddba-afd3-489e-836b-0bc287256294" />

__Test connectivity to private endpoint__
In here we use the vm to connect to the storage accounts across the private endpoint using __Microsoft Azure Storage Explorer__. 
Go to the vm on the portal.
In Connect, select Bastion.
Enter credentials and select connect.
Open Windows PowerShell on the server after you connect
Enter
```
nslookup <storage-account-name>.blob.core.windows.net
```
Replace <storage-account-name> with the name of the storage account you created in the previous steps. The output of the command should return A private IP address of 10.0.0.10 is returned for the east asia storage account name.
Do the same for UAE SA.

<img width="892" height="462" alt="image" src="https://github.com/user-attachments/assets/7887638d-78d5-4977-8f06-62880afb4d6b" />

Install Microsoft Azure Storage Explorer on the virtual machine.
Open Storage Explorer, and select Power plug symbol to open the Select Resource dialog box in the left-hand toolbar.
Select __Storage account or service__  

<img width="1422" height="770" alt="image" src="https://github.com/user-attachments/assets/47b6665f-030b-4745-96d5-51ee516d4444" />

In the Select Connection Method screen, select Connection string.
Paste the connection string from the storage account.
Select Connect
Do the same for the other SA as well.
Now on the Storage Explorer you'd be able to see the two storage accs lised along with the created containers. 
Upload an image file to the container of east asia.
<img width="1073" height="591" alt="image" src="https://github.com/user-attachments/assets/7cfe2dbe-8b79-418e-a4b7-f926ac88bd88" />

Now to copy files from asia to UAE, we need to set up peering between the two vnets.

__Establish VNet Peering:__
Navigate to one of your VNets in the Azure portal.
Select "Peerings" under "Settings."
Click "+ Add" to create a new peering.
Configure the peering from the current VNet to the VNet in the other region.
Ensure "Allow forwarded traffic" is enabled on both sides of the peering if you intend for resources in one VNet to reach resources in the other VNet through the peering.


Repeat the process from the other VNet to establish a bidirectional peering.
Verify that the peering status shows "Connected" in both directions.

<img width="1113" height="562" alt="image" src="https://github.com/user-attachments/assets/2ba3ea9d-e227-4af5-84b5-19fdbedf97c0" />


__Use Storage Explorer to Move Blobs__

Navigate to the source container in East Asia SA
Right-click → Copy
Navigate to the destination container in UAE SA
Right-click → Paste

Note: This is a client-side copy. Data flows through the VM.

<img width="997" height="518" alt="image" src="https://github.com/user-attachments/assets/b0ac761d-91db-4ffb-aa7b-8a7f12841b81" />


