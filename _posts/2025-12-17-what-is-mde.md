# What is Microsoft Defender for Endpoint

Introduction

---

## Contents

## MDE Threat Protection Stack

The MDE Threat Protection Stack is essentially all the components that contibute to protecting the endpoints.

### Core Defender Vulnerability Management

With Core Defender Vulnerability Management, the purpose is to identify vulnerabilities and misconfigurations in the endpoints of the tenant. With this information, the following components are enhanced:
- Endpoint Detection and Response (EDR) insights are provided by correlating with endpoint vulnerabilities.
- Device vulnerability context is added during investigations.
- A remediation process is facilitated via Intune or Configuration Manager.

### Attack Surface Reduction

Attack Surface Reduction (ASR) is the first line of defense in the world of MDE. ASR, as its name suggests, applies a series of security configurations and protections which aim at minimizing the potential entry points for attackers. This is done by blocking different series of actions which are known to be frequently used by attacks, but rarely used for legitimate purposes.

#### Attack Surface Reduction Rules
ASR Rules is one of the protection measures included under ASR. These protect against common malware behaviors and risk application actions. The ASR Rules can be separeted in a few different categories based on what protection they provide, and they are the following:
|Polymorphic threats|Lateral movement & credential theft|Productivity apps rules|Email rules|Script rules|Misc rules|
|-|-|-|-|-|-|
|Block executable files from running unless they meet a prevalence (1,000 machines), age, or trusted list criteria|Block process creations originating from PSExec and WMI commands|Block Office apps from creating executable content|Block executable content from email client and webmail|Block execution of potentially obfuscated scripts|Block abuse of exploited vulnerable signed drivers|
|Block untrusted and unsigned processes that run from USB|Block credential stealing from the Windows local security authority subsystem (lsass.exe)[2]|Block Office apps from creating child processes|Block only Office communication applications from creating child processes|Block JS/VBS from launching downloaded executable content|Block webshell creation for servers|
|Use advanced protection against ransomware|Block persistence through WMI event subscription|Block Office apps from injecting code into other processes|Block Office communication apps from creating child processes| | |
|Block use of copied of impersonated system tools| Block rebooting machine in Safe Mode|Block Adobe Reader from creating child processes| | | |
|||Block Win32 API calls from office macros||||

