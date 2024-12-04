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

---

## Forcing Redeployment of Win32 Apps

### Deleting All Assigned Apps for a User
To delete all applications assigned to a specific user:
```powershell
$Path = "HKLM:SOFTWARE\Microsoft\IntuneManagementExtension\Win32Apps"
$UserObjectID = "18ba2977-ea61-4547-8e8b-e9cbbced8719"  # Replace with actual User Object ID
Get-Item -Path $Path\$UserObjectID | Remove-Item -Recurse -Force
```powershell

Deleting a Specific App Assignment
To delete a specific application:

Remove the app ID under the user’s registry key.
Parse IME logs to locate and delete the corresponding GRS key.
Script Example:

Kopier kode
$Path = "HKLM:SOFTWARE\Microsoft\IntuneManagementExtension\Win32Apps"
$UserObjectID = "efd4c448-e6f1-46fa-b083-d87e60ea1274"
$AppID = "8ea44431-bb08-460c-b881-52bdff6a7128"

function GetAppGRSHash {
    param ([string] $appId)

    $logFiles = Get-ChildItem -Path "$env:ProgramData\Microsoft\IntuneManagementExtension\Logs" -Filter "IntuneManagementExtension*.log" -File |
                Sort-Object LastWriteTime -Descending

    foreach ($log in $logFiles) {
        $appMatch = Select-String -Path $log.FullName -Pattern "\[Win32App\]\[GRSManager\] App with id: $appId is not expired." -Context 0, 1
        if ($appMatch) {
            $hashLine = Get-Content $log.FullName | Select-Object -Skip $appMatch.LineNumber -First 1
            return ($hashLine -split " = ")[1]
        }
    }
    Write-Error "Unable to find GRS hash for App ID $appId."
}

$GRSHash = GetAppGRSHash -appId $AppID

Remove-Item -Path "$Path\$UserObjectID\$AppID" -Recurse -Force
Remove-Item -Path "$Path\$UserObjectID\GRS\$GRSHash" -Recurse -Force

# Restart IME Service
Get-Service -DisplayName "Microsoft Intune Management Extension" | Restart-Service
Finding Application IDs in Intune
The application ID can be located in the Intune portal URL while viewing the application details. Alternatively, use PowerShell with the Microsoft Graph API to retrieve app IDs:

Connect to Microsoft Graph:
powershell
Kopier kode
Connect-MSGraph -ForceInteractive
Retrieve App IDs:
powershell
Kopier kode
# Get all Apps and their IDs
$Apps = Get-DeviceAppManagement_MobileApps 
$Apps | Select-Object displayName, id

# Filter Apps by Name
$Apps = Get-DeviceAppManagement_MobileApps -Filter "contains(displayName, 'App Name')"
$Apps | Select-Object displayName, id
Additional Notes
Uninstall Existing Files: Ensure that files or registry keys checked by the detection rule are removed before forcing redeployment.
Policy Sync: Sometimes, a policy sync is required after making changes to ensure redeployment.
Log Files: IME logs are located at:
makefile
Kopier kode
C:\ProgramData\Microsoft\IntuneManagementExtension\Logs
References
Microsoft Intune Management Extension
Microsoft Graph API Documentation
GRS Parsing Samples
Call4Cloud – Intune Management

Kilder:
https://www.powershellgallery.com/packages/IntuneStuff/1.6.0
https://github.com/ztrhgf/useful_powershell_functions/blob/master/INTUNE/Invoke-IntuneWin32AppRedeploy.ps1 <br>
https://doitpshway.com/force-redeploy-of-intune-applications-using-powershell <br>
