# Module Description
Built a PowerShell script that connects to Microsoft Entra ID via Microsoft Graph API, scans every enabled user account in the tenant, checks their registered authentication methods, identifies anyone relying solely on a password with no MFA configured, automatically adds flagged users to a security group, and triggers an MFA Registration Campaign. Exports a prioritized report of at-risk accounts.

**Step 1 — Connected to Microsoft Graph**
- Connected to Entra ID tenant using two scopes
- User.Read.All — permission to read all user accounts
- UserAuthenticationMethod.Read.All — permission to read MFA methods per user
```powershell
Connect-MgGraph -Scopes "User.Read.All", "UserAuthenticationMethod.Read.All" -NoWelcome
Write-Host "Connected to Graph" -ForegroundColor Green
```
**Step 2 — Pulled all users from the tenant**
- Retrieved all 102 user accounts from the tenant
- Used -Property to only request the four fields needed — more efficient than pulling everything
- Confirmed count with Write-Host
```powershell
$users = Get-MgUser -All -Property "Id,DisplayName,UserPrincipalName,AccountEnabled"
Write-Host "Total users found: $($users.Count)" -ForegroundColor Green
```
**Step 3 — Created two empty results lists**
- Created $noMFA to collect users with no MFA registered
- Created $hasMFA to collect users with MFA enabled
- Two separate lists so counts and exports can be handled independently
```powershell
$noMFA  = [System.Collections.Generic.List[PSCustomObject]]::new()
$hasMFA = [System.Collections.Generic.List[PSCustomObject]]::new()
```
**Step 4 — Looped through every user and skipped disabled accounts**
- Looped through all 102 users one at a time
- Skipped any disabled accounts immediately using continue — no point checking MFA on accounts that can't sign in
```powershell
foreach ($user in $users) {
if ($user.AccountEnabled -eq $false) {continue}
```
**Step 5 — Pulled MFA methods for each user**
- Called Graph API once per user to get their registered authentication methods
- Used $user.Id as the unique identifier to query the right account
- Returns a list of all methods registered — password, authenticator app, phone, etc.
```powershell
$methods = Get-MgUserAuthenticationMethod -UserId $user.Id
```
**Step 6 — Checked method count and classified each user**
- Count of 1 means only a password is registered — no MFA at all
- Count greater than 1 means at least one additional method is registered — MFA enabled
- Printed NO MFA users in red and HAS MFA users in green
- Added each user to the correct list as a custom object with five properties
```powershell
if ($methods.Count -eq 1) {
Write-Host "NO MFA: $($user.DisplayName) | $($user.UserPrincipalName)" -ForegroundColor Red

$noMFA.Add([PSCustomObject]@{
	DisplayName = $user.DisplayName
	UserPrincipalName = $user.UserPrincipalName
	AccountEnabled = $user.AccountEnabled
	MFAStatus = "No MFA"
	MethodCount = $methods.Count
})
} else {
Write-Host "HAS MFA: $($user.DisplayName) | $($user.UserPrincipalName)" -ForegroundColor Green

$hasMFA.Add([PSCustomObject]@{
	DisplayName = $user.DisplayName
	UserPrincipalName = $user.UserPrincipalName
	AccountEnabled = $user.AccountEnabled
	MFAStatus = "MFA Enabled"
	MethodCount = $methods.Count
})
}
Write-Host "`nScan complete" -ForegroundColor Green
Write-Host "No MFA: $($noMFA.Count)" -ForegroundColor Red
Write-Host "MFA Enabled: $($hasMFA.Count)" -ForegroundColor Green
```
**Step 7 — Exported no MFA users to CSV**
- Exported only the $noMFA list — the at-risk accounts
- Saved to Desktop as NoMFA_Report.csv
- Report includes: DisplayName, UserPrincipalName, AccountEnabled, MFAStatus, MethodCount
```powershell
$noMFA | Export-Csv -Path "$HOME/Desktop/NoMFA_Report.csv" -NoTypeInformation
```
**Step 8 — Calculated enabled account total for summary**
- Filtered the full users list to only enabled accounts
- Used .Count to get the number
- Used in summary to show how many active accounts were actually scanned
```powershell
$totalEnabled = ($users | Where-Object AccountEnabled -eq $true).Count
```
**Step 9 — Printed formatted summary**
- Printed a clean summary box showing total users, enabled accounts, MFA coverage and gap
- Green for MFA enabled count, red for no MFA count
```powershell
Write-Host "====================================="
Write-Host " MFA SCAN SUMMARY"
Write-Host " Total users found: $($users.Count)"
Write-Host " Enabled accounts : $totalEnabled"
Write-Host " Have MFA         : $($hasMFA.Count)"
Write-Host " No MFA           : $($noMFA.Count)"
```
**Step 10 — Reconnected with additional scope for group management**
- Disconnected existing session to clear permissions
- Reconnected adding Group.ReadWrite.All — needed to create groups and add members
```powershell
Connect-MgGraph -Scopes "User.Read.All","UserAuthenticationMethod.Read.All","Group.ReadWrite.All" -NoWelcome
```
**Step 11 — Created security group via PowerShell**
- Created security group directly from PowerShell without touching the portal
- Group name: No MFA Users
- Set SecurityEnabled: True and MailEnabled: False — security group not a distribution list
- Group ID returned: d956dc52-0279-498c-bda3-df34b1f61fd2
```powershell
$group = New-MgGroup -DisplayName "No MFA Users" 
    -Description "Users with no MFA registered - auto populated by script" 
    -MailEnabled:$false 
    -SecurityEnabled:$true 
    -MailNickname "NoMFAUsers"
```
<img width="899" height="116" alt="Screenshot 2026-07-25 at 14 44 51" src="https://github.com/user-attachments/assets/1a7a8e89-936c-4b41-9969-338598ee73e2" />

**Step 12 — Added automation loop to add flagged users to group**
- Looped through every user in the $noMFA list
- Checked if user was already a member using Get-MgGroupMember
- If already a member — incremented $skipped counter
- If not a member — added using New-MgGroupMember and incremented $added counter
- Wrapped each operation in try/catch to handle errors without crashing the script
- $added++ and $skipped++ track running totals throughout the loop
```powershell
$groupId = "d956dc52-0279-498c-bda3-df34b1f61fd2"
$added   = 0
$skipped = 0
foreach ($user in $noMFA) {
    $mgUser = Get-MgUser -UserId $user.UserPrincipalName
    try {
        $existing = Get-MgGroupMember -GroupId $groupId | Where-Object { $_.Id -eq $mgUser.Id }
        if ($existing) { $skipped++ }
        else {
            New-MgGroupMember -GroupId $groupId -DirectoryObjectId $mgUser.Id
            $added++
        }
    } catch {
        Write-Host "Failed: $($_.Exception.Message)" -ForegroundColor Yellow
    }
}
```
**Step 13 — Ran the full automated script**
- Script scanned all 102 users
- Found 90 users with no MFA
- Automatically added all 90 to the No MFA Users security group
- Final summary: Users added: 90 | Already in: 0

<img width="741" height="804" alt="Screenshot 2026-07-25 at 14 43 56" src="https://github.com/user-attachments/assets/9271f53b-b270-4b50-b4a4-e2d2c556c478" />
<img width="360" height="882" alt="Screenshot 2026-07-25 at 14 44 17" src="https://github.com/user-attachments/assets/ad9db6b8-3ae2-4ae2-992f-c5f24acbef2b" />

**Step 14 — Targeted the group in MFA Registration Campaign**
- Opened Entra portal → Protection → Authentication Methods → Registration Campaign
- Under Include → selected the No MFA Users group created by the script
- Campaign now automatically prompts all 90 flagged users to register MFA on next sign in
- No manual intervention needed — script finds the users, group gets populated, campaign targets the group

<img width="2844" height="1702" alt="Image 7-25-26 at 14 52" src="https://github.com/user-attachments/assets/f29d717a-98aa-4804-97ed-db5d3999fd9c" />
<img width="2854" height="1148" alt="Image 7-25-26 at 14 52 (1)" src="https://github.com/user-attachments/assets/4165c205-c4c9-433a-bb5c-562e4b4f8347" />

**Step 15 — Created custom Authentication Strength policy**
- Opened Entra portal → Protection → Authentication Methods → Authentication Strengths
- Created new custom strength named Authentication Strengthening
- Enabled only strong phishing resistant methods:
- ✅ Passkeys (FIDO2)
- ✅ Microsoft Authenticator Phone Sign-in (Passwordless)
- Disabled weak methods:
- ❌ SMS
- ❌ Email OTP
- ❌ Voice calls
- ❌ Password only
<img width="2832" height="680" alt="Image 7-25-26 at 14 55" src="https://github.com/user-attachments/assets/7de42e25-e024-4932-a099-50f264c031b1" />

**Youtube video explaining project**
https://www.youtube.com/watch?v=E2qwvQav27k&t=190s
