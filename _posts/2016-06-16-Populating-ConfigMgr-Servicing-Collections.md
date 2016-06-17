---
layout: single
title: "Populating ConfigMgr Servicing Collections"
tags: 
 - ConfigMgr
 - Windows 10
 - WaaS
 - Servicing
---

One of the main challenges that I see with servicing Windows 10 in any organization is determining when devices will receive the update. While evaluating different approaches I used the following criteria to choose the best solution for me:

* Minimal collection administration 
* Limit the number of Insider builds to my team and select partners
* Enable end-users to opt-in to and opt-out of Current Branch
* For most of our assets I don't care when they get updated to the latest CBB release as long as we minimize the risk by spreading out the update
* Collections already exist for devices/users that we care about (C-Level, SVP, etc) which can be used as excludes and targeted for updating on a different schedule

The solution that I kept coming back to fits the *"Automate the mundane so we can focus on the complex"* mantra. 

### Approach
During the workstation build a step will create an dummy program which will show up in Programs and Features. This program will inventoried by SCCM alongside all the other programs which saves us from extending hardware inventory further. The entry specifies what CBB ring a device is assigned to in the version attribute. From there query based collections will be created based on the on the inventoried data. If needed this process can also re-executed on devices if you decide to change the number of rings or want to reset the environment. This approach will take care of 95+% of the devices in my environment, however depending on how many users opt-in to Current Branch we may manually define a pilot device collection. 

In addition to the automatic ring assignment we will deploy an application to the Application Catalog which will allow users to opt-in to Current Branch or change their Current Branch For Business ring. We don't expect that most users taking advantage of this, however we hope that there's enough interest that we only need to source a handful of users to be part of our pilot group.

For this post I'm going to focus on the automatic ring assignment and will detail out the Current Branch opt-in and ring override method later. 

### Implementation
I've decided to group my assets into 10 Current Branch For Business rings, however you can easily adjust this number based on your requirements. In order to distribute the assets across the 10 rings I converted the hostname into an integer then using the modulus operator then assign the device into a group based on the result. Based on my testing the exact method used to convert the hostname to an integer doesn't matter as long as you consistently apply it to all the devices in scope. The rings will not not be perfectly equal, however they are within a close enough range that I found acceptable. 

Here's the code that I used to auto assign a device to a ring then create a Add Remote Program Entry

```powershell
$numberOfRings = 10
[int]$iComputerName
$ComputerName = (Get-CimInstance Win32_ComputerSystem).Name
foreach ($character in $ComputerName.ToCharArray()) {
    $iComputerName += [int][char]$character
}
$AssignedRing = $iComputerName % $numberOfRings

Write-Output "Assigning $ComputerName To $AssignedRing"

$RegPath = 'HKLM:\SOFTWARE\Microsoft\Windows\CurrentVersion\Uninstall\Windows 10 Upgrade Ring'
New-Item -Path $RegPath
New-ItemProperty -Path $RegPath -Name 'DisplayName' -Value 'Windows 10 Upgrade Ring' -PropertyType String
New-ItemProperty -Path $RegPath -Name 'DisplayVersion' -Value $AssignedRing -PropertyType String
New-ItemProperty -Path $RegPath -Name 'DisplayIcon' -Value 'msiexec.exe' -PropertyType String
New-ItemProperty -Path $RegPath -Name 'NoModify' -Value 00000001 -PropertyType DWord
New-ItemProperty -Path $RegPath -Name 'NoRepair' -Value 00000001 -PropertyType DWord
New-ItemProperty -Path $RegPath -Name 'EstimatedSize' -Value 00000000 -PropertyType DWord
New-ItemProperty -Path $RegPath -Name 'Publisher' -Value 'Company Name' -PropertyType String
New-ItemProperty -Path $RegPath -Name 'UninstallString' -Value 'msiexec.exe' -PropertyType String
New-ItemProperty -Path $RegPath -Name 'NoRemove' -Value 00000001 -PropertyType DWord
New-ItemProperty -Path $RegPath -Name 'HelpLink' -Value 'http://wiki.company.com/wiki/index.php/WaaS' -PropertyType String
```
Here's an example of the entry which is created in Programs and Features. I chose to place it here as it's out-of-the-way, but still accessible. 
![Programs&FeaturesExample](/images/Ring.png)

Here's a sample of the WQL that can be used to create the different collections. 

```sql
select SMS_R_SYSTEM.ResourceID,SMS_R_SYSTEM.ResourceType,SMS_R_SYSTEM.Name,SMS_R_SYSTEM.SMSUniqueIdentifier,
SMS_R_SYSTEM.ResourceDomainORWorkgroup,SMS_R_SYSTEM.Client from SMS_R_System inner join SMS_G_System_ADD_REMOVE_PROGRAMS_64 
on SMS_G_System_ADD_REMOVE_PROGRAMS_64.ResourceId = SMS_R_System.ResourceId inner join SMS_G_System_OPERATING_SYSTEM on
SMS_G_System_OPERATING_SYSTEM.ResourceId = SMS_R_System.ResourceId where SMS_G_System_ADD_REMOVE_PROGRAMS_64.DisplayName = 
"Windows 10 Upgrade Ring" and SMS_G_System_ADD_REMOVE_PROGRAMS_64.Version = "4" and SMS_G_System_OPERATING_SYSTEM.Caption like "Microsoft Windows 10%"
```

Kaido Järvemets has an excellent [blog](http://blog.coretech.dk/kaj/creating-configmgr-servicing-plans-with-powershell/ "Kaido's Blog: Creating ConfigMgr Servicing Plans with PowerShell") post where he shows how to use the new New-CMWindowsServicingPlan cmdlet to implement a servicing plan within Configuration Manager 1511 or newer. His post can be easily updated to apply the WQL queries to the collections.

### Author's Note
This is my first of what I hope to be many blog posts. I hope that you've enjoyed reading it and it inspires you to develop a process to populate your own Servicing Collections. If you liked what you read please help spread the word by sharing the post. 

For a long time I've read the numerous blogs posts on OSD, SCCM and Windows. I've been helped and inspired by so many of them and it's been long overdue (in my opinion) to start giving back to the community that I've benefited greatly from.

-Mike
