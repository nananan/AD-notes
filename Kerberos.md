- [Kerberos](#kerberos)

# Kerberos
È un protocollo di autenticazione centralizzato che usa dei "ticket" invece delle password. Funziona sulla porta 88 ed è il sistema predefinito per i domini Windows dal 2000. Il vantaggio principale è che la password dell'utente non viaggia mai sulla rete.
Come funziona (semplificato)

- L'utente chiede un "documento d'identità" a un server centrale (il KDC)
- Dopo aver provato la propria identità, riceve un TGT (Ticket Granting Ticket), cioè appunto il suo "documento"
- Ogni volta che vuole accedere a un servizio, presenta questo TGT
- Il server centrale gli rilascia allora un ticket temporaneo specifico per quel servizio
- Il servizio riceve il ticket e decide se concedere o meno l'accesso

> Il servizio di concessione dei ticket (TGS) di Kerberos si basa su un ticket di concessione dei ticket (TGT) valido. Parte dal presupposto che, se un utente possiede un TGT valido, deve aver dimostrato la propria identità.

## Vantaggi rispetto a NTLM (il vecchio sistema)
Prima di Kerberos si usava NTLM, dove l'hash della password rimaneva in memoria. Se un hacker comprometteva una macchina, poteva usare quell'hash per accedere a tutto (attacco Pass-The-Hash). Con Kerberos i ticket sono limitati a specifiche macchine e hanno una scadenza, il che riduce significativamente il danno potenziale.

## Lato sicurezza
Nonostante sia più sicuro di NTLM, esistono comunque attacchi come il Pass-The-Ticket o il famoso Golden Ticket attack, ma con limitazioni maggiori rispetto agli attacchi sul vecchio sistema.


## Autenticazione Kerberos

### Le 3 entità coinvolte
- Utente – chi vuole accedere a un servizio
- KDC (Key Distribution Center) – il server centrale che conosce le credenziali di tutti
- Servizio – la risorsa a cui l'utente vuole accedere

![alt text](images/kerberos/kdc_all_keys.png)