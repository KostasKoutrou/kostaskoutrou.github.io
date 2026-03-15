
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

Additionally, the community.proxmox ansible collection requires the Python libraries `proxmoxer` and `requests` to talk to the Proxmox API. To install them, run one of the following:

```bash
# If you are on Ubuntu/Debian in WSL
sudo apt update
sudo apt install python3-proxmoxer python3-requests

# OR if you prefer pip:
pip3 install proxmoxer requests
```

Without these libraries, the following error is shown when trying to run the Ansible playbook:

<img alt="image" src="https://github.com/user-attachments/assets/fb69ef22-ac75-41a8-bc5f-1d0bc0032e24" />

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

After setting up Ansible, the next step is to write and execute the Ansible Playbooks to apply the configurations to Proxmox, which were initially applied manually and described on my [previous post](https://kostaskoutrou.github.io/2026/02/02/cyber-range-as-code-part1.html) and part 1 of this series. The Ansible playbook can be found [here](https://github.com/KostasKoutrou/kostas-seclab/blob/master/ansible/proxmox_config.yml), and is also depicted below, with more details in the comments and afterwards:

```yaml
---
#before running, run the following:
# export PROXMOX_HOST="192.168.0.50"
# export PROXMOX_USER="packer@pve"
# export PROXMOX_TOKEN_ID="packer-token"
# export PROXMOX_TOKEN_SECRET="<token_secret>"

- name: Configure Proxmox via API
  hosts: proxmox
  connection: local # because this play configures Proxmox via API, with this keyword the playbook is executed at the ansible control node instead of SSH'ing and running it locally on the managed node.
  gather_facts: false

  tasks:
    - name: Set DNS # Task to configure the central DNS settings of Proxmox
      community.proxmox.proxmox_node:
        node_name: kkproxmox
        dns:
          dns1: 1.1.1.1
          dns2: 8.8.8.8
          search: kostas.local
      # delegate_to: localhost # this is not needed if the "connection: local" is at the start of the playbook

    - name: Set Network # Task to create the Virtual Network Bridges in Proxmox
      community.proxmox.proxmox_node_network:
        node: kkproxmox
        autostart: true
        iface_type: bridge
        iface: "{{ item.iface }}"
        cidr: "{{ item.cidr }}"
        comments: "{{ item.comments }}"
        bridge_ports: "{{ item.bridge_ports }}"
        gateway: "{{ item.gateway }}"
      loop:
        - { iface: "vmbrWAN10", cidr: "10.0.10.2/24", comments: "WAN" , bridge_ports: "" , gateway: "" }
        - { iface: "vmbrDMZ20", cidr: "10.0.20.2/24", comments: "DMZ" , bridge_ports: "" , gateway: "" }
        - { iface: "vmbrIZ30", cidr: "10.0.30.2/24", comments: "Internal Zone" , bridge_ports: "" , gateway: "" }
        - { iface: "vmbrEUZ40", cidr: "10.0.40.2/24", comments: "End User Zone" , bridge_ports: "" , gateway: "" }
        - { iface: "vmbr0", cidr: "192.168.0.50/24", comments: "Bridge to home router" , bridge_ports: "nic1" , gateway: "192.168.0.1" }
      
    - name: Set Network interfaces # Task to connect the physical interface to the physical network
      community.proxmox.proxmox_node_network:
        node: kkproxmox
        iface_type: eth
        iface: nic1
        comments: "Physical Interface of Proxmox to home router"

    - name: Apply Network # Apply the above network configurations
      community.proxmox.proxmox_node_network:
        node: kkproxmox
        state: "apply"

    - name: Set Firewall Aliases # Set firewall aliases to be used for the firewall rules below
      community.proxmox.proxmox_firewall:
        level: cluster
        aliases:
          - name: subnet10
            cidr: "10.0.0.0/8"
          - name: subnet172
            cidr: "172.16.0.0/12"
          - name: subnet192
            cidr: "192.168.0.0/16"
    
    - name: Create Firewall Security Groups # security groups to be applied on the different Proxmox levels
      community.proxmox.proxmox_firewall:
        level: cluster
        group_conf: true # Whether security group should be created or deleted
        state: present # create the group
        group: "{{ item.group }}"
      loop:
        - { group: "allowguesttraffic" }
        - { group: "blockhomenwtraffic" }
    
    - name: Set Firewall Security Groups rules # configure rules of the above security groups
      community.proxmox.proxmox_firewall:
        level: group
        state: present # Create/update/delete firewall rules or security group.
        update: true # If state=present and if one or more rule/alias/ipset already exists it will update them.
        group: "{{ item.group }}"
        rules: "{{ item.rules }}"
      loop: 
        - group: allowguesttraffic # allow/not control traffic from the lab's subnets.
          rules:
            - type: in
              action: ACCEPT
              source: dc/subnet10
              pos: 0
              log: nolog
              enable: true
        - group: blockhomenwtraffic # block traffic to the home physical network.
          rules:
            - type: in
              action: ACCEPT
              source: dc/subnet192
              pos: 0
              log: nolog
              enable: false # disabled but kept just in case, because proxmox has this rule by default
              comment: "Allow ProxMox Management"
              dest: 192.168.0.50
              dport: 8006
              proto: tcp
            - type: in
              action: ACCEPT
              source: dc/subnet192
              pos: 1
              log: nolog
              enable: true
              comment: "Allow OPNSense Management"
              dest: 192.168.0.51
              dport: 443
              proto: tcp
            - type: out
              action: REJECT
              pos: 2
              log: nolog
              enable: true
              comment: "Block Local Traffic"
              dest: dc/subnet192

    - name: Apply Security Groups # apply security groups for both cluster and node levels
      community.proxmox.proxmox_firewall:
        level: "{{ item.level }}"
        node: "{{ item.node }}"
        update: true
        state: present
        rules:
          - action: blockhomenwtraffic # same rule for both cluster and node levels
            pos: 0
            type: group
            enable: true
      loop:
        - { level: cluster, node: "" }
        - { level: node, node: kkproxmox }

    # - name: Configure Update Repositories - left for future update

    #The below 2 tasks are useful if it is needed to print the current configuration
    # - name: Get Firewall Config
    #   community.proxmox.proxmox_node_info:
    #     # level: node
    #     # node: kkproxmox
    #   register: debug_data

    # - name: Show debug data
    #   debug:
    #     var: debug_data

- name: Configure Proxmox via SSH
  hosts: proxmox
  gather_facts: false

  tasks:
    - name: Check current Cluster Firewall status
      ansible.builtin.command: pvesh get /cluster/firewall/options --output-format json
      register: cluster_fw_status
      changed_when: false

    - name: Enable Firewall on Cluster
      ansible.builtin.command: pvesh set /cluster/firewall/options -enable 1
      # Only run this command IF the 'enable' value is not 1 (or if it doesn't exist yet)
      when: (cluster_fw_status.stdout | from_json).enable | default(0) | int != 1

    - name: Check current Node Firewall status
      ansible.builtin.command: pvesh get /nodes/kkproxmox/firewall/options --output-format json
      register: node_fw_status
      changed_when: false

    - name: Enable Firewall on Node
      ansible.builtin.command: pvesh set /nodes/kkproxmox/firewall/options -enable 1
      # Only run this command IF the 'enable' value is not 1 (or if it doesn't exist yet)
      when: (node_fw_status.stdout | from_json).enable | default(0) | int != 1
```

With this playbook, the following Proxmox settings are configured:

1. DNS Settings
2. Network Interfaces
3. Network Configuration
4. Firewall Rules
5. Enabling Firewall

One point to note is that the playbooks consists of two plays:

1. Configuring Proxmox via API: This is the bulk of the configuration, because the Proxmox Community Ansible collection supports most of the configurations required.
2. Configuring Proxmox via SSH: This contains tasks that cannot be executed using the avaialble Ansible collection. It includes executing manual commands on Proxmox using its CLI `pvesh`.

To run this playbook, the following command is executed:

`ansible-playbook -i inventory.ini proxmox_config.yml`

Running the playbook outputs the following:

<img alt="image" src="https://github.com/user-attachments/assets/e19c0be3-adcb-4af4-a3b1-30725f455332" />

As shown in the screenshot, the configuration is already applied, and all tasks output status `ok`. The two tasks that output `skipped` are the ones executed via SSH. Because they manually set values on Proxmox, the `when` keyword is used to skip the execution if the value is already set. Otherwise, the task would always output status `changed`, making the total output of the playbook confusing to read.

#### Issues met and resolutions

**CRLF vs LF in VS Code**

In my current setup, I am runnig VS Code on Windows (my host machine) and running all the code using Windows Subsystem for Linux (WSL). There is a discrepancy was to how the new line character is saved in Windows and expected to be read in Linux. More specifically:

1. Windows saves a new line as CRLF (\\r\\n).
2. Linux expects a new line to be LF (\\n).

When Ansible (running in Linux) reads the `inventory.ini` file, it sees the carriage return as part of the text. It reads the group name as `[proxmox\r]`. The playbook, on the other hand asks for `proxmox`. Since `proxmox` does not equal `proxmox\r`, Ansible skips it.

To fix this, in VS Code, the new line character needs to be changed on the bottom right of the window for CRLF to LF:

<img alt="image" src="https://github.com/user-attachments/assets/350a5d79-058d-419c-965f-f55f6dbe0adc" />

## Automate OPNSense

### OPNSense Packer

As described on my first post of the series, the idea of this lab is to build VMs automatically in 3 steps:

1. Using Packer, build a relatively blank template (more on the "relatively" part later)
2. Use that template with Terraform to provision VMs ready to be configured by
3. Ansible, where the rest of the configuration and final touches happens

Regarding OPNSense specifically, which is actually a custom machine based on FreeBSD, unfortunately the design towards automation is not complete. Therefore, several workarounds were needed in order to achieve automated deployment.

First of all, in order to get the initial "blank" state of the machine, the most controllable way to do that was by configuring an OPNSense VM manually exactly up to the point where it functions and is ready to be used by Terraform, without any more settings configured. These configurations are:

1. Assigning the WAN interface
2. Assigning the WAN interface IP with DHCP (this will be changed with Terraform)
3. Enabling SSH for management
4. Installing QEMU agent so that Proxmox will be able to read the IP address assigned to the VM
5. Auto-start the QEMU agent service

After applying the above configurations manually to the OPNSense VM, the configuration of the machine was exported to a file `config.xml`.

<img alt="image" src="https://github.com/user-attachments/assets/163b3fef-0b37-4b4d-8afe-c9cc63b4a9af" />

This config.xml file contains all the information required for a VM ready to be used by Terraform.

The idea is to use the config.xml and import it on a new OPNSense VM by utilizing its "Configuration Importer" feature, where during boot time you can select to import a config.xml file from an external drive (in this case a CD created by Packer containing the above exported `config.xml`).

The packer script can be found under the [project's repository](https://github.com/KostasKoutrou/kostas-seclab/blob/master/packer/opnsense/opnsense.pkr.hcl), and is also presented below

```terraform
packer {
  required_plugins {
    name = {
      version = "~> 1"
      source  = "github.com/hashicorp/proxmox"
    }
  }
}

# Declare variables, we will pull them later in the packer build command
variable "proxmox_api_url" { type = string }
variable "proxmox_api_token_id" { type = string }
variable "proxmox_api_token_secret" {
  type      = string
  sensitive = true
}

source "proxmox-iso" "opnsense" { #Resource type and local name
  proxmox_url = var.proxmox_api_url
  username    = var.proxmox_api_token_id
  token       = var.proxmox_api_token_secret
  # Skip TLS Verification for self-signed certificates
  insecure_skip_tls_verify = true
  qemu_agent = true # Default is true anyway
  node = "kkproxmox"
  vm_id = 1001
  vm_name = "opnsense-template"
  ssh_username = "root"
  ssh_password = "opnsense" # Default root password, can be changed later.
  ssh_timeout = "20m"
  cores = 4
  memory = 4096 # RAM must be more than 3GB, otherwise the boot_command is different and will not work
  os = "other" # for FreeBSD, you choose "other"
  cpu_type = "host"
  scsi_controller = "virtio-scsi-single"


  boot_iso {
    # type = "scsi"
    type = "ide"
    iso_file = "local:iso/OPNsense-25.7-dvd-amd64.iso" # ISO stored locally on Proxmox. In the future this can be changed to downloading from the internet.
    iso_checksum = "sha256:e4c178840ab1017bf80097424da76d896ef4183fe10696e92f288d0641475871"
    unmount = true
  }

  additional_iso_files { # this will created a cd in "cd1", which will be selected in the boot_command
    cd_content = {
    "conf/config.xml" = templatefile("${path.root}/conf/config.xml", {
      dynamic_ssh_key = base64encode(file("~/.ssh/id_rsa.pub"))}) # pull the public SSH key, base64 encode it, and write it in config.xml
    }
    cd_label = "config"
    iso_storage_pool = "local"
  }
  

  network_adapters {
    model  = "virtio"
    bridge = "vmbr0" # Will change it in the Terraform script, this is only for packer.
  }

  disks {
    disk_size    = "20G"
    storage_pool = "local-lvm"
    type         = "scsi"
    ssd          = true
  }

  boot_command = [
    # There is already a 10 sec wait for boot, adding another 12.
    # Start configuration importer and select cd1 where the cd_content is stored.
    "<wait12s><enter><wait5s>cd1<enter><wait25s>",

    # Put the default credentials to install the OS.
    "installer<enter><wait2s>",
    "opnsense<enter><wait10s>",
    
    # Going through the installation options:
    # Accept default Keymap and ZFS installation.
    "<enter><wait2s><enter><wait10s><enter><wait2s><spacebar><wait1s><enter><wait1s>",
    
    # Confirm formatting (Yes) and wait 2 minutes for installation.
    "<left><wait1s><enter><wait120s>",
    
    # Do not change password and reboot.
    "<down><wait1s><enter><wait1s><enter>",

    # Enable qemu agent service autostart and Update from console to latest version,
    # because qemu requires the latest OPNSense version.
    "<wait45s>root<enter><wait2s>opnsense<enter><wait5s>",
    "8<enter><wait2s>sysrc qemu_guest_agent_enable='YES'<wait1s><enter><wait1s>exit<enter><wait1s>",
    "12<enter><wait6s>y<enter><wait3s>q"
  ]
}

build {
  sources = ["source.proxmox-iso.opnsense"]
}
```

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
