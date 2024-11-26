# Intune_Win32app-ForceRedeploy
## Guide til redeploy af udrullet Intune Win32app <br>
1. Kør PowerShell som administrator
2. PS: Install-Module IntuneStuff -Force
3. PS: Import-Module IntuneStuff
4. PS: Connect-MgGraph
5. Vælg din MS-konto i browser popup
6. PS: Invoke-IntuneWin32Appredeploy -getDataFromIntune
7. Markér den App som du vil redeploy og klik "Ok" <br>
