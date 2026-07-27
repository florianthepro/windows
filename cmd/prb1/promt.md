aktueller befehl:
```
powershell -NoProfile -ExecutionPolicy Bypass -Command "$ErrorActionPreference='Stop'; $admin='.\Administrator'; $items=(Get-ItemProperty 'HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*','HKLM:\Software\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*','HKCU:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*' -ErrorAction SilentlyContinue | Where-Object { $_.DisplayName -and $_.DisplayName.Trim() -ne '' } | Select-Object -ExpandProperty DisplayName -Unique); $filter=''; $selIndex=0; Write-Host 'Tippe = Suche | Strg+U = Uninstall | Strg+Q = Quit' -ForegroundColor Cyan; while($true){ $key=[Console]::ReadKey($true); if($key.Key -eq 'Escape'){break}; if(($key.Modifiers -band [ConsoleModifiers]::Control) -ne 0){ if($key.Key -eq 'U'){ $hits=$items | Where-Object { $_ -like ('*'+$filter+'*') } | Sort-Object; if(-not $hits){ Write-Host 'Keine Treffer (für Uninstall).' -ForegroundColor Yellow; continue }; $sel=$hits | Select-Object -First 1; $uninstProps=Get-ItemProperty 'HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*','HKLM:\Software\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*','HKCU:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*' -ErrorAction SilentlyContinue | Where-Object { $_.DisplayName -eq $sel } | Select-Object -First 1; $uninst=$uninstProps.QuietUninstallString; if([string]::IsNullOrWhiteSpace($uninst)){ $uninst=$uninstProps.UninstallString }; if([string]::IsNullOrWhiteSpace($uninst)){ Write-Host ('Kein UninstallString/QuietUninstallString für: '+$sel) -ForegroundColor Yellow; continue }; $cmd='cmd.exe /c \"'+$uninst+'\"'; runas /user:$admin $cmd; Write-Host ('Uninstall gestartet: '+$sel) -ForegroundColor Green; continue } elseif($key.Key -eq 'Q'){ Write-Host 'Quit.' -ForegroundColor Cyan; break } else { continue } } else { if($key.Key -eq 'Backspace'){ if($filter.Length -gt 0){ $filter=$filter.Substring(0,$filter.Length-1) } } elseif($key.Key -eq 'Enter'){ continue } else { $ch=$key.KeyChar; if($ch -and -not [char]::IsControl($ch)){ $filter += $ch } } }; $hits=$items | Where-Object { $_ -like ('*'+$filter+'*') } | Sort-Object; if($hits.Count -gt 20){ $hits=$hits | Select-Object -First 20 }; [Console]::Clear(); Write-Host 'Tippe = Suche | Strg+U = Uninstall | Strg+Q = Quit' -ForegroundColor Cyan; Write-Host ('Tippe: ' + $filter) -ForegroundColor Cyan; if(-not $hits){ Write-Host ' (keine Treffer)' -ForegroundColor Yellow } else { for($i=0; $i -lt $hits.Count; $i++){ Write-Host (($i+1).ToString()+'. '+$hits[$i]) } } }"
```
---
tst:
```
Tippe = Suche | Strg+U = Uninstall | Strg+Q = Quit
Tippe: Fire
1. M
Syntax von RUNAS:

RUNAS [ [/noprofile | /profile] [/env] [/savecred | /netonly] ]
        /user:<Benutzername> Programm

RUNAS [ [/noprofile | /profile] [/env] [/savecred] ]
        /smartcard [/user:<Benutzername>] Programm

RUNAS [ [/machine:<MachineType>] ] /trustlevel:<TrustLevel> program

   /noprofile        Legt fest, dass das Benutzerprofil nicht geladen werden
                     soll. Führt dazu, dass die Anwendung schneller geladen
                     wird. Dies kann bei einigen Anwendungen zu Fehlern führen.
   /profile          Legt fest, dass das Benutzerprofil geladen werden soll.
                     Dies ist die Standardeinstellung.
   /env              Verwendet die aktuelle Umgebung anstatt der des Benutzers.
   /netonly          Falls Anmeldeinformationen nur für den Remotezugriff
                     gültig sind.
   /savecred         Verwendet Anmeldeinformationen, die von einem anderen
   /smartcard        Falls Anmeldeinformationen von einer Smartcard zur
                     Verfügung gestellt werden.
   /user             <Benutzername> muss in der Form Benutzer@Domäne oder
                     Domäne\Benutzer angegeben werden.
   /showtrustlevels  Zeigt die Vertrauensstufen an, die als Argumente zu
                     /trustlevel verwendet werden können.
   /trustlevel       <Stufe> sollte eine der in /showtrustlevels aufgelisteten
                     Stufen sein.
   /machine          gibt die Computerarchitektur des Prozesses an.
                     <MachineType> sollte einer von x86|amd64|arm|arm64 sein.
   Programm           befehlszeile für EXE.  Beispiele finden Sie unten.

Beispiele:
> runas /noprofile /user:mymachine\administrator cmd
> runas /profile /env /user:mydomain\admin "mmc %windir%\system32\dsa.msc"
> runas /env /user:Benutzer@Domäne.Microsoft.com "notepad \"Meine Datei.txt\""

HINWEIS: Geben Sie das Benutzerkennwort nur ein, wenn Sie dazu aufgefordert
werden.
Hinweis: /profile ist nicht mit /netonly kompatibel.
NOTE:  /savecred ist mit /smartcard nicht kompatibel.
Uninstall gestartet: Mozilla Firefox (x64 de)
```

beachte:
nur 1 zusätzliche md (answer.md) dem repo beifügen
1. name bei suche abgeschnitten bzw beim programm
2. uninstall sollte sauber mit runas admin sein
3. erzeuge answer.md die einen einzeiligen ps befehl enthält der das umsetzt
```
