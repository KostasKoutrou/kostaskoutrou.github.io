# Cyber Range as Code: Automating Security Lab with IaC - Part 1

## Introduction

What is a Cyber Range?

> A [Cyber Range](https://www.ibm.com/think/topics/cyber-range) is a virtual environment for cybersecurity training, testing, and research that simulates real-world networks and cyberattacks.

I always wanted to build a security homelab, where I will have the freedom and the infrastructure to build, secure, attack, monitor, and defend an full-on infrastructure. At the same time, I want to build all of this "as code". I really believe is converting everything as code, because this approach solves many issues of manual work, like manual maintenance, configuration drift, complex change history, and no version control. It also facilitates the ability to rebuild the infrastructure as quickly as possible. I was thinking of naming this project "Security Playground" at first, but I thought "Cyber Range as Code" was more catchy and it grasped the two main concepts that I want to focus at for this project.

The end goal of this project is to **build and maintain a real-world infrastructure** to:

- **Test attack scenarios** and
  - Monitor the result of the attack first-hand
  - Review the generated logs to understand what happened on the backend, what could be detected and created as a detection rule
- Implement **security standards** like ISO27001, NIS2, NIST, and CIS
- **Architect a secure infrastructure**: This will be a never ending goal, as, even in a relatively simple infrastructure like the one which will be built under this project, there always is something to improve in terms of security architecture.
- **Recover seamlessly**: **All the infrastructure must be deployed automatically**. The end goal is if it is ever wanted to deploy the infrastructure on a new server, the deployment will be done with the minimum number of manual work. This is where **the project leans heavily into IaC**, as it will be explained more in the next sections.

This project covers a lot of concepts about IaC, security hardening, security standards, blue teaming, and red teaming. Hopefully the end result will allow other security professionals deploy this infrastructure and easily get to security testing.

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
  PaC will something that will be implemented at a later phase of the project, after the IaC phase is at a mature state.
- **Detection as Code (DaC)**: [DaC](https://www.legitsecurity.com/aspm-knowledge-base/detection-as-code) enables the writing, maintenance, and automation of the threat detection logic as if it were software code, making security a built-in part of the development pipeline. Similar to PaC, DaC will start being implemented at a later phase of the project.

## Tech stack

For this project, a wide variety of systems and technologies will be used, as the purpose is, among others, to simulate a real-world IT infrastructure.

The following technologies, for now, are strong contenders to be deployed. Since the project is just starting now, some technologies are only thought of as ideas, and may or may not actually be deployed when the time comes. I am looking for recommendations, though. Send me a message if you have something to recommend:

- [**Proxmox**](https://www.proxmox.com/en/): The whole infrastructure will be deployed on a physical server running Proxmox Virtual Environment (PVE) as a Type-1 Hypervisor. The infrastructure will be built with VMs and Containers.
- For the **IaC** aspect, the following 3 tools will be used:
  - [**Packer**](https://developer.hashicorp.com/packer): Packer is a community tool for **creating identical machine images** for multiple platforms from a single source configuration. What Packer essentially does is
  1. it takes an ISO file of an OS
  2. installs it on a temporary VM
  3. applies any defined action during the installation (language, locale, hard disk to install the OS, etc.)
  4. applies any configuration defined (e.g., IP address)
  5. installs any package defined and
  6. converts that VM to a template.
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
    - Host visibiltiy with [Elastic agent](https://www.elastic.co/elastic-agent)
    - Centralized management with [Elastic Fleet](https://www.elastic.co/docs/reference/fleet)
- [**ModSecurity**](https://modsecurity.org/) + [**OWASP Core Rule Set (CRS)**](https://coreruleset.org/): ModSecurity is an open source cross-platform **Web Application Firewall (WAF)**. Since its version 3 release, it now works as a standalone module which provies the capability to load/interpret rules written in the ModSecurity SecRules format and apply them to HTTP content provided by the web application via ModSecurity Connectors. This will prove useful, since in its previous module it worked as an Apache module only, while now it is more independent of the web server solution that it protects. ModSecurity on its own does not provide detection/protection rules. The OWASP CRS is a set of generic attack detection rules to be used with ModSecurity.
- The Actual Infrastructure the will be secured and attacked:
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

![NetworkDesign-FW Rules drawio](https://github.com/user-attachments/assets/f6528984-e477-4a61-9b26-40c389cea53c)

explain the graph.
it is not final, we add more as we go.

## PoC

install proxmox + config
install opnsense + config

IaC PoC
packer/terraform/ansible
with pitfalls.

## Next Steps

Make proxmox and opnsense into IaC.

## Conclusion
