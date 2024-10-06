To update the hostname of a node in an Azure Kubernetes Service (AKS) cluster, you can modify the hostname directly on the Virtual Machine Scale Set (VMSS) nodes. However, the change may only persist until the node is recreated or scaled down, as the AKS cluster can dynamically create and delete nodes. If you need a persistent hostname format for your AKS nodes, you might want to incorporate a script or a method that runs during node initialization to consistently set the hostname.

Steps to Update the Hostname of an AKS Node:

Option 1: Update Hostname Temporarily

If you just want to change the hostname temporarily (until the node is rebooted or replaced), you can SSH into the node and manually change the hostname.

1. SSH into the Node: Follow the steps I provided earlier to SSH into the AKS node:

ssh -i ~/.ssh/id_rsa azureuser@<node-ip>


2. Update the Hostname: Use the following command to update the hostname temporarily:

sudo hostnamectl set-hostname <new-hostname>


3. Verify the Change: Verify that the hostname has changed using:

hostname

However, this change will be reverted when the node is replaced, recreated, or rebooted. If you want the change to persist, use Option 2 below.



Option 2: Update Hostname Persistently Using Custom Script in VMSS

For persistent hostname updates, you can use the Custom Script Extension or cloud-init to automate the hostname change whenever a node is created or rebooted.

1. Create a Custom Script: Prepare a script to set the hostname based on a pattern or increment (e.g., server001, server002, etc.):

#!/bin/bash
hostnamectl set-hostname <new-hostname-pattern>

For example, if you want a hostname based on an environment variable (e.g., node ID):

#!/bin/bash
NODE_ID=$(curl -s -H "Metadata:true" "http://169.254.169.254/metadata/instance/compute/name?api-version=2021-02-01&format=text")
hostnamectl set-hostname "$NODE_ID"


2. Store the Script: You can store this script in Azure Blob Storage or any accessible location that the AKS nodes can reach.


3. Apply the Script Using VMSS Custom Script Extension: Use the az vmss extension set command to apply the script as part of the VM scale set. This ensures that every time a node is created or restarted, the hostname is updated.

az vmss extension set \
  --resource-group <node-resource-group> \
  --vmss-name <vmss-name> \
  --name CustomScript \
  --publisher Microsoft.Azure.Extensions \
  --version 2.1 \
  --settings '{"fileUris": ["<custom-script-url>"], "commandToExecute": "bash myscript.sh"}'

Replace <node-resource-group> with the resource group of your AKS node.

Replace <vmss-name> with the name of the VM scale set.

Replace <custom-script-url> with the location of your script (e.g., Azure Blob Storage URL).

Replace myscript.sh with the actual script filename.



4. Update the VMSS: Once the custom script extension is added, update the VMSS to ensure all new nodes will run the script:

az vmss update \
  --resource-group <node-resource-group> \
  --name <vmss-name>



This will set the hostname of the node automatically whenever a new node is added to the AKS cluster or the VM is rebooted.

Option 3: Cloud-Init for Persistent Hostname Setting (At Node Creation)

You can also use cloud-init to configure the hostname during the creation of the VM instance. This approach is useful when using custom node images in AKS.

Define a cloud-init YAML configuration that sets the hostname based on the environment or VM information, and ensure it's applied during the node provisioning process.


Considerations:

Autoscaling and Node Replacements: In an AKS environment, nodes can be frequently added or removed due to autoscaling. Any hostname change should account for this dynamic behavior.

VMSS Model Update: Ensure any custom hostname logic is included in the VMSS model so that every new node is configured consistently.


Let me know if you'd like help with the specific script logic or further details!

