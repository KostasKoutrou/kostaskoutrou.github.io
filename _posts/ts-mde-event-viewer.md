# Troubleshooting MDE - Event Viewer

## Introduction

Inspired by https://learn.microsoft.com/en-us/defender-endpoint/attack-surface-reduction-windows-events#custom-xml-templates-for-attack-surface-reduction-events

TOC

## Event Viewer general info and logs to look for and what they include

event tables to look for and what they include

A great thing about the Windows Event Viewer is that all of its logs are stored as XML data, so custom views can be created using XPath queries, where only the needed events can be filtered out. XPath (XML Path Language) is a query language used to find and extract specific information from an XML document.

Using XPath allows for creating shareable filtered views where only the detections and blocks done by Defender are shown, which results in similar reports to the ones created using the KQL queries shown in the previous post [Using KQL to identify detections from MDE - Getting to know MDE Part 2](https://kostaskoutrou.github.io/2026/01/06/using-kql-for-mde.html).

In the next sections, XPath queries are shown for each MDAV feature. These can be used to facilitate troubleshooting towards identifying false positive detections and blocks. More details on the specific XPath queries is found in the sections below.


## MDAV Changes logging

An interesting and useful event related to all Microsoft Defender Antivirus (MDAV) features is Event ID 5007 with description "Event when settings are changed". A lot of events with Event ID 5007 are logged because it includes any change happening to the MDAV configuration, such as:

- Signature/Engine updates
- Policy syncs from Group Policy, Intune, SCCM
- Cloud-Delivered Protection Adjustments
- Any MDAV feature changing states (e.g., ASR rule switching from Audit mode to Block mode)

Investigating these logs is useful to identify suspicious tampering of the MDAV configuration, like disabling ASR rules or Real-Time protection, as well as to troubleshoot MDAV behavior that started unexpectedly, such as to identify when an ASR rule switched from Audit mode to Block mode.

To search for Event ID 5007, which can be found under 

## ASR rules

Attack Surface Reduction (ASR) rules in MDAV protect against risky software behavior commonly exploited, such as:

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

The following XPath query will filter our ASR rule detections, blocks, and user overrides:

```xml
<QueryList>
  <Query Id="0" Path="Microsoft-Windows-Windows Defender/Operational">
   <Select Path="Microsoft-Windows-Windows Defender/Operational">*[System[(EventID=1121 or EventID=1122 or EventID=1129)]]</Select>
   <Select Path="Microsoft-Windows-Windows Defender/WHC">*[System[(EventID=1121 or EventID=1122 or EventID=1129)]]</Select>
  </Query>
</QueryList>
```

If you just want to find blocks only, just keep Event ID 1121:

```xml
<QueryList>
  <Query Id="0" Path="Microsoft-Windows-Windows Defender/Operational">
   <Select Path="Microsoft-Windows-Windows Defender/Operational">*[System[(EventID=1121)]</Select>
   <Select Path="Microsoft-Windows-Windows Defender/WHC">*[System[(EventID=1121)]]</Select>
  </Query>
</QueryList>
```

What these XPath queries do is the following:

1. Default path to look for is `Microsoft-Windows-Windows Defender/Operational`. This is needed to me stated even though other paths are specified below because when Microsoft build the XML structure for Event Viewer, they designed the `<Query>` node to act as the primary "container". The schema rules dictate that this container must declare a starting point.
2. `Select Path="Microsoft-Windows-Windows Defender/Operational">`: Search in the path `Microsoft-Windows-Windows Defender/Operational`
3. `*\[System\[\(EventID=1121 or EventID=1122 or EventID=1129\)\]\]`:
  1. *: Search at any root event, i.e., grab all events in the log as a starting point.
  2. The brackets contents in XPath are used to apply a filter to the item just before it, i.e., all the events.
  3. System 

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
