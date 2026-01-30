# MDE Performance Troubleshooting - Getting to know MDE Part 3

## Introduction

A frequently reported issue in MDE is its performance impact on machines. Many engineers have seen the task manager of a machine looking like the following, and are asked to fix it.

<img alt="image" src="https://github.com/user-attachments/assets/4e3f30e4-2b9c-4731-ab5b-989b04eb2b49" />



Welcome to the 3rd part of the blog post series “Getting to know MDE”!

In this series, each post contains a different component/feature/methodology when it comes to understanding and managing Microsoft Defender for Endpoint (MDE).

> In this post, the focus is on troubleshooting issues that have a performance impact on machines caused by Defender processes.

All the information is based on official documentation of Microsoft, which you can find [here](https://learn.microsoft.com/en-us/defender-endpoint/).

So let's get started.

---

## Contents

* TOC
{:toc}

---

## Common reasons for higher CPU by MDAV

There may be several reasons which cause MDAV to utilize a higher percentage of CPU power. In this section, a few common reasons are briefly described.

### Binaries not signed

When a binary (exe, dll, etc.) that is not digitally signed is launched, MDAV will start a Real-Time Protection Scan.

Generally, properly identifying such cases is difficult. If you know a robust method, please let me know. The KQL table `DeviceFileCertificateInfo` can be used to identify certificate-related issues:

```kql
DeviceFileCertificateInfo
| where TimeGenerated > ago(30d)
| where IsTrusted == 0
```

The following KQL query can also be run to potentially identify cases of unsigned binaries being executed.

```kql
DeviceProcessEvents
| where TimeGenerated > ago(1d)
| where SHA1 != ""
| where FolderPath contains ":"
| project TimeGenerated, DeviceName, DeviceId, SHA1, FileName, FolderPath, ProcessCommandLine, InitiatingProcessFileName
| join kind=leftouter (
DeviceFileCertificateInfo
| where TimeGenerated > ago(30d) //putting 30 days because these events are not generated for every execution of a binary.
| project SHA1, IsSigned, IsTrusted, Signer, Issuer, DeviceId
) on SHA1 and DeviceId
| project TimeGenerated, DeviceName, DeviceId, SHA1, FileName, FolderPath, ProcessCommandLine, IsSigned, IsTrusted, Signer, Issuer, InitiatingProcessFileName
| where isnull(IsSigned) or IsTrusted == 0 or IsSigned == 0
| summarize count() by DeviceName, DeviceId, SHA1, FileName, Signer, Issuer, FolderPath, IsSigned, IsTrusted
| sort by FileName asc, DeviceName asc
```

The results of the detected files should then be filtered to see if they are actually signed, and what Defender mentions about their signature information. This can be done by clicking on the hashes in the resulting table, and checking the signature information:

<img alt="image" src="https://github.com/user-attachments/assets/e6b3f143-cade-4e43-aed4-d3b1449bd3d0" />

If the binaries have certificates signed by CAs not trusted by Microsoft or need to be excluded for any reason, they can be added to the Certificates Indicators allow list.

<img alt="image" src="https://github.com/user-attachments/assets/bb9da54c-ddbb-4e71-8ef4-6d311a207956" />

### Obfuscated scripts

Obfuscated scripts require much more CPU to be scanned, so obfuscation should be used only if necessary.

### Not letting MDAV cache finish before sealing a VDI image

If creating a VDI image for a persistent or non-persistent image, ensure that the cache maintenance completes before sealing the image. To do this:

1. Open up the Task Scheduler mmc (taskschd.msc).
2. Expand Task Scheduler Library > Microsoft > Windows > Windows Defender, and then right-click on Windows Defender Cache Maintenance.
3. Select Run, and let the scheduled task finish.

### Misspelled exclusions

Double check that the exclusions are spelled correctly.

### Path exclusions only work for scanning flows

Behavior Monitoring and Network Real-time Inspection can still cause performance issues. As a workaround, one of the following can be done:

- Add the exe or dll to the Indicators file hash allow list
- Add the certificate to the Indicators certificates allow list
- Add MDAV exclusions for the process, too.

### File hash computation

If the File Hash computation feature is enabled, file hashes for every executable file that is scanned is computed, if it wasn’t previously computed. This has a performance cost especially when copying large files from a network share. Keep in mind that this feature is a prerequisite for File Hash Indicators in Defender.

<img alt="image" src="https://github.com/user-attachments/assets/9628f6f6-4fcf-4d07-9b10-c1adf7119ec3" />

If it is decided that it is not needed, it can be disabled via PowerShell or any management solution used in the organization (Intune, AD GPO, Configuration Management, etc.). To disable it with PowerShell run the following cmdlet:

`Set-MpPreference -EnableFileHashComputation $false`

### Scheduled scanning

When it comes to scheduled scanning, the following scan settings should be checked, depending on the way they are pushed to machines:

- Configure low CPU priority for scheduled scans. This lowers the scheduled scan thread priority from 9 to 8.
- Specify CPU usage limit per scan from 50 to 20 or 30.
- Start scheduled scan when the device is idle with `ScanOnlyIfIdle`.
- Specify the interval to run quick scans per day to `Not configured`.
- Specify the time for a daily quick scan to a time when the machine is least used.
- Consider disabling the Scheduled Scan settings. The Daily Quick Scan is enough:
    - Specify the scan type to use for a scheduled scan to `Not configured`.
    - Specify the time of day to run scheduled scan to `Not configured`.
    - Specify the day of the week to `Not configured`.

More information about how to configure the scheduled scan settings can be found in [Microsoft's do**cumenta**tion](https://learn.microsoft.com/en-us/defender-endpoint/configure-advanced-scan-types-microsoft-defender-antivirus).

### Scan after security intelligence updates

By default, a scan happens after MDAV receives its latest security intelligence updates. If scheduled scans are enabled, maybe this is not needed and can be disabled.

To disable it in Group Policy (or another management tool, such as MDM), go to Computer Configuration > Administrative Templates > Microsoft Defender Antivirus > Security Intelligence Updates, and set Turn on scan after security intelligence update to `Disabled`.

### Conflicts with other security software

If other security software, like AV, EDR, and DLP, are used, the proper exclusions need to be defined on both MDE and the other software's side.

More info about using third party security software with MDE can be found in [Microsoft Defender Antivirus compatibility with other security products](https://learn.microsoft.com/en-us/defender-endpoint/microsoft-defender-antivirus-compatibility) and exclusions that need to be added can be found in Microsoft's [MDE Exclusion List](https://learn.microsoft.com/en-us/defender-endpoint/switch-to-mde-phase-2?tabs=Windows#step-2-add-microsoft-defender-for-endpoint-to-the-exclusion-list-for-your-existing-solution)

## Troubleshooting Mode

An easy way to investigate if Defender could be a reason for performance issues is to disable Real-Time Protection. If Tamper Protection is on, then Troubleshooting mode can be used on the machine to allow Tamper Protection to be disabled.

After Troubleshooting Mode is turned on, the following cmdlets can be run to disable Real-Time Protection:

1. Disable Tamper Protection: `Set-MPPreference -DisableTamperProtection $true`
2. Confirm that Tamper Protection is disabled: `Get-MpPreference | fl DisableTamperProtection` - The output should have a value of "True".
3. Disable Real-Time Protection: `Set-MPPreference -DisableRealtimeMonitoring $true`
4. Confirm that Real-Time Protection is disabled: `Get-MpPreference | fl DisableRealtimeMonitoring` - The output should have a value of "True".

Afterwards, a check should be done on the machine again to see if the issue is resolved. If it is, then Defender Antivirus should be investigated further, by following the steps described in the next sections. Otherwise, there could be another reason for the CPU usage of Defender, and probably a ticket to Microsoft should be opened for further investigation.

More info on PowerShell cmdlet for configuring Defender settings can be found [here](https://learn.microsoft.com/en-us/powershell/module/defender/set-mppreference?view=windowsserver2025-ps).

More info on Troubleshooting Mode can be found [here](https://learn.microsoft.com/en-us/defender-endpoint/enable-troubleshooting-mode).

In the following link, additional scenarios where Troubleshooting Mode may help are described [Troubleshooting mode scenarios](https://learn.microsoft.com/en-us/defender-endpoint/troubleshooting-mode-scenarios).

## Performance Analyzer

Performance Analyzer is a tool that helps in determining which files, file extensions, and processes may be causing performance issues in machines during Defender Antivirus scans. This information can be used as input towards potentially defining Defender exclusions.

Running the Performance Analyzer is straight-forward, by running the following cmdlet as an admin:

`New-MpPerformanceRecording -RecordTo <recording.etl>`

The `RecordTo` parameter defines the file where the Performance Analyzer recording is written.

The following is an example of the output of running that cmdlet:

<img alt="image" src="https://github.com/user-attachments/assets/c843946d-e3ce-4175-a27b-8d109881f483" />

The idea is to "turn on" Performance Analyzer, and while it is running and monitoring, the problematic scenario is reproduced. If it is a general issue which cannot be reproduced on demand, but happens sporadically, the only way to analyze that is by leaving Performance Analyzer running and hoping that the problematic behavior reappears.

After capturing the problematic behavior, the recording can be stopped by pressing `Enter`, which results in the following output:

<img alt="image" src="https://github.com/user-attachments/assets/1b914672-b7be-4e9c-8b72-02c369963feb" />

The next step is to parse the Performance Analyzer's report. To do this the cmdlet `Get-MpPerformanceReport` is used. This cmdlet has a lot of different available parameters, which can be viewed at Microsoft's [Performance Analyzer Reference](https://learn.microsoft.com/en-us/defender-endpoint/performance-analyzer-reference).

Below, a series of cmdlets which can help pinpoint the issue are depicted:

To view the overview of the recording:

`Get-MpPerformanceReport -Path .\recording.etl -Overview`

<img alt="image" src="https://github.com/user-attachments/assets/ecaebaf3-2885-4a86-837c-54b80b1f7e09" />

To view the top 20 scans, paths, extensions, and processes:

`Get-MpPerformanceReport -Path .\recording.etl -TopScans 20 -TopPaths 20 -TopExtensions 20 -TopProcesses 20`

The screenshot is cut short because the output is too long:

<img alt="image" src="https://github.com/user-attachments/assets/996b5778-0df9-4519-98c3-836e5a8cb373" />

After running the above command, if a specific category of top values or a specific value per-se is of interest, it is possible to then dive a bit deeper. For example, if a specific extension seems to be causing a lot of scans, to dive deeper the following cmdlet can be run:

`Get-MpPerformanceReport -Path .\recording.etl -TopExtensions 20 -TopScansPerExtension 5 -TopPathsPerExtension 5 -TopScansPerPathPerExtension 5 -TopProcessesPerExtension 5 -TopScansPerProcessPerExtension 5 -TopScansPerFilePerExtension 5 -TopFilesPerExtension 5`

This cmdlet uses all the parameters that end with "PerExtension", and will produce a much longer report focused on the top extensions, and for each top extension, its top scans, paths, processes and files. This can help in pinpointing what is causing these scans, which may lead to either resolving the source of the issue or defining exclusions in MDE.

<img alt="image" src="https://github.com/user-attachments/assets/cb938f45-dc6a-409d-b87c-f8480b080082" />

The report can get lengthy. It is possible to make the output machine readable with the `-Raw` parameter, and then possibly convert it to exportable formats, like JSON, which can then be exported, saved, and analyzed:

`Get-MpPerformanceReport -Path .\recording.etl -TopExtensions 20 -TopScansPerExtension 5 -TopPathsPerExtension 5 -TopScansPerPathPerExtension 5 -TopProcessesPerExtension 5 -TopScansPerProcessPerExtension 5 -TopScansPerFilePerExtension 5 -TopFilesPerExtension 5 -Raw | ConvertTo-Json`

<img alt="image" src="https://github.com/user-attachments/assets/e780314d-7a8d-477f-a74e-f717087d4fd5" />

### Run Performance Analyzer via Live Response

It is also possible to run Performance Analyzer via Live Response, by creating a custom script and uploading it to the Live Response library.

A prerequisite is to allow unsigned scripts to run it, by going to the MDE Portal > System > Settings > Endpoints > Live Response unsigned script execution

<img alt="image" src="https://github.com/user-attachments/assets/f61f7ff8-c91d-4813-a938-67bb9652a91e" />

The following script will need to be saved in a file. For this post, the file will be named "LivePerfAnalyser.ps1"

```powershell
param (
    [Parameter(Mandatory=$false)]
    [int]$Seconds = 120
)

$Hostname = $env:COMPUTERNAME
$Datetime = Get-Date -Format "yyyyMMdd_HHmmss"
$OutputPath = "C:\Temp\PerformanceRecording_${Hostname}_${Datetime}.etl"

Write-Host "Initiating recording for $Seconds seconds..."
New-MpPerformanceRecording -Seconds $Seconds -RecordTo $OutputPath

Write-Host "Performance Recording for $Hostname for $Seconds seconds was written at $OutputPath"
```

The steps to run Performance Analyzer via Live Response are:

1. Initiate a Live Response session for the endpoint that needs troubleshooting
2. Upload the script to library by clicking on:

    <img alt="image" src="https://github.com/user-attachments/assets/cec0377f-98b5-4277-84d4-0b33f3cc6d58" />

3. Run the Performance Analyzer with the following command in Live Response: `run LivePerfAnalyzer.ps1 -parameters "-Seconds 120"`. You can change the value of `Seconds` to the time period in seconds that you need to run the analyzer, or skip the parameters completely, with the default value being 120 seconds.
4. Try to reproduce the activity that causes performance issues on the endpoint. Wait for the designated time.
5. After the Performance Analyzer finishes, the following is printed:

    <img alt="image" src="https://github.com/user-attachments/assets/bdffe3dd-437d-422e-b039-3ec73fb4fe35" />

6. Get the file from the path written in the script output: `getfile \<recordingpath\recording.etl\>`
7. After getting the file, the analysis steps described in the previous section can then be followed.

## MPLog file parsing for performance impact

The Microsoft Protection Log (MPLog) file is one of a few files found under the path `C:\ProgramData\Microsoft\Windows Defender\Support`. Under this path, a few files which contain different Defender logs can be found, including:

- MPDetection
- MPDeviceControl
- MPDlpLog
- MPLog
- MPScanSkip
- MpWppCoreTracing
- MpWppTracing

These files are very useful when investigating what Defender and its different detection mechanisms did. When it comes to performance troubleshooting, the `MPLog` file contains a few useful logs, among others, which can assist in detecting processes which experienced high scanning activity, potentially impacting their performance. Credits to the related [article](https://www.crowdstrike.com/en-us/blog/how-to-use-microsoft-protection-logging-for-forensic-investigations/) by CrowdStrike explaining these logs. Remember that there is always the option of getting these files via Live Response and the `getfile` command.

These logs have the following format:

`2025-12-31T15:45:19.964 ProcessImageName: Razer Synapse Service Process.exe, Pid: 9104, TotalTime: 810, Count: 224, MaxTime: 15, MaxTimeFile: \Device\HarddiskVolume3\Program Files (x86)\Razer\Synapse3\UserProcess\Razer Synapse Service Process.exe.config, EstimatedImpact: 53%`

Let's go through this line:

|Field Name|Description|Example|
|-|-|-|
|Date (not stated)|Timestamp of event|2025-12-31T15:45:19.964|
|ProcessImageName|Process Image Name|Razer Synapse Service Process.exe|
|Pid|Process ID of the Process|9104|
|TotalTime|The summary of all time periods spent in scans of files accessed by this process (milliseconds)|810|
|Count|The number of scanned files accessed by this process|224|
|MaxTime|The longest time spent in scanning a single file accessed by this process|15|
|MaxTimeFile|The file which was scanned for `MaxTime` milliseconds|\Device\HarddiskVolume3\Program Files (x86)\Razer\Synapse3\UserProcess\Razer Synapse Service Process.exe.config|
|EstimatedImpact|This value shows the impact that MDAV had on the performance of the above process. It is the percentage of (Total time spent in scans of files accessed by this process)/(Total time which this process experienced scan activity). For example, if you open a large folder in File Explorer, all the files in it are being scanned via Real-Time Protection. The time it takes until the last file is finished being scanned was 10 seconds, but the time that MDAV was actually scanning files was 3 seconds. Here the Estimated Impact is about 3/10 = 30%.|53%|

Therefore, using the above logs and the `EstimatedImpact` value, it is possible to review processes which experience heavy scanning load. Note that this does not mean necessarily that these processes result in higher CPU for the machine, but rather that that specific process experienced a performance hit due to scanning.

The following PowerShell script parses the MPLog file, takes only the logs which include the `EstimatedImpact` values, and exports it all to a CSV.

```powershell
$logfile = Read-Host "Enter log MPLog file"

# Extract relevant log entries, remove whitespaces, and process data
$logs = Get-Content $logfile | `
    Select-String "EstimatedImpact" | ` # Select lines that contain the string "EstimatedImpact"
    Select-String -Pattern '%$' | ` # Select lines that end with "%". Sometimes the logs are not recorded properly and 2 logs are written in the same line, which makes it difficult to parse.
    ForEach-Object {
        # Remove all whitespace from the line
        $_ -replace '\s', ''
    } | `
    ForEach-Object {
        # Add a "Date:" prefix to each log entry
        "Date:" + $_
    } | `
    ForEach-Object {
        # Insert a comma after the 30th character for consistent formatting
        $_.Insert(30, ',')
    }

# Convert the cleaned logs into structured objects for CSV export
$logObjects = $logs | ForEach-Object {
    # Split the cleaned line into fields (key-value pairs separated by ':')
    $fields = $_ -split ',' | ForEach-Object {
        $pair = $_ -split ':'
        @{$pair[0] = $pair[1]}
    }

    # Combine all key-value pairs into a single hash table
    $object = @{}
    foreach ($field in $fields) {
        foreach ($key in $field.Keys) {
            $object[$key] = $field[$key]
        }
    }

    # Remove the '%' from the "EstimatedImpact" field if it exists
    if ($object["EstimatedImpact"]) {
        $object["EstimatedImpact"] = $object["EstimatedImpact"] -replace '%', ''
    }

    # Return the final object as a PSCustomObject
    [PSCustomObject]$object
}

# Export the processed data to a CSV file
$outfile = $logfile -replace '\.log$', '.csv'
$logObjects | Export-Csv -Path $outfile -NoTypeInformation -Encoding UTF8

Write-Host "Process Logs exported to $outfile"
Read-Host -Prompt "Press Enter to exit"
```

Below is an example output of the above script:

<img alt="image" src="https://github.com/user-attachments/assets/3792a576-513c-4929-80a9-b7612bc16966" />

After exporting the CSV file, it is useful to sort by `TotalTime` to find processes that tend to result in a lot of scans. For example, below, after sorting by `TotalTime`, it can be seen that the Process `RazerCentral.exe` tends to access a lot of files which are scanned for a long time:

<img alt="image" src="https://github.com/user-attachments/assets/624a4b94-b337-4628-bf57-491489d0dab7" />

Another option is to sort by `EstimatedImpact` to view processes which are affected a lot by MDAV scans:

<img alt="image" src="https://github.com/user-attachments/assets/db1be4db-92a2-4453-99f3-583ab42d39f6" />

A similar analysis can also be done by inserting a Pivot Table, and adding `ProcessImageName` in Rows, and `TotalTime` or `EstimatedImpact` in Values, on which you can sort again by the `Sum of TotalTime` or `Sum of EstimatedImpact`. This will produce a table like the following:

<img alt="image" src="https://github.com/user-attachments/assets/60a864b3-db49-42c4-8569-95d58670bac8" />

As shown above, the MPLog is yet another way of identifying processes which may impact the performance of the machine or may suffer from MDAV scans themselves.

## Conclusion

Sometimes, MDE and its components may utilize a higher-than-normal CPU percentage. Identifying the root cause can prove difficult. Hopefully, with this post it will be a little bit easier.

Stick around for the next posts, where we will dive deep into looking Windows Event Logs, and also may move to other Microsoft Security products like Purview.

I hope to see you back in the next one!
