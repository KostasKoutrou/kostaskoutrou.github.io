

## Automate Proxmox

Regarding automating the configuration of Proxmox, this essentially includes:

### Initial Setup

This step includes the installation of Proxmox on a machine, setting up the network configuration, i.e., IP Address, Gateway, DNS, and setting up the root user credentials. You can always take a look at the description of the configuration to be achieved in my [previous post](https://kostaskoutrou.github.io/2026/02/02/cyber-range-as-code-part1.html).

One point of improvement here is to potentially automate the installation parameters. This is supported by Proxmox with the [Automated Installation](https://pve.proxmox.com/wiki/Automated_Installation). This would be a very interesting improvement point towards a fully hands-off installation and configuration. But this will be left as a future exercise.

After the initial setup and reaching the point of being able to sign in to the Proxmox GUI, the next step is to set up Proxmox so that Ansible can communicate with it.

This means:

1. Create a Proxmox firewall rule to allow SSH Access: This is not needed, as this rule exists by default when enabling firewall on Proxmox, see [Default firewall rules](https://pve.proxmox.com/pve-docs/pve-admin-guide.html#pve_firewall_default_rules).
2. Create role for all automation steps: The following permissions are required for all steps of this project:



4. Create user:
5. API key

Proxmox Ansible
Install collections
Add SSH public key on proxmox's trusted keys

Add envars

Describe issues in word

Describe yml

Automate OPNSense

OPNSense Packer

method used config.xml, config one OPNSense manually and export and config.xml to be used by packer.

config.xml has dynamic parameters.

describe code

describe issues and resolutions from word

OPNSense Terraform

describe code and issues and resolutions

OPNSense Ansible

what was installed to run it

describe code and issues and resolutions

next steps

conclusion
