# Using KQL to identify detections from MDE

## Introduction


You can always refer to the DeviceEvents Schema reference directly from the Advanced Hunting page in Defender:

<img alt="image" src="https://github.com/user-attachments/assets/d43c04c7-c212-49b4-8e5e-767a6b20decd" />


### Attack Surface Reduction Rules Detections

Each ASR Rule has its own GUID. This will need to be used when configuring, e.g., a GPO to enable ASR rules in Audit or Block mode for machines. The ASR Rule to GUID matrix can be found in Microsoft's [Attack surface reduction rules reference](https://learn.microsoft.com/en-us/defender-endpoint/attack-surface-reduction-rules-reference#asr-rule-to-guid-matrix)

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

