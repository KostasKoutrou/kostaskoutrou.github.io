
The code of the project can be find in my [GitHub Repository](https://github.com/KostasKoutrou/kostas-seclab)

## Automate Proxmox

Regarding automating the configuration of Proxmox, this essentially includes:

### Initial Setup

This step includes the installation of Proxmox on a machine, setting up the network configuration, i.e., IP Address, Gateway, DNS, and setting up the root user credentials. You can always take a look at the description of the configuration to be achieved in my [previous post](https://kostaskoutrou.github.io/2026/02/02/cyber-range-as-code-part1.html).

One point of improvement here is to potentially automate the installation parameters. This is supported by Proxmox with the [Automated Installation](https://pve.proxmox.com/wiki/Automated_Installation). This would be a very interesting improvement point towards a fully hands-off installation and configuration. But this will be left as a future exercise.

After the initial setup and reaching the point of being able to sign in to the Proxmox GUI, the next step is to set up Proxmox so that Ansible can communicate with it.

This means:

1. Create a Proxmox firewall rule to allow SSH Access: This is not needed, as this rule exists by default when enabling firewall on Proxmox, see [Default firewall rules](https://pve.proxmox.com/pve-docs/pve-admin-guide.html#pve_firewall_default_rules).
2. Create role for all automation steps: The following permissions are required for all steps of this project:

  <img alt="image" src="https://github.com/user-attachments/assets/65d09a28-516a-4173-928b-13e1692482c5" />

3. Create user: The pakcer user created in the initial lab is used. It is a simple user and the above role is assigned to it.

  <img alt="image" src="https://github.com/user-attachments/assets/84bc030f-e6b3-4e47-abd1-139a2d7e77f5" />

4. API key: An API key was generated and assigned to the above user, in order for Ansible to execute API calls to Proxmox.

  <img alt="image" src="https://github.com/user-attachments/assets/d8031359-89ed-46b9-97b7-8933de6a5e9a" />

### Proxmox Ansible

In this step, the Ansible Control Node is prepared for Proxmox configurations.

The Community Proxmox Ansible Collection was used for this project, which can be found [here](https://galaxy.ansible.com/ui/repo/published/community/proxmox/docs/).

To install the collection the following command was executed:

`ansible-galaxy collection install community.proxmox`

<img alt="image" src="https://github.com/user-attachments/assets/c604ab1e-eb4b-4887-8687-82597fafe540" />

Since Ansible executes SSH using certificate-based authentication, the public key of the Ansible Control Node is needed to be added on Proxmox's trusted keys. Since the public key is the one that exists on the physical PC currently running all the commands, the easiest way to do this task is with the following command:

`ssh-copy-id -i ~/.ssh/id_rsa.pub root@192.168.0.50`

After running the command, it is now possible to ping Proxmox via Ansible:

<img alt="image" src="https://github.com/user-attachments/assets/4dd29f39-14cb-4b8a-9f4e-93211562291d" />

In the above command, the `inventory.ini` file includes machines which are in the scope of configurations via Ansible. For now it only includes the Proxmox host:

```ini
[proxmox]
192.168.0.50 ansible_user=root
```

Because the Community Proxmox Ansible Collection requires a few specific [parameters](https://galaxy.ansible.com/ui/repo/published/community/proxmox/content/module/proxmox_node/#parameters) for every task being run, it is also supported to configure these parameters as Environment Variables:

|Parameter|Comments|
|-|-|
|api_hoststring / required|Specify the target host of the Proxmox VE cluster.<br>Uses the PROXMOX_HOST environment variable if not specified.|
|api_passwordstring|Specify the password to authenticate with.<br>Uses the PROXMOX_PASSWORD environment variable if not specified.|
|api_portinteger|Specify the target port of the Proxmox VE cluster.<br>Uses the PROXMOX_PORT environment variable if not specified.|
|api_token_idstring|Specify the token ID.<br>Uses the PROXMOX_TOKEN_ID environment variable if not specified.|
|api_token_secretstring|Specify the token secret.<br>Uses the PROXMOX_TOKEN_SECRET environment variable if not specified.|
|api_userstring / required|Specify the user to authenticate with.<br>Uses the PROXMOX_USER environment variable if not specified.|

Therefore, the following Envars are set before running Proxmox Ansible playbooks:

```bash
export PROXMOX_HOST="192.168.0.50"
export PROXMOX_USER="packer@pve"
export PROXMOX_TOKEN_ID="packer-token"
export PROXMOX_TOKEN_SECRET="<token_secret>"
```

Describe yml

After setting up Ansible, the next step is to write and execute the Ansible Playbooks to apply the configurations to Proxmox, which were initially applied manually and escribed on my [previous post](https://kostaskoutrou.github.io/2026/02/02/cyber-range-as-code-part1.html) and part 1 of this series. The Ansible playbook can be found [here](https://github.com/KostasKoutrou/kostas-seclab/blob/master/ansible/proxmox_config.yml), and is also depicted below:

```yaml

```

How to run
Screenshots of it running

Describe issues in word


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
