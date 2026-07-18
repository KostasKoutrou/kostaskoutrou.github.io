# Using Windows Events to identify detections from MDE - Getting to know MDE Part 4

## Introduction

Welcome to the 4th part of the blog post series "Getting to know MDE"!

In this series, each post contains a different component/feature/methodology when it comes to understanding and managing Microsoft Defender for Endpoint (MDE).

> In this post, the focus is on **Windows Events: What Event IDs to look for to investigate for blocks/detections done by MDE, and using XPath queries to quickly filter for the needed Events.**

In each section, a brief description of a different MDE capability is described, including what Event IDs to look for, and an XPath query to use when searching for events of that capability.

If you want to get a tl;dr XPath query from this blog post which includes all the detections, check the last section.

This post was inspired by the previous post regarding [Using KQL to identify detections from MDE](https://kostaskoutrou.github.io/2026/01/06/using-kql-for-mde.html#device-control), as well as the [XPath queries provided by Microsoft](https://learn.microsoft.com/en-us/defender-endpoint/attack-surface-reduction-windows-events#custom-xml-templates-for-attack-surface-reduction-events), which were expanded upon to search for all Defender features.

So let's get started.

---

## Contents

* TOC
{:toc}

---

## Event Viewer general info and logs to look for and what they include

Before starting with the Windows Events of each MDE capability, to get an idea of how a Windows Event is structured, an example of an Event reporting a detection and quarantining of a file which Defender detected as a threat is provided below:

```xml
<Event xmlns="http://schemas.microsoft.com/win/2004/08/events/event">
  <System>
    <Provider Name="Microsoft-Windows-Windows Defender" Guid="{11cd958a-c507-4ef3-b3f2-5fd9dfbd2c78}" />
    <EventID>1117</EventID>
    <Version>0</Version>
    <Level>4</Level>
    <Task>0</Task>
    <Opcode>0</Opcode>
    <Keywords>0x8000000000000000</Keywords>
    <TimeCreated SystemTime="2026-01-22T11:02:01.1593627Z" />
    <EventRecordID>37116</EventRecordID>
    <Correlation ActivityID="{5768510e-0445-477f-bf75-2a37805d0ef0}" />
    <Execution ProcessID="5224" ThreadID="24524" />
    <Channel>Microsoft-Windows-Windows Defender/Operational</Channel>
    <Computer>DESKTOP-ABC</Computer>
    <Security UserID="S-1-5-18" />
  </System>
  <EventData>
    <Data Name="Product Name">Microsoft Defender Antivirus</Data>
    <Data Name="Product Version">4.18.25110.6</Data>
    <Data Name="Detection ID">{CE415EB5-1414-4FDE-B466-541D57C3035A}</Data>
    <Data Name="Detection Time">2026-01-22T10:59:42.891Z</Data>
    <Data Name="Threat ID">2147536539</Data>
    <Data Name="Threat Name">Exploit:HTML/IframeRef.gen</Data>
    <Data Name="Severity ID">5</Data>
    <Data Name="Severity Name">Severe</Data>
    <Data Name="Category ID">30</Data>
    <Data Name="Category Name">Exploit</Data>
    <Data Name="FWLink">https://go.microsoft.com/fwlink/?linkid=37020&amp;name=Exploit:HTML/IframeRef.gen&amp;threatid=2147536539&amp;enterprise=0</Data>
    <Data Name="Status Code">3</Data>
    <Data Name="Status Description"></Data>
    <Data Name="State">2</Data>
    <Data Name="Source ID">2</Data>
    <Data Name="Source Name">System</Data>
    <Data Name="Process Name">Unknown</Data>
    <Data Name="Detection User">NT AUTHORITY\SYSTEM</Data>
    <Data Name="Path">containerfile:_E:\phishing_datasets.zip; file:_E:\phishing_datasets.zip-&gt;phishing_datasets/private-phishing4.mbox-&gt;(IframeRefI)</Data>
    <Data Name="Origin ID">1</Data>
    <Data Name="Origin Name">Local machine</Data>
    <Data Name="Execution ID">0</Data>
    <Data Name="Execution Name">Unknown</Data>
    <Data Name="Type ID">2</Data>
    <Data Name="Type Name">Generic</Data>
    <Data Name="Pre Execution Status">0</Data>
    <Data Name="Action ID">2</Data>
    <Data Name="Action Name">Quarantine</Data>
    <Data Name="Error Code">0x00000000</Data>
    <Data Name="Error Description">The operation completed successfully. </Data>
    <Data Name="Post Clean Status">0</Data>
    <Data Name="Additional Actions ID">0</Data>
    <Data Name="Additional Actions String">No additional actions required</Data>
    <Data Name="Remediation User">NT AUTHORITY\SYSTEM</Data>
    <Data Name="Security intelligence Version">AV: 1.443.793.0, AS: 1.443.793.0, NIS: 1.443.793.0</Data>
    <Data Name="Engine Version">AM: 1.1.25110.1, NIS: 1.1.25110.1</Data>
  </EventData>
</Event>
```

It may look like the event includes a lot of information initially, but in reality most of it is not really useful for typical troubleshooting. Also, Windows Event Viewer parses these logs and presents them in a user-friendly format:

<img alt="image" src="https://github.com/user-attachments/assets/2546a011-ddba-4d45-8ca0-13ddfa722073" />

Every Windows Event log is divided into two main sections:

- The `<System>` Block (The Envelope): This contains the **standard metadata** generated by Windows. It tells when it happened, what computer it happened on (DESKTOP-ABC), and where the log is saved (Windows Defender/Operational).
- The `<EventData>` Block (The Letter): This contains the **custom payload** for the specific event. For Windows Defender, this is where to find the exact details about the malware, its location, and what the antivirus decided to do about it.

###XPath queries

A great thing about the Windows Event Viewer is that **all of its logs are stored as XML data with the strictly enforced XML schema** depicted previously, so custom views can be created using XPath queries, where only the needed events are filtered. XPath (XML Path Language) is a query language used to find and extract specific information from an XML document.

Using XPath allows for creating shareable filtered **views where only the detections and blocks done by Defender are shown**, which results in similar reports to the ones created using the KQL queries shown in the previous post [Using KQL to identify detections from MDE - Getting to know MDE Part 2](https://kostaskoutrou.github.io/2026/01/06/using-kql-for-mde.html).

All the XPath queries used here have the following format:

```xml
<QueryList>
  <Query Id="0" Path="Microsoft-Windows-Windows Defender/Operational">
   <Select Path="Microsoft-Windows-Windows Defender/Operational">*[System[(EventID=... or EventID=... or ...)]]</Select>
   <Select> more Paths filters... </Select>
  </Query>
</QueryList>
```

The only difference between the queries are **Windows Event channel paths** and the **Event IDs** searched for. To not repeat the explanation of the query under each section, a general explanation is provided here:

1. Default path to look for is `Microsoft-Windows-Windows Defender/Operational`. This is needed to be stated even though other paths are specified below because the schema rules dictate that this container must declare a starting point. Even if you try to skip it, when you save the Custom View in Windows Event Viewer, a path will be added automatically.
2. `Select Path="Microsoft-Windows-Windows Defender/Operational">`: Search in the path `Microsoft-Windows-Windows Defender/Operational`
3. `*\[System\[\(EventID=... or EventID=... or EventID=...\)\]\]`:
  1. `*`: Search at any root event, i.e., grab all events in the log as a starting point.
  2. `[...]`: The brackets contents in XPath are used to apply a filter to the item just before it, i.e., the asterisk, i.e., all the events.
  3. `System`: As mentioned before, every Windows Event is split into two main XML sections: `<System>` (metadata like time, provider and Event ID) and `<EventData>` (the details of what happened). Here we are filtering the `<System>` section.
  4. The brackets again filter to the item just before, i.e., the `<System>` section.
  5. `EventID=...`: With this condition the specified Event IDs are filtered.

There are 2 ways to import an XPath query in Windows Event Viewer:

**1) Create a Custom View**

Open Event Viewer and click on `Create Custom View...`.

<img alt="image" src="https://github.com/user-attachments/assets/7820ffb5-0d1e-4da3-b4b8-b1c3039f54c6" />

Click on the `XML` tab and check the `Edit query manually` checkbox.

<img alt="image" src="https://github.com/user-attachments/assets/d041f4e2-770c-4f39-a261-f0333497e8f8" />

Click Yes on the warning pop-up:

<img alt="image" src="https://github.com/user-attachments/assets/e1d23a8b-a1a9-44b4-add2-b6a9a3cdf4af" />

You can now paste the XPath queries provided on this post and save them accordingly.

**2) Import a Custom View**

If you have saved any of the XPath queries as XML files, you can directly import them by clicking on the `Import Custom View...` in Windows Event Viewer:

<img alt="image" src="https://github.com/user-attachments/assets/a01dc539-1f02-4046-ace8-56a72dd01539" />

After importing the custom view, you can export the results to a CSV, txt, XML, or EVTX (Event Files) format by clicking on `Save All Events in Custom View As...`:

<img alt="image" src="https://github.com/user-attachments/assets/4cc67355-9e5c-45b9-9572-52888c83e356" />

In the next sections, XPath queries are provided for each Defender feature. These can be used to facilitate troubleshooting towards identifying false positive detections and blocks. More details on the specific XPath queries are found in the sections below.

## Demo Files

If there are no Events for a specific feature and there is a need to produce some for confirmation of the query working as expected, there are [demo files provided by Microsoft](https://learn.microsoft.com/en-us/defender-endpoint/defender-endpoint-demonstrations), which can be used to force Defender detections and produce Windows Events.

## MDAV Changes logging

Starting with the logging of changes to configurations, an interesting and useful event related to all Microsoft Defender Antivirus (MDAV) features is Event ID 5007 with description "Event when settings are changed". A lot of events with Event ID 5007 are logged because it includes any change happening to the MDAV configuration, such as:

- Signature/Engine updates
- Policy syncs from Group Policy, Intune, SCCM
- Cloud-Delivered Protection Adjustments
- Any MDAV feature changing states (e.g., ASR rule switching from Audit mode to Block mode)

Investigating these logs is useful to identify suspicious tampering of the MDAV configuration, like disabling ASR rules or Real-Time protection, as well as to troubleshoot MDAV behavior that started unexpectedly, such as to identify when an ASR rule switched from Audit mode to Block mode.

In addition to the Event ID above, there are the following Event IDs related to changing settings in Defender:

|Event ID|Description|
|:-:|-|
|5000|Real-time protection enabled: Microsoft Defender Antivirus real-time protection scanning for malware and other potentially unwanted software was enabled.|
|5001|Real-time protection disabled: Microsoft Defender Antivirus real-time protection scanning for malware and other potentially unwanted software was disabled.|
|5004|Microsoft Defender Antivirus real-time protection feature configuration changed.|
|5007|The antimalware platform configuration changed.|
|5009|Antispyware enabled: Microsoft Defender Antivirus enabled scanning for malware and other potentially unwanted software.|
|5010|Antispyware disabled: Microsoft Defender Antivirus scanning for malware and other potentially unwanted software is disabled.|
|5011|Malware protection enabled: Microsoft Defender Antivirus scanning for viruses is enabled.|
|5012|Malware protection disabled: Microsoft Defender Antivirus scanning for viruses is disabled.|

To search for these Event IDs, which can be found under **Applications and Services Logs > Microsoft > Windows > Windows Defender > Operational**, the following XPath query can be used:

```xml
<QueryList>
  <Query Id="0" Path="Microsoft-Windows-Windows Defender/Operational">
   <Select Path="Microsoft-Windows-Windows Defender/Operational">
    *[System[EventID=5000 or EventID=5001 or EventID=5004 or 
    EventID=5007 or (EventID &gt;= 5009 and EventID &lt;= 5012)]]
  </Select>
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
|1121|Event when an ASR rule fires in block mode|
|1122|Event when an ASR rule fires in audit mode|
|1129|Event when the user overrides an ASR rule block in warn mode|

The following XPath query will search for ASR rule detections, blocks, and user overrides:

```xml
<QueryList>
  <Query Id="0" Path="Microsoft-Windows-Windows Defender/Operational">
   <Select Path="Microsoft-Windows-Windows Defender/Operational">
    *[System[(EventID=1121 or EventID=1122 or EventID=1129)]]
   </Select>
  </Query>
</QueryList>
```

If **blocks only** are needed, just keep Event ID 1121:

```xml
<QueryList>
  <Query Id="0" Path="Microsoft-Windows-Windows Defender/Operational">
   <Select Path="Microsoft-Windows-Windows Defender/Operational">
    *[System[(EventID=1121)]
   </Select>
  </Query>
</QueryList>
```

## Controlled Folder Access (CFA)

The basic idea with CFA is that a set of directories (folders) and a set of applications are defined. Only that set of apps is allowed to process in any way the set of directories. For more info, click [here](https://learn.microsoft.com/en-us/defender-endpoint/controlled-folder-access-overview)

When it comes to Windows Event logs, the CFA events are located under **Applications and Services Logs > Microsoft > Windows > Windows Defender > Operational**. The following event IDs are related to CFA:

|Event ID|Description|
|:-:|-|
|1124|Audited controlled folder access event|
|1123|Blocked controlled folder access event|
|1127|Controlled Folder Access blocked an untrusted process from potentially modifying disk sectors. In general, these events are more severe, because they mean that an attempt was made to write directly to the hard drive, bypassing the file system.|
|1128|Audited controlled folder access sector write block event. In general, these events are more severe, because they mean that an attempt was done to write directly to the hard drive, bypassing the file system.|

The following XPath query will filter for all detection and block events:

```xml
<QueryList>
  <Query Id="0" Path="Microsoft-Windows-Windows Defender/Operational">
   <Select Path="Microsoft-Windows-Windows Defender/Operational">
    *[System[(EventID=1123 or EventID=1124 or EventID=1127 or EventID=1128)]]
   </Select>
  </Query>
</QueryList>
```

For blocks only, use the following:

```xml
<QueryList>
  <Query Id="0" Path="Microsoft-Windows-Windows Defender/Operational">
   <Select Path="Microsoft-Windows-Windows Defender/Operational">
    *[System[(EventID=1123 or EventID=1127)]]
   </Select>
  </Query>
</QueryList>
```

## Device Control

[Device Control](https://learn.microsoft.com/en-us/defender-endpoint/device-control-overview) enables controls related to usage and installation of peripheral (USB/Bluetooth) or other devices with endpoints.

When it comes to Windows Event logs, for Device Control there is no specific set of events or a specific path where all the events reside. It is more complicated to identify block events. I could not find any specific set of events to look for. The most relevant events are logged under

- **Security**
- **Microsoft > Windows > Kernel > PnP > Configuration**
- **Microsoft > Windows > DeviceSetupManager > Admin**
- **Microsoft > Windows > PrintService > Admin**

|Event ID|Description|
|:-:|-|
|6423|The system blocked the installation of a hardware device (USB, Bluetooth radio, etc.) due to device installation restrictions.|
|871|A print job was explicitly blocked by Defender Device Control or Removable Storage Access Control.|
|808|The print spooler blocked a plug-in module from loading, often due to untrusted driver execution policies.|
|215|A printer driver failed to install due to a security policy or restriction.|

The XPath query for the Event IDs above is this following:

```xml
<QueryList>
  <Query Id="0" Path="Security">
    <Select Path="Security">
      *[System[EventID=6423]]
    </Select>
    <Select Path="Microsoft-Windows-PrintService/Admin">
      *[System[(EventID=871 or EventID=808 or EventID=215)]]
    </Select>
  </Query>
</QueryList>
```

## Exploit Protection

Exploit Protection helps protect against malware that uses exploits to infect devices and spread. It consists of many mitigations that can be applied to either the whole OS or individual apps. For more info, click [here](https://learn.microsoft.com/en-us/defender-endpoint/exploit-protection).

When it comes to Windows Event logs, most Exploit Protection events are located in the Windows Event log under **Security-Mitigations > Kernel Mode and Security-Mitigations > User Mode**, while some are located in **WER-Diagnostics > Operational** and **Win32k > Operational**. The following event IDs are related to CFA:

**Security-Mitigations > Kernel Mode** and **Security-Mitigations > User Mode**
|Event ID|Description|
|:-:|-|
|1|Arbitrary Code Guard (ACG) audit|
|2|rbitrary Code Guard (ACG) enforce|
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
|13|Export Address Filtering (EAF) audit|
|14|Export Address Filtering (EAF) enforce|
|15|Export Address Filtering+ (EAF+) audit|
|16|Export Address Filtering+ (EAF+) enforce|
|17|Import Address Filtering (IAF) audit|
|18|Import Address Filtering (IAF) enforce|
|19|Return-Oriented Programming (ROP) StackPivot audit|
|20|Return-Oriented Programming (ROP) StackPivot enforce|
|21|Return-Oriented Programming (ROP) CallerCheck audit|
|22|Return-Oriented Programming (ROP) CallerCheck enforce|
|23|Return-Oriented Programming (ROP) SimExec audit|
|24|Return-Oriented Programming (ROP) SimExec enforce|

**WER-Diagnostics > Operational**
|Event ID|Description|
|:-:|-|
|5|Control Flow Guard (CFG) Block|

**Win32k > Operational**
|Event ID|Description|
|:-:|-|
|260|Untrusted Font|

The following XPath query will filter for all detection and block events for Exploit Protection:

```xml
<QueryList>
  <Query Id="0" Path="Microsoft-Windows-Security-Mitigations/KernelMode">
   <Select Path="Microsoft-Windows-Security-Mitigations/KernelMode">
    *[System[(EventID &gt;= 1 and EventID &lt;= 24) or EventID=260]]
   </Select>
   <Select Path="Microsoft-Windows-Security-Mitigations/UserMode">
    *[System[(EventID &gt;= 1 and EventID &lt;= 24) or EventID=260]]
   </Select>
   <Select Path="Microsoft-Windows-Win32k/Operational">
    *[System[(EventID &gt;= 1 and EventID &lt;= 24) or EventID=260]]
   </Select>
   <Select Path="Microsoft-Windows-WER-Diagnostics/Operational">
    *[System[(EventID &gt;= 1 and EventID &lt;= 24) or EventID=260]]
   </Select>
  </Query>
</QueryList>
```

For blocks only, the following XPath query can be used:

```xml
<QueryList>
  <Query Id="0" Path="Microsoft-Windows-Security-Mitigations/KernelMode">
    <Select Path="Microsoft-Windows-Security-Mitigations/KernelMode">
      *[System[(EventID=2 or EventID=4 or EventID=6 or EventID=8 or
      EventID=10 or EventID=12 or EventID=14 or EventID=16 or
      EventID=18 or EventID=20 or EventID=22 or EventID=24)]]
    </Select>
    <Select Path="Microsoft-Windows-Security-Mitigations/UserMode">
      *[System[(EventID=2 or EventID=4 or EventID=6 or EventID=8 or
      EventID=10 or EventID=12 or EventID=14 or EventID=16 or
      EventID=18 or EventID=20 or EventID=22 or EventID=24)]]
    </Select>
    <Select Path="Microsoft-Windows-WER-Diagnostics/Operational">
      *[System[(EventID=5)]]
    </Select>
    <Select Path="Microsoft-Windows-Win32k/Operational">
      *[System[(EventID=260)]]
    </Select>
  </Query>
</QueryList>
```

## Network Protection, Web Protection, and SmartScreen

The combination of Network Protection, Web Protection, and SmartScreen provides the ability to block access to malicious websites, custom indicators, and web content filtering. For more info, click [here](https://learn.microsoft.com/en-us/defender-endpoint/network-protection).

When it comes to Windows Event logs, the Network Protection events are located in the Windows Event log under **Applications and Services Logs > Microsoft > Windows > Windows Defender > Operational**. The following event IDs are related to Network Protection:

|Event ID|Description|
|:-:|-|
|1125|Event when network protection fires in audit mode|
|1126|Event when network protection fires in block mode|

The following XPath query can be used to detect these events:

```xml
<QueryList>
 <Query Id="0" Path="Microsoft-Windows-Windows Defender/Operational">
  <Select Path="Microsoft-Windows-Windows Defender/Operational">
  *[System[(EventID=1125 or EventID=1126)]]
  </Select>
 </Query>
</QueryList>
```

If only the block events are needed, the audit events can be emitted:

```xml
<QueryList>
 <Query Id="0" Path="Microsoft-Windows-Windows Defender/Operational">
  <Select Path="Microsoft-Windows-Windows Defender/Operational">
  *[System[(EventID=1126)]]
  </Select>
 </Query>
</QueryList>
```

## Tamper Protection

[Tamper Protection](https://learn.microsoft.com/en-us/defender-endpoint/prevent-changes-to-security-settings-with-tamper-protection) blocks any changes attempted to several MDAV settings.

When it comes to events generated, the only Event ID that matters for Tamper Protection is located under **Applications and Services Logs > Microsoft > Windows > Windows Defender > Operational**, with Event ID 5013. The message of this Event ID is "Tamper protection blocked a change to Microsoft Defender Antivirus", and states which setting change was blocked.

The XPath query for this Event ID is the following:

```xml
<QueryList>
 <Query Id="0" Path="Microsoft-Windows-Windows Defender/Operational">
  <Select Path="Microsoft-Windows-Windows Defender/Operational">
    *[System[(EventID=5013)]]
  </Select>
 </Query>
</QueryList>
```

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

These Event IDs are logged under **Applications and Services Logs > Microsoft > Windows > Windows Defender > Operational.** Regarding the first Event IDs couples, they log  the same event, but the 100x ones are older Event IDs for Windows 7, Windows 8, while the 111x ones are the new ones introduced in Windows 10.

The XPath query to filter for those events is the following:

```xml
<QueryList>
  <Query Id="0" Path="Microsoft-Windows-Windows Defender/Operational">
   <Select Path="Microsoft-Windows-Windows Defender/Operational">
    *[System[(EventID=1006 or EventID=1007 or EventID=1008 or
    EventID=1011 or EventID=1012 or EventID=1015 or EventID=1116 or
    EventID=1117 or EventID=1118 or EventID=1119)]]
   </Select>
  </Query>
</QueryList>
```

As described in the Event IDs table, these events log also events of Potentially Unwanted Software/Applications (PUA). If the events **not** related to PUA are needed, an additional filter is needed to be applied in the "Category Name" field of the EventData. The following XPath query can be used:

```xml
<QueryList>
  <Query Id="0" Path="Microsoft-Windows-Windows Defender/Operational">
   <Select Path="Microsoft-Windows-Windows Defender/Operational">
    *[EventData[Data[@Name='Category Name']!='Potentially Unwanted Software'] and 
    System[(EventID=1006 or EventID=1007 or EventID=1008 or EventID=1011 or 
    EventID=1012 or EventID=1015 or EventID=1116 or EventID=1117 or 
    EventID=1118 or EventID=1119)]]
   </Select>
  </Query>
</QueryList>
```

## Potentially Unwanted Apps (PUA)

PUA is a category of software that can:

- Cause the machine to run slowly
- Display unexpected apps
- Install other software that might be unwanted

For more info, click [here](https://learn.microsoft.com/en-us/defender-endpoint/detect-block-potentially-unwanted-apps-microsoft-defender-antivirus).

As mentioned in the previous section, PUA is logged under the same Event IDs as MDAV. Therefore, if events related only PUA are needed, an additional filter is needed to be applied in the "Category Name" field of the EventData. The XPath query that can be used is the following:

```xml
<QueryList>
  <Query Id="0" Path="Microsoft-Windows-Windows Defender/Operational">
   <Select Path="Microsoft-Windows-Windows Defender/Operational">
    *[EventData[Data[@Name='Category Name']='Potentially Unwanted Software'] and 
    System[(EventID=1006 or EventID=1007 or EventID=1008 or EventID=1011 or 
    EventID=1012 or EventID=1015 or EventID=1116 or EventID=1117 or 
    EventID=1118 or EventID=1119)]]
   </Select>
  </Query>
</QueryList>
```

## App Control for Business and AppLocker

[AppLocker](https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/applocker/applocker-overview) helps control which apps and files users can run. [App Control for Business](https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/appcontrol-and-applocker-overview) also known as **Windows Defender Application Control** (WDAC), formerly known as **Configurable Code Integrity** (CCI), is a newer solution for the same purpose, and provides more granular controls.

When it comes to Windows Events, AppLocker logs events under **Application and Services Logs > Microsoft > Windows > AppLocker**, while App Control logs its events under the AppLocker path as well as **Application and Services Logs > Microsoft > Windows > CodeIntegrity > Operational**. The events are quite granular, where there are categories based on

- the file type of the event
- whether the file that is about to be ran is there because of a Managed Installer (Intune, SCCM, etc.)
- the feature that is controlling the file execution (AppLocker or WDAC)

More info:

- [AppLocker events](https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/applocker/using-event-viewer-with-applocker)
- [App Control for Business events](https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/operations/event-id-explanations)

For AppLocker only, the following Event IDs are of interest:

|Event ID|Description|
|:-:|-|
|8000|AppID policy conversion failed. This indicates that the policy wasn’t applied correctly to the computer.|
|8001|The AppLocker policy was applied successfully to this computer.|
|8002|File was allowed to run. (.exe and .dll files.). When a process launches that matches a managed installer rule, this event is raised with PolicyName = MANAGEDINSTALLER found in the event Details.|
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

For App Control only, the following Event IDs are of interest (the table information was taken from the previously providing link regarding [Understanding App Control events](https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/operations/event-id-explanations):

|Event ID|Description|
|:-:|-|
|8028|This event indicates that a script host, such as PowerShell, queried App Control about a file the script host was about to run. Since the policy was in audit mode, the script or MSI file should have run, but wouldn't have passed the App Control policy if it was enforced.|
|8029|This event is the enforcement mode equivalent of event 8028.|
|8036|COM object was prevented from running.|
|8037|This event indicates that a script host checked whether to allow a script to run, and the file passed the App Control policy.|
|8038|Signing information event correlated with either an 8028 or 8029 event. One 8038 event is generated for each signature of a script file. Contains the total number of signatures on a script file and an index as to which signature it is. Unsigned script files generate a single 8038 event with TotalSignatureCount 0. These events are correlated with 8028 and 8029 events and can be matched using the Correlation ActivityID found in the System portion of the event.|
|8039|This event indicates that a packaged app (MSIX/AppX) was allowed to install or run because the App Control policy is in audit mode. But, it would have been blocked if the policy was enforced.|
|8040|This event indicates that a packaged app was prevented from installing or running due to the App Control policy.|

There are also some logs under **Applications and Services logs > Microsoft > Windows > CodeIntegrity > Operational**, focused on App Control policy activation and the control of executables, dlls, and drivers.

|Event ID|Explanation|
|:-:|-|
|3004|This event isn't common and may occur with or without an App Control policy present. It typically indicates a kernel driver tried to load with an invalid signature. For example, the file may not be WHQL-signed ([Windows Hardware Quality Labs](https://learn.microsoft.com/en-us/windows-hardware/drivers/install/whql-test-signature-program)) on a system where WHQL is required.  This event is also seen for kernel- or user-mode code that the developer opted-in to (/INTEGRITYCHECK)[https://learn.microsoft.com/en-us/cpp/build/reference/integritycheck-require-signature-check?view=msvc-170] but isn't signed correctly.|
|3033|This event may occur with or without an App Control policy present and should occur alongside a 3077 event if caused by App Control policy. It often means the file's signature is revoked or a signature with the Lifetime Signing EKU has expired. Presence of the Lifetime Signing EKU is the only case where App Control blocks files due to an expired signature. Try using option 20 Enabled:Revoked Expired As Unsigned in your policy along with a rule (for example, hash) that doesn't rely on the revoked or expired cert.  This event also occurs if code compiled with Code Integrity Guard (CIG) tries to load other code that doesn't meet the CIG requirements.|
|3034|This event isn't common. It's the audit mode equivalent of event 3033.|
|3076|This event is the main App Control block event for audit mode policies. It indicates that the file would have been blocked if the policy was enforced.|
|3077|This event is the main App Control block event for enforced policies. It indicates that the file didn't pass your policy and was blocked.|
|3089|This event contains signature information for files that were blocked or audit blocked by App Control. One of these events is created for each signature of a file. Each event shows the total number of signatures found and an index value to identify the current signature. Unsigned files generate a single one of these events with TotalSignatureCount of 0. These events are correlated with 3004, 3033, 3034, 3076 and 3077 events. You can match the events using the Correlation ActivityID found in the System portion of the event.|
|3090|Optional This event indicates that a file was allowed to run based purely on ISG or managed installer.|
|3091|This event indicates that a file didn't have ISG or managed installer authorization and the App Control policy is in audit mode.|
|3092|This event is the enforcement mode equivalent of 3091.|

Lastly, for running files installed via a Managed Installer (Intune, SCCM, etc.), there are some events which are AppLocker events, but may be triggered by either AppLocker or App Control. More specifically, the Managed Installer feature was initially built for AppLocker. The following two are AppLocker components:

1. The [SmartlockerFilter](https://revertservice.com/10/applockerfltr/) driver that watches the installations and identifies files created by authorized installers
2. The [AppID](https://learn.microsoft.com/en-us/windows/security/application-security/application-control/app-control-for-business/applocker/configure-the-application-identity-service) service that verifies the tags.

These components are used to trust Managed Installers, and track applications which exist in the machine because of a Managed Installer. They can then be used either by AppLocker itself or by App Control to allow or block the execution of Managed Installer managed applications. In this case, the following events are of interest:

|Event ID|Explanation|
|:-:|-|
|8030|ManagedInstaller check SUCCEEDED during Appid verification of *. A user tried to run a program. Windows checked for the hidden stamp, found it, and confirmed that the trusted Managed Installed installed it. The program is allowed to run.|
|8031|SmartlockerFilter detected file * being written by process *. The trusted deployment tool is actively downloading or installing a program. The Smartlocker filter driver is watching the tool write the files to the hard drive and is actively stamping them with the hidden "trusted" tag.|
|8032|ManagedInstaller check FAILED during Appid verification of *. A user tried to run a program. Windows checked for the hidden stamp, but it wasn't there. This means the user downloaded it themselves from the internet or brought it in on a USB drive. Because the security policy is set to Enforce, the program is blocked.|
|8033|ManagedInstaller check FAILED during Appid verification of * . Allowed to run due to Audit AppLocker Policy. This is the equivalent of Event ID 8032, but the program is not blocked, but only audited.|
|8034|ManagedInstaller Script check FAILED during Appid verification of *. A script tried to run, but it lacked the hidden stamp from the trusted Managed Installer. The script is blocked.|
|8035|ManagedInstaller Script check SUCCEEDED during Appid verification of *. A script tried to run, Windows found the hidden stamp by the trusted Managed Installer, and the script is allowed to run.|

For `AppLocker`, the XPath query that can be used is the following:

```xml
<QueryList>
  <Query Id="0" Path="Microsoft-Windows-AppLocker/EXE and DLL">
    <Select Path="Microsoft-Windows-AppLocker/EXE and DLL">
      *[System[(EventID &gt;= 8000 and EventID &lt;= 8008) or 
      (EventID &gt;= 8020 and EventID &lt;= 8027) or 
      (EventID &gt;= 8030 and EventID &lt;= 8035)]]
    </Select>
    <Select Path="Microsoft-Windows-AppLocker/MSI and Script">
      *[System[(EventID &gt;= 8000 and EventID &lt;= 8008) or 
      (EventID &gt;= 8020 and EventID &lt;= 8027) or 
      (EventID &gt;= 8030 and EventID &lt;= 8035)]]
    </Select>
    <Select Path="Microsoft-Windows-AppLocker/Packaged app-Deployment">
      *[System[(EventID &gt;= 8000 and EventID &lt;= 8008) or 
      (EventID &gt;= 8020 and EventID &lt;= 8027) or 
      (EventID &gt;= 8030 and EventID &lt;= 8035)]]
    </Select>
    <Select Path="Microsoft-Windows-AppLocker/Packaged app-Execution">
      *[System[(EventID &gt;= 8000 and EventID &lt;= 8008) or 
      (EventID &gt;= 8020 and EventID &lt;= 8027) or 
      (EventID &gt;= 8030 and EventID &lt;= 8035)]]
    </Select>
  </Query>
</QueryList>
```


For `App Control`, the XPath query that can be used is the following:

```xml
<QueryList>
  <Query Id="0" Path="Microsoft-Windows-CodeIntegrity/Operational">
    <Select Path="Microsoft-Windows-CodeIntegrity/Operational">
      *[System[EventID=3004 or EventID=3033 or EventID=3034 or EventID=3076 or 
      EventID=3077 or (EventID &gt;= 3089 and EventID &lt;= 3092)]]
    </Select>
    <Select Path="Microsoft-Windows-AppLocker/MSI and Script">
      *[System[EventID=8028 or EventID=8029 or
      (EventID &gt;= 8036 and EventID &lt;= 8040)]]
    </Select>
    <Select Path="Microsoft-Windows-AppLocker/Packaged app-Deployment">
      *[System[EventID=8028 or EventID=8029 or
      (EventID &gt;= 8036 and EventID &lt;= 8040)]]
    </Select>
    <Select Path="Microsoft-Windows-AppLocker/Packaged app-Execution">
      *[System[EventID=8028 or EventID=8029 or
      (EventID &gt;= 8036 and EventID &lt;= 8040)]]
    </Select>
  </Query>
</QueryList>
```

For blocks only, for both features, the following XPath query can be used:

```xml
<QueryList>
  <Query Id="0" Path="Microsoft-Windows-CodeIntegrity/Operational">
    <!-- AppLocker and App Control 1 -->
    <Select Path="Microsoft-Windows-CodeIntegrity/Operational">
      *[System[(EventID=3004 or EventID=3033 or 
      EventID=3077 or EventID=3092)]]
    </Select>
    <!-- AppLocker and App Control 2 -->
    <Select Path="Microsoft-Windows-AppLocker/EXE and DLL">
      *[System[(EventID=8004 or EventID=8007 or EventID=8022 or 
      EventID=8025 or EventID=8027 or EventID=8029 or 
      EventID=8032 or EventID=8034 or EventID=8036 or EventID=8040)]]
    </Select>
    <!-- AppLocker and App Control 3 -->
    <Select Path="Microsoft-Windows-AppLocker/MSI and Script">
      *[System[(EventID=8004 or EventID=8007 or EventID=8022 or 
      EventID=8025 or EventID=8027 or EventID=8029 or 
      EventID=8032 or EventID=8034 or EventID=8036 or EventID=8040)]]
    </Select>
    <!-- AppLocker and App Control 4 -->
    <Select Path="Microsoft-Windows-AppLocker/Packaged app-Deployment">
      *[System[(EventID=8004 or EventID=8007 or EventID=8022 or 
      EventID=8025 or EventID=8027 or EventID=8029 or 
      EventID=8032 or EventID=8034 or EventID=8036 or EventID=8040)]]
    </Select>
    <!-- AppLocker and App Control 5 -->
    <Select Path="Microsoft-Windows-AppLocker/Packaged app-Execution">
      *[System[(EventID=8004 or EventID=8007 or EventID=8022 or 
      EventID=8025 or EventID=8027 or EventID=8029 or 
      EventID=8032 or EventID=8034 or EventID=8036 or EventID=8040)]]
    </Select>
  </Query>
</QueryList>
```

## MDE

When it comes to Microsoft Defender for Endpoint (MDE) itself, SENSE is the internal name used to refer to the behavioral sensor that powers MDE. The Windows Events created for SENSE can be found under the path **Applications and Services Logs > Microsoft > Windows > SENSE > Operational**. Information about the events created can be found in Microsoft's documentation: [Review events and errors using Event Viewer](https://learn.microsoft.com/en-us/defender-endpoint/event-error-codes
). The events created are only useful for troubleshooting purposes related to the SENSE service not operating as expected, but they are not related to any blocking activities. Therefore, they will not be documented in this post.

## General XML for all detections

Because sometimes there is no certainty as to which Defender feature may have blocked something, or the need may be to prove that Defender did not actually block anything, the following XPath query filters for any block event done by any Defender feature:

```xml
<QueryList>
  <Query Id="0" Path="Microsoft-Windows-Windows Defender/Operational">
    <!-- ASR blocks -->
    <Select Path="Microsoft-Windows-Windows Defender/Operational">
        *[System[(EventID=1121)]]
    </Select>
    <!-- CFA blocks -->
    <Select Path="Microsoft-Windows-Windows Defender/Operational">
        *[System[(EventID=1123 or EventID=1127)]]
    </Select>
    <!-- Device Control 1 -->
    <Select Path="Security">
        *[System[EventID=6423]]
    </Select>
    <!-- Device Control 2 -->
    <Select Path="Microsoft-Windows-PrintService/Admin">
       *[System[(EventID=871 or EventID=808 or EventID=215)]]
    </Select>
    <!-- Exploit Protection 1 -->
    <Select Path="Microsoft-Windows-Security-Mitigations/KernelMode">
      *[System[(EventID=2 or EventID=4 or EventID=6 or EventID=8 or
      EventID=10 or EventID=12 or EventID=14 or EventID=16 or
      EventID=18 or EventID=20 or EventID=22 or EventID=24)]]
    </Select>
    <!-- Exploit Protection 2 -->
    <Select Path="Microsoft-Windows-Security-Mitigations/UserMode">
      *[System[(EventID=2 or EventID=4 or EventID=6 or EventID=8 or
      EventID=10 or EventID=12 or EventID=14 or EventID=16 or
      EventID=18 or EventID=20 or EventID=22 or EventID=24)]]
    </Select>
    <!-- Exploit Protection 3 -->
    <Select Path="Microsoft-Windows-WER-Diagnostics/Operational">
      *[System[(EventID=5)]]
    </Select>
    <!-- Exploit Protection 4 -->
    <Select Path="Microsoft-Windows-Win32k/Operational">
      *[System[(EventID=260)]]
    </Select>
    <!-- Network Protection -->
    <Select Path="Microsoft-Windows-Windows Defender/Operational">
      *[System[(EventID=1126)]]
    </Select>
    <!-- Microsoft Defender Antivirus Action Taken (both malware and PUA) -->
    <Select Path="Microsoft-Windows-Windows Defender/Operational">
      *[System[(EventID=1007 or EventID=1117)]]
    </Select>
    <!-- AppLocker and App Control 1 -->
    <Select Path="Microsoft-Windows-CodeIntegrity/Operational">
      *[System[(EventID=3004 or EventID=3033 or 
      EventID=3077 or EventID=3092)]]
    </Select>
    <!-- AppLocker and App Control 2 -->
    <Select Path="Microsoft-Windows-AppLocker/EXE and DLL">
      *[System[(EventID=8004 or EventID=8007 or EventID=8022 or 
      EventID=8025 or EventID=8027 or EventID=8029 or 
      EventID=8032 or EventID=8034 or EventID=8036 or EventID=8040)]]
    </Select>
    <!-- AppLocker and App Control 3 -->
    <Select Path="Microsoft-Windows-AppLocker/MSI and Script">
      *[System[(EventID=8004 or EventID=8007 or EventID=8022 or 
      EventID=8025 or EventID=8027 or EventID=8029 or 
      EventID=8032 or EventID=8034 or EventID=8036 or EventID=8040)]]
    </Select>
    <!-- AppLocker and App Control 4 -->
    <Select Path="Microsoft-Windows-AppLocker/Packaged app-Deployment">
      *[System[(EventID=8004 or EventID=8007 or EventID=8022 or 
      EventID=8025 or EventID=8027 or EventID=8029 or 
      EventID=8032 or EventID=8034 or EventID=8036 or EventID=8040)]]
    </Select>
    <!-- AppLocker and App Control 5 -->
    <Select Path="Microsoft-Windows-AppLocker/Packaged app-Execution">
      *[System[(EventID=8004 or EventID=8007 or EventID=8022 or 
      EventID=8025 or EventID=8027 or EventID=8029 or 
      EventID=8032 or EventID=8034 or EventID=8036 or EventID=8040)]]
    </Select>
  </Query>
</QueryList>
```

##Conclusion

Sometimes, it is complicated to understand what Defender blocked, when, and why the block happened. Having tools like Windows Events or Defender Advanced Hunting logs does help, but there is an overwhelming amount of logs.

By creating a few queries ready to be used and filter for only the needed events, it cuts a few steps in the troubleshooting process and makes it a little bit quicker. This post focused on making such queries for the Windows Events.

Hopefully they will be used and help with issues appearing related to Defender, or to prove that Defender was not an issue.

Thank you for being here.

On to the next one!
