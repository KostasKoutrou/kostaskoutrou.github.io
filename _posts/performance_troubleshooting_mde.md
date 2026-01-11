# Title

## Introduction

## Contents


## Some common reasons for higher CPU by MDAV:

**Binaries not signed**: When a binary (exe, dll, etc.) that is not digitally signed is launched, MDAV will start a Real-Time Protection Scan.

Generally, properly identifying such cases is difficult. If you know a robust method, please let me know. You can use the KQL table `DeviceFileCertificateInfo` to identify certificate-related issues:

```kql
DeviceFileCertificateInfo
| where TimeGenerated > ago(30d)
| where IsTrusted == 0
```

You can also run the following KQL query to potentially identify running of such cases.

```kql
DeviceProcessEvents
| where TimeGenerated > ago(1d)
| where SHA1 != ""
| where FolderPath contains ":"
| project TimeGenerated, DeviceName, DeviceId, SHA1, FileName, FolderPath, ProcessCommandLine, InitiatingProcessFileName
| join kind=leftouter (
DeviceFileCertificateInfo
| where TimeGenerated > ago(30d) //putting 30 days because these events are not generated for every executation of a binary.
| project SHA1, IsSigned, IsTrusted, Signer, Issuer, DeviceId
) on SHA1 and DeviceId
| project TimeGenerated, DeviceName, DeviceId, SHA1, FileName, FolderPath, ProcessCommandLine, IsSigned, IsTrusted, Signer, Issuer, InitiatingProcessFileName
| where isnull(IsSigned) or IsTrusted == 0 or IsSigned == 0
| summarize count() by DeviceName, DeviceId, SHA1, FileName, Signer, Issuer, FolderPath, IsSigned, IsTrusted
| sort by FileName asc, DeviceName asc
```

Then you will have to manual click through the results of the detected files to see if they are actually signed, and what Defender mentions about their signature information.

## MPLog file parsing for performance impact
