# Troubleshooting MDE - Event Viewer

## Introduction

TOC

## Event Viewer general info and logs to look for and what they include

event tables to look for and what they include

A great thing about the event viewer is that custom views can be created with XML files, where only the needed events from the needed logs can be filtered out. This allows for creating filtered views where only the detections and blocks done by Defender are shown, which results in similar reports to the ones created using the KQL queries shown in the previous post [Using KQL to identify detections from MDE - Getting to know MDE Part 2](https://kostaskoutrou.github.io/2026/01/06/using-kql-for-mde.html). More details on the specific XML in the sections below.

## ASR rules

Attack Surface Reduction (ASR) rules in Microsoft Defender Antivirus (MDAV) protect against risky software behavior commonly exploited, such as:

- Running unsigned drivers or signed by an untrusted CA.
- Injecting code into other processes.
- Running Office Macros.
- Running software that has low prevalence across organizations.

For more info about ASR rules, visit [Microsoft's relevant documentation](https://learn.microsoft.com/en-us/defender-endpoint/attack-surface-reduction-rules-overview).

When it comes to Windows Event logs, the ASR rule events are located in the Windows Event log under **Applications and Services Logs > Microsoft > Windows > Windows Defender > Operational**. The following event IDs are related to ASR rules:

|Event ID|Description|
|-|-|
|1121|Event when rule fires in block mode|
|1122|Event when rule fires in audit mode|
|1129|Event when user overrides block in warn mode|
|5007|Event when settings are changed|

Note that a lot of events with Event ID 5007 occur

## CFA

## Device Control

## Exploit Protection

## Network Protection, Web Protection, and SmartScreen

## Tamper Protection

## MDAV Detections

https://learn.microsoft.com/en-us/defender-endpoint/troubleshoot-microsoft-defender-antivirus

## PUA

## WDAC and AppLocker

## General XML for all detections

## MDE

https://learn.microsoft.com/en-us/defender-endpoint/event-error-codes

Applications and Services Logs > Microsoft > Windows > SENSE and select Operational.

SENSE is the internal name used to refer to the behavioral sensor that powers Microsoft Defender for Endpoint.
