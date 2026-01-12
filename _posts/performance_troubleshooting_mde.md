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

## MPLog file parsing for performance impact

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
