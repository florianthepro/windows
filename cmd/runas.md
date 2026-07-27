```
runas /user:Administrator "cmd.exe /k"
```
```
runas /user:Administrator "cmd.exe /c start appwiz.cpl"
```
- <`user`>: `user`/`domain\user`/`.\user` (lokales Konto)/`computer\user`
- <`Administrator`>: `Administrator`/`Name` (Kontoname)
- <`cmd.exe`>: `cmd.exe`/`appwiz.cpl`/`Programm` (Beispielprogramm)
- <`/k`>: `/k`/`/c`/`/s`/`/d` (als Argument für `cmd.exe`)
  - <`/K`> führt den Befehl aus und lässt die Konsole danach offen
  - <`/c`> führt den Befehl aus und beendet danach die Konsole
  - <`/s`> schaltet die Verarbeitung von Sonderzeichen/Zeichenfolgen für den folgenden Befehl ein
  - <`/d`> ignoriert AutoRun-Details
