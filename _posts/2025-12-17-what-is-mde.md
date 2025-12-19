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

MDAV works with Microsoft Cloud services, also known as Microsoft Advanced Protection Service (MAPS). With these, next-gen technologies provide quick identification of new threats. This is done by MDAV uploading samples of metadata or the samples themselves to allow Cloud protection to identify of the samples are malicious.

#### .3 Tamper Protection

When Tamper Protection is enabled on a machine, the following MDAV setting cannot be changed by anyone, not even by applying a new GPO or a setting in Intune:

- Virus and threat protection remains enabled.
- Real-time protection remains turned on.
- Behavior monitoring remains turned on.
- Antivirus protection, including IOfficeAntivirus (IOAV) remains enabled.
- Cloud protection remains enabled.
- Security intelligence updates occur.
- Automatic actions are taken on detected threats.
- Notifications are visible in the Windows Security app on Windows devices.
- Archived files are scanned.
- Exclusions can't be modified or added

#### .4 Block at first sight (BAFS)

BAFS is a simple feature in MDE. When enabled, if a file or executable is never seen before by MDE and it is identified as suspicious, the opening action or the execution of it is blocked until a verdict is received from the cloud-delivered protection.

#### .5 Anti-Malware Scan Interface (AMSI) integration with MDAV

AMSI provides the capability to inspect PowerShell and other scripts, even if there is obfuscation applied on them. MDE utilizes AMSI to protect against fileless malware, dynamic script-based attacks, and other threats of this nature.

#### .6 MDAV Components and Technologies

MDAV utilizes multiples engines to be able to detect and prevent a wide range of threats and attacker techniques. The following diagram shows the different engines used:

<img width="975" height="401" alt="image" src="https://github.com/user-attachments/assets/05ee9ded-66da-4284-b3e6-07a945c83178" />

As shown above, there is a balance between engines running locally on the client which provide real time protection, and engines being provided via cloud-delivered protection which handle the heavy load when needed. With this graph the necessity of the cloud-delivered protection is also highlighted, as a machine gets a plethora of additional capabilities when it is enabled.

In the following table, the engines are described briefly:

|On the Client|In the Cloud|
|-|-|
|**ML engine**: Lightweight ML engine. It has specialized models for specific file types commonly abused, like PE files, PowerShell, macros, JS, PDF, etc.|**Metadata-based ML engine**: Specialized ML models analyze a featurized description of suspicious files sent by the client. Stacked ensemble classifiers combine results to make a verdict. You can read more on how it works here: https://www.microsoft.com/security/blog/2019/07/25/new-machine-learning-model-sifts-through-the-good-to-unearth-the-bad-in-evasive-malware/|
|**Behavior monitoring engine**: Monitors for potential attacks post-execution. It observes process behaviors, including behavior sequence at runtime, to identify and block activities based on predetermined rules.|**Behavior-based ML engine**: Suspicious behavior sequences are used to trigger to analyze the process tree behavior using ML models. Monitored attack techniques span the attack chain, like exploits, elevation, persistence, lateral movement, and exfiltration.|
|**Memory scanning engine**: Scans memory space used by a running process to expose malicious behavior that could be hiding with code obfuscation.|**AMSI-paired ML engine**: Client-side and cloud-side pairs analyze scripts behavior pre and post execution to detect threats like fileless and in-memory attacks.|
|**AMSI integration engine**: Enables detection of files and in-memory attacks, defeating code obfuscation. This blocks malicious behavior of scripts client-side.|**File classification ML engine**: Multi-class, deep neural network classifiers examine full file contents. Suspicious files are held from running and submitted to the cloud protection service for classification.|
|**Heuristics engine**: Rules identify file characteristics that have similarities with known malicious characteristics to catch new threats or modifications of known ones.|**Detonation-based ML engine**: Suspicious files are detonated in a sandbox. Deep learning classifiers analyze the observed behavior to block attacks.|
|**Emulation engine**: Dynamically unpacks malware and examine how they would behave at runtime. Checks both during runtime and the memory content after, finding malware packers and polymorphic malware.|**Reputation ML engine**: Reputation sources and models from all Microsoft are queried to block threats linked to malicious/suspicious URLs, domains, emails, and files. Sources include SmartScreen, MDO, and others through the Microsoft Intelligent Security Graph.|
|**Network engine**: Network activity is inspected.|**Smart rules engine**: Smart rules identify threats based on researcher expertise and collective knowledge of threats.|
|**CommandLine Scanning engine**: Scans command lines of all processes before they execute.|**CommandLine ML engine**: ML models scan suspicious command lines in the cloud.|

