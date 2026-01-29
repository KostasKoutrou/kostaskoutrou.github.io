# Cyber Range as Code: Automating Security Lab with IaC - Part 1

## Introduction

what is a cyber range?

Purpose is to create a lab to:

- test attacks
- implementing standards like iso, nist, cis
- hardening infrastructure
- All Infra is automated: if i get a new server, i can be up and running with the minimum number of manual work

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
