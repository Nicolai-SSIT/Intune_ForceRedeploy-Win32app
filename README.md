# Intune_Win32app-ForceRedeploy
## Guide til redeploy af udrullet Intune Win32app <br>
1. Kør PowerShell som administrator
2. PS: Install-Module IntuneStuff -Force
3. PS: Import-Module IntuneStuff -Force
4. Hvis den melder fejl kør: Set-ExecutionPolicy, derefter RemoteSigned og kør step 3 igen.
5. PS: Connect-MgGraph
6. Vælg din MS-konto i browser popup
7. PS: Invoke-IntuneWin32Appredeploy -getDataFromIntune
8. Markér den App som du vil redeploy og klik "Ok"

 Hvis den fryser, åben kør og indsæt "intunemanagementextension://syncapp" for at gennnemtvinge Intune udrulning <br>


Kilder:
https://www.powershellgallery.com/packages/IntuneStuff/1.6.0
https://github.com/ztrhgf/useful_powershell_functions/blob/master/INTUNE/Invoke-IntuneWin32AppRedeploy.ps1 <br>
https://doitpshway.com/force-redeploy-of-intune-applications-using-powershell <br>
