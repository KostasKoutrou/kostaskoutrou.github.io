# Using KQL to identify detections from MDE

## Introduction

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

|DeviceEvents ActionType|Description|Exploit Protection Measure|
|-|-|-|
|ExploitGuardAcgAudited or ExploitGuardAcgEnforced|Arbitrary code guard (ACG) **detected** or **blocked** an attempt to modify code page permissions or create unsigned code pages.|Arbitrary code guard (ACG)|
|ExploitGuardChildProcessAudited or ExploitGuardChildProcessBlocked|Exploit protection **detected** or **blocked** the creation of a child process|Don’t allow child processes|
|ExploitGuardEafViolationAudited or ExploitGuardEafViolationBlocked|Export address filtering (EAF) blocked possible exploitation activity.|Export address filtering (EAF)|
|ExploitGuardIafViolationAudited or ExploitGuardIafViolationBlocked|Import address filtering (IAF) **detected** or **blocked** possible exploitation activity.|Import address filtering (IAF)|
|ExploitGuardLowIntegrityImageAudited or ExploitGuardLowIntegrityImageBlocked|Exploit protection **detected** or **blocked** the launch of a process from a low-integrity file.|Block low integrity images|
|ExploitGuardNonMicrosoftSignedAudited or ExploitGuardNonMicrosoftSignedBlocked|Exploit protection **detected** or **blocked** the launch of a process from an image file that is not signed by Microsoft.|
|ExploitGuardRopExploitAudited or ExploitGuardRopExploitBlocked|Exploit protection blocked possible return-object programming (ROP) exploitation.|
|ExploitGuardSharedBinaryAudited or ExploitGuardSharedBinaryBlocked|Exploit protection detected or blocked the launch of a process from a file in a remote shared file.|
|ExploitGuardWin32SystemCallAudited or ExploitGuardWin32SystemCallBlocked|Exploit protection **detected** or **blocked** a call to the Windows system AIP|
|||
|||
|||
|||
|||
|||
|||
|||
|||
|||
|||
|||
|||
