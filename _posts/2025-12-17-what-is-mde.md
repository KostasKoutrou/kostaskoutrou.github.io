# What is Microsoft Defender for Endpoint

Introduction

---

## Contents

## Graph of all measures

## MDE Threat Protection Stack

The MDE Threat Protection Stack is essentially all the components that contibute to protecting the endpoints.

### Core Defender Vulnerability Management

With Core Defender Vulnerability Management, the purpose is to identify vulnerabilities and misconfigurations in the endpoints of the tenant. With this information, the following components are enhanced:
- Endpoint Detection and Response (EDR) insights are provided by correlating with endpoint vulnerabilities.
- Device vulnerability context is added during investigations.
- A remediation process is facilitated via Intune or Configuration Manager.

### Attack Surface Reduction (ASR)

ASR is the first line of defense in the world of MDE. ASR, as its name suggests, applies a series of security configurations and protections which aim at minimizing the potential entry points for attackers. This is done by blocking different series of actions which are known to be frequently used by attacks, but rarely used for legitimate purposes.

#### .1 Attack Surface Reduction Rules

ASR Rules is one of the protection measures included under ASR. These protect against common malware behaviors and risk application actions. The ASR Rules can be separeted in a few different categories based on what protection they provide, and they are the following:

|Polymorphic threats|Lateral movement & credential theft|Productivity apps rules|Email rules|Script rules|Misc rules|
|-|-|-|-|-|-|
|Block executable files from running unless they meet a prevalence (1,000 machines), age, or trusted list criteria|Block process creations originating from PSExec and WMI commands|Block Office apps from creating executable content|Block executable content from email client and webmail|Block execution of potentially obfuscated scripts|Block abuse of exploited vulnerable signed drivers|
|Block untrusted and unsigned processes that run from USB|Block credential stealing from the Windows local security authority subsystem (lsass.exe)|Block Office apps from creating child processes|Block only Office communication applications from creating child processes|Block JS/VBS from launching downloaded executable content|Block webshell creation for servers|
|Use advanced protection against ransomware|Block persistence through WMI event subscription|Block Office apps from injecting code into other processes|Block Office communication apps from creating child processes| | |
|Block use of copied of impersonated system tools| Block rebooting machine in Safe Mode|Block Adobe Reader from creating child processes| | | |
|||Block Win32 API calls from office macros||||

#### .2 Controlled Folder Access (CFA)

CFA is a measure that mostly protects against attacks related to ransomware.
1. With CFA essentially you define a set of directories (folders) and a set of applications. Only that set of apps is allowed to process in any way the set of directories.
2. There is already a default set of directories and processes in CFA which can be used immediately.
3. Apps are added to the list automatically when they are highly prevalent in the organization and haven't displayed any behavior deemed malicious.
4. In the default folder list, the folders Documents, Pictures, Videos, Videos, Music, and Favorites are included.

#### .3 Device Control

Device Control enables controls related to usage and installation of peripheral (USB/Bluetooth) or other devices with endpoints. Common scenarios include:
- Control access to USB devices
  - Configure device installation restrictions
  - Control access to removable media
- BitLocker control: Block usage of removable media if BitLocker is disabled
- Control access to printers
- Bluetooth services control: Allowing advertising, discovery, preparing prompting

#### .4 Exploit Protection

Exploit Protection helps protect against malware that uses exploits to infect devices and spread. It consist of many mitigations that can be applied to either the whole OS or individual apps. It protects by default some behaviors which are correlated to exploits, like an app accessing parts of memory which it should be able to. The protection measures vary from simple which can be enabled OS-wide and in audit mode to test, all the way to complex protection which protect only per-application and cannot be turned on audit mode but only be turned on immediately.
Exploit Protection provides the following protection measures:
1. Control flow guard (CFG)
1. Data Execution Prevention (DEP)
1. Force randomization for images (Mandatory ASLR)
1. Randomize memory allocations (Bottom-Up ASLR)
1. Validate exception chains (SEHOP)
1. Validate heap integrity
1. Arbitrary code guard (ACG)
1. Block low integrity images
1. Block remote images
1. Block untrusted fonts
1. Code integrity guard
1. Disable extension points
1. Disable Win32k system calls
1. Don't allow child processes
1. Export address filtering (EAF)
1. Import address filtering (IAF)
1. Simulate execution (SimExec)
1. Validate API invocation (CallerCheck)
1. Validate handle usage
1. Validate image dependency integrity
1. Validate stack integrity (StackPivot)
1. Hardware-enforced stack protection

#### .5 Web and Network Protection

##### .5.1 Web Protection

Web Protection in MDE includes the following capabilities:
1. Custom Indicators: When defined in MDE portal, with the Web Protection feature enabled, the endpoints are blocked when accessing the Blocked Indicators (URLs/IPs).
2. Web Threat Protection: It stops access to phishing, malicious, untrusted, or low-reputation sites.
3. Web Content Filtering (WCF): This provides the ability to block access of websites based on categories (e.g., gambling, torrenting, cloud storage, etc.).

##### .5.2 Network Protection

Network Protection:
1. Protects devices by preventing connections to malicious or suspicious sites.
2. Expands MS Defender SmartScreen to block all outbound HTTP(S) traffic to poor-reputation destinations.
3. Extends Web Protection, which works only on Microsoft Edge browser, to the OS level.
4. Is a core component to Web Content Filtering.
5. Provides visibility and blocking of IoCs when used with EDR. Indicators defined in MDE portal are blocked only if Network Protection is enabled.

### Next-Generation Protection

Under Next-Gen Protection is where most of the more "advanced" protection mechanisms reside to block emerging threats.

#### .1 Microsoft Defender Antivirus (MDAV)

Here is where the famous MDAV lives. With it, process creation events and file download events from the internet are monitored. It not only uses its signature-based engine, but also predictive technologies such as machine learning and cloud-delivered protection to find attacks. If working offline, the latest dynamic intelligence from the Intelligence Security Graph is provision regularyl throughout the day.

#### .2 Cloud Portection and MDAV

To identify new threats dynamically, Next-Gen Protection technologies work with
1. AI systems which are using machine learning models
2. Large sets of interconnected data in the Microsoft Intelligent Security Graph.

MDAV works with Microsoft Cloud services, also known as Microsoft Advanced Protection Service (MAPS). 
