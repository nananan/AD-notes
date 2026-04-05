# AD Basi

## Local Account vs Domain Account
- Un **local account** esiste solo sulla macchina locale — è salvato nel SAM database di quel PC. Non ha nessuna visibilità nel dominio, funziona solo per autenticarsi localmente. Esempi tipici sono Administrator, Guest, o account creati manualmente sulla macchina.

- Un **domain account** è salvato nel database di Active Directory sul Domain Controller. Funziona su qualsiasi macchina joinata al dominio, e le sue credenziali vengono validate dal DC tramite Kerberos (o NTLM come fallback).

> Un caso importante da tenere a mente è l'account **Administrator** locale: se nell'ambiente non è stato deployato **LAPS**, spesso tutte le macchine hanno la stessa password di Administrator locale — il che significa che compromettere una macchina ti dà accesso a tutte le altre tramite Pass-the-Hash.
> In molti ambienti aziendali, quando il reparto IT fa il deploy delle macchine Windows, le configurano tutte con la stessa password per l'account Administrator locale, perché è più comodo da gestire. Quindi se hai l'**hash NTLM dell'Administrator locale** di PC-01, puoi provare a fare Pass-the-Hash verso PC-02, PC-03, ecc. — e se la password è la stessa (cosa frequente), entri su tutte senza conoscere la password in chiaro. Questo è esattamente il problema che LAPS (Local Administrator Password Solution) risolve: genera una password diversa e randomica per ogni macchina, salvandola in AD. Se LAPS è deployato, l'hash che hai dumpa su PC-01 non funziona da nessun'altra parte.

### Verifica LAPS
Se restituisce risultati, LAPS è deployato

```
Get-DomainComputer | select name, ms-mcs-admpwd, ms-mcs-admpwdexpirationtime
```

> Puoi leggere la password solo se hai i permessi giusti:
>
> ms-mcs-admpwd vuoto = non hai accesso, non che LAPS non c'è


## Local Account non Admin
Un local account non-admin non ti dà accesso remoto ad altre macchine per due motivi:
- Il primo è che è un account locale — esiste solo su quella macchina, quindi anche se la password fosse identica su un'altra macchina, Windows le tratta come account separati senza relazione tra loro.
- Il secondo è che anche volendo connetterti remotamente (via SMB, WinRM, ecc.), hai bisogno di privilegi amministrativi sulla macchina target. Un utente standard viene bloccato da UAC per le connessioni remote, anche se conosce la password.

> L'unica eccezione teorica sarebbe se quell'utente locale non-admin ha accesso a qualche share specifica sulla macchina target — ma è uno scenario molto raro e comunque limitato a quella risorsa, non ti dà una shell.


## Computer
I computer (workstation) possono essere anche non joinati al dominio.

Il comando `net user`:

- Su una macchina joinata al dominio → mostra gli utenti locali e puoi specificare /domain per vedere quelli del dominio.
- Su una macchina non joinata → mostra solo gli utenti locali, e se provi a fare net user /domain ottieni un errore perché non c'è nessun DC a cui parlare. Il "dominio" che vede è semplicemente il nome del workgroup (di default WORKGROUP). Se sei su una macchina non joinata al dominio, net non ti darà informazioni AD — devi prima essere su una macchina dentro il dominio per enumerarlo.

### Workgroup
- Il workgroup è il predecessore del dominio Active Directory — un modo più semplice e primitivo di raggruppare computer in una rete locale.
- In un workgroup ogni macchina è completamente autonoma: gestisce i propri account, le proprie password, i propri permessi. Non c'è nessun server centrale (nessun DC), nessuna autenticazione centralizzata, nessuna policy di gruppo. Se vuoi accedere a una risorsa su un'altra macchina del workgroup, quella macchina deve avere un account locale con le tue stesse credenziali.
- È quello che trovi tipicamente in ambienti casalinghi o piccoli uffici con 2-3 PC. Appena l'azienda cresce e ha bisogno di gestire decine o centinaia di macchine in modo centralizzato, passa ad Active Directory.
- Quando installi Windows su una macchina nuova e non la joini a nessun dominio, Windows la mette automaticamente in un workgroup chiamato WORKGROUP — è solo un nome di default, non significa nulla di speciale. Ecco perché il comando net lo mostra quando non sei in un dominio.

Per verificare se il pc è nel workgroup o joinata ad un dominio:
```
systeminfo | findstr /i "domain"
```

- Se vedi Domain: WORKGROUP — sei in un workgroup
- Se vedi un nome tipo Domain: corp.local — sei joinato a un dominio AD.


