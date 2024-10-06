To achieve the behavior where changing the hostname inside the VM automatically updates the computer name shown in the Azure console, you'll need to use a process that updates both the hostname at the OS level and the metadata that Azure uses for the VM’s name.

Unfortunately, Azure does not automatically synchronize the VM’s hostname with the computer name shown in the portal. The computer name in Azure is typically set when the VM is created, and any changes made to the hostname after creation are not reflected in the Azure portal. To ensure that both the hostname and the computer name in the Azure console are updated automatically, you would need to implement a custom solution.

Here’s an approach you can take to achieve this:

1. Automatically Sync Hostname and Azure Computer Name

To ensure that when you update the hostname inside the VM it also reflects in the Azure portal, you can use a custom script extension that:

Detects hostname changes in the VM.

Updates the Azure computer name to match the new hostname.


Steps:

Step 1: Write a Script to Update Hostname and Sync with Azure

Create a script that:

Changes the hostname inside the VM.

Updates Azure’s metadata to reflect the new hostname.


Example of update_computer_name.sh:

#!/bin/bash

# Set the new hostname (use the parameter passed to the script or a default value)
NEW_HOSTNAME=$1

# Check if a new hostname was provided
if [ -z "$NEW_HOSTNAME" ]; then
  echo "No hostname provided. Exiting."
  exit 1
fi

# Step 1: Set the hostname locally
echo "Setting hostname to $NEW_HOSTNAME"
sudo hostnamectl set-hostname "$NEW_HOSTNAME"

# Step 2: Update /etc/hosts to reflect the new hostname
sudo sed -i "s/127.0.1.1.*/127.0.1.1 $NEW_HOSTNAME/g" /etc/hosts

# Step 3: Use the Azure CLI to update the VM's computer name in the Azure console
# Ensure that the Azure CLI is installed and logged in (az login)
VM_NAME=$(az vm list --query "[?osProfile.computerName=='$(hostname)'].name" -o tsv)
RESOURCE_GROUP=$(az vm list --query "[?osProfile.computerName=='$(hostname)'].resourceGroup" -o tsv)

echo "Updating Azure computer name to $NEW_HOSTNAME for VM $VM_NAME in Resource Group $RESOURCE_GROUP"
az vm update --resource-group "$RESOURCE_GROUP" --name "$VM_NAME" --set osProfile.computerName="$NEW_HOSTNAME"

# Step 4: Reboot the VM to ensure all services use the new hostname
sudo reboot

In this script:

The hostname is updated locally using hostnamectl.

The /etc/hosts file is updated for proper name resolution.

The Azure CLI is used to update the computer name in the Azure portal.

The VM is rebooted to apply the changes across the system.


Step 2: Run the Script Automatically on VM or VMSS Nodes

You can use the Azure Custom Script Extension or cloud-init to ensure that the script is executed whenever a new VM or VM scale set instance is created, or when the hostname is updated.

Using Custom Script Extension: To run the script on a VM or VM scale set instance, you can apply it using the Azure Custom Script Extension:

az vmss extension set \
  --resource-group <node-resource-group> \
  --vmss-name <vmss-name> \
  --name CustomScript \
  --publisher Microsoft.Azure.Extensions \
  --version 2.1 \
  --settings '{"fileUris": ["<script-url>"], "commandToExecute": "bash update_computer_name.sh aks-worker-001"}'

Replace:

<node-resource-group> with the resource group of the VM scale set.

<vmss-name> with the name of the VM scale set.

<script-url> with the URL to your script.


Step 3: Validate the Results

1. Test the Hostname Change: SSH into the VM and run the script to change the hostname:

sudo bash update_computer_name.sh new-hostname


2. Check the Console: After the script runs and the VM reboots, check the Azure portal to confirm that the computer name has been updated to match the new hostname.



Key Points:

Azure’s computer name is typically only set at VM creation. Changing the hostname within the VM does not automatically update it in the Azure console.

You can create a custom solution, such as a script that updates both the hostname inside the VM and the Azure computer name using the Azure CLI.

Rebooting the VM after the changes ensures that the new hostname is applied consistently across services.


This approach provides a way to keep the computer name in sync with any hostname changes inside the VM. Let me know if you need further clarification or help with setting this up!

