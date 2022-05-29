---
layout: post
title: Stop-FailedService.ps1
---

### something exciting

Some information about the exciting thing

- [something exciting](#something-exciting)
  - [Script](#script)
  - [Download](#download)
  - [Report Issues](#report-issues)

<small><i><a href='http://ecotrust-canada.github.io/markdown-toc/'>Table of contents generated with markdown-toc</a></i></small>

---

#### Script

```powershell
function Stop-FailedService {
    [CmdletBinding()]
    param (
        [Parameter()]
        [string[]]
        $ComputerName,

        [Parameter()]
        [string]
        $ProcessName
    )
    foreach ($Computer in $ComputerName) {
        $Process = Get-CimInstance -ClassName "CIM_Process" -Namespace "root/CIMV2" -ComputerName "$Computer" | Where-Object -Property Name -Like ($ProcessName + ".exe")
        IF ($null -ne $Process) {
            $returnval = $process.Dispose()
            $processid = $process.handle
            if ($returnval.returnvalue -eq 0) {
                Write-Output "The process $ProcessName `($processid`) terminated successfully on Server $Computer"
            }
            else {
                Write-Error -Message "The process $ProcessName `($processid`) termination has some problems on Server $Computer"
            }
        }
        Else {
            Write-Error -Message "Process Not Running On Server $Computer"
        }
    }
}
```

<span style="font-size:11px;"><a href="#"><i class="fas fa-caret-up" aria-hidden="true" style="color: white; margin-right:5px;"></i>Back to Top</a></span>

#### Download

Please feel free to copy parts of the script or if you would like to download the entire script, simple click the download button. You can download the complete repository in a zip file by clicking the Download link in the menu bar on the left hand side of the page.

<button class="btn" type="submit" onclick="window.open('http://agamar.domain.leigh-services.com:4000/powershell/functions/myProfile/Stop-FailedService.ps1')">
    <i class="fa fa-cloud-download-alt">
    </i>
        Download
</button>

---

#### Report Issues

You can report an issue or contribute to this site on <a href="https://github.com/BanterBoy/scripts-blog/issues">GitHub</a>. Simply click the button below and add any relevant notes. I will attempt to respond to all issues as soon as possible.

<!-- Place this tag where you want the button to render. -->

<a class="github-button" href="https://github.com/BanterBoy/scripts-blog/issues/new?title=Stop-FailedService.ps1&body=There is a problem with this function. Please find details below." data-show-count="true" aria-label="Issue BanterBoy/scripts-blog on GitHub">Issue</a>

---

<span style="font-size:11px;"><a href="#"><i class="fas fa-caret-up" aria-hidden="true" style="color: white; margin-right:5px;"></i>Back to Top</a></span>

<a href="/menu/_pages/myProfile.html">
    <button class="btn">
        <i class='fas fa-reply'>
        </i>
            Back to myProfile
    </button>
</a>

[1]: http://ecotrust-canada.github.io/markdown-toc
[2]: https://github.com/googlearchive/code-prettify