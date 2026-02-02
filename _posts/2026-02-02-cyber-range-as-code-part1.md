# Cyber Range as Code: Automating Security Lab with IaC - Part 1

## Introduction

What is a Cyber Range?

> A [Cyber Range](https://www.ibm.com/think/topics/cyber-range) is a virtual environment for cybersecurity training, testing, and research that simulates real-world networks and cyberattacks.

I always wanted to build a security homelab, where I will have the freedom and the infrastructure to build, secure, attack, monitor, and defend a full-on infrastructure. At the same time, I want to build all of this "as code". I really believe in converting everything to "as code", because this approach solves many issues of manual work, like manual maintenance, configuration drift, complex change history, and no version control. It also facilitates the ability to rebuild the infrastructure as quickly as possible. I was thinking of naming this project "Security Playground" at first or something like that, but I thought "Cyber Range as Code" was more catchy and it grasped the two main concepts that I want to focus on for this project.

The end goal of this project is to **build and maintain a real-world infrastructure** to:

- **Test attack scenarios** and
  - Monitor the result of the attack first-hand
  - Review the generated logs to understand what happened on the backend, what could be detected and created as a detection rule
- Implement **security standards** like ISO27001, NIS2, NIST, and CIS
- **Architect a secure infrastructure**: This will be a never ending goal, as, even in a relatively simple infrastructure like the one which will be built under this project, there always is something to improve in terms of security architecture.
- **Recover seamlessly**: **All the infrastructure must be deployed automatically**. The end goal is seamless recovery: If it is ever needed to redeploy the infrastructure on a new server, it can be done with minimal manual effort. This is where **the project leans heavily into IaC**, as it will be explained more in the next sections.

This project covers a lot of concepts about IaC, security hardening, security standards, blue teaming, and red teaming. Hopefully the end result will allow other security professionals to deploy this infrastructure and easily get to security testing.

In this first post, a **high level design** of the end result is described, of the **tech stack** that is planned to be used, as well as **an early PoC** regarding the infrastructure and its deployment using IaC.

So let's get started!

---

## Contents

* TOC
{:toc}

---

## Automated Infrastructure

As mentioned above, one core aspect of this project is to be able to **have the whole infrastructure deployed automatically**. This means that the following practices will be utilized.
- **Infrastructure as Code (IaC)**: All the infrastructure will be managed and provisioned **using code and configuration files**, instead of manual processes. This enables automation, consistency, version control, and swift, repeatable deployments. In the case specifically of this Cyber Range project, this will also allow quick recovery in case of an attack resulting in part of the infrastructure or the whole infrastructure being compromised and damaged unrecoverably.
- For security hardening and implementing security standards, in addition to IaC, the following practices will also be utilized:
  - **Policy as Code (PaC)**: PaC ([ref1](https://www.paloaltonetworks.com/cyberpedia/what-is-policy-as-code), [ref2](https://developer.hashicorp.com/sentinel/docs/concepts/policy-as-code) , [ref3](https://www.hashicorp.com/en/blog/policy-as-code-explained)) is **the practice of defining, updating, sharing and enforcing policies using code**. With this approach, when compared to manual processes to manage policies, the benefits are:
    - _Sandboxing_: Policies provide guardrails for other automated systems. By defining Policies as Code, **the verification by the policies is automated**, reducing the time needed of manual work.
    - _Codification_: Because the policy is defined as code, it is possible to **describe the logic about the policy directly on the code with comments**, which results in better understand and knowledge sharing of these policies.
    - _Version Control_: The benefits of version control are well known (history, diffs, pull requests, etc.).
    - _Testing_: Because the policies are defined as code, they can be tested by utilizing **automated testing** such as through a CI/CD pipeline. This allows for testing if a policy will result in the expected outcome before deploying to production.
    - _Automation_: Similarly to IaC, with PaC, tools can be used to **automatically deploy the policies to specified systems**.
    - _Compliance as Code (CaC) / Security as Code (SaC)_: This is where the concepts of [CaC](https://www.puppet.com/blog/compliance-as-code) and [SaC](https://www.isaca.org/resources/news-and-trends/isaca-now-blog/2024/security-as-code-a-key-building-block-for-devsecops) are also related.
    - Some **examples** of what rules can be enforced as policy with PaC are the following:
      - All 'victim' VMs must be on the isolated VLAN, not the management VLAN.
      - No single VM can be assigned more than 4 CPU cores or 8GB RAM.
      - All VMs must clone from the template X.
      - Terraform must not use the default OS users to perform actions.
      - Specific fields in Terraform must not be left empty.
      - Every VM must have a description field explaining its purpose.
  PaC is something that will be implemented at a later phase of the project, after the IaC phase is at a mature state.
- **Detection as Code (DaC)**: [DaC](https://www.legitsecurity.com/aspm-knowledge-base/detection-as-code) enables the writing, maintenance, and automation of the threat detection logic as if it were software code, making security a built-in part of the development pipeline. Similar to PaC, DaC will start being implemented at a later phase of the project.

## Tech stack

For this project, a wide variety of systems and technologies will be used, as the purpose is, among others, to simulate a real-world IT infrastructure.

The following technologies, for now, are strong contenders to be deployed. Since the project is just starting now, some technologies are only thought of as ideas, and may or may not actually be deployed when the time comes. I am looking for recommendations, though. Send me a message if you have something to recommend:

- [**Proxmox**](https://www.proxmox.com/en/): The whole infrastructure will be deployed on a physical server running Proxmox Virtual Environment (PVE) as a Type-1 Hypervisor. The infrastructure will be built with VMs and Containers.
- For the **IaC** aspect, the following 3 tools will be used:
  - [**Packer**](https://developer.hashicorp.com/packer): Packer is a community tool for **creating identical machine images** for multiple platforms from a single source configuration. Essential, Packer:
  1. Takes an ISO file of an OS
  2. Installs it on a temporary VM
  3. Applies any defined action during the installation (language, locale, hard disk to install the OS, etc.)
  4. Applies any configuration defined (e.g., IP address)
  5. Installs any package defined and
  6. Converts that VM to a template.
    That template is made for the platform on which the temporary VM was created on. In the case of this project, since the VM is created in Proxmox, the template will be a Proxmox VM template. **This template is then used by Terraform to provision VMs which will have that configuration ready immediately**. Packer provides the freedom of utilizing any ISO and converting it to a template exactly for the needs of the task at hand.
  - [**Terraform**](https://developer.hashicorp.com/terraform): HashiCorp Terraform is an **infrastructure as code tool** that enables the definition of both cloud and on-prem resources in human-readable configuration files that can be versioned, reused, and shared. Terraform can manage low-level components like compute, storage, and networking resources, as well as high-level components like DNS entries and SaaS features. **In the context of the project, the templates created by Packer will used by Terraform to provision all the infrastructure defined**. Note here that many of the configurations of the template defined by Packer can be changed during the provisioning by Terraform (e.g., RAM, CPU cores, installed packages, etc.), which provides more freedom during provisioning. But with Packer the benefit with the resulting template is that there is no need to install the packages already installed and execute any time consuming action every time a new VM is provisioned, because it was already done during the Packer VM template creation.
  - [**Ansible**](https://docs.ansible.com/): Ansible is an automation language which allows for automating essentially any IT task. **For the project, Ansible is used for further configuring all the VMs provisioned by Terraform**, like installing additional packages, starting services, connections, etc. Ansible uses SSH or WinRM to execute remote commands on machines. One of the benefits of Ansible is what is called **Idempotency**. This means that Ansible only makes changes if necessary, preventing unintended side effects. An Ansible playbook is written in YAML, and in a playbook the final desired state of the target machine is described. Then it is up to Ansible to make any changes or no changes depending on whether the target machine already has the final desired state.
- [**OPNsense**](https://opnsense.org/): OPNsense is an open-source next-gen grade firewall and routing platform, which brings all the features provided by commercial products to the open-source world. It is the most widely used open-source firewall, provides all the features necessary for this project, and **will be used as the central firewall**, controlling the traffic among the different subnets and machines.
  - [**Suricata**](https://suricata.io/): Suricata is an open-source network analysis and threat detection software. **Suricata will be the network-based IDS/IPS solution of this project**. **OPNsense provides an integration with Suricata**, which will allow for an experience of a next-gen firewall in just a few steps.
- **SIEM/XDR**: When it comes to the XDR solution to be deployed for this project, a few alternatives are taken into consideration:
  - [**Wazuh**](https://wazuh.com/): Wazuh is an open-source security platform, which brings a lot of features under the XDR and SIEM umbrella. This includes:
    - Configuration Assessment
    - Malware Detection
    - File Integrity Monitoring
    - Threat Hunting
    - Log Data Analysis
    - Vulnerability Detection
    - Incident Response
    - Regulatory Compliance
    - IT Hygiene
    - Containers Security
    - Posture Management
    - Workload Protection
  - [**Microsoft Defender XDR**](https://www.microsoft.com/en-us/security/business/siem-and-xdr/microsoft-defender-xdr#capabilities): One of the most widely used XDR solutions in the market. It is a full commercial XDR solution, and it would require multiple posts to describe it fully. While this is not a free solution, hopefully I will figure out how to implement it in a lab environment because I would be really interested in including Microsoft Defender XDR in this Cyber Range project.
  - [**Security Onion**](https://securityonionsolutions.com/): Security Onion is a free platform providing a series of features, including:
    - Network visibility using Suricata
    - Intrusion detection honeypots based on [OpenCanary](https://github.com/thinkst/opencanary)
    - Log management with the [Elastic Stack](https://www.elastic.co/elastic-stack)
    - File extraction with [Zeek](https://zeek.org/) or Suricata
    - Full packet capture with [Stenographer](https://docs.securityonion.net/en/2.4/stenographer.html)
    - File analysis with [Strelka](https://github.com/target/strelka)
    - Host visibility with [Elastic agent](https://www.elastic.co/elastic-agent)
    - Centralized management with [Elastic Fleet](https://www.elastic.co/docs/reference/fleet)
- [**ModSecurity**](https://modsecurity.org/) + [**OWASP Core Rule Set (CRS)**](https://coreruleset.org/): ModSecurity is an open source cross-platform **Web Application Firewall (WAF)**. Since its version 3 release, it now works as a standalone module which provides the capability to load/interpret rules written in the ModSecurity SecRules format and apply them to HTTP content provided by the web application via ModSecurity Connectors. This will prove useful, since in its previous module it worked as an Apache module only, while now it is more independent of the web server solution that it protects. ModSecurity on its own does not provide detection/protection rules. The OWASP CRS is a set of generic attack detection rules to be used with ModSecurity.
- The Actual **Infrastructure** that will be secured and attacked will include:
  - **Linux servers**:
    - [**DVWA**](https://github.com/digininja/DVWA): Damn Vulnerable Web Application (DVWA) is a PHP/MariaDB web application which **can be configured to be vulnerable against different types of web-based attacks**. This will be used for simulating attacks, as well as attempting to protect the application with different measures even though it is vulnerable.
    - [**OWASP Juice Shop**](https://owasp.org/www-project-juice-shop/): Similar to DVWA, it is another vulnerable web application.
    - [**OWASP WebGoat**](https://owasp.org/www-project-webgoat/): Another Java-based vulnerable application.
  - **Windows**: In order to get close to a real-world infrastructure, different Windows components will also need to be configured:
    - Active Directory Domain Services
    - Windows Workstations
    - Windows Servers with different configured roles
- For the **Red Team side**, a few potential and very useful candidates are the following:
  - [**Atomic Red Team**](https://www.atomicredteam.io/): It is an open source library of tests designed to test the applied security controls.
  - [**MITRE Caldera**](https://caldera.mitre.org/): Caldera is a much more complex framework than Atomic Red Team. It is a cybersecurity framework which can enable the following:
    - _Autonomous Adversary Emulation_: It is possible to build a specific threat profile and launch it in a network to see vulnerable points.
    - _Test and Evaluation of Detection, Analytic and Response Platforms_: It provides automated testing of cyber defense measures.
    - _Manual Red-Team Engagements_: It augments existing offensive toolsets.
  - [**PurpleSharp**](https://www.purplesharp.com/en/latest/): It is an adversary simulation tool **focused on Windows Active Directory** Environments. It currently supports 47 MITRE ATT&CK techniques.
  - [**Injection Monkey**](https://github.com/guardicore/monkey): Maintained by Akamai (originally by GuardiCore), it is an adversary emulation platform, where the basic idea is that a "worm" is dropped on a machine, and it tries to spread to every other machine in the network using common exploits and weak passwords.
- **Policy as Code**: In the later stages of the project, after the IaC part is at a mature state, the next step will be to proceed with the Policy as Code. For now, the potential candidates for PaC are the following:
  - [**Open Policy Agent**](https://www.openpolicyagent.org/): OPA is a general-purpose policy engine that unifies policy enforcement across the stack. It uses a **high-level declarative language that allows the specification of policies for a wide range of use cases**. In the case of the project, it will mostly be used for security hardening and compliance aspects.
  - [**HashiCorp Sentinel**](https://www.hashicorp.com/en/sentinel): Sentinel is a **Policy as Code framework for HashiCorp products**, defining what is allowed and what is prohibited. In the context of the project, Sentinel could be used for Packer and Terraform, but cannot be extended for Ansible playbooks.
- **Detection as Code**: Similarly to PaC, DaC is something that will be implemented later on. A few candidates identified are the following:
  - [**Sigma Rules**](https://sigmahq.io/docs/guide/about.html): Sigma rules are YAML files that contain all the information required to detect specified malicious behaviour when inspecting log files.
  - [**YARA**](https://virustotal.github.io/yara/): YARA is a tool that assists in defining malware samples. A YARA rule includes meta-information about the malware, and a set of strings and conditions to detect the malware (signatures).

## Architecture

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

## Initial Configuration / PoC

In this section, **an initial configuration of the infrastructure is described, with only the bare minimum of components**, as well as a **PoC of the infrastructure** working as described in the previous sections, i.e., with IaC practices.

All the code used in this PoC can be found under the project's repository [kostas-seclab](https://github.com/KostasKoutrou/kostas-seclab).

### Proxmox Configuration

The installation of Proxmox is a typical installation of any OS. Because the physical machine is connected to the home router, the following network configuration was added:

- **IP Address**: Static 192.168.0.50/24 - Outside of the DHCP range of the home router, but in the same subnet.
- **Gateway**: 192.168.0.1 - The IP Address of the home router.
- **DNS**: 1.1.1.1 - Used Cloudflare's general use DNS IP address.

<img alt="image" src="https://github.com/user-attachments/assets/d4e74abb-2a23-48a6-8d1a-d5d63a1840a8" />

#### Network Configuration

Proxmox VE is using the [Linux network stack](https://pve.proxmox.com/wiki/Network_Configuration). A Linux bridge interface (usually named _vmbrX_) is needed to connect guests to the underlying physical network. It can be thought of as a **virtual switch** which the **guests** and **physical interfaces** are connected to.

The network configuration for the project utilized Linux Bridges to create the defined zones.

<img alt="image" src="https://github.com/user-attachments/assets/8121de5d-694b-4668-a7d9-fabdb06bc79b" />

In the screenshot above, the utilized objects are the following:

|Name|Type|Description|
|-|-|
|nic1|Network Device|This is the **physical interface** which physically connects from the Proxmox VE server to the home router, enabling connectivity to the home PC and to the internet.|
|vmbr0|Linux Bridge|It is through this Linux Bridge that the Proxmox VE server actually **connects to the home router**, as depicted in the `Ports/Slaves` column, where the `nic1` is defined. The default gateway for the Proxmox server is also defined through this bridge.|
|vmbrDMZ20|Linux Bridge|**The Linux Bridge for the DMZ**. The "20" in the name is there for convenience of knowing which subnet the zone has (10.0.20.0/24). The IP of the bridge is 10.0.20.2/24, because the .1 will be assigned to the OPNsense interface.|
|vmbrEUZ40|Linux Bridge|**The Linux Bridge for the End Users zone**. The "40" in the name is there for convenience of knowing which subnet the zone has (10.0.40.0/24). The IP of the bridge is 10.0.40.2/24, because the .1 will be assigned to the OPNsense interface.|
|vmbrIZ30|Linux Bridge|**The Linux Bridge for the Internal Zone**. The "30" in the name is there for convenience of knowing which subnet the zone has (10.0.30.0/24). The IP of the bridge is 10.0.30.2/24, because the .1 will be assigned to the OPNsense interface.|

#### Proxmox Firewall

In addition to the OPNsense which will operate as the central Firewall of the infrastructure, **the [firewall](https://pve.proxmox.com/pve-docs/pve-admin-guide.html#chapter_pve_firewall) of Proxmox was also enabled**, with the purpose of isolating the Infrastructure from being able to access the home router local network. Think of it as a "Defence in Depth" approach.

Before explaining how firewalling works in Proxmox, let's briefly review how the VMs and nodes are structured in Proxmox:

<img width="253" height="343" alt="image" src="https://github.com/user-attachments/assets/ffaf5587-0b5b-45f2-b509-ecace5c234fd" />

In the above screenshot this structure is depicted. More specifically, there is:

- **Datacenter**: This is the top level of abstraction. Under datacenter all the proxmox servers are listed. If a Proxmox cluster was created, there would be more than one, but since in the project only one physical server is used, there is only one entry.
- **Proxmox VE Nodes (Host)**: The "kkproxmox" node is the physical server running the Proxmox VE.
- **VMs under each Proxmox VE Node**: Under each Proxmox VE node, the different resources deployed are listed. This includes VMs, VM templates, storage, network, etc.

The Proxmox VE firewall groups the network into the above **logical zones** (there is also a zone related to SDN which is out of scope for this PoC). The firewall as a functionality can be enabled on any of the zones described above. For example, if the firewall is needed to be enabled on a VM, then the firewall needs to be enabled on the Datacenter, the Proxmox node hosting that VM, the VM itself, and on each virtual network interface of that VM. Also, when a firewall rule is created on one of the levels described above, it applies to all the levels under it. For example, if a rule is create on the Proxmox VE Node, then it applies to all the VMs under that Node.

In Proxmox, firewall rules can be defined for different directions:

- **In**: Traffic that is arriving in a zone
- **Out**: Traffic that is leaving a zone
- **Forward**: Traffic that is passing through a zone. In the host zone this can be routed traffic (when the host is acting as a gateway or performing NAT). At a VNet-level this affects all traffic that is passing by a VNet, including traffic from/to bridged network interfaces.

There are default rules applied when enabling firewall. For `In` traffic the default rule is _Deny_ and for `Out` traffic, the default rule is _Allow_.

For the project, the following configuration is applied:

Firstly, on the **Datacenter** zone, Aliases were created for the private subnet IP ranges:

<img alt="image" src="https://github.com/user-attachments/assets/888a88f4-2792-4810-8073-50c6801cc75d" />

Two `Security Groups` were then created, which contain groups of firewall rules, which can be enforced on any zone afterwards, for easier application to multiple zones.

The first `Security Group` is the following, and contains rules to allow access to the management IP of OPNsense (the creation of which is decribed on a later section), and to reject traffic to the local subnet of the home network. This is to isolate the infrastructure from being able to reach out to the local subnet directly. Access to the internet is still allowed. Rule No. 0 was initially created to allow access to the Proxmox management IP, but it was later found that Proxmox includes [Default firewall rules](https://pve.proxmox.com/pve-docs/pve-admin-guide.html#pve_firewall_default_rules) that already permit this traffic, so the rule is now disabled.

<img alt="image" src="https://github.com/user-attachments/assets/c20b282f-9d3d-4619-a04e-6b94ac85624d" />

The second `Security Group` is the following, and contains a rule to allow traffic between the local subnets of the infrastructure. This traffic will be controlled further by the OPNsense.

<img alt="image" src="https://github.com/user-attachments/assets/02d58251-9355-420a-b321-d4c5b6c0dda2" />

After creating the two Security Groups, they were enforced on every zone:

On the **Proxmox** host only the first one is needed, because traffic towards the infrastructure local subnets does not enter or leave the host directly:

<img alt="image" src="https://github.com/user-attachments/assets/7d5ad664-c918-4ba1-b93f-d9f9111dc5a4" />

On the **OPNsense**, the configuration of which will be shown in a later section, both rules were added.

<img alt="image" src="https://github.com/user-attachments/assets/40c41f29-6796-42b0-acf5-d12e8bf56e3f" />

### Manual OPNsense Installation

Before moving to full IaC mode, as a PoC, OPNsense was installed manually on Proxmox.

The first step is to download the ISO from the [download page](https://opnsense.org/download/). Afterwards, the ISO is needed to be uploaded to Proxmox:

<img alt="image" src="https://github.com/user-attachments/assets/67ffe8d6-accb-4e93-b06d-3195513579db" />

#### VM Creation

When creating a VM in Proxmox, there are many options to select related to the OS, the system architecture, Disks architecture, etc. The configuration applied to OPNsense VM is shown in the following screenshots. Note that most selection were the default ones, because there is no significant difference to matter in the context of the project.

General Settings

<img alt="image" src="https://github.com/user-attachments/assets/cd0e382e-4f94-4795-9ca2-f3aa827ef87e" />

OS Settings: This is where the OPNsense ISO is selected, and also the guest OS type.

<img alt="image" src="https://github.com/user-attachments/assets/0b64421a-13e1-4701-9c52-d437e631da81" />

System Settings

<img alt="image" src="https://github.com/user-attachments/assets/2e577ee7-eac2-4616-8adf-2597fe1f3755" />

Disks Settings

<img alt="image" src="https://github.com/user-attachments/assets/aaa288a8-5220-4fff-a0bf-0f2faced53df" />

CPU Settings

<img alt="image" src="https://github.com/user-attachments/assets/ebd5a435-f05c-41af-a84b-478085422e48" />

Memory Settings: A useful setting which will be experimented with when deploying more VMs is the "**Ballooning Device**" setting along with the Minimum memory. [Memory ballooning](https://pve.proxmox.com/wiki/Dynamic_Memory_Management#Ballooning) allows you to have your guest **dynamically change its memory usage** by evicting unused memory during run time. It reduces the impact your guest can have on memory usage of your host by giving up unused memory back to the host. The Proxmox VE host can loan ballooned memory to a busy VM. The VM decides which processes or cache pages to swap out to free up memory for the balloon. The VM (Windows or Linux) knows best which memory regions it can give up without impacting performance of the VM. The Minimum memory setting defines the minimum memory in MiB which will never be freed up for use by other VMs. For now the Minimum memory is set to the actual memory size, so essentially there is no memory ballooning for the VM.

<img alt="image" src="https://github.com/user-attachments/assets/fc30cda5-8ba9-4a3c-ba16-42e678646265" />

Network Settings: As mentioned before, the WAN interface of the OPNsense will be connected to the same Linux Bridge as the Proxmox host and the home router. After creating the VM, more network interfaces will be created.

<img alt="image" src="https://github.com/user-attachments/assets/3fb8f750-6eeb-42c6-9655-2531f4a7a7e7" />

#### VM Installation

Installation with ZFS. After reading about ZFS vs UFS, it seems the ZFS handles unexpected power loss better while UFS may lead to data corruption, but it has a slightly higher RAM usage.

<img alt="image" src="https://github.com/user-attachments/assets/651a47cf-bf7e-42b6-9267-8418c56907e2" />

ZFS Configuration: We have only 1 disk for now, so stripe it is:

<img alt="image" src="https://github.com/user-attachments/assets/cf67261c-b823-48ee-b0fb-61770a7ebd58" />

Select the virtual disk created in the VM creation steps:

<img alt="image" src="https://github.com/user-attachments/assets/dc3616c7-72c1-40b9-9229-20cc09be9df7" />

Changed the root password and completed the install:

<img alt="image" src="https://github.com/user-attachments/assets/c178efdf-1d0f-4d36-bd2d-3d7cbf32457d" />

After installation, through the CLI, the interface and IP address assignments were completed:

Select option 1

<img alt="image" src="https://github.com/user-attachments/assets/0ea1c108-6d9a-4595-9ba7-724fa4e55197" />

The interface is recommended to be assigned automatically as it is the only interface at the moment.

<img alt="image" src="https://github.com/user-attachments/assets/a1423bc1-4028-44bd-880c-50ee19c3401a" />

After the interface assignment is completed, the IP address assignment is next (select option 2).

<img alt="image" src="https://github.com/user-attachments/assets/c9cca315-f284-453e-90ca-a499af903de0" />


<img alt="image" src="https://github.com/user-attachments/assets/e80f61a4-681b-4327-adc5-a595977ce7d1" />

In order for Proxmox to be able to exchange information between the host and guest, and to execute commands in the guest, the [Qemu guest agent](https://pve.proxmox.com/wiki/Qemu-guest-agent) needs to be installed.

<img alt="image" src="https://github.com/user-attachments/assets/77b766f5-c5ee-43e5-94cc-097b6ed2cc33" />

Then the service needs to be enabled:

<img alt="image" src="https://github.com/user-attachments/assets/2e205c47-d277-4ae7-8e52-22959f646553" />

#### OPNsense network interfaces

As shown in the network diagram, 3 more network interfaces were created on OPNsense, and connected to the corresponding Linux Bridges:

<img alt="image" src="https://github.com/user-attachments/assets/79b3738f-da4e-4b51-a154-1e853a8a411c" />

Then, each interface was assigned and enabled in OPNsense as well:

<img alt="image" src="https://github.com/user-attachments/assets/2ad5f191-43f1-45a3-b8b3-4f0e73848451" />

Each interface was assigned its corresponding IP address:

<img alt="image" src="https://github.com/user-attachments/assets/f9dca3e4-66eb-4122-9761-50a87c39dd5c" />

#### OPNsense Firewall rules

For firewall rules, since this is just a PoC, only 2 rules were created:

A rule to allow access to the Web GUI at the management IP of OPNsense.

<img alt="image" src="https://github.com/user-attachments/assets/2d9cf8b0-4864-48d2-97ca-decd084f5cae" />

A rule to allow machine in the End User Zone to access anything.

<img alt="image" src="https://github.com/user-attachments/assets/c16b0eb9-058e-43d4-bae4-6129eb3c430f" />

#### OPNsense DHCP

Additionally, DHCP was configured for the End User Zone:

<img alt="image" src="https://github.com/user-attachments/assets/012f58c0-194b-4aba-bf85-b9b7acdd88ff" />

### Packer Configuration

In order for Packer to work, the following was implemented:

#### Prepare Proxmox for Packer

**Create "Packer" Role and User on Proxmox:**

For the Role:

1. Go to Datacenter -> Permissions -> Roles.
2. Click `Create`. Name it `PackerRole`.
3. Add these privileges: VM.Allocate, VM.Config.HWType, VM.Config.CPU, VM.Config.Memory, VM.Config.Network, VM.Config.Disk, VM.Monitor, VM.Audit, VM.PowerMgmt, Datastore.AllocateSpace, Datastore.Audit, VM.Config.Options, SDN.Use VM.Backup+Clone+Console, VM.Config.CDROM, VM.Config.CloudInit, and VM.GuestAgent.Audit

<img alt="image" src="https://github.com/user-attachments/assets/7353549f-d79e-4013-84b8-b7483de64cb5" />

For the User:

1. Go to Datacenter -> Permissions -> Users.
2. Click `Add`. User name: `packer`, Realm: pve (more info at [Proxmox VE authentication](https://pve.proxmox.com/wiki/User_Management) - Authentication Realms).

<img alt="image" src="https://github.com/user-attachments/assets/aad65f22-b8fa-4c5d-ab13-f9eeaafc5241" />

Assign permissions to the User:

1. Go to Datacenter -> Permissions.
2. Click `Add` -> User Permission.
3. Path: / (This gives permission for the whole datacenter).
4. User: packer@pve.
5. Role: PackerRole.

<img alt="image" src="https://github.com/user-attachments/assets/54ed22d9-b447-4fc4-81f6-e0774f71a938" />

**Generate API Token**

This is the "Password" Packer will use to talk to the Proxmox API.
1.	Go to Datacenter -> Permissions -> API Tokens.
2.	Click `Add`.
3.	Select User: packer@pve.
4.	Token ID: packer-token.
5.	Uncheck "Privilege Separation" (this ensures the token has the same rights as the user `packer`).
6.	Click `Add`.
CRITICAL: Proxmox will show you the Token Secret **only** at the time of its creation. If it is missed at that time, a new token must be created.

<img alt="image" src="https://github.com/user-attachments/assets/8058d9e5-93be-4788-aa32-8612f96b8f12" />

#### Preparing packer

To install Packer, the [official method](https://developer.hashicorp.com/packer/tutorials/docker-get-started/get-started-install-cli) resulted in some errors, and the method describe in this [stackoverflow post](https://stackoverflow.com/questions/75254685/gpg-error-https-apt-releases-hashicorp-com-bionic-inrelease-the-following-si) worked properly, which includes running the following commands and will be used to install Terraform, too.

```bash
# Source - https://stackoverflow.com/a
# Posted by Thilina Ashen Gamage
# Retrieved 2026-01-28, License - CC BY-SA 4.0

# GPG is required for the package signing key
sudo apt install gpg

# Download the signing key to a new keyring
wget -O- https://apt.releases.hashicorp.com/gpg | gpg --dearmor | sudo tee /usr/share/keyrings/hashicorp-archive-keyring.gpg

# Verify the key's fingerprint
gpg --no-default-keyring --keyring /usr/share/keyrings/hashicorp-archive-keyring.gpg --fingerprint

# The fingerprint must match 798A EC65 4E5C 1542 8C8E 42EE AA16 FCBC A621 E701, which can also be verified at https://www.hashicorp.com/security under "Linux Package Checksum Verification".

# Add the HashiCorp repo
echo "deb [signed-by=/usr/share/keyrings/hashicorp-archive-keyring.gpg] https://apt.releases.hashicorp.com $(lsb_release -cs) main" | sudo tee /etc/apt/sources.list.d/hashicorp.list

# apt update successfully
sudo apt update

sudo apt install packer
sudo apt install terraform
```

However, because I am currently running bash via Windows Subsystem for Linux (WSL), it is difficult to grab the IP address of the physical PC, which makes packer fail when opening the temporary web server to give the `user-data` file. Therefore, for this iteration we are running Packer on Windows, and for the next phase we will probably add the `user-data` file to Proxmox directly for more seamless implementation.

To install Packer for windows, you just download it from [here](https://developer.hashicorp.com/packer/install), unzip it, and add the path of the exe to the Path environment variable of the PC:

<img width="630" height="720" alt="image" src="https://github.com/user-attachments/assets/38936949-0060-46ef-bbf5-1b4eea0b5021" />

<img alt="image" src="https://github.com/user-attachments/assets/016dc5ee-a194-465e-b68e-22f942dc74f3" />

To organize the Packer directory, a `packer` path was created, under which all the other files and directories were created.

For the PoC, 2 Ubuntu VMs will be created. One will be under the `End user Zone` and the other under the `WAN Zone`. Because Packer creates ready-to-provision images, an image is to be created for every OS planned to be provisioned with Terraform. Therefore, under the `packer` path, a sub-path was created named `ubuntu-2404` for this purpose.

All the created files are found under the project's repository [kostas-seclab](https://github.com/KostasKoutrou/kostas-seclab)

**Create Credentials**

The following `packer/credentials.pkrvars.hcl` file was created, which contains credentials to be used in Packer, and later on in Terraform as well.

<img width="565" height="105" alt="image" src="https://github.com/user-attachments/assets/74f29d00-3b08-483e-b739-e6d59740de6d" />

**Create `user-data` file**

A YAML file `packer/ubuntu-2404/http/user-data` was created. When running Packer, the PC becomes a temporary web server, and the machine pulls the `user-data` file from the PC from the `http` path.

To better understand how the `user-data` and the next files are involved during the image building using Packer, this is a great point where the tool called [**Cloud-init**](https://cloudinit.readthedocs.io/en/latest/) should be briefly described. **Cloud-init is the industry standard multi-distribution method for cross-platform cloud instance initialization**. It is supported across all major public cloud providers, provisioning systems for private cloud infrastructure, and bare-metal installations.

When a VM boots, cloud-init runs and provides the necessary glue between launching a cloud instance and connecting to it so that it works as expected. It looks for metadata provided by proxmox or terraform and performs different initialization tasks automatically, including:

- Network: Sets static IP or DHCP config
- Identity: changes the hostname from ubuntu-template to what is defined
- Security: injects public SSH keys to log in without password
- Growth: expands the disk partition to fill the size you gave it in proxmox.

Cloud-init grabs the `ubuntu.pkr.hcl` which is described next for instructions for proxmox and the `user-data` file for instructions for the guest (the OS).

But first, the `user-data` file is shown below:

```yaml
#cloud-config
autoinstall:
  version: 1
  identity:
    hostname: ubuntu-template
    username: lab-admin
    password: "$6$wFmQrqy8bMHGTQ.O$1WWGjLd3buuOov83OY7zJbdw5Z9Gx4C3ueH04GZGHzqz6h7Jy0TelUUOisEt/1GJwQifYKYKVfj17vkd0mk0f0"
  user-data:
    disable_root: false
  locale: en_US.UTF-8
  timezone: UTC
  keyboard:
    layout: us
  ssh:
    install-server: true #install openssh
    allow-pw: true #allow password authentication in ssh
  packages:
    - qemu-guest-agent #qemu guest agent so that proxmox retrieves info like IP address.
    - cloud-init #needed so the VM is configurable via Terraform.
  storage:
    layout:
      name: direct #use the entire virtual disk as one big partition.
```

In the `user-data` file, the `autoinstall` directive is the set of instructions for the Ubuntu Subiquity installer. It provides all the information required during the initial installation of the ubuntu server, including:

- The hostname
- The admin user
- Local
- Timezone
- Keyboard layout
- SSH settings
- Packages to be installed
- Storage settings

Let's look now at the `ubuntu.pkr.hcl` file:

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
variable "ubuntu_pw" {
  type = string
  sensitive = true
}

source "proxmox-iso" "ubuntu-server" { #Resource type and local name
  proxmox_url = var.proxmox_api_url
  username    = var.proxmox_api_token_id
  token       = var.proxmox_api_token_secret

  # Skip TLS Verification for self-signed certificates
  insecure_skip_tls_verify = true
  # qemu_agent = true # Default is true anyway

  node    = "kkproxmox"
  vm_id   = 1000
  vm_name = "ubuntu-2404-template"

  # iso_file = "local:iso/ubuntu-24.04.3-live-server-amd64.iso"

  boot_iso {
    # type = "scsi"
    type = "ide"
    iso_file = "local:iso/ubuntu-24.04.3-live-server-amd64.iso"
    iso_checksum = "sha256:c3514bf0056180d09376462a7a1b4f213c1d6e8ea67fae5c25099c6fd3d8274b"
    unmount = true
  }

  cores = 4
  memory = 4096

  network_adapters {
    model  = "virtio"
    bridge = "vmbr0" # Will probably change it in the Terraform script, this is only for packer.
  }

  disks {
    disk_size    = "20G"
    storage_pool = "local-lvm"
    type         = "scsi"
    ssd          = true
  }

  cloud_init = true # add an empty Cloud-Init CDROM driver after the VM has been converted to a template.
  cloud_init_storage_pool = "local-lvm" # Name of the Proxmox storage pool to store the Cloud-Init CDROM on.

  boot_command = [
    "<esc><wait>", "e<wait>",
    "<down><down><down><end>",
    " autoinstall cloud-config-url=http://{{ .HTTPIP }}:{{ .HTTPPort }}/user-data ds='nocloud-net;s=http://{{.HTTPIP}}:{{.HTTPPort}}/'",
    "<f10>"
  ]

  http_directory = "http"
  ssh_username   = "lab-admin"
  ssh_password = "${var.ubuntu_pw}"
  ssh_timeout    = "20m"
}

build {
  sources = ["source.proxmox-iso.ubuntu-server"]

  provisioner "shell" {
    # execute_command = "echo ${var.ubuntu_pw}| sudo -S sh -c '{{ .Vars }} {{ .Path }}'"
    execute_command = "echo ${var.ubuntu_pw}| {{.Vars}} sudo -S -E sh -eux '{{.Path}}'"
    inline = [
      "echo 'Waiting for cloud-init to complete...'",
      "while [ ! -f /var/lib/cloud/instance/boot-finished ]; do echo 'Still waiting...'; sleep 2; done",
      "echo 'Cloud-init completed successfully'",
      "echo 'Cleaning up...'",
      "rm -rf /var/lib/apt/lists/*",
      "rm -rf /tmp/*",
      "rm -rf /var/tmp/*",
      "cloud-init clean --logs --machine-id --seed --configs all"
    ]
  }
}
```
{% endraw %}

This file is called the `Packer Template`, and include all the information for how to build the template. It includes:

{% raw %}
- The **required packer plugins** for this template. In this case, the Proxmox plugin is required only, to define the rest of the information and know how to communicate with the Proxmox API.
- **Variable declaration to be used in the packer build** in the next lines of the file. Note here that at this point only the declaration of the variables is done. The actual value of the variables will be provided via the `credentials.pkrvars.hcl` file during the packer build execution.
- The `source` block is the core logic. It defines the different values that the VM should have, like CPU cores, memory size, ssh settings, the ISO file to use, the VM name, where to install the VM in proxmox, etc. The values that are not explicitly defined here have default values set provided by the proxmox plugin.
  - `boot_command`: A very valuable part of the `source` block is the `boot_command`. In it, the actual keyboard presses for the installation are defined. With these boot commands, what is done is that the installation file is edited to pull the installation parameters from the `user-data` file described above. More specifically, it contains Linux Kernel Boot Parameters before the installation starts:
    - `autoinstall`: This tells the Ubuntu installer (Subiquity) to run in automated mode, instead of going through the manual installation process of selecting language etc.
    - The variables {{ .HTTPIP }} and {{ .HTTPPort }}: these are the IP of the PC and the port that temporarily serves the `user-data` file.
    - `cloud-config-url`: This parameter tells the installer where the file of `user-data` is.
    - `ds`: This stands for "Data Source" and Cloud-init uses its contents:
      - `nocloud-net`: This tells Cloud-init that the data source is not Public cloud (Azure/AWS/GCP etc.) and to look at the local network for the file.
      - `s=`: This stand for "seed from". It tells Cloud-init where to find the `user-data` and `meta-data` (another file not used for this PoC) files.
- The `build` block is the execution part of packer. The `source` block defines the VM, while the `build` block actually runs it. Here is where the [**Provisioners**](https://developer.hashicorp.com/packer/docs/provisioners) live, through which it is possible to execute actions on the machine image and configure it after booting. They can be used to install packages, patch the kernel, create users, download application code, etc. In the PoC, the `shell` provisioner is used to clean up the machine and reinitialize cloud-init so that it reruns during the Terraform provisioning of the VM (cloud-init runs only once on the machine unless its configuration is cleaned).
{% endraw %}

#### Packer pitfalls, solutions, and lessons learnt

Some tools that helped in troubleshooting Packer:
1. Set the envar `$env:PACKER_LOG=1` to make running packer verbose. This shows many hints as to what is going wrong.
2. If the VM does not proceed with the autoinstaller, and reaches the manual installation flow, you can use `Ctrl + Alt + F2` in the Proxmox console to **jump to a root shell**. This allows to
  1. **review logs of the subiquity and cloud-init**. Two useful paths for checking are `/var/log/installer/subiquity-server-debug.log` and `/var/log/syslog`.
  2. **test network connections**, to confirm, for example, that you can reach the temporary packer server on the PC running packer, or that the internet is reachable.
3. **Check the hardware resource usage of the VM** being installed in Proxmox. Maybe there is overuse of the resource and may need to be increased.

Some **pitfalls and lessons** learnt I met while setting up packer were the following:

{% raw %}
1. **Proxmox 500 Error**: After enabling verbose mode in packer (by setting `$env:PACKER_LOG=1`), a repeated message of 500 "Qemu must be running to read the IP address of the machine" kept popping up. I thought this meant that something was wrong, but actually this is expected and Packer is actually waiting for Qemu guest agent to be installed so that it can read the VM's IP Address from proxmox so it can SSH to it. If it never moves that point, then the QEMU guest agent was not installed, or, most probably, the autoinstaller did not start.
2. **The "Interactive Menu" problem**:
  - **The issue**: The VM kept reaching the manual language selection screen instead of starting the automated install.
  - **The cause**: The installer could not find or parse the autoinstall instructions.
  - **The resolution**: In the `boot_command`, an explicit definition of the autoinstall instructions needed to be defined using the `cloud-config-url` parameter.
3. **YAML formatting**
  - **The issue**: The installer found the user-data file but ignored the settings, falling back to manual installation.
  - **The cause**: Ubuntu ignores the user-data file if it does not have the "#cloud-config" comment at the first line.
  - **The lesson**: read the documentation of the requirements of different configuration files.
4. **Validation crashes**:
 - **The issue**: The installer crashed with an "Unknown keyboard layout 'en'" error (1st screenshot), and crashed while attempting to run timedatectl list-timezones (2nd screenshot).
 - **The cause and resolution**: Some parameters in the `user-data` file are mandatory and do not have default values when not explicitly defined. These include the timezone and keyboard layout parameters, which were not initally defined. Once added to the `user-data` file, the crashes were fixed.
  Keyboard layout error:
  <img alt="image" src="https://github.com/user-attachments/assets/e1db920c-62d7-4366-95da-2dea2d3b3689" />
  Timezone error: 
  <img alt="image" src="https://github.com/user-attachments/assets/f58cd53a-93ec-4de2-8d53-074a1fc7733b" />
5. **Hardware resource constraints**
  - **The issue**: The installation appeared to hang or move extremely slowly without showing errors.
  - **The cause**: Initially the default memory and CPU values were left in the packer template file, which means 1 CPU core and 512MB memory. Especially for the memory size, this is a very low value for an installation of Ubuntu server.
  - **The solution and lesson**: Modern installers require more thant the default minimal resources to complete. Increase the values for 4 CPU cores and 4096MB memory resolved the issue.
6. **SSH Handshake and authentication**
 - The issue: The OS installed successfully, but Packer failed to connect with the following repeating error:
```plaintext
2026/01/26 10:59:47 packer-plugin-proxmox_v1.2.3_x5.0_windows_amd64.exe plugin: 2026/01/26 10:59:47 [INFO] Attempting SSH connection to 192.168.0.110:22...
2026/01/26 10:59:47 packer-plugin-proxmox_v1.2.3_x5.0_windows_amd64.exe plugin: 2026/01/26 10:59:47 [DEBUG] reconnecting to 
TCP connection for SSH
2026/01/26 10:59:47 packer-plugin-proxmox_v1.2.3_x5.0_windows_amd64.exe plugin: 2026/01/26 10:59:47 [DEBUG] handshaking with SSH
2026/01/26 10:59:47 packer-plugin-proxmox_v1.2.3_x5.0_windows_amd64.exe plugin: 2026/01/26 10:59:47 [DEBUG] SSH handshake err: ssh: handshake failed: ssh: unable to authenticate, attempted methods [none], no supported methods remain
2026/01/26 10:59:47 packer-plugin-proxmox_v1.2.3_x5.0_windows_amd64.exe plugin: 2026/01/26 10:59:47 [DEBUG] Detected authentication error. Increasing handshake attempts.
```
  - **The cause and resolution**: the `ssh_password` value in the packer template is used for this connection, and was not initially defined. Upon its definition, the issue was resolved.
7. **sudo asking password on shell provisioner in packer**
  - **The issue**: When running the "shell" provisioner in Packer build, it asked for a manual input of the user password to execute the sudo commands.
  - **The cause**: the "shell" provisioner has a variable called `execute_command`. What shell provisioner does essential is that it converts all the commands provided to a script file, and then executes that script with a predetermined `execute_command`. This can be edited to satisfy any need.
  - **The solution**: The `execute_command` value was changed to `"echo ${var.ubuntu_pw}| {{.Vars}} sudo -S -E sh -eux '{{.Path}}'"`, which pulls the password and inputs it for when sudo asks for it.
8. **Cloud-init status --wait not working in shell provisioner**
  - **The issue**: In the shell provisioner, when trying to clean up the cloud-init state, at first it was tried to run the "cloud-init status --wait" command, to wait for cloud-init to complete before reinitializing. This command would return status code 2 which means that the cloud-init process has not finished, while shell provisioner only accepts status code 0 as a "non-error" status code, so Packer would crash and mention this as an error.
  - **The Solution**: The /var/lib/cloud/instance/boot-finished file is now being checked instead to verify that cloud-init finished running.
{% endraw %}

#### Packer build and PoC result

To run packer, the following commands are used, executed at the path of `packer/ubuntu-2404/`:

```powershell
packer init .
packer validate -var-file="../credentials.pkrvars.hcl" . # To Validate the packer configuration before building
packer build -var-file="../credentials.pkrvars.hcl" . # To Build the template
```

Running the packer validate command results in the following:

<img width="697" height="55" alt="image" src="https://github.com/user-attachments/assets/209c4b78-27e1-49e1-b411-b01bb6e044e8" />


Running the packer build command results in the following:

<img alt="image" src="https://github.com/user-attachments/assets/acf0f158-9ece-4ab3-b59d-a71c2c222fcd" />

While the VM is being installed, in the Console window it is possible to review what is being run:

<img alt="image" src="https://github.com/user-attachments/assets/3d421b7d-5c40-4d06-8eb4-f5d63302e1c9" />

After successfully running Packer, the result is a Proxmox template, ready to be cloned. This template is used by Terraform, as shown in the next section.

<img alt="image" src="https://github.com/user-attachments/assets/862f0292-2481-4aa2-8862-6ba033da0d88" />

### Terraform Configuration

In order for Terraform to work, the following was implemented:

#### Prepare Proxmox for Terraform

**Update "Packer" Role in Proxmox for Terraform**

For simplicity, the same user used in Packer is used for Terraform. Therefore, the permissions required for Terraform were provided to the previously created `PackerRole` role. The permissions required are described [here](https://registry.terraform.io/providers/Telmate/proxmox/latest/docs), and are the following:

- Datastore.AllocateSpace
- Datastore.AllocateTemplate
- Datastore.Audit
- Pool.Allocate
- Pool.Audit
- Sys.Audit
- Sys.Console
- Sys.Modify
- VM.Allocate
- VM.Audit
- VM.Clone
- VM.Config.CDROM
- VM.Config.Cloudinit
- VM.Config.CPU
- VM.Config.Disk
- VM.Config.HWType
- VM.Config.Memory
- VM.Config.Network
- VM.Config.Options
- VM.Migrate
- VM.PowerMgmt
- SDN.Use

#### Preparing Terraform

To organize the Terraform directory, a `terraform` path was created, under which all the other files and directories were created.

**For the PoC, 2 Ubuntu VMs will be created**. One will be under the `End user Zone` and the other under the `WAN Zone`.

All the created files are found under the project's repository [kostas-seclab](https://github.com/KostasKoutrou/kostas-seclab)

For terraform, the following files were created:

The `terraform/variables.tf` file contains **variables to be used by terraform**:

```terraform
variable "proxmox_api_url" {
    type = string
    description = "The URL for the Proxmox API (e.g., https://192.168.0.50:8006/api2/json)"
}

variable "proxmox_api_token_id" {
    type = string
    sensitive = true
    description = "The API Token ID (e.g., root@pam!terraform)"
}

variable "proxmox_api_token_secret" {
    type = string
    sensitive = true
    description = "The API Token Secret (UUID)"
}
```

The `terraform/provider.tf` file contains information regarding **what provider terraform will connect to** (AWS, Azure Google Cloud, Proxmox, etc. - In our case it is proxmox, using the [Telmate/proxmox](https://registry.terraform.io/providers/Telmate/proxmox/latest/docs) provider), as well as **the configuration to connect to the provider**. The variables for the provider are pulled from the `terraform/variables.tf` definitions. The execution of `terraform apply` is where the source and the actual values of the varaibles will be defined.

```terraform
terraform {
  required_providers {
    proxmox = {
        source = "Telmate/proxmox"
        version = "3.0.2-rc07"
    }
  }
}

provider "proxmox" {
    pm_api_url = var.proxmox_api_url
    pm_api_token_id = var.proxmox_api_token_id
    pm_api_token_secret = var.proxmox_api_token_secret

    pm_tls_insecure = true
}
```

The `terraform/main.tf` file contains **the blocks of what exactly to build**. In our case of building 2 test Ubuntu servers using the template built by Packer, the main.tf looks like this:

```terraform
resource "proxmox_vm_qemu" "test_server" { # Resource type and resource name
    name = "terraform-vm-01" # Name of the VM
    target_node = "kkproxmox" # Proxmox node to build the VM on

    clone = "ubuntu-2404-template" # Clone the ubuntu template built by Packer

    agent = 1 #enable QEMU guest agent
    
    cpu {
        cores = 2
        sockets = 1
        type = "host"
    }
    memory = 2048
    scsihw = "virtio-scsi-pci"
    # bootdisk = "scsi0"

    # This is added to the default ciuser "ubuntu". "ssh-keygen -t rsa" was used to generate the key pair. This is needed for Ansible to SSH with.
    sshkeys = "${file("~/.ssh/id_rsa.pub")}"

    disk { # The cloudinit disk needs to be defined explicitly for it to be mounted temporarily for cloud-init to run.
        slot    = "ide0"
        type    = "cloudinit"
        storage = "local-lvm"
    }

    disk {
        slot = "scsi0"
        size = "20G"
        type = "disk"
        storage = "local-lvm"
        format = "raw"
    }

    startup_shutdown { # These were defined because these are the values that Proxmox puts on the VMs by default while Terraform tried to put different values, so every time that "terraform apply" would run, it would try to change them, for Proxmox to change them back right after.
        order = -1
        shutdown_timeout = -1
        startup_delay = -1
    }

    network { # This VM connects to the WAN Zone Linux Bridge
        id = 0
        model = "virtio"
        bridge = "vmbr0"
    }

    os_type = "cloud-init" # This is used together with the "cloudinit" disk to apply the cloud-init related configuration (the following commands).
    ipconfig0 = "ip=dhcp" # get IP from the DHCP
    skip_ipv6 = true # do not get IPv6
}

resource "proxmox_vm_qemu" "test_server1" { # the 2nd VM
    name = "terraform-vm-02" # Name of the VM
    target_node = "kkproxmox" # Proxmox node to build the VM on

    clone = "ubuntu-2404-template" # Clone the ubuntu template built by Packer

    agent = 1 #enable QEMU guest agent
    
    cpu {
        cores = 2
        sockets = 1
        type = "host"
    }
    memory = 2048
    scsihw = "virtio-scsi-pci"
    # bootdisk = "scsi0"

    disk { # The cloudinit disk needs to be defined explicitly for it to be mounted temporarily for cloud-init to run.
        slot    = "ide0"
        type    = "cloudinit"
        storage = "local-lvm"
    }

    disk {
        slot = "scsi0"
        size = "20G"
        type = "disk"
        storage = "local-lvm"
        format = "raw"
    }

    startup_shutdown { # These were defined because these are the values that Proxmox puts on the VMs by default while Terraform tried to put different values, so every time that "terraform apply" would run, it would try to change them, for Proxmox to change them back right after.
        order = -1
        shutdown_timeout = -1
        startup_delay = -1
    }

    network { # This VM connects to the End User Zone Linux Bridge
        id = 0
        model = "virtio"
        bridge = "vmbrEUZ40"
    }

    os_type = "cloud-init" # This is used together with the "cloudinit" disk to apply the cloud-init related configuration (the following commands).
    ipconfig0 = "ip=dhcp" # get IP from the DHCP
    skip_ipv6 = true # do not get IPv6
}
```

#### Terraform pitfalls, solutions, and lessons learnt

Some pitfalls and lessons learnt I met while setting up Terraform were the following:

1.	**Stale Cloud-Init State**
 - **The issue**: After deploying a VM with Terraform, Proxmox showed the correct name (e.g., terraform-vm-01), but **the internal OS console still showed the template's original name (ubuntu-server)**. The VM ignored the new configuration.
 - **The Cause**: **cloud-init is designed to run only once**. During the Packer build process, cloud-init ran to set up the initial template and left a "marker file" indicating it was finished. When the template was cloned via Terraform, the new VM saw the marker and immediately went back to sleep, ignoring the new instructions from Terraform.
 - **The Fix**: **A "cleanup" shell provisioner was added** at the end of the Packer configuration (ubuntu.pkr.hcl), which runs the command `cloud-init clean --logs --machine-id --seed --configs all` to fully reinitialized the cloud-init status.
 - **The Lesson**: **Templates must be stateless**. If the state is not reset before saving a template, every clone will wake up thinking it has already been configured.
2. **Duplicate Machine IDs**
 - **The issue (Potential)**: While fixing the previous issue, a risk was identified where **multiple VMs might end up with the same IP address from the DHCP server**.
 - **The Cause**: **Linux generates a unique /etc/machine-id upon installation**. If this isn't reset during the Packer build, every clone shares the same ID, causing network conflicts.
 - **The Fix**: In the `cloud-init clean` command in Packer, **the flag `--machine-id` was added to reset the machine id as well**.
 - **The Lesson**: Simply changing a hostname isn't enough. **Unique system identifiers (GUIDs, Machine IDs) must be regenerated for every new instance** to avoid collisions on the network level.
3.	**No Cloud-Init Drive**
 - **The issue**: Terraform was successfully creating the VM resource, but **the OS configuration (hostname, user data) was never applied**. When manually mounting a Cloud-Init drive fixed this issue.
 - **The Cause**: **The Packer template was missing the `cloud_init = true` instruction**. Consequently, the template was created without a Cloud-Init drive. Without this drive, Terraform had no physical medium to insert the configuration data for the OS to read. Additionally, in the Terraform main.tf file, the "cloudinit" was also needed to be explicitly defined.
 - **The Fix**: The source "proxmox-iso" block in Packer was updated to explicitly **include the `cloud-init` parameter,** and Terraform to include the "cloudinit" disk.

#### Terraform apply and PoC result

To run terraform, the following commands were used:

```bash
terraform init # To initialize terraform
terraform plan -var-file=../packer/credentials.pkrvars.hcl # To validate that the terraform config is ready to be applied.
terraform apply -var-file=../packer/credentials.pkrvars.hcl # To apply the Terraform config and build the defined VMs.
```

The `credentials.pkrvars.hcl` file was used as the var-file, because it contains the same info required for Terraform to run.

Unlike Packer, Terraform was run in WSL, because the ssh key defined in the Terraform `main.tf` file will be used by Ansible in the next step, and Ansible cannot be run in Windows, only in Linux.

Running `terraform init` results in the following:

<img width="565" height="275" alt="image" src="https://github.com/user-attachments/assets/a0dcbe99-67c6-485c-b3db-5b8faaf158d8" />

Running `terraform plan -var-file="../packer/credentials.pkrvars.hcl"` outputs many lines which show the configuration to be applied. Below is a screenshot of part of the output:

<img alt="image" src="https://github.com/user-attachments/assets/7a5b9108-aeb8-46e5-b9ca-1a66b3a9fee5" />

Running the `terraform apply -var-file="../packer/credentials.pkrvars.hcl"` shows the same information as the `terraform plan`, which is the configuration to be applied, and asks for confirmation in the end. After typing "yes", the VMs are being provisioned, and the final output is the following:

<img width="593" height="523" alt="image" src="https://github.com/user-attachments/assets/f34fb350-5e3e-4e79-ad68-2372d76f4987" />

After 2 minutes, in Proxmox we can see the provisioned VMs up and runnning:

<img alt="image" src="https://github.com/user-attachments/assets/dc1a165b-e520-4022-ab24-bec391d18158" />

Now that the VMs are provisioned, the next step is to apply further configurations using Ansible.

### Ansible

Ansible cannot be run in Windows PowerShell, so WSL was used.

To install Ansible, use `sudo apt install ansible`

Ansible uses ssh keys to SSH to machine and apply configurations. To generated the ssh key pair, run `ssh-keygen -t rsa`:

<img width="808" height="605" alt="image" src="https://github.com/user-attachments/assets/ed8b1b3b-7b85-4118-a156-09add815c186" />

Then, in the Terraform main.tf file, the public key was added to one of the machines:

`sshkeys = "${file("~/.ssh/id_rsa.pub")}"`

For Ansible, the following files were created:

`ansible/inventory.ini`: This file contains the list of machines with their IP addresses for Ansible to connect to them.

```ini
[webservers]
192.168.0.104 ansible_user=ubuntu
```

`ansible/install_nginx.yml`: This file contains the state for this PoC of the configuration to be applied to the machines of inventory.ini. For the purposes of the PoC, only an Nginx server is installed on one of the two Linux Servers.

```yaml
---
- name: Configure Webservers
  hosts: all
  become: true #sudo

  tasks:
    - name: Update apt cache
      apt:
        update_cache: yes
        cache_valid_time: 3600

    - name: Install Nginx
      apt:
        name: nginx
        state: present

    - name: Start Nginx service
      service:
        name: nginx
        state: started
        enabled: yes
```

`ansible/ansible.cfg`: This file override the global Ansible settings:

```ini
[defaults]
inventory = ./inventory.ini
host_key_checking = False
```

The ansible.cfg file had to be set by setting the environment variable (because we are running with WSL): `export ANSIBLE_CONFIG=./ansible.cfg`. To confirm, run `ansible --version`, which should show the `config file` value:

<img alt="image" src="https://github.com/user-attachments/assets/7ac8a653-ee6a-4c07-82fa-f6b17f0a801e" />

#### Running Ansible

To run Ansible with the configuration defined above, the following command was run: `ansible-playbook install_nginx.yml`

<img alt="image" src="https://github.com/user-attachments/assets/4a414b9c-6f01-4da1-9984-69ad5cf00a17" />

Due to Ansible's idempotency, running the command again will show that everything is ok and no changes were applied:

<img alt="image" src="https://github.com/user-attachments/assets/2fe0b7d3-f372-434c-94ec-3f3ab3f059d1" />

After running Ansible, the Nginx can be reached from the browser

<img width="765" height="265" alt="image" src="https://github.com/user-attachments/assets/ae5c13d6-23e0-4c80-ae2f-b095bf0873f4" />

## Next Steps

So now the flow is complete. A VM template was created using Packer, VMs were provisioned with Terraform, and further configurations were applied using Ansible. This was a PoC to verify the flow can be used for the rest of the project.

**The next steps** will be to reach a fully recoverable and fully IaC state from start up to the point which has been reached now manually.

This means:

1. Configuring all the **Proxmox settings** described in this post using either Ansible or any other automation tool (even a script).
2. Configuring **OPNsense** using Packer, Terraform and Ansible. A Packer template will be built using OPNsense's ISO, Terraform will be used to provision the OPNsense with the settings which were applied manually, and Ansible will be used to apply any configuration to it, including Firewall rules, DHCP, installing Packages, etc.

## Conclusion

Building a Cyber Range as Code is going to be a complex project to pull off fully "as Code". It will require testing and several iterations in order for it work properly.

However, this project will be a great lesson towards:

1. Learning IaC (Packer, Terraform, Ansible).
2. Applying security hardening and security standards to multiple machines using IaC practices.
3. Developing methodologies for security testing and reviewing the results.

This post included the plan for the project. It included different ideas for potential technologies to be used, as well as the architecture to be built. In the next posts the implementation will start.

Hopefully you picked something of interest and will follow along the progress of this project.

See you in the next one.
