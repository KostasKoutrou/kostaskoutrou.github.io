# Using KQL to identify detections from MDE

## Introduction

### ASR Rules Detections

Each ASR Rule has its own GUID. This will need to be used when configuring, e.g., a GPO to enable ASR rules in Audit or Block mode for machines. The ASR Rule to GUID matrix can be found in Microsoft's [Attack surface reduction rules reference](https://learn.microsoft.com/en-us/defender-endpoint/attack-surface-reduction-rules-reference#asr-rule-to-guid-matrix)

The following KQL query parses the AdditionalFields column in order to extract the ASR Rule GUID.

```kql
DeviceEvents
| where ActionType startswith 'Asr'
| extend RuleId=extractjson("$Ruleid", AdditionalFields, typeof(string))
```

