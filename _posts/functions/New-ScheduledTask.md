---
layout: post
title: New-ScheduledTask.ps1
---

- [Description](#description)
  - [Script](#script)
  - [Download](#download)
  - [Report Issues](#report-issues)

---

### Description

**_BanterBoy:_** Explain what the script does and why it is useful.

**_GitHub Copilot:_**

---

#### Script

```powershell
<#
Establish Secure Remote Session
Create Script
 - Create new directory - SoloScripts
 - Create new file - ScheduledRestart.ps1
 - Populate contents of new file with reboot script
 Write Event Log message confirming creation of files
 Schedule Task to run script at selected interval
#>

[CmdletBinding(DefaultParameterSetName = 'default')]
param(
    [Parameter(Mandatory = $false,
        HelpMessage = "Please enter your API Key",
        ValueFromPipeline = $True,
        ValueFromPipelineByPropertyName = $True)]
    [Alias('ak')]
    [string[]]$ApiKey,

    [Parameter(Mandatory = $false,
        HelpMessage = "Please enter your API Key",
        ValueFromPipeline = $True,
        ValueFromPipelineByPropertyName = $True)]
    [Alias('ak')]
    [string[]]$ApiKey,

    [Parameter(Mandatory = $false,
        HelpMessage = "Please enter your API Key",
        ValueFromPipeline = $True,
        ValueFromPipelineByPropertyName = $True)]
    [Alias('ak')]
    [string[]]$ApiKey,

    [Parameter(Mandatory = $True,
        HelpMessage = "Please select your scheduled time",
        ValueFromPipeline = $false,
        ValueFromPipelineByPropertyName = $false)]
    [Alias('st')]
    [string]$Title = 'Select Time'
)
Write-Output "================ $Title ================"
Write-Output "1: Press '1' for Mid-day."
Write-Output "2: Press '2' for Quarter Past."
Write-Output "3: Press '3' for Half Past."
Write-Output "Q: Press 'Q' to quit."

BEGIN {
    # [Net.ServicePointManager]::SecurityProtocol = [Net.SecurityProtocolType]::Tls12
    function New-Entry {
        $NewEvent = [PSCustomObject]@{
            LogName   = 'System'
            Source    = 'EventLog'
            EntryType = 'Information'
            EventId1  = 4668
            EventId2  = 4666
            Message1  = 'The daily PC Restart folder/files have been created.'
            Message2  = 'The daily PC Restart Scheduled task has been created.'
        }
    }
}

PROCESS {
    $Username = Read-Host "Enter Username (domain\username)"
    $Password = Read-Host "Enter Password" -AsSecureString
    $Credentials = New-Object System.Management.Automation.PSCredential($Username, $Password)
    $Server = Read-Host "Enter Server (FQDN or IP)"
    Enter-PSSession -ComputerName $Server -Credential $Credentials -Authentication Negotiate
    Wait-Event -Timeout 5
    Write-Output "You are PSRemoting to $Server as $Username"
    Wait-Event -Timeout 5

    Set-Location "C:\"
    New-Item -Path "C:" -Name SoloScripts -ItemType Directory
    New-Item -Path "C:\SoloScripts" -Name ScheduledRestart.ps1 -ItemType File

    $Content = { $params = [PSCustomObject]@{
            LogName   = 'System'
            Source    = 'EventLog'
            EntryType = 'Information'
            EventId   = 6006
            Message   = 'The daily PC Restart has taken place in order to update starter and leaver changes'
        }
        Write-EventLog -LogName $params.LogName -Source $params.Source -EntryType $params.EntryType -EventId $params.EventId -Message $params.Message
        Restart-Computer -Force
    }

    Set-Content -Path "C:\SoloScripts\ScheduledRestart.ps1" -Value $Content
    Write-EventLog -LogName $NewEvent.LogName -Source $NewEvent.Source -EntryType $NewEvent.EntryType -EventId $NewEvent.EventId1 -Message $NewEvent.Message1

    $taskName = 'Solo Restart'
    $scriptPath = "C:\SoloScripts\ScheduledRestart.ps1"

    $NewEvent = [PSCustomObject]@{
        LogName   = 'System'
        Source    = 'EventLog'
        EntryType = 'Information'
        EventId1  = 4668
        EventId2  = 4666
        Message1  = 'The daily PC Restart folder/files have been created.'
        Message2  = 'The daily PC Restart Scheduled task has been created.'
    }

    do {
        New-ScheduledTask
        $input = Read-Host "Please make a selection"
        switch ($input) {
            '1' {
                if ($input -eq '1') {
                    $Midday = (Get-Date -Hour 12 -Minute 00 -Second 0 -Millisecond 0).ToShortTimeString()
                    $Action = New-ScheduledTaskAction -Execute "Powershell.exe" -Argument "-executionpolicy bypass -noprofile -file $scriptPath"
                    $Trigger = New-ScheduledTaskTrigger -Daily -At $Midday
                    Register-ScheduledTask -TaskName $taskName -Trigger $Trigger -Action $Action -Description "A daily PC Restart is required in order to to update starter and leaver changes within the filemaker application." -RunLevel Highest -Force
                    Write-EventLog -LogName $NewEvent.LogName -Source $NewEvent.Source -EntryType $NewEvent.EntryType -EventId $NewEvent.EventId2 -Message $NewEvent.Message2
                    'Task has been scheduled for Mid-day'
                }
            } '2' {
                if ($input -eq '2') {
                    $QuarterPast = (Get-Date -Hour 12 -Minute 00 -Second 0 -Millisecond 0).AddMinutes(15).ToShortTimeString()
                    $Action = New-ScheduledTaskAction -Execute "Powershell.exe" -Argument "-executionpolicy bypass -noprofile -file $scriptPath"
                    $Trigger = New-ScheduledTaskTrigger -Daily -At $QuarterPast
                    Register-ScheduledTask -TaskName $taskName -Trigger $Trigger -Action $Action -Description "A daily PC Restart is required in order to to update starter and leaver changes within the filemaker application." -RunLevel Highest -Force
                    Write-EventLog -LogName $NewEvent.LogName -Source $NewEvent.Source -EntryType $NewEvent.EntryType -EventId $NewEvent.EventId2 -Message $NewEvent.Message2
                    'Task has been scheduled for Quarter Past'
                }
            } '3' {
                if ($input -eq '3') {
                    $HalfPast = (Get-Date -Hour 12 -Minute 00 -Second 0 -Millisecond 0).AddMinutes(30).ToShortTimeString()
                    $Action = New-ScheduledTaskAction -Execute "Powershell.exe" -Argument "-executionpolicy bypass -noprofile -file $scriptPath"
                    $Trigger = New-ScheduledTaskTrigger -Daily -At $HalfPast
                    Register-ScheduledTask -TaskName $taskName -Trigger $Trigger -Action $Action -Description "A daily PC Restart is required in order to to update starter and leaver changes within the filemaker application." -RunLevel Highest -Force
                    Write-EventLog -LogName $NewEvent.LogName -Source $NewEvent.Source -EntryType $NewEvent.EntryType -EventId $NewEvent.EventId2 -Message $NewEvent.Message2
                    'Task has been scheduled for Half Past'
                }
            } 'q' {
                return
            }
        }
        pause
    }
    until ($input -eq 'q')

}

END {}
```

<span style="font-size:11px;"><a href="#"><i class="fas fa-caret-up" aria-hidden="true" style="color: white; margin-right:5px;"></i>Back to Top</a></span>

---

#### Download

Please feel free to copy parts of the script or if you would like to download the entire script, simple click the download button. You can download the complete repository in a zip file by clicking the Download link in the menu bar on the left hand side of the page.

<button class="btn" type="submit" onclick="window.open('/PowerShell/functions/New-ScheduledTask.ps1')">
    <i class="fa fa-cloud-download-alt">
    </i>
        Download
</button>

---

#### Report Issues

You can report an issue or contribute to this site on <a href="https://github.com/BanterBoy/scripts-blog/issues">GitHub</a>. Simply click the button below and add any relevant notes. I will attempt to respond to all issues as soon as possible.

<!-- Place this tag where you want the button to render. -->

<a class="github-button" href="https://github.com/BanterBoy/scripts-blog/issues/new?title=New-ScheduledTask.ps1&body=There is a problem with this function. Please find details below." data-show-count="true" aria-label="Issue BanterBoy/scripts-blog on GitHub">Issue</a>

---

<span style="font-size:11px;"><a href="#"><i class="fas fa-caret-up" aria-hidden="true" style="color: white; margin-right:5px;"></i>Back to Top</a></span>

<a href="/menu/_pages/functions.html">
    <button class="btn">
        <i class='fas fa-reply'>
        </i>
            Back to Functions
    </button>
</a>

[1]: http://ecotrust-canada.github.io/markdown-toc
[2]: https://github.com/googlearchive/code-prettify
