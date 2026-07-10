# Troubleshooting MDE - Event Viewer

## Introduction

Inspired by https://learn.microsoft.com/en-us/defender-endpoint/attack-surface-reduction-windows-events#custom-xml-templates-for-attack-surface-reduction-events

TOC

## Event Viewer general info and logs to look for and what they include

event tables to look for and what they include

how an event XML is formatted (System and EventData).

A great thing about the Windows Event Viewer is that all of its logs are stored as XML data, so custom views can be created using XPath queries, where only the needed events can be filtered out. XPath (XML Path Language) is a query language used to find and extract specific information from an XML document.

Using XPath allows for creating shareable filtered views where only the detections and blocks done by Defender are shown, which results in similar reports to the ones created using the KQL queries shown in the previous post [Using KQL to identify detections from MDE - Getting to know MDE Part 2](https://kostaskoutrou.github.io/2026/01/06/using-kql-for-mde.html).

In the next sections, XPath queries are shown for each MDAV feature. These can be used to facilitate troubleshooting towards identifying false positive detections and blocks. More details on the specific XPath queries is found in the sections below.

How to import custom XPath queries:

## XPath query explanation

All the XPath queries used here have the following format:

```xml
<QueryList>
  <Query Id="0" Path="Microsoft-Windows-Windows Defender/Operational">
   <Select Path="Microsoft-Windows-Windows Defender/Operational">*[System[(EventID=... or EventID=... or ...)]]</Select>
   <Select Path="Microsoft-Windows-Windows Defender/WHC">*[System[(EventID=... or EventID=... or ...)]]</Select>
  </Query>
</QueryList>
```

The only difference between the queries are **Windows Event channel paths** and the **Event IDs** searched for. To not repeat the explanation of the query under each section, a general explanation is provided here:

1. Default path to look for is `Microsoft-Windows-Windows Defender/Operational`. This is needed to me stated even though other paths are specified below because when Microsoft build the XML structure for Event Viewer, they designed the `<Query>` node to act as the primary "container". The schema rules dictate that this container must declare a starting point.
2. `Select Path="Microsoft-Windows-Windows Defender/Operational">`: Search in the path `Microsoft-Windows-Windows Defender/Operational`
3. `*\[System\[\(EventID=... or EventID=... or EventID=...\)\]\]`:
  1. `*`: Search at any root event, i.e., grab all events in the log as a starting point.
  2. The brackets contents in XPath are used to apply a filter to the item just before it, i.e., all the events.
  3. `System`: Every Windows event is split into two main XML sections: `<System>` (metadata like time, provider and Event ID) and `<EventData>` (the details of what happened). Here we are filtering the `<System>` section.
  4. The brackets again filter to the item just before, i.e., the `<System>` section.
  5. `EventID=...`: With this condition the specified Event IDs are filtered.

The queries searches in the following paths, depending on the MDAV feature:
1. `Microsoft-Windows-Windows Defender/Operational`: This channel logs all the actual events related to MDAV. The full list of all Event IDs can be found [here](https://learn.microsoft.com/en-us/defender-endpoint/troubleshoot-microsoft-defender-antivirus).
2. `Microsoft-Windows-Windows Defender/WHC`: WHC stands for Windows Health Center, and is a channel which provides mostly informational events about the state of the Windows Defender process. I have not found any official documentation regarding what exactly is reported in this channel, but if you have more information, please let me know.

The reason to search for both paths is for redundancy and if for any reason an event is logged in the WHC channel instead of the Operational channel.

## Demo Files

If there are no Events for a specific feature and there is a need to produce some for confirmation of the query working as expected, there are [demo files provided by Microsoft](https://learn.microsoft.com/en-us/defender-endpoint/defender-endpoint-demonstrations), which can be used to force MDAV detections and produce Windows Events.

## MDAV Changes logging

An interesting and useful event related to all Microsoft Defender Antivirus (MDAV) features is Event ID 5007 with description "Event when settings are changed". A lot of events with Event ID 5007 are logged because it includes any change happening to the MDAV configuration, such as:

- Signature/Engine updates
- Policy syncs from Group Policy, Intune, SCCM
- Cloud-Delivered Protection Adjustments
- Any MDAV feature changing states (e.g., ASR rule switching from Audit mode to Block mode)

Investigating these logs is useful to identify suspicious tampering of the MDAV configuration, like disabling ASR rules or Real-Time protection, as well as to troubleshoot MDAV behavior that started unexpectedly, such as to identify when an ASR rule switched from Audit mode to Block mode.

Event ID 5000
Symbolic name: MALWAREPROTECTION_RTP_ENABLED

Message: Real-time protection is enabled.
Event ID 5001
Symbolic name: MALWAREPROTECTION_RTP_DISABLED

Message: Real-time protection is disabled.
Event ID 5004
Symbolic name: MALWAREPROTECTION_RTP_FEATURE_CONFIGURED

Message: The real-time protection configuration changed.
Event ID 5009
Symbolic name: MALWAREPROTECTION_ANTISPYWARE_ENABLED

Message: Scanning for malware and other potentially unwanted software is enabled.
Event ID 5010
Symbolic name: MALWAREPROTECTION_ANTISPYWARE_DISABLED

Message: Scanning for malware and other potentially unwanted software is disabled.
Event ID 5011
Symbolic name: MALWAREPROTECTION_ANTIVIRUS_ENABLED

Message: Scanning for viruses is enabled.

Event ID 5012
Symbolic name: MALWAREPROTECTION_ANTIVIRUS_DISABLED

Message: Scanning for viruses is disabled.

To search for Event ID 5007, which can be found under **Applications and Services Logs > Microsoft > Windows > Windows Defender > Operational**, the following XPath query can be used:

```xml
<QueryList>
  <Query Id="0" Path="Microsoft-Windows-Windows Defender/Operational">
   <Select Path="Microsoft-Windows-Windows Defender/Operational">*[System[(EventID=5007)]]</Select>
   <Select Path="Microsoft-Windows-Windows Defender/WHC">*[System[(EventID=5007)]]</Select>
  </Query>
</QueryList>
```

## Attack Surface Reduction (ASR) rules

Attack Surface Reduction (ASR) rules in MDAV protect against risky software behavior commonly exploited, such as:

- Running unsigned drivers or signed by an untrusted CA.
- Injecting code into other processes.
- Running Office Macros.
- Running software that has low prevalence across organizations.

For more info about ASR rules, visit [Microsoft's relevant documentation](https://learn.microsoft.com/en-us/defender-endpoint/attack-surface-reduction-rules-overview).

When it comes to Windows Event logs, the ASR rule events are located in the Windows Event log under **Applications and Services Logs > Microsoft > Windows > Windows Defender > Operational**. The following event IDs are related to ASR rules:

|Event ID|Description|
|:-:|-|
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

If you just want to **find blocks only**, just keep Event ID 1121:

```xml
<QueryList>
  <Query Id="0" Path="Microsoft-Windows-Windows Defender/Operational">
   <Select Path="Microsoft-Windows-Windows Defender/Operational">*[System[(EventID=1121)]</Select>
   <Select Path="Microsoft-Windows-Windows Defender/WHC">*[System[(EventID=1121)]]</Select>
  </Query>
</QueryList>
```

## Controlled Folder Access (CFA)

The basic idea with CFA is that you define a set of directories (folders) and a set of applications. Only that set of apps is allowed to process in any way the set of directories. For more info, click [here](https://learn.microsoft.com/en-us/defender-endpoint/controlled-folder-access-overview)

When it comes to Windows Event logs, the CFA events are located in the Windows Event log under **Applications and Services Logs > Microsoft > Windows > Windows Defender > Operational**. The following event IDs are related to CFA:

|Event ID|Description|
|:-:|-|
|1124|Audited controlled folder access event|
|1123|Blocked controlled folder access event|
|1127|Controlled Folder Access blocked an untrusted process from potentially modifying disk sectors.|
|1128|Audited controlled folder access sector write block event|

The following XPath query will filter for all detection and block events:

```xml
<QueryList>
  <Query Id="0" Path="Microsoft-Windows-Windows Defender/Operational">
   <Select Path="Microsoft-Windows-Windows Defender/Operational">*[System[(EventID=1123 or EventID=1124 or EventID=1127 or EventID=1128)]]</Select>
   <Select Path="Microsoft-Windows-Windows Defender/WHC">*[System[(EventID=1123 or EventID=1124 or EventID=1127 or EventID=1128)]]</Select>
  </Query>
</QueryList>
```

For blocks only, use the following:

```xml
<QueryList>
  <Query Id="0" Path="Microsoft-Windows-Windows Defender/Operational">
   <Select Path="Microsoft-Windows-Windows Defender/Operational">*[System[(EventID=1123 or EventID=1127)]]</Select>
   <Select Path="Microsoft-Windows-Windows Defender/WHC">*[System[(EventID=1123 or EventID=1127)]]</Select>
  </Query>
</QueryList>
```

## Device Control

basic info

When it comes to Windows Event logs, the CFA events are located in the Windows Event log under **Applications and Services Logs > Microsoft > Windows > Windows Defender > Operational**. The following event IDs are related to CFA:

## Exploit Protection

Exploit Protection helps protect against malware that uses exploits to infect devices and spread. It consists of many mitigations that can be applied to either the whole OS or individual apps. Exploit Protection helps protect against malware that uses exploits to infect devices and spread. It consists of many mitigations that can be applied to either the whole OS or individual apps. For more info, click [here](https://learn.microsoft.com/en-us/defender-endpoint/exploit-protection).

When it comes to Windows Event logs, most Exploit Protection events are located in the Windows Event log under **Security-Mitigations > Kernel Mode and Security-Mitigations > User Mode**, while some are located in **WER-Diagnostics > Operational** and **Win32k > Operational**. The following event IDs are related to CFA:

**Security-Mitigations > Kernel Mode and Security-Mitigations > User Mode**
|Event ID|Description|
|:-:|-|
|1|ACG audit|
|2|ACG enforce|
|3|Don't allow child processes audit|
|4|Don't allow child processes block|
|5|Block low integrity images audit|
|6|Block low integrity images block|
|7|Block remote images audit|
|8|Block remote images block|
|9|Disable win32k system calls audit|
|10|Disable win32k system calls block|
|11|Code integrity guard audit|
|12|Code integrity guard block|
|13|EAF audit|
|14|EAF enforce|
|15|EAF+ audit|
|16|EAF+ enforce|
|17|IAF audit|
|18|IAF enforce|
|19|ROP StackPivot audit|
|20|ROP StackPivot enforce|
|21|ROP CallerCheck audit|
|22|ROP CallerCheck enforce|
|23|ROP SimExec audit|
|24|ROP SimExec enforce|

**WER-Diagnostics > Operational**
|Event ID|Description|
|:-:|-|
|5|CFG Block|

**Win32k > Operational**
|Event ID|Description|
|:-:|-|
|260|Untrusted Font|

The following XPath query will filter for all detection and block events:

```xml
<QueryList>
  <Query Id="0" Path="Microsoft-Windows-Security-Mitigations/KernelMode">
   <Select Path="Microsoft-Windows-Security-Mitigations/KernelMode">*[System[Provider[@Name='Microsoft-Windows-Security-Mitigations' or @Name='Microsoft-Windows-WER-Diag' or @Name='Microsoft-Windows-Win32k' or @Name='Win32k'] and ( (EventID &gt;= 1 and EventID &lt;= 24)  or EventID=5 or EventID=260)]]</Select>
   <Select Path="Microsoft-Windows-Win32k/Concurrency">*[System[Provider[@Name='Microsoft-Windows-Security-Mitigations' or @Name='Microsoft-Windows-WER-Diag' or @Name='Microsoft-Windows-Win32k' or @Name='Win32k'] and ( (EventID &gt;= 1 and EventID &lt;= 24)  or EventID=5 or EventID=260)]]</Select>
   <Select Path="Microsoft-Windows-Win32k/Contention">*[System[Provider[@Name='Microsoft-Windows-Security-Mitigations' or @Name='Microsoft-Windows-WER-Diag' or @Name='Microsoft-Windows-Win32k' or @Name='Win32k'] and ( (EventID &gt;= 1 and EventID &lt;= 24)  or EventID=5 or EventID=260)]]</Select>
   <Select Path="Microsoft-Windows-Win32k/Messages">*[System[Provider[@Name='Microsoft-Windows-Security-Mitigations' or @Name='Microsoft-Windows-WER-Diag' or @Name='Microsoft-Windows-Win32k' or @Name='Win32k'] and ( (EventID &gt;= 1 and EventID &lt;= 24)  or EventID=5 or EventID=260)]]</Select>
   <Select Path="Microsoft-Windows-Win32k/Operational">*[System[Provider[@Name='Microsoft-Windows-Security-Mitigations' or @Name='Microsoft-Windows-WER-Diag' or @Name='Microsoft-Windows-Win32k' or @Name='Win32k'] and ( (EventID &gt;= 1 and EventID &lt;= 24)  or EventID=5 or EventID=260)]]</Select>
   <Select Path="Microsoft-Windows-Win32k/Power">*[System[Provider[@Name='Microsoft-Windows-Security-Mitigations' or @Name='Microsoft-Windows-WER-Diag' or @Name='Microsoft-Windows-Win32k' or @Name='Win32k'] and ( (EventID &gt;= 1 and EventID &lt;= 24)  or EventID=5 or EventID=260)]]</Select>
   <Select Path="Microsoft-Windows-Win32k/Render">*[System[Provider[@Name='Microsoft-Windows-Security-Mitigations' or @Name='Microsoft-Windows-WER-Diag' or @Name='Microsoft-Windows-Win32k' or @Name='Win32k'] and ( (EventID &gt;= 1 and EventID &lt;= 24)  or EventID=5 or EventID=260)]]</Select>
   <Select Path="Microsoft-Windows-Win32k/Tracing">*[System[Provider[@Name='Microsoft-Windows-Security-Mitigations' or @Name='Microsoft-Windows-WER-Diag' or @Name='Microsoft-Windows-Win32k' or @Name='Win32k'] and ( (EventID &gt;= 1 and EventID &lt;= 24)  or EventID=5 or EventID=260)]]</Select>
   <Select Path="Microsoft-Windows-Win32k/UIPI">*[System[Provider[@Name='Microsoft-Windows-Security-Mitigations' or @Name='Microsoft-Windows-WER-Diag' or @Name='Microsoft-Windows-Win32k' or @Name='Win32k'] and ( (EventID &gt;= 1 and EventID &lt;= 24)  or EventID=5 or EventID=260)]]</Select>
   <Select Path="System">*[System[Provider[@Name='Microsoft-Windows-Security-Mitigations' or @Name='Microsoft-Windows-WER-Diag' or @Name='Microsoft-Windows-Win32k' or @Name='Win32k'] and ( (EventID &gt;= 1 and EventID &lt;= 24)  or EventID=5 or EventID=260)]]</Select>
   <Select Path="Microsoft-Windows-Security-Mitigations/UserMode">*[System[Provider[@Name='Microsoft-Windows-Security-Mitigations' or @Name='Microsoft-Windows-WER-Diag' or @Name='Microsoft-Windows-Win32k' or @Name='Win32k'] and ( (EventID &gt;= 1 and EventID &lt;= 24)  or EventID=5 or EventID=260)]]</Select>
  </Query>
</QueryList>
```

## Network Protection, Web Protection, and SmartScreen

The combination of Network Protection, Web Protection, and SmartScreen provide the ability to block access to malicious websites, custom indicators, and web content filtering. For info, click [here](https://learn.microsoft.com/en-us/defender-endpoint/network-protection).

When it comes to Windows Event logs, the Network Protection events are located in the Windows Event log under **Applications and Services Logs > Microsoft > Windows > Windows Defender > Operational**. The following event IDs are related to Network Protection:

|Event ID|Description|
|:-:|-|
|1125|Event when network protection fires in audit mode|
|1126|Event when network protection fires in block mode|

```xml
<QueryList>
 <Query Id="0" Path="Microsoft-Windows-Windows Defender/Operational">
  <Select Path="Microsoft-Windows-Windows Defender/Operational">*[System[(EventID=1125 or EventID=1126)]]</Select>
  <Select Path="Microsoft-Windows-Windows Defender/WHC">*[System[(EventID=1125 or EventID=1126)]]</Select>
 </Query>
</QueryList>
```

If only the block events are needed, the audit events can be emitted:

```xml
<QueryList>
 <Query Id="0" Path="Microsoft-Windows-Windows Defender/Operational">
  <Select Path="Microsoft-Windows-Windows Defender/Operational">*[System[(EventID=1126)]]</Select>
  <Select Path="Microsoft-Windows-Windows Defender/WHC">*[System[(EventID=1126)]]</Select>
 </Query>
</QueryList>
```

## Tamper Protection

Event ID 5013
Symbolic name: MALWAREPROTECTION_SCAN_CANCELLED

Message: Tamper protection blocked a change to Microsoft Defender Antivirus.

Description: If Tamper protection is enabled then any attempt to change any of Defender's settings is blocked. Event ID 5013 is generated and states which setting change was blocked.

## Microsoft Defender Antivirus (MDAV) Detections

MDAV is the core component of Defender. With it, process creation events and file download events from the internet are monitored. It not only uses its signature-based engine, but also predictive technologies such as machine learning and cloud-delivered protection to find attacks. For more info, click [here](https://learn.microsoft.com/en-us/defender-endpoint/microsoft-defender-antivirus-windows)

Regarding Event IDs, there is a thorough documentation [here](https://learn.microsoft.com/en-us/defender-endpoint/troubleshoot-microsoft-defender-antivirus). The Event IDs which mostly relate to detections and blocks are the following:

|Event ID|Description|
|:-:|-|
|1006/1116|The antimalware platform found malware or other potentially unwanted software.|
|1007/1117|The antimalware platform performed an action to protect your system from malware or other potentially unwanted software.|
|1008/1118|The antimalware platform attempted to perform an action to protect your system from malware or other potentially unwanted software, but the action failed.|
|1011|The antimalware platform deleted an item from quarantine.|
|1012|The antimalware platform couldn't delete an item from quarantine.|
|1015|The antimalware platform detected suspicious behavior.|
|1119|The antimalware platform encountered a critical error when trying to take action on malware or other potentially unwanted software.|

These Event IDs are logged under **Applications and Services Logs > Microsoft > Windows > Windows Defender > Operational.**

The XPath query to filter for those events is the following:

```xml
<QueryList>
  <Query Id="0" Path="Microsoft-Windows-Windows Defender/Operational">
   <Select Path="Microsoft-Windows-Windows Defender/Operational">*[
    System[(EventID=1006 or EventID=1007 or EventID=1008 or
    EventID=1011 or EventID=1012 or EventID=1015 or EventID=1116 or
    EventID=1117 or EventID=1118 or EventID=1119)]
    ]
    </Select>
  </Query>
</QueryList>
```

As described in the Event IDs table, these events log also events of Potentially Unwanted Software/Applications (PUA). If only events **not** related to PUA are needed, the following XPath query can be used:

```xml
<QueryList>
  <Query Id="0" Path="Microsoft-Windows-Windows Defender/Operational">
   <Select Path="Microsoft-Windows-Windows Defender/Operational">*[
    EventData[Data[@Name='Category Name']!='Potentially Unwanted Software'] and 
    System[(EventID=1006 or EventID=1007 or EventID=1008 or EventID=1011 or 
    EventID=1012 or EventID=1015 or EventID=1116 or EventID=1117 or 
    EventID=1118 or EventID=1119)]
    ]
    </Select>
  </Query>
</QueryList>
```

## Potentially Unwanted Apps (PUA)

PUA is a category of software that can:

- Cause the machine to run slowly
- Display unexpected apps
- Install other software that might be unwanted

For more info, click [here](https://learn.microsoft.com/en-us/defender-endpoint/detect-block-potentially-unwanted-apps-microsoft-defender-antivirus)

As mentioned in the previous section, PUA is logged under the same Event IDs as MDAV. Therefore, if events related only PUA are needed, an additional filter is needed to be applied in the "Category Name" field of the EventData 

```xml
<QueryList>
  <Query Id="0" Path="Microsoft-Windows-Windows Defender/Operational">
   <Select Path="Microsoft-Windows-Windows Defender/Operational">*[
    EventData[Data[@Name='Category Name']='Potentially Unwanted Software'] and 
    System[(EventID=1006 or EventID=1007 or EventID=1008 or EventID=1011 or 
    EventID=1012 or EventID=1015 or EventID=1116 or EventID=1117 or 
    EventID=1118 or EventID=1119)]
    ]
    </Select>
  </Query>
</QueryList>
```

## App Control for Business and AppLocker

[AppLocker](https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/applocker/applocker-overview) helps control which apps and files users can run. [App Control for Business](https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/appcontrol-and-applocker-overview) also known as **Windows Defender Application Control** (WDAC), formerly known as **Configurable Code Integrity ** (CCI), is a newer solution for the same purpose, and provides more granular controls.

When it comes to Windows Events, both features log their Events under **Application and Services Logs > Microsoft > Windows > AppLocker**. The events are quite granular, where there are categories based on

- the the file type of the event
- whether the file that is about to be ran is there because of a Managed Installer (Intune, SCCM, etc.)
- the feature that is controlling the file execution (AppLocker or WDAC)

More info regarding [AppLocker events](https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/applocker/using-event-viewer-with-applocker) and [App Control for business events](https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/operations/event-id-explanations), follow the corresponding links.

For AppLocker only, the following Event IDs are of interest:

|Event ID|Description|
|:-:|-|
|8000|AppID policy conversion failed. This indicates that the policy wasn’t applied correctly to the computer.|
|8001|The AppLocker policy was applied successfully to this computer.|
|8002|File was allowed to run. (.exe and .dll files.)|
|8003|File was allowed to run but would have been prevented from running if the AppLocker policy were enforced. (.exe and .dll files.)|
|8004|File was prevented from running. (.exe and .dll files.)|
|8005|File was allowed to run. (script or .msi files)|
|8006|File was allowed to run but would have been prevented from running if the AppLocker policy were enforced. (script or .msi files)|
|8007|File was prevented from running. (script or .msi files)|
|8008|File: AppLocker component not available on this SKU. Indicates that the Windows edition does not support AppLocker.|
|8020|File was allowed to run. (packaged apps .appx, .msix)|
|8021|File was allowed to run but would have been prevented from running if the AppLocker policy were enforced. (packaged apps .appx, .msix)|
|8022|File was prevented from running.  (packaged apps .appx, .msix)|
|8023|File was allowed to be installed. (packaged apps .appx, .msix)|
|8024|File was allowed to install but would have been prevented from running if the AppLocker policy were enforced. (packaged apps .appx, .msix)|
|8025|File was prevented from installing. (packaged apps .appx, .msix)|
|8027|No packaged apps can be executed while Exe rules are being enforced and no Packaged app rules have been configured.|

For WDAC only, the following Event IDs are of interest:

|Event ID|Description|
|:-:|-|
|8028|File was allowed to run but would have been prevented if the Config CI policy were enforced (script or .msi).|
|8029|File was prevented from running due to Config CI policy (script or .msi).|
|8036|File was prevented from running due to Config CI policy (COM object).|
|8037|* passed Config CI policy and was allowed to run.|
|8038|Publisher info: Subject: * Issuer: * Signature index * (* total)|
|8039|Package family name * version * was allowed to install or update but would have been prevented if the Config CI policy|
|8040|Package family name * version * was prevented from installing or updating due to Config CI policy|

For running files installed via a Managed Installer (Intune, SCCM, etc.), the following events are of interest:



## General XML for all detections

add all block event IDs in one query, and try to put an easy filter for date and time range and maybe see if you can put a name filter to search for specific applications.

## MDE

https://learn.microsoft.com/en-us/defender-endpoint/event-error-codes

Applications and Services Logs > Microsoft > Windows > SENSE and select Operational.

SENSE is the internal name used to refer to the behavioral sensor that powers Microsoft Defender for Endpoint.
