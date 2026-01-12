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

You can also add Certificates to the Indicators allow list.

**Using different files as databases**: Files like HTML Applications (HTA), or Compiled HTML (CHM), if MDAV has to scan complex file formats, it will use much more CPU. Consider using actual databased if needed to save info.

**Obfuscated scripts**: Obfuscated script required much more CPU to be scanned, so obfuscation should be used only if necessary.

**Not letting MDAV cache finish before sealing the image**: If you are creating a VDI image for a persistent or non-persistent image, make sure the cache maintenance completes before sealing the image. To do this:

1. Open up the Task Scheduler mmc (taskschd.msc).
2. Expand Task Scheduler Library > Microsoft > Windows > Windows Defender, and then right-click on Windows Defender Cache Maintenance.
3. Select Run, and let the scheduled task finish.

**Misspelled exclusions**: Double check that the exclusions as spelled correctly.

**Path exclusions only work for scanning flows**: Behavior MOnitoring and Network Real-time Inspection can still cause performance issues. As a workaround:

- Either add the exe or dll to the Indicators file hash allow list, or
- Add the certificate to the Indicators certificates allow list, or
- Add MDAV exclusions for the process, too.

**File hash computation**: If you enable the File Hash computation feature, computes file hashes for every executable file that is scanned if it wasn’t previously computed. This has a performance cost especially when copying large files from a network share. This feature is needed when blocking file Indicators (IoCs) in defender. Keep in mind that this feature is a prerequisite for File Hash Indicators.

(probably not useful) To determine which component might be using CPU:

RTP
Scheduled scanning
Scan after sec int updates
COnflict with other sec SW
Scanning large files / number of files

## Performance Analyzer

Some examples which can be useful
Start with general search per category, and if you find a pattern in a category, then you can dig deeper.

## Process Monitor


## Windows Performance Recorder (WPRUI)

[Reference](https://learn.microsoft.com/en-us/windows-hardware/test/wpt/introduction-to-wpr)

Check the other troubleshooting section 13.

## MPLog file parsing for performance impact

The Microsoft Protection Log (MPLog) file is one of a few files found under the path `C:\ProgramData\Microsoft\Windows Defender\Support`. Under this path, a few files which contain Defender logs can be found, including:

- MPDetection
- MPDeviceControl
- MPDlpLog
- MPLog
- MPScanSkip
- MpWppCoreTracing
- MpWppTracing

These files are very useful when investigating what Defender and its different detection mechanisms did. When it comes to performance troubleshooting, the MPLog file contains a few useful lines, among others, which can assist in detecting processes and files which took a heavy toll on the machine's performance. Credits to the related [article](https://www.crowdstrike.com/en-us/blog/how-to-use-microsoft-protection-logging-for-forensic-investigations/) by CrowdStrike explaining these columns.

These lines have the following format:

`2025-12-31T15:45:19.964 ProcessImageName: Razer Synapse Service Process.exe, Pid: 9104, TotalTime: 810, Count: 224, MaxTime: 15, MaxTimeFile: \Device\HarddiskVolume3\Program Files (x86)\Razer\Synapse3\UserProcess\Razer Synapse Service Process.exe.config, EstimatedImpact: 53%`

Let's go through this line:

|Field Name|Description|Example|
|-|-|-|
|Date (not stated)|Timestamp of event|2025-12-31T15:45:19.964|
|ProcessImageName|Process Image Name|Razer Synapse Service Process.exe|
|Pid|Process ID of the Process|9104|
|TotalTime|The summary of all time periods spent in scans of files accessed by this process (milliseconds)|810|
|Count|The number of scanned files accessed by this process|810|
|MaxTime|The longest time spent in scanning a single file accessed by this process|15|
|MaxTimeFile|The file which was scanned for `MaxTime` milliseconds|\Device\HarddiskVolume3\Program Files (x86)\Razer\Synapse3\UserProcess\Razer Synapse Service Process.exe.config|
|EstimatedImpact|This value shows the impact that MDAV had on the performance of the above process. It is the percentage of (Total time spent in scans of files accessed by this process)/(Total time which this process experienced scan activity). For example, if you open a large file in File Explorer, all the files in it are being scanned via Real-Time Protection. The time it takes until the last file is finished being scanned was 10 seconds, but the time that MDAV was actually scanning files was 3 seconds. Here the Estimated Impact is about 3/10 = 30%.|53%|

Therefore, using the above lines and the `EstimatedImpact` value, it is possible to review processes which experience heavy scanning load. Note that this does not mean necessarily that these processes result in higher CPU for the machine.

The following PowerShell script parses the MPLog file, takes only the lines which include the `EstimatedImpact` values, and exports it all in a CSV.

```powershell
# Function to get the log file via dialog
function Get-FileName($initialDirectory) {
    [System.Reflection.Assembly]::LoadWithPartialName("System.windows.forms") | Out-Null
    $OpenFileDialog = New-Object System.Windows.Forms.OpenFileDialog
    $OpenFileDialog.initialDirectory = $initialDirectory
    $OpenFileDialog.filter = "All files (*.*)| *.*"
    $OpenFileDialog.ShowDialog() | Out-Null
    $OpenFileDialog.filename
}

# Prompt the user to select a Windows Defender log file
Write-Host "Select Windows Defender Log File to export Impactful Processes..."
$logfile = Get-FileName
Write-Host "File Selected: $logfile"

# Extract relevant log entries, remove whitespaces, and process data
$logs = Get-Content $logfile | `
    Select-String "EstimatedImpact" | `
    Select-String -Pattern '%$' | `
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

A similar analysis can also be done by inserting a Pivot Table, and adding `ProcessImageName` in Rows, and `TotalTime` in Values, on which you can sort again by the Sum of TotalTime. This will produce a table like the following:

<img alt="image" src="https://github.com/user-attachments/assets/60a864b3-db49-42c4-8569-95d58670bac8" />

As shown above, the MPLog is yet another way of identifying processes which may impact the performance of the machine or may suffer from MDAV scans themselves.
