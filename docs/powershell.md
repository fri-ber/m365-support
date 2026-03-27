---
layout: default
title: PowerShell Scripts
nav_section: powershell
nav_section_label: PowerShell
---

# ⚡ PowerShell Scripts

Nützliche Scripts für den IT-Alltag mit Active Directory, Exchange Online und Microsoft Graph.

---

## Vorbereitung

### Execution Policy setzen

```powershell
Set-ExecutionPolicy RemoteSigned -Scope CurrentUser
```

### Module installieren

```powershell
# Active Directory (RSAT)
# → Windows Features → RSAT: Active Directory DS Tools

# Exchange Online
Install-Module ExchangeOnlineManagement -Scope CurrentUser

# Microsoft Graph
Install-Module Microsoft.Graph -Scope CurrentUser

# Azure AD (Legacy, aber noch verbreitet)
Install-Module AzureAD -Scope CurrentUser
```

### Verbindungen herstellen

```powershell
# Exchange Online
Connect-ExchangeOnline -UserPrincipalName admin@firma.ch

# Microsoft Graph
Connect-MgGraph -Scopes "User.Read.All", "Directory.Read.All"

# Alle Sessions trennen
Disconnect-ExchangeOnline -Confirm:$false
Disconnect-MgGraph
```

---

## Active Directory

### Passwort-Ablauf prüfen

```powershell
# User anzeigen, deren Passwort in X Tagen abläuft
$DaysUntilExpiry = 14

Get-ADUser -Filter * -Properties PasswordLastSet, PasswordNeverExpires |
  Where-Object { -not $_.PasswordNeverExpires } |
  Select-Object Name, SamAccountName,
    @{ Name='ExpiresOn'; Expression={ $_.PasswordLastSet.AddDays(90) } } |
  Where-Object { $_.ExpiresOn -lt (Get-Date).AddDays($DaysUntilExpiry) } |
  Sort-Object ExpiresOn |
  Format-Table -AutoSize
```

### Deaktivierte User aufräumen

```powershell
# Deaktivierte AD-User in "Deaktiviert"-OU verschieben
$ZielOU = "OU=Deaktiviert,DC=firma,DC=ch"

Get-ADUser -Filter { Enabled -eq $false } -Properties LastLogonDate |
  Where-Object { $_.DistinguishedName -notlike "*Deaktiviert*" } |
  ForEach-Object {
    Move-ADObject -Identity $_.DistinguishedName -TargetPath $ZielOU
    Write-Host "Verschoben: $($_.Name)" -ForegroundColor Yellow
  }

Write-Host "✅ Fertig" -ForegroundColor Green
```

### Inaktive User finden

```powershell
# User die sich seit X Tagen nicht eingeloggt haben
$InactiveDays = 90
$InactiveDate  = (Get-Date).AddDays(-$InactiveDays)

Get-ADUser -Filter { LastLogonDate -lt $InactiveDate -and Enabled -eq $true } `
  -Properties LastLogonDate, Department |
  Select-Object Name, SamAccountName, LastLogonDate, Department |
  Sort-Object LastLogonDate |
  Export-Csv "./Inaktive-User.csv" -NoTypeInformation -Encoding UTF8

Write-Host "✅ Export: Inaktive-User.csv" -ForegroundColor Green
```

### Gruppen-Mitgliedschaft exportieren

```powershell
# Alle Mitglieder einer AD-Gruppe exportieren
$GruppenName = "IT-Admins"

Get-ADGroupMember -Identity $GruppenName -Recursive |
  Get-ADUser -Properties Department, Title |
  Select-Object Name, SamAccountName, Department, Title |
  Export-Csv "./$GruppenName-Mitglieder.csv" -NoTypeInformation -Encoding UTF8
```

---

## Exchange Online

### Shared Mailbox Berechtigungen setzen

```powershell
Connect-ExchangeOnline

$Mailbox = "support@firma.ch"
$User    = "hans.muster@firma.ch"

# Full Access (mit Auto-Mapping)
Add-MailboxPermission -Identity $Mailbox `
  -User $User -AccessRights FullAccess -AutoMapping $true

# Send As
Add-RecipientPermission -Identity $Mailbox `
  -Trustee $User -AccessRights SendAs -Confirm:$false

Write-Host "✅ Berechtigungen gesetzt für $User" -ForegroundColor Green
```

### Mailbox-Statistiken abrufen

```powershell
# Postfachgrösse und Elementanzahl
Get-Mailbox -ResultSize Unlimited |
  Get-MailboxStatistics |
  Select-Object DisplayName, TotalItemSize, ItemCount |
  Sort-Object TotalItemSize -Descending |
  Select-Object -First 20 |
  Format-Table -AutoSize
```

### E-Mail-Weiterleitungen prüfen

```powershell
# Alle aktiven Weiterleitungen (potentielles Sicherheitsrisiko!)
Get-Mailbox -ResultSize Unlimited |
  Where-Object { $_.ForwardingSmtpAddress -ne $null } |
  Select-Object DisplayName, UserPrincipalName, ForwardingSmtpAddress, DeliverToMailboxAndForward |
  Export-Csv "./Weiterleitungen.csv" -NoTypeInformation -Encoding UTF8

Write-Host "⚠️  Weiterleitung-Report erstellt: Weiterleitungen.csv" -ForegroundColor Yellow
```

---

## Microsoft Graph (MS Graph API)

### Lizenz-Report

```powershell
Connect-MgGraph -Scopes "User.Read.All", "Directory.Read.All"

# Alle lizenzierten User
Get-MgUser -All -Filter "assignedLicenses/`$count ne 0" `
  -ConsistencyLevel eventual -CountVariable count |
  Select-Object DisplayName, UserPrincipalName, AccountEnabled |
  Export-Csv "./Lizenzen-Report.csv" -NoTypeInformation -Encoding UTF8

Write-Host "✅ Lizenzen-Report.csv erstellt ($count User)" -ForegroundColor Green
```

### Sign-In Logs abrufen

```powershell
Connect-MgGraph -Scopes "AuditLog.Read.All"

# Letzte 100 fehlgeschlagene Logins
Get-MgAuditLogSignIn -Filter "status/errorCode ne 0" -Top 100 |
  Select-Object UserDisplayName, UserPrincipalName, CreatedDateTime,
    @{ Name='Error'; Expression={ $_.Status.FailureReason } },
    IPAddress, Location |
  Sort-Object CreatedDateTime -Descending |
  Format-Table -AutoSize
```

### Gäste ohne Aktivität entfernen

```powershell
Connect-MgGraph -Scopes "User.ReadWrite.All", "AuditLog.Read.All"

$InactiveDays = 180
$Cutoff = (Get-Date).AddDays(-$InactiveDays).ToString("yyyy-MM-ddTHH:mm:ssZ")

# Gäste die sich seit X Tagen nicht eingeloggt haben
$InactiveGuests = Get-MgUser -All `
  -Filter "userType eq 'Guest' and signInActivity/lastSignInDateTime le $Cutoff" `
  -Property DisplayName, UserPrincipalName, SignInActivity

$InactiveGuests | ForEach-Object {
  Write-Host "Entfernen: $($_.DisplayName)" -ForegroundColor Red
  # Remove-MgUser -UserId $_.Id  # ← auskommentiert für Sicherheit!
}

Write-Host "✅ $($InactiveGuests.Count) inaktive Gäste gefunden" -ForegroundColor Yellow
```

> ⚠️ `Remove-MgUser` ist auskommentiert — erst nach Überprüfung aktivieren!

---

## Nützliche Snippets

### Ausgabe als GridView (GUI)

```powershell
Get-ADUser -Filter * -Properties Department |
  Select-Object Name, SamAccountName, Department |
  Out-GridView -Title "AD User" -PassThru
```

### Passwort sicher als SecureString

```powershell
$Pw = Read-Host "Neues Passwort" -AsSecureString
Set-ADAccountPassword -Identity "user.name" -NewPassword $Pw -Reset
```

### Transcript (Protokoll) starten

```powershell
Start-Transcript -Path "C:\Logs\ps-session-$(Get-Date -Format 'yyyyMMdd-HHmm').log"
# ... Script hier ...
Stop-Transcript
```
