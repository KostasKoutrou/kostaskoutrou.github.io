# Using KQL to identify detections from MDE

## Introduction

Describe that we are looking at DeviceEvents table in advanced hunting, and filtering for different ActionType.

You can always refer to the DeviceEvents Schema reference directly from the Advanced Hunting page in Defender:

<img alt="image" src="https://github.com/user-attachments/assets/d43c04c7-c212-49b4-8e5e-767a6b20decd" />

If you want to get a tl;dr KQL query from this blog post, check the last section.

### DeviceEvents table

The detections by MDE are reported under the DeviceEvents table in Advanced Hunting.

Two important columns of the DeviceEvents table are the following:

- ActionType: The `ActionType` column shows what action triggered the `DeviceEvents` event. When it comes to MDE, the `ActionType` column shows which MDE detection tirggered the event.
- AdditionalFields: The `AdditionalFields` column contains, as its name suggests, additional information regarding the event which does not fit any of the other columns. When it comes to MDE detections, the `AdditionalFields` column contains necessary information about the detection, such as whether something was blocked or audited, or what policy triggered the event. It is in JSON format, and, depending on what is being searched for, the `AdditionalFields` JSON data will need to be parsed in order to retrieve the correct information. In the queries below, there are some examples to get a better idea of how to parse the column.

### Attack Surface Reduction Rules Detections

When looking for ASR Rules detections, keep the following in mind:

For each ASR Rule detection, there can be one of three ActionTypes in the DeviceEvents table, depending on the enabled state of the ASR Rule for the machine:

|ActionType|Description|
|-|-|
|Asr\<RuleName\>Audited|ASR Rule of <RuleName> was triggered but did not block.|
|Asr\<RuleName\>Blocked|ASR Rule of <RuleName> was triggered and blocked.|
|Asr\<RuleName\>WarnBypassed|ASR Rule of <RuleName> was triggered in Warn mode, and the user excluded themselves from it.|

In the following table the ASR Rules action types are listed and the ASR rule that they correspond to.

|ActionType|ASR Rule|
|-|-|
|AsrAbusedSystemToolAudited<br>AsrAbusedSystemToolBlocked<br>AsrAbusedSystemToolWarnBypassed|Block use of a copied or impersonated system tools|
|AsrAdobeReaderChiledProcessAudited<br>AsrAdobeReaderChiledProcessBlocked<br>AsrAdobeReaderChiledProcessWarnBypassed|Block Adobe Reader from creating child processes|
|AsrExecutableEmailContentAudited<br>AsrExecutableEmailContentBlocked<br>AsrExecutableEmailContentWarnBypassed|Block Launching of executable content from email attachment|
|AsrExecutableOfficeContentAudited<br>AsrExecutableOfficeContentBlocked<br>AsrExecutableOfficeContentWarnBypassed|Block Office applications from creating executable content|
|AsrLsassCredentialTheftAudited<br>AsrLsassCredentialTheftBlocked<br>AsrLsassCredentialTheftWarnBypassed|Block credential stealing from the Windows local security authority subsystem (lsass.exe)|
|AsrObfuscatedScriptAudited<br>AsrObfuscatedScriptBlocked<br>AsrObfuscatedScriptWarnBypassed|Block execution of potentially obfuscated scripts|
|AsrOfficeChildProcessAudited<br>AsrOfficeChildProcessBlocked<br>AsrOfficeChildProcessWarnBypassed|Block all Office applications from creating child processes|
|AsrOfficeCommAppChildProcessAudited<br>AsrOfficeCommAppChildProcessBlocked<br>AsrOfficeCommAppChildProcessWarnBypassed|Block Office communication application from creating child processes|
|AsrOfficeMacroWin32ApiCallsAudited<br>AsrOfficeMacroWin32ApiCallsBlocked<br>AsrOfficeMacroWin32ApiCallsWarnBypassed|Block Win32 API calls from Office macro|
|AsrOfficeProccessInjectionAudited<br>AsrOfficeProccessInjectionBlocked<br>AsrOfficeProccessInjectionWarnBypassed|Block Office applications from injecting code into other processes|
|AsrPersistenceThroughWmiAudited<br>AsrPersistenceThroughWmiBlocked<br>AsrPersistenceThroughWmiWarnBypassed|Block persistence through WMI event subscription|
|AsrPsexecWmiChildProcessAudited<br>AsrPsexecWmiChildProcessBlocked<br>AsrPsexecWmiChildProcessWarnBypassed|Block Process Creations originating from PSExec & WMI commands|
|AsrRandomwareAudited<br>AsrRandomwareBlocked<br>AsrRandomwareWarnBypassed|Use advanced protection against ransomware|
|AsrSafeModeRebootAudited<br>AsrSafeModeRebootBlocked<br>AsrSafeModeRebootWarnBypassed|Block rebooting machine in Safe Mode|
|AsrScriptExecutableDownloadAudited<br>AsrScriptExecutableDownloadBlocked<br>AsrScriptExecutableDownloadWarnBypassed|Block JavaScript or VBScript from launching downloaded executable content|
|AsrUntrustedExecutableAudited<br>AsrUntrustedExecutableBlocked<br>AsrUntrustedExecutableWarnBypassed|Block executable files from running unless they meet a prevalence, age, or trusted list criteria|
|AsrUntrustedUsbProcessAudited<br>AsrUntrustedUsbProcessBlocked<br>AsrUntrustedUsbProcessWarnBypassed|Block untrusted and unsigned processes that run from USB|
|AsrVulnerableSignedDriverAudited<br>AsrVulnerableSignedDriverBlocked<br>AsrVulnerableSignedDriverWarnBypassed|Block abuse of in-the-wild exploited vulnerable signed drivers|
|AsrWebShellOnServerAudited<br>AsrWebShellOnServerBlocked<br>AsrWebShellWarnBypassed (does not have the "OnServer" substring like the other 2)|Block Webshell creation for Servers|

Each ASR Rule has its own GUID. GUIDs are needed when configuring, e.g., a GPO to enable ASR rules in Audit or Block mode for machines. The ASR Rule to GUID matrix can be found in Microsoft's [Attack surface reduction rules reference](https://learn.microsoft.com/en-us/defender-endpoint/attack-surface-reduction-rules-reference#asr-rule-to-guid-matrix)

The KQL query to search for ASR Rule detections is the following:

```kql
DeviceEvents
| where ActionType startswith 'Asr'
| extend RuleId=extractjson("$Ruleid", AdditionalFields, typeof(string)) //Parse the AdditionalFields column in order to extract the ASR Rule GUID.
```

### Controlled Folder Access Detections

When looking for CFA detections, keep the following in mind:

CFA has only two Action Types reported in DeviceEvents:

- ControlledFolderAccessViolationAudited
- ControlledFolderAccessViolationBlocked

The KQL query to search for CFA detections is the following:

```kql
DeviceEvents
//If you want only blocks, remove the 'ControlledFolderAccessViolationAudited'.
| where ActionType in ('ControlledFolderAccessViolationAudited','ControlledFolderAccessViolationBlocked')
```

### Device Control

When looking for Device Control detections, keep the following in mind:

Device Control has six `ActionType`s reported in `DeviceEvents`:

|DeviceEvents ActionType|Description|
|-|-|
|BluetoothPolicyTriggered|A Bluetooth service llowed or blocked by a device control policy.|
|PnPDeviceAllowed|Device control allowed a trusted plug and play (PnP) device. Note that this is the default event when Device Control is enabled but no block policies are configured (ADD MORE INFO HERE)|
|PnPDeviceBlocked|Device control blocked an untrusted plug and play (PnP) device.|
|PrintJobBlocked|Device control prevented an untrusted printer from printing.|
|RemovableStorageFileEvent|Removable storage file activity matched a device control removable storage access control policy.|
|RemovableStoragePolicyTriggered|Device control detected an attempted read/write/execute event from a removable storage device.|

```kql
DeviceEvents
| where ActionType in ("BluetoothPolicyTriggered","PnPDeviceBlocked",
"PrintJobBlocked","RemovableStorageFileEvent","RemovableStoragePolicyTriggered") //You may want to search for "PnPDeviceAllowed", too, after configuring Device Control policies, to make sure that the desired activities are allowed. But for checking for Device Control Blocks, it is not needed.
```

### Exploit Protection

Ignore DeviceEvents with ActionType `ExploitGuardNetworkProtectionAudited` and `ExploitGuardNetowkrProtectionBlocked` when checking for Exploit Protection, as these Action Types correspond to detection done by Network Protection, which is a different ASR, and will be described in the next section.

```kql
DeviceEvents
| where ActionType startswith ("ExploitGuard
```

Note that some Exploit Protection measures do not create events, because they do not detect. For example, with Mandatory ASLR and Bottom-up ASLR, a program's code and libraries and loaded at a random memory address instead of a predictable one. This measure does not detect anything to create an event for.

Microsoft's [documentation](https://learn.microsoft.com/en-us/defender-endpoint/exploit-protection#exploit-protection-and-advanced-hunting) does not include the Control Flow Guard (CFG) DeviceEvents ActionTypes, it only includes the Exploit Protection measures ActionTypes actually starting with the substring `ExploitGuard`. The actual complete list is found in the following table.

|DeviceEvents ActionType|Description|Exploit Protection Measure|
|-|-|-|
|ExploitGuardAcgAudited<br>ExploitGuardAcgEnforced|Arbitrary code guard (ACG) **detected** or **blocked** an attempt to modify code page permissions or create unsigned code pages.|Arbitrary code guard (ACG)|
|ExploitGuardChildProcessAudited<br>ExploitGuardChildProcessBlocked|Exploit protection **detected** or **blocked** the creation of a child process|Don’t allow child processes|
|ExploitGuardEafViolationAudited<br>ExploitGuardEafViolationBlocked|Export address filtering (EAF) blocked possible exploitation activity.|Export address filtering (EAF)|
|ExploitGuardIafViolationAudited<br>ExploitGuardIafViolationBlocked|Import address filtering (IAF) **detected** or **blocked** possible exploitation activity.|Import address filtering (IAF)|
|ExploitGuardLowIntegrityImageAudited<br>ExploitGuardLowIntegrityImageBlocked|Exploit protection **detected** or **blocked** the launch of a process from a low-integrity file.|Block low integrity images|
|ExploitGuardNonMicrosoftSignedAudited<br>ExploitGuardNonMicrosoftSignedBlocked|Exploit protection **detected** or **blocked** the launch of a process from an image file that is not signed by Microsoft.|Code integrity guard|
|ExploitGuardRopExploitAudited<br>ExploitGuardRopExploitBlocked|Exploit protection blocked possible return-object programming (ROP) exploitation.|Simulate execution (SimExec)<br>Validate API invocation (CallerCheck)<br>Validate stack integrity (StackPivot)|
|ExploitGuardSharedBinaryAudited<br>ExploitGuardSharedBinaryBlocked|Exploit protection detected or blocked the launch of a process from a file in a remote shared file.|Block remote images|
|ExploitGuardWin32SystemCallAudited<br>ExploitGuardWin32SystemCallBlocked|Exploit protection **detected** or **blocked** a call to the Windows system AIP|Disable Win32k system calls|
|ControlFlowGuardViolation|Control Flow Guard terminated an application after detecting an invalid function call|Control flow guard (CFG)|

Unfortunately, it seems that for the following Exploit Protection measures, there are no Device Events created. If you know more details, please let me know:

- Data Execution Prevention (DEP)
- Validate exception chains (SEHOP)
- Validate heap integrity
- Block untrusted fonts
- Disable extension points
- Validate handle usage
- Validate image dependency integrity
- Hardware-enforced stack protection

