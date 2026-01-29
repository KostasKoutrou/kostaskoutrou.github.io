# Cyber Range as Code: Automating Security Lab with IaC - Part 1

## Introduction

What is a Cyber Range?

> A [Cyber Range](https://www.ibm.com/think/topics/cyber-range) is a virtual environment for cybersecurity training, testing, and research that simulates real-world networks and cyberattacks.

Purpose is to create a lab to:

- test attacks
- implementing standards like iso, nist, cis
- hardening infrastructure
- Recover seamlessly. All Infra is automated: if i get a new server, i can be up and running with the minimum number of manual work

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

## All Infra is automated

- IaC
- For hardening and implementing standards:
  - compliance as code
  - policy as code
- detection as code

## Tech stack

- physical server
- proxmox
- packer
- terraform
- ansible
- openpolicy agent
- opnsense + suricata for ips
- wazuh / MDXDR
- linux servers
  - DVWA / juice shop / owasp webgoat
  - mysql / postgresql
  - nginx
- AD and windows workstations
- atomic red / kali linux

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
