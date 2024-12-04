# Intune_Win32app-ForceRedeploy
## Guide til force redeploy af udrullet Intune Win32app <br>


# Forcing Redeployment of Applications and Scripts in Microsoft Intune

This repository provides step-by-step guidance and sample scripts to force the redeployment of Win32 applications and PowerShell scripts assigned through Microsoft Intune. This is especially useful for testing, troubleshooting detection rules, or reassigning applications to devices.

## Table of Contents
- [Overview](#overview)
- [Win32 Apps – Background Information](#win32-apps--background-information)
- [Forcing Redeployment of Win32 Apps](#forcing-redeployment-of-win32-apps)
  - [Deleting All Assigned Apps for a User](#deleting-all-assigned-apps-for-a-user)
  - [Deleting a Specific App Assignment](#deleting-a-specific-app-assignment)
  - [Parsing Logs for GRS Key](#parsing-logs-for-grs-key)
- [Finding Application IDs in Intune](#finding-application-ids-in-intune)
- [Additional Notes](#additional-notes)
- [References](#references)

---

## Overview

Forcing the redeployment of applications or scripts in Intune is a common need during:
- **Testing:** Verifying required assignments or detection rules.
- **Troubleshooting:** Resolving issues with compliance or application installation.
- **Reassignment:** Updating devices with the latest configurations or revisions.

When a Win32 app is installed via Intune, the Microsoft Intune Management Extension (IME) agent keeps track of deployments in the registry. By deleting specific registry keys and logs, you can force redeployment.

---

## Win32 Apps – Background Information

- Win32 apps are often configured with the **Install Behavior** set to `System`, ensuring installation without requiring administrative rights for the user.
- The IME agent manages deployments and tracks them in the following registry key:
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\IntuneManagementExtension\Win32Apps

Each user has a unique subkey corresponding to their **Azure AD User Object ID**, and under this key are subkeys for each application and its **Global Retry Schedule (GRS)**.

Under the Win32Apps key, you find one sub key for each user, where the key name is the same as the user object id in Azure Ad. If you want to force a reinstall of all apps deployed, you can simply delete the user id key. But if you want to force a reinstall of a single app, you need to delete the app id as well as it's corresponding GRS (Global Retry Schedule key). Both located under the user key. Here is an example:

![Alt text](https://www.deploymentresearch.com/wp-content/uploads/2021/12/UserGuidInWin32AppsKey.png)

Win32Apps registry key sample from a machine enrolled into Intune

![Alt text](https://www.deploymentresearch.com/wp-content/uploads/2022/07/image-2.png)

GRS Key

On the first screenshot above, the red rectangle is the user key, and the blue rectangle is one of the deployed apps. Based on this info, if I wanted to reinstall all apps, I could run this PowerShell script which deletes all app IDs as well as the GRS keys:

```
# Delete all apps for a user
$Path = "HKLM:SOFTWARE\Microsoft\IntuneManagementExtension\Win32Apps"
$UserObjectID = "18ba2977-ea61-4547-8e8b-e9cbbced8719"
Get-Item -Path $Path\$UserObjectID | Remove-Item -Recurse -Force
```
If I wanted to reinstall a single app, I would first delete the single application id instead, and then I would have to locate the right GRS key and delete that one. The GRS key is found by parsing the IME log file. Below is sample script to do all of this:

```
# Sample to delete a single app
#
# Note: Don't got forget to delete any files/installs that the detection method uses on your machine
# Deleting specific application based on its object id
$Path = "HKLM:SOFTWARE\Microsoft\IntuneManagementExtension\Win32Apps"
$UserObjectID = "efd4c448-e6f1-46fa-b083-d87e60ea1274"
$AppID = "8ea44431-bb08-460c-b881-52bdff6a7128"

function GetAppGRSHash {
    param (
        [Parameter(Mandatory = $true)]
        [string] $appId
    )

    $intuneLogList = Get-ChildItem -Path "$env:ProgramData\Microsoft\IntuneManagementExtension\Logs" -Filter "IntuneManagementExtension*.log" -File | sort LastWriteTime -Descending | select -ExpandProperty FullName

    if (!$intuneLogList) {
        Write-Error "Unable to find any Intune log files. Redeploy will probably not work as expected."
        return
    }

    foreach ($intuneLog in $intuneLogList) {
        $appMatch = Select-String -Path $intuneLog -Pattern "\[Win32App\]\[GRSManager\] App with id: $appId is not expired." -Context 0, 1
        if ($appMatch) {
            foreach ($match in $appMatch) {
                $Hash = ""
                $LineNumber = 0
                $LineNumber = $match.LineNumber
                $Hash = ((Get-Content $intuneLog | Select-Object -Skip $LineNumber -First 1) -split " = ")[1]
                if ($hash) {
                    $hash = $hash.Replace('+','\+')
                    return $hash
                }
            }
        }
    }

    Write-Error "Unable to find App '$appId' GRS hash in any of the Intune log files. Redeploy will probably not work as expected"
}

$GRSHash = GetAppGRSHash -appId $AppID

(Get-ChildItem -Path $Path\$UserObjectID) -match $AppID | Remove-Item -Recurse -Force

(Get-ChildItem -Path $Path\$UserObjectID\GRS) -match $GRSHash | Remove-Item -Recurse -Force

# Restart the IME Service
Get-Service -DisplayName "Microsoft Intune Management Extension" | Restart-Service
```

Note #1: Make sure you also uninstall the existing application or remove whatever the application detection rule is configured to look for. Sometimes an Intune policy sync is also required.

Note #2: When deleting a single application, you have to use a wildcard match, since the registry key actually contains the revision of the app as well.

As for finding the application id, you can see it in the browser address bar when viewing the application in Intune, or you can use the below PowerShell script. Just remove the trailing _1 from the app registry key when searching for a matching GUID:
![Alt text](https://www.deploymentresearch.com/wp-content/uploads/2022/06/AppIDinURL-848x241.png)

Commands to see all Intune Win32Apps enrolled into the tenant:
```
# Connect to Microsoft Graph 
# Requires the Microsoft.Graph.Intune module to be installed
Connect-MgGraph -ForceInteractive

# Get all Apps and their id
$Apps = Get-DeviceAppManagement_MobileApps 
$Apps | select displayName, id

# Get Apps, their size in MB, and their id. Filter on App Name
$Apps = Get-DeviceAppManagement_MobileApps -Filter "contains(displayName, '100 MB Single File')"
$Apps | select displayName, @{Label="Size in MB";Expression={[math]::Round(($_.size/1MB),2)}}, id 
```

Kilder:
https://www.deploymentresearch.com/force-application-reinstall-in-microsoft-intune-win32-apps/
https://doitpshway.com/force-redeploy-of-intune-applications-using-powershell <br>
