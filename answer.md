# answer

Korrigierter Einzeiler für das interaktive Uninstall-Tool (`cmd/prb1/promt.md`).

## Behobene Fehler
1. **Name abgeschnitten (`Tippe: Fire` → `1. M`)**
   Bei genau einem Treffer liefert `Where-Object` einen einzelnen String statt eines
   Arrays; `$hits[$i]` indexiert dann *in den String* und gibt nur das erste Zeichen
   zurück. Fix: beide `$hits`-Zuweisungen mit `@(...)` zu einem Array erzwingen.
2. **`runas` schlug fehl (Syntax-Hilfe wurde ausgegeben)**
   `cmd.exe /c "..."` mit dem bereits zitierten UninstallString erzeugte verschachtelte
   Anführungszeichen, die beim Aufruf von `runas` zerbrachen. Fix: der Uninstall-Befehl
   wird in eine temporäre `.cmd` geschrieben und über ein Base64-kodiertes
   `powershell -EncodedCommand` gestartet – die `runas`-Kommandozeile enthält damit
   keine Anführungszeichen mehr und wird sauber als `Administrator` ausgeführt.

## Einzeiler
```
powershell -NoProfile -ExecutionPolicy Bypass -Command "$ErrorActionPreference='Stop'; $admin='Administrator'; $keys='HKLM:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*','HKLM:\Software\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\*','HKCU:\Software\Microsoft\Windows\CurrentVersion\Uninstall\*'; $items=(Get-ItemProperty $keys -ErrorAction SilentlyContinue | Where-Object { $_.DisplayName -and $_.DisplayName.Trim() -ne '' } | Select-Object -ExpandProperty DisplayName -Unique); $filter=''; Write-Host 'Tippe = Suche | Strg+U = Uninstall | Strg+Q = Quit' -ForegroundColor Cyan; while($true){ $key=[Console]::ReadKey($true); if($key.Key -eq 'Escape'){break}; if(($key.Modifiers -band [ConsoleModifiers]::Control) -ne 0){ if($key.Key -eq 'U'){ $hits=@($items | Where-Object { $_ -like ('*'+$filter+'*') } | Sort-Object); if($hits.Count -eq 0){ Write-Host 'Keine Treffer (fuer Uninstall).' -ForegroundColor Yellow; continue }; $sel=$hits[0]; $p=Get-ItemProperty $keys -ErrorAction SilentlyContinue | Where-Object { $_.DisplayName -eq $sel } | Select-Object -First 1; $u=$p.QuietUninstallString; if([string]::IsNullOrWhiteSpace($u)){ $u=$p.UninstallString }; if([string]::IsNullOrWhiteSpace($u)){ Write-Host ('Kein UninstallString fuer: '+$sel) -ForegroundColor Yellow; continue }; $inner='$f=Join-Path $env:TEMP (''u''+$PID+''.cmd''); Set-Content -LiteralPath $f -Value '+[char]39+$u.Replace([char]39,[char]39+[char]39)+[char]39+' -Encoding OEM; & $env:ComSpec /c $f; Remove-Item -LiteralPath $f -Force'; $enc=[Convert]::ToBase64String([Text.Encoding]::Unicode.GetBytes($inner)); $prog='powershell -NoProfile -EncodedCommand '+$enc; runas /user:$admin $prog; Write-Host ('Uninstall als Administrator gestartet: '+$sel) -ForegroundColor Green; continue } elseif($key.Key -eq 'Q'){ Write-Host 'Quit.' -ForegroundColor Cyan; break } else { continue } } else { if($key.Key -eq 'Backspace'){ if($filter.Length -gt 0){ $filter=$filter.Substring(0,$filter.Length-1) } } elseif($key.Key -eq 'Enter'){ continue } else { $ch=$key.KeyChar; if($ch -and -not [char]::IsControl($ch)){ $filter += $ch } } }; $hits=@($items | Where-Object { $_ -like ('*'+$filter+'*') } | Sort-Object); if($hits.Count -gt 20){ $hits=$hits | Select-Object -First 20 }; [Console]::Clear(); Write-Host 'Tippe = Suche | Strg+U = Uninstall | Strg+Q = Quit' -ForegroundColor Cyan; Write-Host ('Tippe: ' + $filter) -ForegroundColor Cyan; if(-not $hits){ Write-Host ' (keine Treffer)' -ForegroundColor Yellow } else { for($i=0; $i -lt $hits.Count; $i++){ Write-Host (($i+1).ToString()+'. '+$hits[$i]) } } }"
```

## Test
1. Einzeiler in `cmd.exe` einfügen.
2. `Fire` tippen → Liste zeigt den **vollen** Namen `1. Mozilla Firefox (x64 de)`.
3. `Strg+U` → `runas`-Passwortabfrage für `Administrator` (keine Syntax-Hilfe),
   nach Eingabe startet der Uninstaller erhöht.
4. `Strg+Q` / `Esc` beendet.
