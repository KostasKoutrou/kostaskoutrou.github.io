# Cyber Range as Code: Automating Security Lab with IaC - Part 1

## Introduction

What is a Cyber Range?

> A [Cyber Range](https://www.ibm.com/think/topics/cyber-range) is a virtual environment for cybersecurity training, testing, and research that simulates real-world networks and cyberattacks.

The end goal of this project is to **create** and **maintain** a real-world infrastructure to:

- Test attack scenarios and
  - Monitor the result of the attack first-hand
  - Review the generated logs to understand what happened on the backend, what could be detected and created as a detection rule
- Implement security standards like ISO27001, NIS2, NIST, and CIS
- Architecting a secure infrastructure and security hardening
- Recover seamlessly: All the infrastructure must be deployed automatically. The end goal is if it is ever wanted to deploy the infrastructure on a new server, the deployment will be done with the minimum number of manual work. This is where the project leans heavily into IaC, as it will be explained more in the next sections.

This project covers a lot of concepts about IaC, security hardening, security standards, blue teaming, and red teaming. Hopefully the end result will allow other security professionals deploy this infrastructure and easily get to security testing.

In this post, a high level design of the end result is described, of the tech stack that is planned to be used, as well as an early PoC regarding the infrastructure and its deployment using IaC.

So let's get started!

---

## Contents

* TOC
{:toc}

---

## Automated Infrastructure

As mentioned above, one core aspect of this project is to be able to have the whole infrastructure deployed automatically. This means that the following will be utilized.
- Infrastructure as Code (IaC): All the infrastructure will be managed and provisioned using code and configuration files, instead of manual processes. This enables automation, consistency, version control, and swift, repeatable deployments. In the case specifically of this Cyber Range project, this will also allow quick recovery in case of an attack resulting in part of the infrastructure or the whole infrastructure being compromised and damaged unrecoverably.
- For security hardening and implementing security standards, in addition to IaC, the following practices will also be utilized:
  - Policy as Code (PaC): PaC ([ref1](https://www.paloaltonetworks.com/cyberpedia/what-is-policy-as-code), [ref2](https://developer.hashicorp.com/sentinel/docs/concepts/policy-as-code) , [ref3](https://www.hashicorp.com/en/blog/policy-as-code-explained)) is the practice of defining, updating, sharing and enforcing policies using code. With this approach, instead of relying on manual processes to manage policies, the benefits are:
    - Sandboxing: Policies provide guardrails for other automated systems. By defining Policies as Code, the verification by the policies is automated, reducing the time needed of manual work.
    - Codification: Because the policy is defined as code, it is possible to describe the logic about the policy directly on the code with comments, which results in better understand and knowledge sharing of these policies.
    - Version Control: The benefits of version control are well known (history, diffs, pull requests, etc.).
    - Testing: Because the policies are defined as code, they can be tested by utilizing automated testing such as through a CI/CD pipeline. This allows for testing if a policy will result in the expected outcome before deploying to production.
    - Automation: Similarly to IaC, with PaC, tools can be used to automatically deploy the policies to specified systems.
    - Compliance as Code (CaC) / Security as Code (SaC): This is where the concepts of [CaC](https://www.puppet.com/blog/compliance-as-code) and [SaC](https://www.isaca.org/resources/news-and-trends/isaca-now-blog/2024/security-as-code-a-key-building-block-for-devsecops) are also related.
- Detection as Code (DaC): [DaC](https://www.legitsecurity.com/aspm-knowledge-base/detection-as-code) enables the writing, maintenance, and automation of the threat detection logic as if it were software code, making security a built-in part of the development pipeline.

## Tech stack

For this project, a wide variety of systems and technologies will be used, as the purpose is, among others, to simulate a real-world IT infrastructure.


The initial plan is to deploy the following:

- [Proxmox](https://www.proxmox.com/en/): The whole infrastructure will be deployed on a physical server running Proxmox Virtual Environment (PVE) as a Type-1 Hypervisor. The infrastructure will be built with VMs and Containers.
- For the IaC aspect, the following 3 tools will be used:
  - [Packer](https://developer.hashicorp.com/packer): Packer is a community tool for creating identical machine images for multiple platforms from a single source configuration. What Packer essentially does is it takes an ISO file of an OS, installs it on a temporary VM, applies any defined action during the installation, applies any configuration defined, installs any package defined and converts that VM to a template. That template is made for the platform on which the temporary VM was created on. In the case of this project, since the VM is created in Proxmox, the template will be a Proxmox VM template. This template is then used by Terraform to provision VMs which will have that configuration ready immediately. Packer provides the freedom of utilizing any ISO and converting it to a template exactly for the needs of the task at hand.
  - [Terraform](https://developer.hashicorp.com/terraform): HashiCorp Terraform is an infrastructure as code tool that lets you define both cloud and on-prem resources in human-readable configuration files that you can version, reuse, and share. You can then use a consistent workflow to provision and manage all of your infrastructure throughout its lifecycle. Terraform can manage low-level components like compute, storage, and networking resources, as well as high-level components like DNS entries and SaaS features. In the context of the project, the templates created by Packer will used by Terraform to provision all the infrastructure defined. Note here that many of the configurations of the template defined by Packer can be changed during the provisioning by Terraform (e.g., RAM, CPU cores, installed packages, etc.), which provides more freedom during provisioning. But with Packer the benefit with the resulting template is that there is no need to install the packages already installed and execute any time consuming action every time a new VM is provisioned, because it was already done during the Packer VM template creation.
  - [Ansible](https://docs.ansible.com/): 
- opnsense + suricata for ips
- wazuh / MDXDR
- linux servers
  - DVWA / juice shop / owasp webgoat
  - mysql / postgresql
  - nginx
- AD and windows workstations
- atomic red / kali linux
- openpolicy agent

## Architecture

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
