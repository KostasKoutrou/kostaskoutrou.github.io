# What is Microsoft Defender for Endpoint, for the Endpoint

## Introduction

---

## Contents

* TOC
{:toc}

## Capabilities, visualized


---

## MDE Threat Protection Stack

The MDE Threat Protection Stack is essentially all the components that contibute to protecting the endpoints.

### 1 Attack Surface Reduction (ASR)

ASR is the first line of defense in the world of MDE. ASR, as its name suggests, applies a series of security configurations and protections which aim at minimizing the potential entry points for attackers. This is done by blocking different series of actions which are known to be frequently used by attacks, but rarely used for legitimate purposes.

#### 1.1 Attack Surface Reduction Rules

ASR Rules is one of the protection measures included under ASR. These protect against common malware behaviors and risk application actions. The ASR Rules can be separeted in a few different categories based on what protection they provide, and they are the following:

|Polymorphic threats|Lateral movement & credential theft|Productivity apps rules|Email rules|Script rules|Misc rules|
|:-:|:-:|:-:|:-:|:-:|:-:|
|Block executable files from running unless they meet a prevalence (1,000 machines), age, or trusted list criteria|Block process creations originating from PSExec and WMI commands|Block Office apps from creating executable content|Block executable content from email client and webmail|Block execution of potentially obfuscated scripts|Block abuse of exploited vulnerable signed drivers|
|Block untrusted and unsigned processes that run from USB|Block credential stealing from the Windows local security authority subsystem (lsass.exe)|Block Office apps from creating child processes|Block only Office communication applications from creating child processes|Block JS/VBS from launching downloaded executable content|Block webshell creation for servers|
|Use advanced protection against ransomware|Block persistence through WMI event subscription|Block Office apps from injecting code into other processes|Block Office communication apps from creating child processes| | |
|Block use of copied of impersonated system tools| Block rebooting machine in Safe Mode|Block Adobe Reader from creating child processes| | | |
|||Block Win32 API calls from office macros||||

#### 1.2 Controlled Folder Access (CFA)

CFA is a measure that mostly protects against attacks related to ransomware.
1. With CFA essentially you define a set of directories (folders) and a set of applications. Only that set of apps is allowed to process in any way the set of directories.
2. There is already a default set of directories and processes in CFA which can be used immediately.
3. Apps are added to the list automatically when they are highly prevalent in the organization and haven't displayed any behavior deemed malicious.
4. In the default folder list, the folders Documents, Pictures, Videos, Videos, Music, and Favorites are included.

#### 1.3 Device Control

Device Control enables controls related to usage and installation of peripheral (USB/Bluetooth) or other devices with endpoints. Common scenarios include:
- Control access to USB devices
  - Configure device installation restrictions
  - Control access to removable media
- BitLocker control: Block usage of removable media if BitLocker is disabled
- Control access to printers
- Bluetooth services control: Allowing advertising, discovery, preparing prompting

#### 1.4 Exploit Protection

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

#### 1.5 Web and Network Protection

##### 1.5.1 Web Protection

Web Protection in MDE includes the following capabilities:
1. Custom Indicators: When defined in MDE portal, with the Web Protection feature enabled, the endpoints are blocked when accessing the Blocked Indicators (URLs/IPs).
2. Web Threat Protection: It stops access to phishing, malicious, untrusted, or low-reputation sites.
3. Web Content Filtering (WCF): This provides the ability to block access of websites based on categories (e.g., gambling, torrenting, cloud storage, etc.).

##### 1.5.2 Network Protection

Network Protection:
1. Protects devices by preventing connections to malicious or suspicious sites.
2. Expands MS Defender SmartScreen to block all outbound HTTP(S) traffic to poor-reputation destinations.
3. Extends Web Protection, which works only on Microsoft Edge browser, to the OS level.
4. Is a core component to Web Content Filtering.
5. Provides visibility and blocking of IoCs when used with EDR. Indicators defined in MDE portal are blocked only if Network Protection is enabled.

### 2 Next-Generation Protection

Under Next-Gen Protection is where most of the more "advanced" protection mechanisms reside to block emerging threats.

#### 2.1 Microsoft Defender Antivirus (MDAV)

Here is where the famous MDAV lives. With it, process creation events and file download events from the internet are monitored. It not only uses its signature-based engine, but also predictive technologies such as machine learning and cloud-delivered protection to find attacks. If working offline, the latest dynamic intelligence from the Intelligence Security Graph is provision regularyl throughout the day.

#### 2.2 Cloud Portection and MDAV

To identify new threats dynamically, Next-Gen Protection technologies work with
1. AI systems which are using machine learning models
2. Large sets of interconnected data in the Microsoft Intelligent Security Graph.

MDAV works with Microsoft Cloud services, also known as Microsoft Advanced Protection Service (MAPS). With these, next-gen technologies provide quick identification of new threats. This is done by MDAV uploading samples of metadata or the samples themselves to allow Cloud protection to identify of the samples are malicious.

#### 2.3 Tamper Protection

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

#### 2.4 Block at first sight (BAFS)

BAFS is a simple feature in MDE. When enabled, if a file or executable is never seen before by MDE and it is identified as suspicious, the opening action or the execution of it is blocked until a verdict is received from the cloud-delivered protection.

#### 2.5 Anti-Malware Scan Interface (AMSI) integration with MDAV

AMSI provides the capability to inspect PowerShell and other scripts, even if there is obfuscation applied on them. MDE utilizes AMSI to protect against fileless malware, dynamic script-based attacks, and other threats of this nature.

#### 2.6 MDAV Components and Technologies

MDAV utilizes multiples engines to be able to detect and prevent a wide range of threats and attacker techniques. The following diagram shows the different engines used:

<img alt="image" src="https://github.com/user-attachments/assets/05ee9ded-66da-4284-b3e6-07a945c83178" />
<!--
<img width="975" height="401" alt="image" src="https://github.com/user-attachments/assets/05ee9ded-66da-4284-b3e6-07a945c83178" />
-->
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

#### 2.7 Potentially Unwanted Apps (PUA)

PUA is a category of software that can:

- Cause the machine to run slowly
- Display unexpected apps
- Install other software that might be unwanted

PUA could be:

- Advertising software
- Bundled software that offers to install additional software not related to the original one, and could be not signed by the same entity as the original one
- Software that evades detection

PUA can:

- Increase the risk of the endpoints being infected with actual malware
- Make malware harder to identify
- Cost time and effort to clean up

Both MDAV and SmartScreen provide PUA protection.

#### 2.8 Scheduled Scans

There are three types of scans in MDAV:

- Quick Scan: Scan the locations with high probability of having malware registered to start with the system, including registry keys and known Windows startup folders.
- Full Scan: Start with a Quick Scan, and then scan all mounted fixed disks and removable and network drives. A Full Scan could take a few hours or days to complete.
- Custom Scan: Scan what is provided in input.

In most cases, the recommended approach is to execute only one Full Scan per endpoint, and then only run Quick Scans, because them together with Real-Time Protection being enabled covers all files in an endpoint.

#### 2.9 MDAV Modes

MDAV could be operating in different modes, depending on whether there is another AV solution installed on the endpoint, or based on the policies and security settings applied on the endpoint:

- Active Mode:
 1. MDAV is the main AV solution on the machine.
 2. Files are scanned, and actions are taken, i.e., threats are actively remediated.
- Passive Mode:
 1. MDAV is _not_ the main AV solution on the machine.
 2. While files are still scanned, actions are not taken, i.e., threats are not remediated.
 3. Updates should still be applied, as they improve alerting and performance(!).
- Endpoint Detection and Response (EDR) in Block Mode:
 1. This is an enhancement to Passive Mode, where actions are taken in malicious artifacts detected by EDR capabilites. Actions are taken on post-breach, behavioral EDR detections, which were not detected by the installed AV solution.
 2. Several features are still not available (Active Mode is required):
  - Real-Time Protection
  - Network Protection
  - ASR
  - Indicators
- Disabled or Uninstalled:
 1. MDAV is _not_ the main AV solution on the machine.
 2. Files are _not_ scanned and actions are _not_ taken.
 3. It is recommended to have MDAV in Passive Mode for additional alerting, and if the main AV gets removed for any reason (licensing, uninstall, etc.), then MDAV will switch to Active Mode automatically, to keep protecting the machine.

#### 2.10 Behavioral blocking and containment

This is a feature based on AI/ML to target fileless malware, polymorphic threats, and human-operated attacks. It detects attacks based on their behaviors and process trees. This feature utilizes the following:

- Next-Gen Protection detects threats by analyzing behaviors.
- EDR received signals across your network, devices, and kernel behavior, creating alerts and incidents.
- MDE has visibility in email, data, apps, network, endpoint, and kernel behavior signals received through EDR. MDE correlates these signals, and raises alerts and incidents.

Components of Behavioral blocking and containment:

- ASR Rules prevent predefines common attack behaviors.
- Client behavioral blocking: As suspicious behaviors are detected on devices by MDAV, artifacts such as files or apps are blocked, checked, and remediated automatically. When suspicious behaviors are detected, MDAV sends them and their process tress to the cloud protection service to determine their maliciousness. If detected, it is blocked on the device.
- Feedback-loop blocking, AKA Rapid Protection: With it, when a suspicious behavior or file is detection, information about it is sent to multiple classifiers. The rapid protection loop engine inspects and correlated the info with other signals to identify if to block the file. This results in blocking malware on a device, other devices in the org, and devices in other orgs. So there is this community approach where you will get threat intel from other orgs based on detections.

#### 2.11 UEFI scanning in MDE

MDE now has a UEFI scanner. This helps in attempts of attackers compromosing the boot flow to achieve low-level malware behavior that is hard to detect.

Windows Defender System Guard combats this with hardware-based security features including hypervisor-level attestation and Secure Launch (aka Dynamic Root of Trust (DRTM)).

With the new UEFI scanner, firmware scanning becomes broadly available. It is built-in with MDAV. It gives MDE the ability to scan inside the firmware file system and perform security assessments. It performs dynamic analysis on the firmware it gets from the hardware flash storage. By obtaining the firmware, the scanner is able to parse the firmware, enabling MDE to inspect firmware content at runtime.

#### .12 Early Launch Antimalware (ELAM) and MDAV

ELAM combats early boot threats (e.g., rootkits, or malicious drivers that can hide from detection) by using a driver named Wdboot.sys that starts before other boot-start drivers. ELAM enabled the evaluation of other drivers, and helps the Windows kernel decide whether those drivers should be initialized.
