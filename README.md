# Intune_Win32app-ForceRedeploy
## Guide til force redeploy af udrullet Intune Win32app <br>

Forcing Redeployment of Intune Applications and Scripts
This repository provides instructions and scripts for forcing the redeployment of Win32 applications and PowerShell scripts assigned through Microsoft Intune. This is especially useful during testing, troubleshooting detection rules, or large-scale deployments.

Overview
When applications or scripts are assigned in Intune, the Microsoft Intune Management Extension (IME) tracks their deployment status. To force a redeployment, the corresponding registry keys and logs must be manipulated.

Table of Contents
Scenario
Forcing Redeployment of Win32 Apps
Delete All Assigned Apps for a User
Delete a Specific App Assignment
Locate App ID and GRS Key
Forcing Redeployment of Scripts
Additional Notes
Scenario
Forcing redeployment is helpful in the following scenarios:

Testing: Large-scale testing of required assignments.
Troubleshooting: Verifying detection rules or fixing compliance issues.
Reassignment: Redeploying applications or scripts to devices without modifying configurations in Intune.
Forcing Redeployment of Win32 Apps
IME Registry Key
IME tracks app assignments in the registry under:

Kopier kode
Computer\HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\IntuneManagementExtension\Win32Apps
Each user has a unique subkey under Win32Apps, corresponding to their Azure AD User Object ID. App-specific details are located under this key, along with a Global Retry Schedule (GRS) key.

Delete All Assigned Apps for a User
To force reinstall all apps for a user:

powershell
Kopier kode
$Path = "HKLM:SOFTWARE\Microsoft\IntuneManagementExtension\Win32Apps"
$UserObjectID = "18ba2977-ea61-4547-8e8b-e9cbbced8719"  # Replace with the actual User Object ID
Get-Item -Path $Path\$UserObjectID | Remove-Item -Recurse -Force
Delete a Specific App Assignment
To force reinstall a single app:

Delete the app ID under the user's registry key.
Parse the IME logs to locate and delete the corresponding GRS key.
Sample Script:
powershell
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
Forcing Redeployment of Scripts
Scripts deployed through Intune are tracked in:

Kopier kode
HKEY_LOCAL_MACHINE\SOFTWARE\Microsoft\IntuneManagementExtension\SideCarPolicies\Scripts
To redeploy a specific script:

Locate the script ID in Intune (e.g., via Graph API or the Azure portal).
Delete the registry key associated with the script ID.
Script Example:
powershell
Kopier kode
$ScriptID = "your-script-id"
$Path = "HKLM:SOFTWARE\Microsoft\IntuneManagementExtension\SideCarPolicies\Scripts"
Remove-Item -Path "$Path\$ScriptID" -Recurse -Force
Additional Notes
Finding IDs:

Win32 Apps: The app ID is visible in the browser address bar in the Intune portal.
Scripts: Use Microsoft Graph API to list deployed scripts:
powershell
Kopier kode
Connect-MSGraph -ForceInteractive
Get-DeviceAppManagement_MobileApps | Select-Object displayName, id
Log Files:

IME logs are located at:
makefile
Kopier kode
C:\ProgramData\Microsoft\IntuneManagementExtension\Logs
Uninstall First: Ensure the application's detection method criteria (e.g., installed files or registry keys) are removed to prevent false positives.

Policy Sync: A policy sync may be required after deletion to force redeployment.

References
Microsoft Intune Management Extension
Microsoft Graph API

Kilder:
https://www.powershellgallery.com/packages/IntuneStuff/1.6.0
https://github.com/ztrhgf/useful_powershell_functions/blob/master/INTUNE/Invoke-IntuneWin32AppRedeploy.ps1 <br>
https://doitpshway.com/force-redeploy-of-intune-applications-using-powershell <br>
