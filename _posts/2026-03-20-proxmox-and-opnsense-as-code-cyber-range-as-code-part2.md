# Cyber Range as Code - Part 2 - Automating the base - Hypervisor \& Firewall

## Introduction

Welcome to the 2nd part of the project "Cyber Range as Code", where a Cyber Range is built using Infrastructure as Code and IT automation concepts.

If you want to see the concept as a whole, please take a look at [part 1](https://kostaskoutrou.github.io/2026/02/02/cyber-range-as-code-part1.html).

The basic idea is to use HashiCorp Packer, Terraform, and Ansible as the tools to build a fully functioning Cyber Range, for testing attack scenarios, implementing security standards, and architecting a secure infrastructure. The highlight of the concept is the IaC part. The whole point is to be able to recover seamlessly, i.e., all the infrasturcutre must be deployable automatically. If it is ever needed to redeploy the infrastructure on a new host server, it should be doable with minimal manual effort.

> In **Part 2**, the base is built. This includes the **Hypervisor** itself, where **Proxmox** is used, as well as the central **Firewall** of the Cyber Range, where **OPNSense** is used.

Now that the picture is painted, let's begin.

---

## Contents

* TOC
{:toc}

---

## Brief reminder of project architecture

Below is a brief reminder of the project architecture. For more details, see my previous post [Cyber Range as Code: Automating Security Lab with IaC - Part 1](https://kostaskoutrou.github.io/2026/02/02/cyber-range-as-code-part1.html)

The initial infrastructure architecture is the following. It is important to note here that this is the initial architecture idea, it is not final, and there may very well be changes during implementation.

![NetworkDesign-FW Rules drawio](https://github.com/user-attachments/assets/daeb1cd9-9816-49d0-874e-2836c15d9d86)

As shown above, the architecture is a relatively simple and typical network infrastructure, with the following components:

- The central Firewall, controlling the network traffic among the four network zones:
  1. **Demilitarized Zone (DMZ)**: The zone that exposes services which are to be served to the Internet. In the context of the project, these will be served to the local network, and, most importantly for the project, will be accessible by the "External Attacker", enabling for attack scenarios initiated from the "Internet".
  2. **Internal Zone**: The zone where all there internal servers and services will reside. This includes:
      - Any internal servers, e.g., SQL Servers and AD DC, hosting and serving information which is destined to be consumed by internal resources only.
      - Security Tools, including the SIEM/XDR/Monitoring tools.
  3. **End Users**: The last zone will be for the End Users, where typical workstation VMs will reside, and have defined access to specific servers/services and to the internet.
  4. **WAN Zone**: This is where the "Internet", in the context of the Infrastructure, lives. This is where:
      - The PC from which the Proxmox management will be done, running the Packer, Terraform, and Ansible.
      - The External Attacks will occur from.
- There will also be Internal Attack Simulations, executed from within the different zones, bypassing the firewall ("assume breach").

The code of the project can be find in the project's [GitHub Repository](https://github.com/KostasKoutrou/kostas-seclab)

## Automate Proxmox

Regarding automating the configuration of Proxmox, this essentially includes:

### Initial Setup

This step includes the installation of Proxmox on a machine, setting up the network configuration, i.e., IP Address, Gateway, DNS, and setting up the root user credentials. One point of improvement here is to potentially automate the parameters during the initial installation of Proxmox. This is supported by Proxmox with the [Automated Installation](https://pve.proxmox.com/wiki/Automated_Installation). This would be a very interesting improvement point towards a fully hands-off installation and configuration. But this will be left as a future exercise.

After the initial setup and reaching the point of being able to sign in to the Proxmox GUI, the next step is to set up Proxmox so that Ansible can communicate with it.

This means:

1. **Create a Proxmox firewall rule to allow SSH Access**: This is not needed, as this rule exists by default when enabling firewall on Proxmox, see [Default firewall rules](https://pve.proxmox.com/pve-docs/pve-admin-guide.html#pve_firewall_default_rules).
2. **Create a role for all automation steps**: The following permissions are required for all steps of this project:

    <img alt="image" src="https://github.com/user-attachments/assets/65d09a28-516a-4173-928b-13e1692482c5" />

3. **Create user**: The packer user created in the initial lab (part 1) is used. It is a simple user and the above role is assigned to it.

    <img alt="image" src="https://github.com/user-attachments/assets/84bc030f-e6b3-4e47-abd1-139a2d7e77f5" />

4. **API key**: An API key was generated and assigned to the above user, in order for Ansible to execute API calls to Proxmox.

    <img alt="image" src="https://github.com/user-attachments/assets/d8031359-89ed-46b9-97b7-8933de6a5e9a" />

### Proxmox Ansible

In this step, the Ansible Control Node (i.e., the PC where all the code is written and executed) is prepared for Proxmox configurations.

#### Ansible Collection

[Ansible Collections](https://docs.ansible.com/projects/ansible/latest/collections_guide/index.html) are a distribution format for Ansible content that can include playbooks, roles, modules, and plugins. Essentially, a collection is like a program library containing all the functionalities required for configuring a specific component.

The **Community Proxmox** Ansible Collection was used for this project, which can be found [here](https://galaxy.ansible.com/ui/repo/published/community/proxmox/docs/).

To install the collection the following command was executed:

```bash
ansible-galaxy collection install community.proxmox
```

<img alt="image" src="https://github.com/user-attachments/assets/c604ab1e-eb4b-4887-8687-82597fafe540" />

Additionally, the `community.proxmox` ansible collection requires the Python libraries `proxmoxer` and `requests` to talk to the Proxmox API. To install them, run one of the following:

```bash
# If you are on Ubuntu/Debian in WSL. Since the project is ran on WSL, this was selected:
sudo apt update
sudo apt install python3-proxmoxer python3-requests

# OR if you prefer pip:
pip3 install proxmoxer requests
```

Without these libraries, the following error is shown when trying to run the Ansible playbook:

<img alt="image" src="https://github.com/user-attachments/assets/fb69ef22-ac75-41a8-bc5f-1d0bc0032e24" />

#### Setting up SSH

Since Ansible executes SSH using certificate-based authentication, the public key of the Ansible Control Node is needed to be added on Proxmox's trusted keys. Since the public key is the one that exists on the physical PC currently running all the commands, the easiest way to do this task is with the following command:

`ssh-copy-id -i ~/.ssh/id_rsa.pub root@192.168.0.50`

After running the command, it is now possible to ping Proxmox via Ansible:

<img alt="image" src="https://github.com/user-attachments/assets/4dd29f39-14cb-4b8a-9f4e-93211562291d" />

In the above command, the `inventory.ini` file includes machines which are in the scope of configurations via Ansible. For now it only includes the Proxmox host:

```ini
[proxmox]
192.168.0.50 ansible_user=root
```

#### Initializing Ansible for Proxmox

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

#### Running Ansible to configure Proxmox

After setting up Ansible, the next step is to write and execute the Ansible Playbooks to apply the configurations to Proxmox, which were initially applied manually and described on my [previous post](https://kostaskoutrou.github.io/2026/02/02/cyber-range-as-code-part1.html) and part 1 of this series. The Ansible playbook can be found [here](https://github.com/KostasKoutrou/kostas-seclab/blob/master/ansible/proxmox_config.yml), and is also depicted below, with more details in the comments and afterwards:

{% raw %}
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
{% endraw %}

With this playbook, the following Proxmox settings are configured:

1. DNS Settings
2. Network Interfaces
3. Network Configuration
4. Firewall Rules
5. Enabling Firewall

One point to note is that the playbook consists of two plays:

1. **Configuring Proxmox via API**: This is the bulk of the configuration, because the Proxmox Community Ansible collection supports most of the configurations required.
2. **Configuring Proxmox via SSH**: This contains tasks that cannot be executed using the avaialble Ansible collection. It includes executing manual commands on Proxmox using its CLI `pvesh`.

To run this playbook, the following command is executed:

```bash
ansible-playbook -i inventory.ini proxmox_config.yml
```

Running the playbook outputs the following:

<img alt="image" src="https://github.com/user-attachments/assets/e19c0be3-adcb-4af4-a3b1-30725f455332" />

As shown in the screenshot, the configuration is already applied, and all tasks output status `ok`. The two tasks that output `skipped` are the ones executed via SSH. Because they manually set values on Proxmox, the `when` keyword is used to skip the execution if the value is already set. Otherwise, the task would always output status `changed`, making the total output of the playbook confusing to read.

#### Issues met and resolutions

While developing the Ansible playbook, the follow issues were faced:

**CRLF vs LF in VS Code**

In current setup of writing the code, VS Code is used on Windows (PC) and  Windows Subsystem for Linux (WSL) is used to run all the code. There is a discrepancy on how the 'new line' character is saved in Windows and expected to be read in Linux. More specifically:

1. Windows saves a new line as `CRLF` (\\r\\n).
2. Linux expects a new line to be `LF` (\\n).

When Ansible (running in Linux) reads the `inventory.ini` file, it sees the carriage return as part of the text. It reads the group name as `[proxmox\r]`. The playbook, on the other hand asks for `proxmox`. Since `proxmox` does not equal `proxmox\r`, Ansible skips it.

To fix this, in VS Code, the new line character needs to be changed on the bottom right of the window for `CRLF` to `LF`:

<img alt="image" src="https://github.com/user-attachments/assets/350a5d79-058d-419c-965f-f55f6dbe0adc" />

After configuring Proxmox, the next step is to automatically configure the central Firewall, which is OPNSense.

## Automate OPNSense

### OPNSense Packer

As described on the first post of the series, the idea of this lab is to build VMs automatically in 3 steps:

1. Using **Packer**, build a relatively blank template (more on the "relatively" part later)
2. Use that template with **Terraform** to provision VMs ready to be configured by
3. **Ansible**, where the rest of the configuration and final touches happen.

Regarding OPNSense specifically, which is actually a custom machine based on FreeBSD, unfortunately it is not fully designed to be deployed and configured using automation tools. Therefore, several workarounds were needed in order to achieve an automated deployment.

First of all, in order to get the initial "blank" state of the machine, the most controllable way to do that was by configuring an OPNSense VM manually exactly up to and not any further than the point where it functions and is ready to be used by Terraform, without any more settings configured. These configurations are:

1. Assigning the WAN interface
2. Assigning the WAN interface IP with DHCP (this will be changed with Terraform)
3. Enabling SSH for management
4. Installing the QEMU agent so that Proxmox will be able to read the IP address assigned to the VM
5. Auto-start the QEMU agent service

After applying the above configurations manually to the OPNSense VM, the configuration of the machine was exported to a file `config.xml`, which can be found [here](https://github.com/KostasKoutrou/kostas-seclab/blob/master/packer/opnsense/conf/config.xml)

<img alt="image" src="https://github.com/user-attachments/assets/163b3fef-0b37-4b4d-8afe-c9cc63b4a9af" />

This `config.xml` file contains all the information required for a VM ready to be used by Terraform.

The idea is to use the `config.xml` and import it on a new OPNSense VM by utilizing its [Configuration Importer](https://docs.opnsense.org/manual/install.html#opnsense-importer) feature, where during boot time there is the option to select to import a `config.xml` file from an external drive (in this case a CD created by Packer containing the above exported `config.xml`).

The packer template can be found under the [project's repository](https://github.com/KostasKoutrou/kostas-seclab/blob/master/packer/opnsense/opnsense.pkr.hcl), and is also presented below, with several comments to explain how it works:

{% raw %}
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
{% endraw %}

Some interesting notes to point out:

**1. config.xml import**

In the CD that is being mounted which contains the `config.xml` file, the SSH public key of the host machine is being written to the config by creating a dynamic variable in the `config.xml`. This is done by using the `templatefile` [function](https://developer.hashicorp.com/terraform/language/functions/templatefile), and providing the path of the public key to it, as shown in the code snippet below:

{% raw %}
```terraform
cd_content = {
"conf/config.xml" = templatefile("${path.root}/conf/config.xml", {
  dynamic_ssh_key = base64encode(file("~/.ssh/id_rsa.pub"))}) # pull the public SSH key, base64 encode it, and write it in config.xml
}
```
{% endraw %}

Note that `cd_content` requires one of the following tools to create the CD: xorriso, mkisofs, hdiutil, oscdimg

In this case, xorriso was installed

```bash
sudo apt install xorriso
```

In the `config.xml` file, in order to make it so that it expects a variable in the SSH key field <authorizedkeys>, the dynamic variable was defined as follows:

{% raw %}
```xml
<user uuid="083dd1a5-394d-441f-aae3-481a4ce478c5">
  <uid>0</uid>
  <name>root</name>
  <disabled>0</disabled>
  <scope>system</scope>
  <expires/>
  <authorizedkeys>${dynamic_ssh_key}</authorizedkeys>
  <otp_seed/>
  <shell/>
  <password>$2y$10$YRVoF4SgskIsrXOvOQjGieB9XqHPRra9R7d80B3BZdbY/j21TwBfS</password>
  <pwd_changed_at/>
  <landing_page/>
  <comment/>
  <email/>
  <apikeys/>
  <priv/>
  <language/>
  <descr>System Administrator</descr>
  <dashboard/>
</user>
```

As shown, for SSH keys, the dynamic variable `${dynamic_ssh_key}` is provided instead of a static one.
{% endraw %}

**2. boot command**

Additionally, the `boot_command` includes all the commands executed via the keyboard during the installation of OPNSense using the configuration importer feature. These commands are described below:

1. Wait enough time to get to the option to use the configuration importer.
2. Write the external drive's name (cd1) to pull the `config.xml` file from.
3. Install the configuration by logging in with the `installer` user.
4. Go through the installation steps (accept default keymap, select ZFS installation, select disk, confirm format and reboot)
5. Wait for the installation and reboot to complete.
6. Open a shell (press option "8"), and pass the variable `qemu_guest_agent_enable='YES'` to auto start the QEMU agent service automatically.
7. Update OPNSense to the latest version (press option "12"), because QEMU requires that. 

The commands to build the Packer template are:

```bash
packer init .
packer validate -var-file="../credentials.pkrvars.hcl" .
packer build -var-file="../credentials.pkrvars.hcl" .
```

The file `credentials.pkrvars.hcl` contains the credentials used by Packer and the other components:

```
proxmox_api_url          = "https://192.168.0.50:8006/api2/json"
proxmox_api_token_id     = "packer@pve!packer-token"
proxmox_api_token_secret = "<proxmox_api_token>"
ubuntu_pw = "lab-admin"
opnsense_pw = "opnsense-admin"
```

The output of running the Packer template build is the following:

<img alt="image" src="https://github.com/user-attachments/assets/f3c1b178-711c-4a50-aa0e-5957f6d95aef" />

#### OPNSense Packer Issues and resolutions

During development of Packer OPNSense, the following issues were identified:

1. **The cloud-init limitation:**
  - **The Issue**: OPNsense is built on FreeBSD and functions as an appliance where the `/conf/config.xml` is the absolute source of truth. It does not officially support standard Linux cloud-init for initial provisioning (like setting IPs or users).
  - **The Solution**: OPNsense’s native Configuration Importer was utilized. By packaging a pre-configured config.xml file into a virtual CD drive, OPNsense digests the configuration during the boot sequence and permanently bakes it into the hard drive during the installation.
2. **QEMU guest agent issues with OPNSense version:**
  - **The Issue**: Packer needs the QEMU Guest Agent to discover the VM's dynamic IP address to connect via SSH. However, the OPNsense base ISO (25.7) was out of date with the plugin repository, meaning the agent would install but would not start its service. It was a "chicken and egg" situation, where it was not possible to SSH in to run the update because the IP could not be detected due to the QEMU agent not running, and it was not possible to get the QEMU agent to run because it was not possible to run the update.
  - **The Solution**: The setup was baked in the `boot_command` command list. Instead of stopping after the installation reboot, the command now continues typing in the console to drop into the shell, enabling the service to auto-start (sysrc qemu_guest_agent_enable='YES'), and trigger a system update via console option 12. This updates the OS and pulls the agent before Packer ever tries to connect.
3. **OPNsense Firewall Blocking Packer**
  - **The Issue**: OPNsense is a default-deny firewall that drops all WAN traffic. Packer needs to connect via the WAN interface (mapped to your Proxmox bridge) to finalize the template.
  - **The Solution**: By ensuring no <lan> interface was strictly mapped initially, OPNsense's safety mechanisms trigger the Anti-Lockout rule on the WAN interface, automatically opening port 22 and allowing Packer/Ansible to connect without the need to add a firewall rule to explicitly allow this traffic. This rule will be created on the next phases anyway.

After finishing with Packer, the result is a "blank" template, ready to be used by Terraform to provision a VM with the final settings.

### OPNSense Terraform

When it comes to using Terraform for provisioning OPNSense, the process was initially straightforward, but, during development, more and more settings were added to the Terraform phase, as it was discovered that Ansible for OPNSense does not provide enough flexibility to implement what was needed. Therefore, more implementation points were moved from the "Ansible phase" to the "Terraform phase".

The code can be found in the [project's repository](https://github.com/KostasKoutrou/kostas-seclab/blob/master/terraform/main.tf) and is also written below:

{% raw %}
```terraform
resource "proxmox_vm_qemu" "c-opnsense" {
    name = "c-opnsense"
    target_node = "kkproxmox"
    clone = "opnsense-template"
    agent = 1 #enable QEMU guest agent
    memory = 4096
    balloon = 4096
    bios = "seabios"
    scsihw = "virtio-scsi-single"
    os_type = "other"
    skip_ipv6 = true

    cpu {
        cores = 4
        sockets = 1
        type = "host"
    }

    disk {
        slot = "scsi0"
        cache = "none"
        discard = true
        iothread = true
        emulatessd = true
        asyncio = "io_uring"
        size = "20G"
        type = "disk"
        storage = "local-lvm"
        format = "raw"
    }

    startup_shutdown {
        order = -1
        shutdown_timeout = -1
        startup_delay = -1
    }

    # Configure the network interfaces
    network {
        id = 0
        model = "virtio"
        bridge = "vmbr0"
        firewall = true
    }
    
    network {
        id = 1
        model = "virtio"
        bridge = "vmbrDMZ20"
        firewall = true
    }

    network {
        id = 2
        model = "virtio"
        bridge = "vmbrIZ30"
        firewall = true
    }

    network {
        id = 3
        model = "virtio"
        bridge = "vmbrEUZ40"
        firewall = true
    }

    # connection used for the provisioners below
    connection {
      type = "ssh"
      user = "root"
      private_key = file("~/.ssh/id_rsa")
      host = self.ssh_host
    }

    # replace the config.xml file with the one below.
    # This one uses dynamic variables to also configure the network interfaces IPs.
    provisioner "file" {
      content = templatefile("${path.module}/template_config_opnsense_lab.xml",
        {
            dynamic_ssh_key = base64encode(file("~/.ssh/id_rsa.pub"))
            wan_if = "vtnet0"
            wan_descr = "WAN"
            wan_ip = "192.168.0.51"
            wan_subnet = "24"
            wan_gw = "WAN_GW"
            dmz20_if = "vtnet1"
            dmz20_descr = "DMZ20"
            dmz20_ip = "10.0.20.1"
            dmz20_subnet = "24"
            iz30_if = "vtnet2"
            iz30_descr = "IZ30"
            iz30_ip = "10.0.30.1"
            iz30_subnet = "24"
            euz40_if = "vtnet3"
            euz40_descr = "EUZ40"
            euz40_ip = "10.0.40.1"
            euz40_subnet = "24"
        }
      )
      destination = "/conf/config.xml"
    }

    # reboot the machine so that the new config.xml is applied
    provisioner "remote-exec" {
      inline = [ 
        "echo 'Injecting Cyber Range Topology by restarting OPNSense. VM will restart in 3 seconds...'",
        "daemon -f /bin/sh -c 'sleep 3; /sbin/reboot'"
       ]
    }
}
```
{% endraw %}

To run the terraform configuration:

```bash
terraform init
terraform plan -var-file=../packer/credentials.pkrvars.hcl
terraform apply -var-file=../packer/credentials.pkrvars.hcl
```

The var-file is the one used also in packer and contains the connection variables so that Terraform can talk to Proxmox.

Running the terraform configuration outputs the following. Note that 2 additional Linux VMs are also provisioned, as they were described in the previous post, but there is no reason to get into them in this post. The interesting lines are the ones starting with `proxmox_vm_qemu.c-opnsense`, because these ones are about the OPNSense VM:

<img alt="image" src="https://github.com/user-attachments/assets/994f387e-e7c4-4cb3-850d-fb33e57f7e5b" />

A few interesting notes to point out:

1. **Configuring settings using the config.xml file**: As mentioned previously, OPNSense is not made to be configured "as-code" natively, i.e., by using Ansible. Therefore, several settings cannot be configured using Ansible. There is effort put by the community towards achieving that, by implementing more API calls. However, for now, a few basic settings, like configuring the network interfaces, were configured using the `config.xml` file of OPNSense. All these workarounds made me think if OPNSense is the right tool for the job, and I thought of switching to something more programmable like [VyOS](https://vyos.io/), but for now I pushed through with OPNSense. The `config.xml` file used can be found [here](https://github.com/KostasKoutrou/kostas-seclab/blob/master/terraform/template_config_opnsense_lab.xml)

2. **Dynamic variables**: As was the case with Packer, too, several variables in the `config.xml` file were written dynamically. The `config.xml` file part where these variables are expected is shown below:

{% raw %}
  ```xml
  <wan>
    <if>${wan_if}</if>
    <descr>${wan_descr}</descr>
    <enable>1</enable>
    <spoofmac />
    <blockbogons>1</blockbogons>
    <ipaddr>${wan_ip}</ipaddr>
    <subnet>${wan_subnet}</subnet>
    <gateway>${wan_gw}</gateway>
  </wan>
  <opt1>
    <if>${dmz20_if}</if>
    <descr>${dmz20_descr}</descr>
    <enable>1</enable>
    <spoofmac />
    <ipaddr>${dmz20_ip}</ipaddr>
    <subnet>${dmz20_subnet}</subnet>
  </opt1>
  <opt2>
    <if>${iz30_if}</if>
    <descr>${iz30_descr}</descr>
    <enable>1</enable>
    <spoofmac />
    <ipaddr>${iz30_ip}</ipaddr>
    <subnet>${iz30_subnet}</subnet>
  </opt2>
  <opt3>
    <if>${euz40_if}</if>
    <descr>${euz40_descr}</descr>
    <enable>1</enable>
    <spoofmac />
    <ipaddr>${euz40_ip}</ipaddr>
    <subnet>${euz40_subnet}</subnet>
  </opt3>
  ```
{% endraw %}

3. **config.xml firewall rules needed**: The Ansible collections used in the next phase require API connectivity and SSH access. Therefore, two firewall rules were needed to be added to the `config.xml` file:

{% raw %}
  ```xml
  <rule uuid="cf7ad3fc-4924-4fa6-b9c5-77e6f5d81746">
    <type>pass</type>
    <interface>wan</interface>
    <ipprotocol>inet</ipprotocol>
    <statetype>keep state</statetype>
    <direction>in</direction>
    <log>1</log>
    <quick>1</quick>
    <protocol>tcp</protocol>
    <source>
      <any>1</any>
    </source>
    <destination>
      <network>wanip</network>
      <port>22</port>
    </destination>
    <description>Allow SSH to OPNSense</description>
  </rule>
  <rule uuid="7409d103-737c-4866-83d8-2c1adc5dbcf7">
    <enabled>1</enabled>
    <statetype>keep</statetype>
    <state-policy/>
    <action>pass</action>
    <quick>1</quick>
    <interfacenot>0</interfacenot>
    <interface>wan</interface>
    <direction>in</direction>
    <ipprotocol>inet</ipprotocol>
    <protocol>TCP</protocol>
    <source_net>wan</source_net>
    <source_not>0</source_not>
    <destination_net>wanip</destination_net>
    <destination_not>0</destination_not>
    <destination_port>443</destination_port>
    <disablereplyto>0</disablereplyto>
    <log>1</log>
    <allowopts>0</allowopts>
    <nosync>0</nosync>
    <nopfsync>0</nopfsync>
    <description>Allow HTTPS to OPNSense WebUI</description>
  </rule>
  ```
{% endraw %}

4. **config.xml API key needed**: The Ansible collections used in the next phase require API connectivity, and therefore an API key. This was generated manually from OPNSense, and the configuration was added to the `config.xml` file:

{% raw %}
  ```xml
  <user uuid="083dd1a5-394d-441f-aae3-481a4ce478c5">
    <uid>0</uid>
    <name>root</name>
    <disabled>0</disabled>
    <scope>system</scope>
    <expires/>
    <authorizedkeys>${dynamic_ssh_key}</authorizedkeys>
    <otp_seed/>
    <shell/>
    <password>$2y$10$YRVoF4SgskIsrXOvOQjGieB9XqHPRra9R7d80B3BZdbY/j21TwBfS</password>
    <pwd_changed_at/>
    <landing_page/>
    <comment/>
    <email/>
    <apikeys>ooyOnJ5Y7NiPxeTXkTxEgM+9steLsU+I+UehjuiXtNdBL0ckTvUQM6PWa5AxpdnUXGLGLtyRQFNCnJI8|$6$$c9GWtGrIy25Ez358v52fDrfyTfD3Q5rnzVlc7je/MKL0EzxnTOrOL0mnSh78O.t6iA8hrtd4.OfsWUvUhJgCl0</apikeys>
    <priv/>
    <language/>
    <descr>System Administrator</descr>
    <dashboard/>
  </user>
  ```
{% endraw %}

5. **Applying the configuration - Reboot the machine**: After provisioning the VM, it has the final configuration written to the `/conf/config.xml` file, but it has not pulled and applied it yet. It needs to restart to apply it. The provisioner `remote-exec` was used to execute a reboot command. However, if the reboot command is executed before the script and the terraform configuration finishes, then terraform will believe that the provisioning failed. Therefore, a daemon was started that will reboot the machine in 3 seconds. So, terraform will be able to finish successfully and consider the machine fully provisioned, and then the machine will reboot to pull the configuration.

{% raw %}
```terraform
provisioner "remote-exec" {
  inline = [ 
    "echo 'Injecting Cyber Range Topology by restarting OPNSense. VM will restart in 3 seconds...'",
    "daemon -f /bin/sh -c 'sleep 3; /sbin/reboot'"
   ]
}
```
{% endraw %}

In conclusion, while there were not any noteworthy issues by using Terraform for OPNSense, Terraform was actually used to solve some issues presented by Ansible, which is described in the next section.

### OPNSense Ansible

As mentioned previously, OPNSense is not designed to be configured through command line automatically. It is a GUI-first firewall. However, efforts have been put towards improving that, and an API is built through which most of its settings can be set.

Considering that the API itself is not yet fully functioning, it was even more difficult to find an Ansible Collection utilizing the API and supporting all the configurations needed to be implemented for this project.

The following Ansible collections were found which were all used in combination:

1. [oxlorg.opnsense](https://ansible-opnsense.oxl.app/): most configurations supported

  To install it, run:
  
  ```bash
  sudo apt install python3-pip
  python3 -m pip install --upgrade httpx
  # latest version:
  ansible-galaxy collection install git+https://github.com/O-X-L/ansible-opnsense.git
  
  # stable/tested version:
  ansible-galaxy collection install git+https://github.com/O-X-L/ansible-opnsense.git,25.7.8
  ## OR
  ansible-galaxy collection install oxlorg.opnsense # This option was selected for this project.
  ```

2. [puzzle.opnsense](https://puzzle.github.io/puzzle.opnsense/collections/puzzle/opnsense/index.html): Does not have many configurations, but it supports network interface assignments.

  To install it, run:
  
  ```bash
  ansible-galaxy collection install puzzle.opnsense
  ```

The Ansible playbook for OPNSense with comments can be found [here](https://github.com/KostasKoutrou/kostas-seclab/blob/master/ansible/opnsense_config.yml) and is provided below:

{% raw %}
```yaml
---
- name: Configure OPNSense via SSH
  hosts: opnsense
  # connection: local  # execute modules on firewall
  gather_facts: false
  
  tasks:
    - name: Assign Interfaces
      puzzle.opnsense.interfaces_assignments:
        description: "{{ item.description }}"
        device: "{{ item.device }}"
        identifier: "{{ item.identifier }}"
      loop:
        - { description: "DMZ20", device: "vtnet1", identifier: "opt1" }
        - { description: "IZ30", device: "vtnet2", identifier: "opt2" }
        - { description: "EUZ40", device: "vtnet3", identifier: "opt3" }
        - { description: "WAN", device: "vtnet0", identifier: "wan" }


- name: Configure OPNSense via API
  hosts: opnsense
  connection: local  # execute modules on controller
  gather_facts: false
  vars:
    # because this collection is designed to be ran on the OPNSense, it was trying to find
    # the default FreeBSD python path, instead of the one running locally. Setting this
    # variable defines the correct path for ansible.
    ansible_python_interpreter: "{{ ansible_playbook_python }}"
  module_defaults:
    group/oxlorg.opnsense.all:
      ssl_verify: false
      firewall: "{{ inventory_hostname }}" # pull the IP directly from invetory.ini
      api_credential_file: "{{ playbook_dir }}/OPNsense.internal_root_apikey.txt" # export of generated API keys from OPNSense

  tasks:
    # OPNSense Ansible and API do not support configuring ISC DHCP, so Kea DHCP was used instead
    - name: Enable Kea DHCP EUZ40
      oxlorg.opnsense.dhcp_general:
        enabled: true
        interfaces: ['opt3']
        fw_rules: true
        
    - name: Set Kea DHCP EUZ40 Subnet
      oxlorg.opnsense.dhcp_subnet:
        subnet: "10.0.40.0/24"
        pools: ["10.0.40.150-10.0.40.200"]
        dns: ["1.1.1.1", "8.8.8.8"]
        ipv: 4

    - name: Set Firewall Rules
      oxlorg.opnsense.rule_multi:
        rules:
        # This rule is not needed because it is needed and was added before Ansible, because
        # this Ansible collection talks via API, i.e., via port 443. It is kept just for future reference.
        # - description: 'Allow HTTPS to OPNSense WebUI' #name: 'Allow HTTPS to OPNSense WebUI'
        #   source_net: 'wan'
        #   destination_net: 'wanip'
        #   destination_port: 443
        #   interface: ['wan']
        #   protocol: 'TCP'
        #   action: 'pass'

        - description: 'EUZ40 allow all traffic'
          source_net: 'any'
          destination_net: 'any'
          # destination_port: leave empty for all
          interface: ['opt3']
          protocol: 'any'
          action: 'pass'
          
        match_fields: ['description']
        reload: true

    # These tasks can be used to retrieve information
    # - name: Listing
    #   oxlorg.opnsense.list:
    #     target: 'rule'
    #   register: existing_entries

    # - name: Printing rules
    #   ansible.builtin.debug:
    #     var: existing_entries.data
```
{% endraw %}

A few interesting notes about the Ansible playbook:

**API key definition**: the API key used to authenticate is pulled from the path `{{ playbook_dir }}/OPNsense.internal_root_apikey.txt`. This file was exported manually from an OPNSense, as described in the previous section, and saved at the path above. It is noteworthy that it is not supported to modify this file at all. The ansible tasks are looking for the format below:

<img alt="image" src="https://github.com/user-attachments/assets/dc8c25bf-00f9-4949-81ce-362e901d6ac3" />

Changing the format in any way results in the following error:

<img alt="image" src="https://github.com/user-attachments/assets/cc474444-4f96-40dd-898b-c27a313a11ad" />

After configuring OPNSense with Ansible, the next step is to configure Proxmox itself for the OPNSense VM.

### Configuring Proxmox for OPNSense with Ansible

In addition to configuring OPNSense itself, some settings need to be configured on Proxmox regarding the hosted OPNSense VM. More specifically, these include configuring the Proxmox firewall settings for the VM.

The Ansible playbook for this purpose is relatively short, can be found [here](https://github.com/KostasKoutrou/kostas-seclab/blob/master/ansible/proxmox_opnsense_config.yml) and is the following:

{% raw %}
```yaml
---
- name: Config Proxmox for OPNSense
  hosts: proxmox
  connection: local
  gather_facts: false

  tasks:

    - name: Config Firewall OPNSense
      community.proxmox.proxmox_firewall:
        level: vm
        vmid: 101
        update: true
        state: present
        rules:
          - action: allowguesttraffic # same rule for both cluster and node levels
            pos: 0
            type: group
            enable: true
          - action: blockhomenwtraffic
            pos: 1
            type: group
            enable: true

    # These tasks can be used to retrieve the current config
    # - name: Config Firewall OPNSense
    #   community.proxmox.proxmox_firewall_info:
    #     level: vm
    #     vmid: 100
    #   register: debug_fw_opnsense

    # - name: Show debug
    #   debug:
    #     var: debug_fw_opnsense
```
{% endraw %}

It is straightforward, and assigned the security groups (created in the previous sections) to the VM.

To run the playbook, as was the case with the playbook for proxmox itself, the envars described in the previous section need to be set firstly. To run the playbook, run:

```bash
ansible-playbook -i inventory.ini proxmox_opnsense_config.yml 
```
Running the playbook outputs the following:

<img alt="image" src="https://github.com/user-attachments/assets/00775ec7-ea55-49ef-ba79-fa437b9d9579" />

Now that Proxmox is configured for the hosted OPNSense VM, this part of the project is finished. Now let's take a look a potential next steps.

## Next Steps

The next steps include implementing some not mandatory final touches on the automation part of Proxmox and OPNSense, including:

1. Automating the installation process of Proxmox itself. This is supported by Proxmox with the [Automated Installation](https://pve.proxmox.com/wiki/Automated_Installation).
2. Automating the users creation to Proxmox
3. Automating the changing of update repositories of Proxmox from the default ones which require a license to the free ones
4. Adding the ISO files to Proxmox, or alternatively switching the Packer templates to download the ISOs from the internet directly instead of trying to grab them from Proxmox locally.
5. Creating a file serving as the single source of truth. All the variables used in the project will be defined in that file, including IP addresses, usernames, etc. The file will be referenced by all code.

Additionally, the next steps include the continuing of the other components described in the first post.

## Conclusion

In this post, the base was built under the context and concept of the project, i.e. utilizing IaC.

As it is probably understood through the post, building each component of this project proved to be more complex than expected, as each component has its own peculiarities and workarounds required in order for it to work in the context of IaC and Configuration Management.
