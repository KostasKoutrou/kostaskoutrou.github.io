# Using KQL to identify detections from MDE

## Introduction

Describe that we are looking at DeviceEvents table in advanced hunting, and filtering for different ActionType.

You can always refer to the DeviceEvents Schema reference directly from the Advanced Hunting page in Defender:

<img alt="image" src="https://github.com/user-attachments/assets/d43c04c7-c212-49b4-8e5e-767a6b20decd" />


### Attack Surface Reduction Rules Detections

Each ASR Rule has its own GUID. This will need to be used when configuring, e.g., a GPO to enable ASR rules in Audit or Block mode for machines. The ASR Rule to GUID matrix can be found in Microsoft's [Attack surface reduction rules reference](https://learn.microsoft.com/en-us/defender-endpoint/attack-surface-reduction-rules-reference#asr-rule-to-guid-matrix)

The ASR Rules ActionTypes come in tuples:

|ActionType|Description|
|-|-|
|Asr<RuleName>Audited|ASR Rule of <RuleName> was triggered but did not block.|
|Asr<RuleName>Blocked|ASR Rule of <RuleName> was triggered and blocked.|
|Asr<RuleName>Bypassed|ASR Rule of <RuleName> was triggered in Warn mode, and the user excluded themselves from it.|

In the following table the ASR Rules action types are listed and the ASR rule that they correspond to.

|ActionType|ASR Rule|
|-|-|
|AsrAbusedSystemToolAudited<br>AsrAbusedSystemToolBlocked<br>AsrAbusedSystemToolBypassed|Block use of a copied or impersonated system tools|
|AsrAdobeReaderChiledProcess<br>AsrAdobeReaderChiledProcess<br>AsrAdobeReaderChiledProcess|Block Adobe Reader from creating child processes|
|AsrExecutableEmailContent<br>AsrExecutableEmailContent<br>AsrExecutableEmailContent|Block Launching of executable content from email attachment|
|AsrExecutableOfficeContent<br>AsrExecutableOfficeContent<br>AsrExecutableOfficeContent|Block Office applications from creating executable content|
|AsrLsassCredentialTheft<br>AsrLsassCredentialTheft<br>AsrLsassCredentialTheft|Block credential stealing from the Windows local security authority subsystem (lsass.exe)|
|AsrObfuscatedScript<br>AsrObfuscatedScript<br>AsrObfuscatedScript|Block execution of potentially obfuscated scripts|
|AsrOfficeChildProcess<br>AsrOfficeChildProcess<br>AsrOfficeChildProcess|Block all Office applications from creating child processes|
||Block Office communication application from creating child processes|
||Block Win32 API calls from Office macro|
||Block Office applications from injecting code into other processes|
||Block persistence through WMI event subscription|
||Block Process Creations originating from PSExec & WMI commands|
||Use advanced protection against ransomware|
||Block rebooting machine in Safe Mode|
||Block JavaScript or VBScript from launching downloaded executable content|
||Block executable files from running unless they meet a prevalence, age, or trusted list criteria|
||Block untrusted and unsigned processes that run from USB|
||Block abuse of in-the-wild exploited vulnerable signed drivers|
||Block Webshell creation for Servers|

The following KQL query parses the AdditionalFields column in order to extract the ASR Rule GUID.

```kql
DeviceEvents
| where ActionType startswith 'Asr'
| extend RuleId=extractjson("$Ruleid", AdditionalFields, typeof(string))
```

### Controlled Folder Access Detections

CFA has only two Action Types reported in DeviceEvents:

- ControlledFolderAccessViolationAudited
- ControlledFolderAccessViolationBlocked

So, searching for CFA DeviceEvents can be done with the following KQL query:

```kql
DeviceEvents
//If you want only blocks, remove the 'ControlledFolderAccessViolationAudited'.
| where ActionType in ('ControlledFolderAccessViolationAudited','ControlledFolderAccessViolationBlocked')
```

### Device Control

Device Control has six Action Types reported in DeviceEvents:

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

