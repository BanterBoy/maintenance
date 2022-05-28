---
layout: post
title: FindOrphanedGPOs.ps1
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
<#
This script will find and print all orphaned Group Policy Objects (GPOs).

Group Policy Objects (GPOs) are stored in two parts:

1) GPC (Group Policy Container). The GPC is where the GPO stores all the AD-related configuration under the
   CN=Policies,CN=System,DC=... container, which is replicated via AD replication.
2) GPT (Group Policy Templates). The GPT is where the GPO stores the actual settings located within SYSVOL
   area under the Policies folder, which is replicated by either File Replication Services (FRS) or
   Distributed File System (DFS).

This script will help find GPOs that are missing one of the parts, which therefore makes it an orphaned GPO.

A GPO typically becomes orphaned in one of two different ways:

1) If the GPO is deleted directly through Active Directory Users and Computers or ADSI edit.
2) If the GPO was deleted by someone that had permissions to do so in AD, but not in SYSVOL. In this case,
   the AD portion of the GPO would be deleted but the SYSVOL portion of the GPO would be left behind.

Although orphaned GPT folders do no harm they do take up disk space and should be removed as a cleanup task.

Lack of permissions to the corresponding objects in AD could cause a false positive. Therefore, verify GPT
folders are truly orphaned before moving or deleting them.

Original script written by Sean Metcalf
http://blogs.metcorpconsulting.com/tech/?p=1076

Release 1.1
Modified by Jeremy@jhouseconsulting.com 29th August 2012

#>

$Domain = [System.DirectoryServices.ActiveDirectory.Domain]::GetCurrentDomain()
# Get AD Domain Name
$DomainDNS = $Domain.Name
# Get AD Distinguished Name
$DomainDistinguishedName = $Domain.GetDirectoryEntry() | Select-Object -ExpandProperty DistinguishedName

$GPOPoliciesDN = "CN=Policies,CN=System,$DomainDistinguishedName"
$GPOPoliciesSYSVOLUNC = "\\$DomainDNS\SYSVOL\$DomainDNS\Policies"

Write-Host -ForegroundColor Green "Finding all orphaned Group Policy Objects (GPOs)...`n"

Write-Host -ForegroundColor Green "Reading GPO information from Active Directory ($GPOPoliciesDN)..."
$GPOPoliciesADSI = [ADSI]"LDAP://$GPOPoliciesDN"
[array]$GPOPolicies = $GPOPoliciesADSI.psbase.children
foreach ($GPO in $GPOPolicies) { [array]$DomainGPOList += $GPO.Name }
#$DomainGPOList = $DomainGPOList -replace("{","") ; $DomainGPOList = $DomainGPOList -replace("}","")
$DomainGPOList = $DomainGPOList | Sort-Object
[int]$DomainGPOListCount = $DomainGPOList.Count
Write-Host -ForegroundColor Green "Discovered $DomainGPOListCount GPCs (Group Policy Containers) in Active Directory ($GPOPoliciesDN)`n"

Write-Host -ForegroundColor Green "Reading GPO information from SYSVOL ($GPOPoliciesSYSVOLUNC)..."
[array]$GPOPoliciesSYSVOL = Get-ChildItem $GPOPoliciesSYSVOLUNC
foreach ($GPO in $GPOPoliciesSYSVOL) { if ($GPO.Name -ne "PolicyDefinitions") { [array]$SYSVOLGPOList += $GPO.Name } }
#$SYSVOLGPOList = $SYSVOLGPOList -replace("{","") ; $SYSVOLGPOList = $SYSVOLGPOList -replace("}","")
$SYSVOLGPOList = $SYSVOLGPOList | sort-object
[int]$SYSVOLGPOListCount = $SYSVOLGPOList.Count
Write-Host -ForegroundColor Green "Discovered $SYSVOLGPOListCount GPTs (Group Policy Templates) in SYSVOL ($GPOPoliciesSYSVOLUNC)`n"

## COMPARE-OBJECT cmdlet note:
## The result indicates where a property value appears:
## The <= sign indicates that the item in question was found in the property set of the first object (reference set) but not found in the property set for the second object.
## The => sign indicates that the item in question was found in the property set of the second object (difference set) but not found in the property set for the first object.
## Use -IncludeEqual to Displays characteristics of compared objects that are equal

# Check for GPTs in SYSVOL that don't exist in AD
[array]$MissingADGPOs = Compare-Object $SYSVOLGPOList $DomainGPOList -passThru | Where-Object { $_.SideIndicator -eq '<=' }
[int]$MissingADGPOsCount = $MissingADGPOs.Count
$MissingADGPOsPCTofTotal = $MissingADGPOsCount / $DomainGPOListCount
$MissingADGPOsPCTofTotal = "{0:p2}" -f $MissingADGPOsPCTofTotal
Write-Host -ForegroundColor Yellow "There are $MissingADGPOsCount GPTs in SYSVOL that don't exist in Active Directory ($MissingADGPOsPCTofTotal of the total)"
if ($MissingADGPOsCount -gt 0 ) {
    Write-Host "These are:"
    $MissingADGPOs
}
Write-Host "`n"

# Check for GPCs in AD that don't exist in SYSVOL
[array]$MissingSYSVOLGPOs = Compare-Object $DomainGPOList $SYSVOLGPOList -passThru | Where-Object { $_.SideIndicator -eq '<=' }
[int]$MissingSYSVOLGPOsCount = $MissingSYSVOLGPOs.Count
$MissingSYSVOLGPOsPCTofTotal = $MissingSYSVOLGPOsCount / $DomainGPOListCount
$MissingSYSVOLGPOsPCTofTotal = "{0:p2}" -f $MissingSYSVOLGPOsPCTofTotal
Write-Host -ForegroundColor Yellow "There are $MissingSYSVOLGPOsCount GPCs in Active Directory that don't exist in SYSVOL ($MissingSYSVOLGPOsPCTofTotal of the total)"
if ($MissingSYSVOLGPOsCount -gt 0 ) {
    Write-Host "These are:"
    $MissingSYSVOLGPOs
}
Write-Host "`n"


# Is there a version match between your Group Policy Object containers and templates?
# This script will check that the version of each GPO in consistent in the Active Directory
# Group Policy Container (GPC) and on each Domain Controller in the Group Policy Template (GPT)

#The following article provides an excellent explanation of how the group policy version number works.
# http://technet.microsoft.com/en-us/library/ff730972.aspx

# You can get an unsupported updated version of GPOTOOL from here:
#http://kb.elmahdy.net/2011/02/gpotool-for-windows-server-2008-r2.html


$ValidGPOS = Compare-Object $DomainGPOList $SYSVOLGPOList -IncludeEqual
foreach ($ValidGPO in $ValidGPOS) {
    Write-Host "`n"
    $GPC = $ValidGPO.InputObject
    $GPCADSI = [ADSI]"LDAP://CN=$GPC,$GPOPoliciesDN"
    $gpcVersion = $GPCADSI.properties.versionnumber[0]
    #$GPCADSI.properties.gPCFileSysPath
    # Get GPT Version from the gpt.ini file and convert it from a string to an integer.
    [int]$gptVersion = (Get-Content "$($GPCADSI.properties.gPCFileSysPath)\gpt.ini" | Where-Object { $_ -like "Version=*" }).Split("=")[1]
    $displayname = $GPCADSI.properties.displayname
    $name = $GPCADSI.properties.name
    Write-Host "DisplayName: $displayname"
    Write-Host "Name: $name"
    Write-Host "GPC Version: $gpcVersion"
    [INT]$UserGPCVersion = "{0:d}" -f [INT]("0x" + [String]::Format("{0:x8}", $gpcVersion).Substring(0, [String]::Format("{0:x8}", $gpcVersion).Length / 2))
    [INT]$MachineGPCVersion = "{0:d}" -f [INT]("0x" + [String]::Format("{0:x8}", $gpcVersion).Substring(4, [String]::Format("{0:x8}", $gpcVersion).Length / 2))
    Write-Host "User GPC Version: $UserGPCVersion"
    Write-Host "Machine GPC Version: $MachineGPCVersion"
    Write-Host "GPT Version: $gptVersion"
    [INT]$UserGPTVersion = "{0:d}" -f [INT]("0x" + [String]::Format("{0:x8}", $gptVersion).Substring(0, [String]::Format("{0:x8}", $gptVersion).Length / 2))
    [INT]$MachineGPTVersion = "{0:d}" -f [INT]("0x" + [String]::Format("{0:x8}", $gptVersion).Substring(4, [String]::Format("{0:x8}", $gptVersion).Length / 2))
    Write-Host "User GPT Version: $UserGPTVersion"
    Write-Host "Machine GPT Version: $MachineGPTVersion"

    if ($gpcVersion -eq $gptVersion) {
        if (!($gpcVersion -eq 0 -AND $gptVersion -eq 0)) {
            $Status = "Match"
            Write-Host -ForegroundColor Green "Status: $Status"
        }
        else {
            $Status = "Empty"
            Write-Host -ForegroundColor Red "Status: $Status"
        }
    }
    else {
        $Status = "Mismatch"
        Write-Host -ForegroundColor Red "Status: $Status"
    }

    # Calculate the size of the GPT in bytes and megabytes as well as getting the folder and file count.
    $colItems = Get-ChildItem "$($GPCADSI.properties.gPCFileSysPath)" �force -recurse
    $FolderCount = $colItems | Where-Object { $_.PSIsContainer } | Measure-Object | Select-Object -Expand Count
    $FileCount = $colItems | Where-Object { !$_.PSIsContainer } | Measure-Object | Select-Object -Expand Count
    $size = ($colItems | Measure-Object -property length -sum)
    $SizeinMB = "{0:N2}" -f ($size.sum / 1MB) + " MB"
    $SizeinBytes = "{0:N0}" -f $size.sum + " bytes"
    Write-Host "Size: $SizeinMB ($SizeinBytes)"
    Write-Host "Contains $FileCount Files, $FolderCount Folders"


    #Get-Hash cmdlet from the PSCX to obtain hash values from files in a folder
    #http://blogs.technet.com/b/heyscriptingguy/archive/2012/05/30/learn-the-easy-way-to-use-powershell-to-get-file-hashes.aspx
    #http://dbadailystuff.com/2013/03/11/get-hash-a-powershell-hash-function/
    #http://poshcode.org/2154
    #https://groups.google.com/forum/#!topic/microsoft.public.windows.powershell/9nZBZVHe6_I
    #http://blogs.technet.com/b/grouppolicy/archive/2012/11/29/group-policy-in-windows-server-2012-infrastructure-status.aspx
    #http://technet.microsoft.com/en-us/library/jj134176.aspx
}
```

<span style="font-size:11px;"><a href="#"><i class="fas fa-caret-up" aria-hidden="true" style="color: white; margin-right:5px;"></i>Back to Top</a></span>

---

#### Download

Please feel free to copy parts of the script or if you would like to download the entire script, simple click the download button. You can download the complete repository in a zip file by clicking the Download link in the menu bar on the left hand side of the page.

<button class="btn" type="submit" onclick="window.open('/PowerShell/scripts/activeDirectory/FindOrphanedGPOs.ps1')">
    <i class="fa fa-cloud-download-alt">
    </i>
        Download
</button>

---

#### Report Issues

You can report an issue or contribute to this site on <a href="https://github.com/BanterBoy/scripts-blog/issues">GitHub</a>. Simply click the button below and add any relevant notes. I will attempt to respond to all issues as soon as possible.

<!-- Place this tag where you want the button to render. -->

<a class="github-button" href="https://github.com/BanterBoy/scripts-blog/issues/new?title=FindOrphanedGPOs.ps1&body=There is a problem with this function. Please find details below." data-show-count="true" aria-label="Issue BanterBoy/scripts-blog on GitHub">Issue</a>

---

<span style="font-size:11px;"><a href="#"><i class="fas fa-caret-up" aria-hidden="true" style="color: white; margin-right:5px;"></i>Back to Top</a></span>

<a href="/menu/_pages/scripts.html">
    <button class="btn">
        <i class='fas fa-reply'>
        </i>
            Back to Scripts
    </button>
</a>

[1]: http://ecotrust-canada.github.io/markdown-toc
[2]: https://github.com/googlearchive/code-prettify
