# Using KQL to identify detections from MDE

## Introduction

### ASR Rules Detections

```kql
DeviceEvents
| where ActionType startswith 'Asr'
```
