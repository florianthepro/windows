```
runas /user:Administrator "cmd.exe /k"
```
```
runas /user:Administrator "appwiz.cpl /k"
```
- `<user>`: `user`/`domain\user`/`.\user` (lokales Konto)/`computer\user`
- `<Administrator>`: `Administrator`/`Name` (Kontoname)
- `<cmd.exe>`: `cmd.exe`/`appwiz.cpl`/`Programm` (Beispielprogramm)
- `<k>`: `/k`/`/c`/`/s`/`/d` (als Argument für `cmd.exe`)
- - w
