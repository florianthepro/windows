# answer (final, nur lesbarer Code)

Interaktives Deinstallations-Tool als reines PowerShell-Skript.

## Ausführen
- Als Datei speichern, z. B. `deinst.ps1`, dann:
  `powershell -NoProfile -ExecutionPolicy Bypass -File deinst.ps1`
- Oder den kompletten Code direkt in ein **PowerShell**-Fenster einfügen.

## Steuerung (einheitlich in allen Menüs)
- **Tippen** = Suche filtern
- **Pfeil hoch / runter** = Auswahl bewegen
- **Enter** = Menü öffnen / Eintrag wählen / Option umschalten
- **Esc** = eine Ebene zurück (oben: beenden)
- **Strg+A** = alle Programme auf vorhandene Deinstaller-Datei prüfen
- **F5** = Programmliste neu aus der Registry laden

## Verhalten
- **Aktionen laufen in separaten Fenstern**: Deinstallation und die Admin-Registry-
  Entfernung (HKLM) öffnen ein eigenes `cmd`-Fenster mit `runas /user:Administrator` –
  das **Hauptfenster bleibt bedienbar**. HKCU-Einträge werden sofort entfernt.
- **Frische Daten**: die Liste wird nach jeder Aktion, bei `Strg+A` und mit `F5` neu
  aus der Registry gelesen, damit kein alter Stand stehen bleibt.
- **Ehrlicher Status** je Programm: vorhanden / FEHLT / Windows Installer (msiexec) /
  kein Befehl / Pfad unklar. Liste ohne Systemkomponenten (wie Einstellungen→Apps).
- **Menü** (Enter): Deinstallieren · Befehl kopieren · Advanced (Flags interaktiv,
  danach zurück ins Menü) · bei verwaisten Einträgen „Nur Registry-Eintrag entfernen".

## Code
```powershell
# Interaktives Deinstallations-Tool
$ErrorActionPreference='Stop'; $admin='Administrator'
$keys='HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*','HKLM:\Software\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*','HKCU:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*'

# Programme FRISCH aus der Registry lesen (wie Einstellungen->Apps, ohne Systemkomponenten)
function Get-Apps{ @(Get-ItemProperty $keys -ErrorAction SilentlyContinue | Where-Object { $_.DisplayName -and $_.DisplayName.Trim() -ne '' -and $_.SystemComponent -ne 1 } | Sort-Object DisplayName -Unique) }

function Get-Cmd($app){ $u=$app.QuietUninstallString; if([string]::IsNullOrWhiteSpace($u)){ $u=$app.UninstallString }; return $u }
function Get-ExeFromCmd($u){ if([string]::IsNullOrWhiteSpace($u)){ return '' }; if($u -match '^\s*"([^"]+)"'){ $e=$matches[1] } elseif($u -match '^\s*(\S+)'){ $e=$matches[1] } else { return '' }; return [Environment]::ExpandEnvironmentVariables($e) }
# Zustand: present|missing|msi|none|unknown
function Get-CmdState($u){ if([string]::IsNullOrWhiteSpace($u)){ return 'none' }; if($u -match '(?i)msiexec'){ return 'msi' }; $e=Get-ExeFromCmd $u; if([string]::IsNullOrWhiteSpace($e)){ return 'unknown' }; if(Test-Path -LiteralPath $e){ return 'present' } else { return 'missing' } }
function Get-StateText($s){ switch($s){ 'present'{'Deinstaller-Datei: vorhanden'} 'missing'{'Deinstaller-Datei: FEHLT'} 'msi'{'Windows Installer (msiexec)'} 'none'{'Kein Deinstallationsbefehl hinterlegt'} default{'Deinstaller-Pfad unklar'} } }
function Test-Orphan($u){ $s=Get-CmdState $u; return ($s -eq 'missing' -or $s -eq 'none') }

# Befehl als Administrator in einem SEPARATEN Fenster starten (Hauptfenster bleibt frei)
function Start-AdminWindow($inner){
  $enc=[Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes($inner))
  Start-Process -FilePath 'cmd.exe' -ArgumentList ('/k runas /user:'+$admin+' powershell -NoProfile -EncodedCommand '+$enc)
}

# Deinstallationsbefehl (beliebige Kommandozeile) erhoeht in separatem Fenster ausfuehren
function Invoke-AsAdmin($cmdline){
  if([string]::IsNullOrWhiteSpace($cmdline)){ Write-Host 'Kein Befehl.' -ForegroundColor Yellow; Start-Sleep -Seconds 1; return }
  $q=$cmdline.Replace("'","''")
  $inner="`$f=Join-Path `$env:TEMP ('u'+`$PID+'.cmd'); Set-Content -LiteralPath `$f -Value '$q' -Encoding OEM; & `$env:ComSpec /c `$f; Remove-Item -LiteralPath `$f -Force"
  Start-AdminWindow $inner
  Write-Host 'Deinstallation in separatem Fenster gestartet (F5 = Liste aktualisieren).' -ForegroundColor Green; Start-Sleep -Seconds 1
}

# Registry-Eintraege entfernen: HKCU sofort, HKLM in separatem Admin-Fenster
function Remove-RegKeys($paths){
  $paths=@($paths | Where-Object { $_ }); if($paths.Count -eq 0){ return }
  $hkcu=@($paths | Where-Object { $_ -match '(?i)HKEY_CURRENT_USER' })
  $hklm=@($paths | Where-Object { $_ -notmatch '(?i)HKEY_CURRENT_USER' })
  foreach($x in $hkcu){ Remove-Item -LiteralPath $x -Recurse -Force -ErrorAction SilentlyContinue }
  if($hklm.Count -gt 0){
    $lits=($hklm | ForEach-Object { "'"+([string]$_).Replace("'","''")+"'" }) -join ','
    $inner='$p=@('+$lits+'); foreach($x in $p){ Remove-Item -LiteralPath $x -Recurse -Force -ErrorAction SilentlyContinue }'
    Start-AdminWindow $inner
  }
  Write-Host ('Entfernt - HKCU sofort: '+$hkcu.Count+' | HKLM (separates Fenster): '+$hklm.Count+'  (F5 = aktualisieren)') -ForegroundColor Green; Start-Sleep -Seconds 1
}

# --- Menue-Bausteine (Pfeil-Navigation, Esc=zurueck) ------------------------
function Read-Choice($header,$options){
  $i=0
  while($true){
    [Console]::Clear(); foreach($h in $header){ Write-Host $h -ForegroundColor Cyan }
    Write-Host 'Pfeile = navigieren | Enter = waehlen | Esc = zurueck' -ForegroundColor DarkGray; Write-Host ''
    for($j=0;$j -lt $options.Count;$j++){ if($j -eq $i){ Write-Host ('> '+$options[$j]) -ForegroundColor Black -BackgroundColor Gray } else { Write-Host ('  '+$options[$j]) } }
    $k=[Console]::ReadKey($true)
    if($k.Key -eq 'UpArrow'){ if($i -gt 0){$i--} } elseif($k.Key -eq 'DownArrow'){ if($i -lt $options.Count-1){$i++} } elseif($k.Key -eq 'Enter'){ return $i } elseif($k.Key -eq 'Escape'){ return -1 }
  }
}
function Confirm-Action($msg){ return ((Read-Choice @($msg) @('Nein','Ja')) -eq 1) }

# Advanced: nur Flags umschalten, gibt den Befehl zurueck ins Aktionsmenue
function Show-Advanced($app,$base){
  $isMsi=$base -match '(?i)msiexec'
  if($isMsi){ $flags=@(@{f='/qn';d='Komplett ohne Oberflaeche (silent)'},@{f='/passive';d='Nur Fortschritt, keine Rueckfragen'},@{f='/norestart';d='Kein automatischer Neustart'},@{f='/l*v "%TEMP%\deinst.log"';d='Ausfuehrliches Logfile schreiben'}) }
  else { $flags=@(@{f='/S';d='NSIS: stille Deinstallation'},@{f='/VERYSILENT';d='Inno-Setup: komplett ohne Dialoge'},@{f='/SUPPRESSMSGBOXES';d='Inno-Setup: Meldungsboxen unterdruecken'},@{f='/norestart';d='Kein Neustart'}) }
  foreach($fl in $flags){ $base=$base.Replace(' '+$fl.f,'') }
  $on=New-Object 'bool[]' $flags.Count; $i=0
  while($true){
    $cmd=$base; for($j=0;$j -lt $flags.Count;$j++){ if($on[$j]){ $cmd+=' '+$flags[$j].f } }
    $rows=@(); for($j=0;$j -lt $flags.Count;$j++){ $rows+=('['+$(if($on[$j]){'x'}else{' '})+'] '+$flags[$j].f.PadRight(20)+$flags[$j].d) }
    $rows+='Zurueck (uebernehmen)'
    [Console]::Clear(); Write-Host ('=== Advanced: '+$app.DisplayName+' ===') -ForegroundColor Cyan
    Write-Host ('Befehl: '+$cmd) -ForegroundColor Green
    Write-Host 'Pfeile = navigieren | Enter = umschalten | Esc/Zurueck = uebernehmen' -ForegroundColor DarkGray; Write-Host ''
    for($r=0;$r -lt $rows.Count;$r++){ if($r -eq $i){ Write-Host ('> '+$rows[$r]) -ForegroundColor Black -BackgroundColor Gray } else { Write-Host ('  '+$rows[$r]) } }
    $k=[Console]::ReadKey($true)
    if($k.Key -eq 'UpArrow'){ if($i -gt 0){$i--} }
    elseif($k.Key -eq 'DownArrow'){ if($i -lt $rows.Count-1){$i++} }
    elseif($k.Key -eq 'Escape'){ return $cmd }
    elseif($k.Key -eq 'Enter'){ if($i -lt $flags.Count){ $on[$i]=-not $on[$i] } else { return $cmd } }
  }
}

# Aktionsmenue fuer ein Programm
function Show-ActionMenu($app){
  $u=Get-Cmd $app
  while($true){
    $state=Get-CmdState $u
    $opts=@()
    if($state -ne 'none'){ $opts+='Deinstallieren (als Administrator)'; $opts+='Deinstall-Befehl in Zwischenablage kopieren'; $opts+='Advanced: Befehl interaktiv anpassen' }
    if($state -eq 'missing' -or $state -eq 'none'){ $opts+='Nur Registry-Eintrag entfernen (verwaist)' }
    $opts+='Zurueck'
    $c=Read-Choice @(('=== '+$app.DisplayName+' ==='),('Befehl: '+$u),(Get-StateText $state),'') $opts
    if($c -lt 0){ return }
    $ch=$opts[$c]
    if($ch -eq 'Zurueck'){ return }
    elseif($ch -like 'Deinstallieren*'){ Invoke-AsAdmin $u; return }
    elseif($ch -like 'Deinstall-Befehl*'){ Set-Clipboard -Value $u; Write-Host 'In Zwischenablage kopiert.' -ForegroundColor Green; Start-Sleep -Seconds 1 }
    elseif($ch -like 'Advanced*'){ $u=Show-Advanced $app $u }
    elseif($ch -like 'Nur Registry*'){ if(Confirm-Action ('Registry-Eintrag von "'+$app.DisplayName+'" wirklich entfernen?')){ Remove-RegKeys @($app.PSPath); return } }
  }
}

# Globales Advanced (Strg+A): alle Programme frisch pruefen
function Show-GlobalAdvanced{
  while($true){
    $apps=Get-Apps
    $data=@(); foreach($a in $apps){ if(Test-Orphan (Get-Cmd $a)){ $data+=$a } }
    $opts=@(); foreach($a in $data){ $opts+=('[verwaist] '+$a.DisplayName) }
    $opts+=('== Alle '+$data.Count+' verwaisten Eintraege entfernen ==')
    $opts+='Zurueck'
    $c=Read-Choice @('=== Advanced: Deinstaller-Dateien pruefen (Strg+A) ===',('Geprueft: '+$apps.Count+' Programme | verwaist: '+$data.Count),'') $opts
    if($c -lt 0){ return }
    $ch=$opts[$c]
    if($ch -eq 'Zurueck'){ return }
    elseif($ch -like '== Alle*'){ if($data.Count -gt 0 -and (Confirm-Action ('Wirklich alle '+$data.Count+' verwaisten Registry-Eintraege entfernen?'))){ Remove-RegKeys @($data | ForEach-Object { $_.PSPath }) } }
    else { $a=$data[$c]; if(Confirm-Action ('Registry-Eintrag von "'+$a.DisplayName+'" entfernen?')){ Remove-RegKeys @($a.PSPath) } }
  }
}

# --- Hauptschleife: Suche (Liste wird frisch gehalten) ----------------------
$apps=Get-Apps; $filter=''; $sel=0
while($true){
  $hits=@($apps | Where-Object { $_.DisplayName -like ('*'+$filter+'*') })
  $view=@($hits | Select-Object -First 20)
  if($sel -ge $view.Count){ $sel=[Math]::Max(0,$view.Count-1) }
  [Console]::Clear()
  Write-Host 'Tippen = Suche | hoch/runter = Auswahl | Enter = Menue | Strg+A = Alle pruefen | F5 = Aktualisieren | Esc = Quit' -ForegroundColor Cyan
  Write-Host ('Suche: '+$filter+'   ('+$apps.Count+' Programme)') -ForegroundColor Cyan
  if($view.Count -eq 0){ Write-Host ' (keine Treffer)' -ForegroundColor Yellow }
  else { for($i=0;$i -lt $view.Count;$i++){ $line=($i+1).ToString()+'. '+$view[$i].DisplayName; if($i -eq $sel){ Write-Host ('> '+$line) -ForegroundColor Black -BackgroundColor Gray } else { Write-Host ('  '+$line) } } }
  $k=[Console]::ReadKey($true)
  if($k.Key -eq 'Escape'){ break }
  elseif($k.Key -eq 'F5'){ $apps=Get-Apps; $sel=0 }
  elseif(($k.Modifiers -band [ConsoleModifiers]::Control) -ne 0 -and $k.Key -eq 'A'){ Show-GlobalAdvanced; $apps=Get-Apps }
  elseif($k.Key -eq 'UpArrow'){ if($sel -gt 0){$sel--} }
  elseif($k.Key -eq 'DownArrow'){ if($sel -lt $view.Count-1){$sel++} }
  elseif($k.Key -eq 'Enter'){ if($view.Count -gt 0){ Show-ActionMenu $view[$sel]; $apps=Get-Apps } }
  elseif($k.Key -eq 'Backspace'){ if($filter.Length -gt 0){ $filter=$filter.Substring(0,$filter.Length-1); $sel=0 } }
  else { $ch=$k.KeyChar; if($ch -and -not [char]::IsControl($ch)){ $filter+=$ch; $sel=0 } }
}
```
