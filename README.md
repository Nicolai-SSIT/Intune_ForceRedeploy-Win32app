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

https://www.deploymentresearch.com/wp-content/uploads/2021/12/UserGuidInWin32AppsKey.png

On the above screenshot, the red rectangle is the user key, and the blue rectangle is one of the deployed apps. Based on this info, if I wanted to reinstall all apps, I could run this PowerShell script which deletes all app IDs as well as the GRS keys:

```
# Delete all apps for a user
$Path = "HKLM:SOFTWARE\Microsoft\IntuneManagementExtension\Win32Apps"
$UserObjectID = "18ba2977-ea61-4547-8e8b-e9cbbced8719"
Get-Item -Path $Path\$UserObjectID | Remove-Item -Recurse -Force
```


Kilder:
https://www.powershellgallery.com/packages/IntuneStuff/1.6.0
https://github.com/ztrhgf/useful_powershell_functions/blob/master/INTUNE/Invoke-IntuneWin32AppRedeploy.ps1 <br>
https://doitpshway.com/force-redeploy-of-intune-applications-using-powershell <br>
