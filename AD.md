# AD Basi

Link Usati:
- https://0xd4y.com/2023/02/28/Active-Directory-Pentesting-Notes/


## Account Computer vs Account Utente
- Gli account computer hanno password randomiche di 120 caratteri che ruotano automaticamente ogni 30 giorni — praticamente impossibili da craccare. 
- Gli account utente usati come service account invece spesso hanno password semplici e che non cambiano mai, tipo Summer2023! — e questo li rende il target perfetto del Kerberoasting.

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


# Kerberos
![alt text](images/image.png)

#### Fase 1 — AS-REQ / AS-REP (client → KDC)
Mario manda la richiesta con il timestamp cifrato. Il KDC verifica chi è Mario e gli rilascia il TGT. Questo TGT è la prova che Mario si è autenticato correttamente — è cifrato con la chiave di krbtgt, quindi Mario non può leggerlo né modificarlo, può solo trasportarlo.

#### Fase 2 — TGS-REQ / TGS-REP (client → KDC, rimanda il TGT)
Mario vuole accedere al file server. Rimanda il TGT al DC dicendo "ho già provato di essere Mario, ora ho bisogno di un ticket specifico per cifs/fileserver". Il DC decifra il TGT con la chiave di krbtgt, verifica che sia valido, e rilascia un TGS (Ticket Granting Service) — un ticket specifico per quel servizio, cifrato con la chiave del file server.

#### Il TGS è firmato con la chiave del Service Owner - Chi è il service owner
- Quando un servizio viene avviato su una macchina Windows, gira nel contesto di sicurezza di un account specifico. Quell'account è il service owner.

> Esempi concreti:
> - il file server gira tipicamente come NT AUTHORITY\SYSTEM — che in AD corrisponde all'account computer della macchina, tipo FILESERVER$. Quindi il TGS per cifs/fileserver è cifrato con l'hash di FILESERVER$.
> - un servizio SQL custom invece può girare come un utente di dominio dedicato, tipo svc-sql@corp.local. Il TGS per MSSQLSvc/sqlserver è cifrato con l'hash di svc-sql.
> - un servizio IIS può girare come svc-web@corp.local. Il TGS per http/webserver è cifrato con l'hash di svc-web.


Quando chiedi un TGS per un servizio, il DC te lo rilascia senza verificare se hai effettivamente accesso a quel servizio — ti dà semplicemente il ticket cifrato. Quel ticket è cifrato con l'hash del service owner.
- Se il service owner è un account utente di dominio (non un account computer), il suo hash è derivato da una password scelta da un umano — quindi potenzialmente debole e craccabile offline.

##### Perché il Client (Mario) rimanda il TGT e non la password?
- Il punto chiave è che Mario rimanda il TGT invece della password perché il TGT è una prova di identità già validata — il DC non deve riautenticare Mario da zero ogni volta. È come avere un badge giornaliero: lo mostri all'ingresso una volta sola al mattino, poi lo usi per aprire tutte le porte interne senza tornare ogni volta alla reception.
- La password non viaggia mai sulla rete dopo la prima fase — questo è uno dei vantaggi fondamentali di Kerberos rispetto a NTLM.

##### Quando il DC riceve il TGT nella fase 2, "verificare che sia valido" significa fare questi controlli:
- Prima cosa: lo decifra con l'hash di krbtgt. Se la decifratura produce dati sensati, significa che il TGT è stato creato da lui stesso — nessun altro conosce quella chiave, quindi non può essere stato falsificato.
- Dentro il TGT decifrato trova:
    - l'identità dell'utente (Mario)
    - il timestamp di quando è stato emesso
    - la scadenza (di default 10 ore)
    - i SID dell'utente e dei suoi gruppi
- Poi controlla:
    - che non sia scaduto
    - che non sia stato emesso nel futuro (clock skew)
    - che l'utente non sia stato disabilitato o bloccato nel frattempo

> L'ultimo punto è interessante — ed è anche una debolezza di Kerberos. Il DC non consulta un database di "TGT revocati". Se Mario viene licenziato e il suo account viene disabilitato, i TGT già emessi rimangono validi fino alla loro scadenza naturale. Per questo il Golden Ticket è così persistente — anche se cambi la password dell'utente, il ticket forgiato continua a funzionare finché non cambi krbtgt.

#### Fase 3 — AP-REQ (client → servizio)
Mario presenta il TGS direttamente al file server. Il file server lo decifra con la propria chiave, verifica che sia valido, e concede l'accesso. Il DC non è più coinvolto.