---
layout: default
title: IT-Support FAQ
nav_section: support
nav_section_label: IT-Support
---

# 🛠️ IT-Support FAQ

Häufige Probleme im IT-Support mit schnellen Lösungen.

---

## Outlook

### Outlook startet nicht / hängt beim Laden

**Ursachen:** Beschädigtes Profil, defekter Add-in, beschädigtes Navigationsfenster

```batch
# Outlook im abgesicherten Modus starten (Add-ins deaktiviert)
outlook.exe /safe

# Navigationsfenster zurücksetzen
outlook.exe /resetnavpane

# Profil reparieren (Autodiscover)
outlook.exe /cleanprofile
```

Wenn das Problem im Safe-Mode nicht auftritt: Add-ins deaktivieren unter
*Datei → Optionen → Add-Ins → COM-Add-Ins verwalten*

### Outlook fragt ständig nach Passwort

1. **Credential Manager** öffnen (Windows-Suche)
2. *Windows-Anmeldeinformationen* → alle `MicrosoftOffice` / `Outlook`-Einträge löschen
3. Outlook neu starten → bei Aufforderung neu anmelden

### Outlook OST-Datei reparieren

```batch
# OST → PST Reparatur-Tool
"C:\Program Files\Microsoft Office\root\Office16\SCANPST.EXE"
```

Pfad je nach Office-Version anpassen. OST-Datei liegt typischerweise unter:
`%LOCALAPPDATA%\Microsoft\Outlook\`

---

## OneDrive

### OneDrive Sync hängt / Fehler

**Schnell-Reset:**
1. OneDrive in Taskleiste → *Pause synchronization*
2. Task-Manager → `OneDrive.exe` beenden
3. OneDrive neu starten (Windows-Suche → OneDrive)

**Vollständiger Reset:**
```batch
%localappdata%\Microsoft\OneDrive\onedrive.exe /reset
```

Danach: OneDrive manuell neu starten und Sync abwarten.

### OneDrive-Speicher voll

```powershell
# OneDrive-Nutzung per Graph abrufen
Connect-MgGraph -Scopes "Files.Read.All"

$drive = Get-MgUserDefaultDrive -UserId "user@firma.ch"
$used  = [math]::Round($drive.Quota.Used / 1GB, 2)
$total = [math]::Round($drive.Quota.Total / 1GB, 2)

Write-Host "OneDrive: $used GB von $total GB belegt"
```

---

## Microsoft Teams

### Teams startet nicht / Absturz beim Start

```batch
# Teams Cache leeren
taskkill /F /IM ms-teams.exe
rd /s /q "%appdata%\Microsoft\Teams\Cache"
rd /s /q "%appdata%\Microsoft\Teams\blob_storage"
rd /s /q "%appdata%\Microsoft\Teams\databases"
rd /s /q "%appdata%\Microsoft\Teams\GPUCache"
rd /s /q "%appdata%\Microsoft\Teams\IndexedDB"
rd /s /q "%appdata%\Microsoft\Teams\Local Storage"
```

Danach Teams neu starten.

### Kein Audio in Meetings

1. Teams → **...** (oben rechts) → *Settings* → *Devices*
2. Richtiges Mikrofon / Lautsprecher wählen
3. Testen mit dem integrierten Test-Anruf (*Make a test call*)

> ⚠️ Bei mehreren Audiogeräten (USB-Headset + integriert): Windows Standard-Gerät und Teams-Gerät müssen übereinstimmen.

### Teams-Meeting ohne Video

Häufige Ursache: Kamera-Zugriff durch Windows blockiert.

1. *Windows-Einstellungen* → *Datenschutz* → *Kamera*
2. **Kamerazugriff für Apps zulassen** → Teams aktivieren

---

## MFA / Authenticator

### MFA-Code kommt nicht an

1. Sicherstellen, dass die **richtige** Authenticator-App verwendet wird (Microsoft Authenticator)
2. Uhrzeit auf dem Smartphone prüfen — muss korrekt/synchronisiert sein!
3. Im **Entra Admin Center** → User auswählen → *Authentication methods* prüfen

### MFA für User zurücksetzen

```powershell
# MFA-Methoden per Graph zurücksetzen
Connect-MgGraph -Scopes "UserAuthenticationMethod.ReadWrite.All"

$userId = "user@firma.ch"

# Alle Methoden auflisten
Get-MgUserAuthenticationMethod -UserId $userId

# Microsoft Authenticator-Methode entfernen (Beispiel)
# Remove-MgUserAuthenticationMicrosoftAuthenticatorMethod -UserId $userId -MicrosoftAuthenticatorAuthenticationMethodId "<ID>"
```

Alternativ im **Entra Admin Center**:
*Users* → User auswählen → *Authentication methods* → *Require re-register MFA*

---

## VPN & Netzwerk

### VPN verbindet sich nicht

```batch
# DNS-Cache leeren
ipconfig /flushdns

# Netzwerk-Stack zurücksetzen (Admin!)
netsh int ip reset
netsh winsock reset

# Danach PC neu starten
```

### Netzlaufwerk verbinden

```batch
# Dauerhaft verbinden
net use Z: \\server\freigabe /persistent:yes

# Mit Anmeldedaten
net use Z: \\server\freigabe /user:DOMAIN\username Passwort /persistent:yes
```

```powershell
# PowerShell Variante
New-PSDrive -Name "Z" -PSProvider FileSystem `
  -Root "\\server\freigabe" -Persist -Credential (Get-Credential)
```

---

## Drucker & Druckspooler

### Drucker offline / Druckaufträge hängen

```powershell
# Druckspooler neu starten (Admin)
Stop-Service Spooler -Force
Remove-Item "$env:SystemRoot\System32\spool\PRINTERS\*" -Force -ErrorAction SilentlyContinue
Start-Service Spooler

Write-Host "✅ Spooler neu gestartet, Queue geleert" -ForegroundColor Green
```

### Drucker per PowerShell hinzufügen

```powershell
# Netzwerkdrucker hinzufügen
Add-Printer -ConnectionName "\\printserver\DruckerName"

# Lokaler Drucker
Add-PrinterPort -Name "IP_192.168.1.100" -PrinterHostAddress "192.168.1.100"
Add-Printer -Name "HP LaserJet" -DriverName "HP LaserJet P2015 PCL6" `
  -PortName "IP_192.168.1.100"
```

---

## Windows / Allgemein

### Anmeldung schlägt fehl (Login-Loop)

Cached Credentials sind nach einer Passwortänderung veraltet:

1. **Credential Manager** öffnen
2. *Windows-Anmeldeinformationen* → alle veralteten Einträge löschen
3. Ggf. `%LOCALAPPDATA%\Microsoft\` Ordner bereinigen

### BitLocker-Recovery Key abrufen

```powershell
# Recovery Key aus AD (on-premise)
Get-ADObject -Filter { objectClass -eq 'msFVE-RecoveryInformation' } `
  -SearchBase "DC=firma,DC=ch" `
  -Properties msFVE-RecoveryPassword |
  Select-Object Name, msFVE-RecoveryPassword
```

Für **Entra ID / Intune**: Entra Admin Center → Geräte → Gerät auswählen → *Recovery keys*

### Windows-Update erzwingen

```powershell
# Windows Update manuell anstoßen
Install-Module PSWindowsUpdate -Scope CurrentUser
Get-WindowsUpdate
Install-WindowsUpdate -AcceptAll -AutoReboot
```

---

## Schnellreferenz

| Problem | Sofort-Massnahme |
|---------|-----------------|
| Outlook hängt | `outlook.exe /safe` |
| OneDrive Sync-Fehler | Reset: `onedrive.exe /reset` |
| Teams kein Audio | Settings → Devices → Gerät prüfen |
| MFA-Code fehlt | Uhrzeit Smartphone prüfen |
| Drucker offline | Spooler neu starten |
| VPN verbindet nicht | `ipconfig /flushdns` |
| Passwort-Loop | Credential Manager leeren |
| SharePoint Zugriff verweigert | Entra → Gruppe prüfen |
