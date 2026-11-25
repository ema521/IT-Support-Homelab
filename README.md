# Homelab per IT Support e networking

Un laboratorio che sto costruendo a casa, un pezzo alla volta, per fare pratica su quello che studio: virtualizzazione, Windows Server, gestione utenti, rete e troubleshooting.

---

## Panoramica

Al momento il laboratorio gira su un **HP EliteDesk 800 G2** con **Proxmox VE**. Più avanti voglio aggiungere un firewall **pfSense** su un **Fujitsu Futro S900** (ha due schede di rete) e uno switch gestito **TP-Link TL-SG108E** per fare pratica con le VLAN.

In parallelo sto seguendo il **Google IT Support Professional Certificate** su Coursera, e molti degli esercizi svolti nascono da lì: studio un argomento e poi provo a metterlo in pratica sul laboratorio.

Nel repository segno come fatte solo le cose che ho fatto davvero. Quello che è ancora in programma è indicato come tale.

N.B.: Gli indirizzi IP e le porte menzionate o le shell riportate sul porfolio, sono tutte informazioni bonificate, quindi non corrispondono ad info reali e sensibili.

---

## Hardware

- **Host principale**: HP EliteDesk 800 G2 (SFF)
  - CPU: Intel Core i7-6700
  - RAM: 32 GB DDR4 (2x16 GB)
  - Storage: SSD NVMe (prima usavo un SSD SATA, ora scollegato)
  - Scheda di rete: Intel I219-LM (driver `e1000e`)
- **PC da cui lavoro**: laptop con Linux Mint
- **Router**: Vodafone Station (rete `192.168.0.0/24`)
- **Access point**: TP-Link WR802N
- **Firewall** (in programma): Fujitsu Futro S900
- **Switch gestito** (in programma): TP-Link TL-SG108E

## Software

| Software | Stato |
|---|---|
| Proxmox VE | In uso |
| Windows Server 2022 | In uso |
| Linux Mint | In uso |
| Remmina (client RDP) | In uso |
| Wireshark | In uso |
| Tailscale | In uso |
| Windows 11 (client da unire al dominio) | In programma |
| Ubuntu Server / Debian | In programma |
| pfSense | In programma |

---

## Com'è collegata la rete

### Adesso

```text
Internet
→ Vodafone Station (router, DHCP — rete 192.168.0.0/24)
    → TP-Link WR802N (access point, stessa rete)
    → HP EliteDesk — Proxmox (+ Tailscale)
        → vmbr0 (bridge di Proxmox)
            → VM SRV01 — Windows Server 2022
    → Laptop Linux Mint
```

Tutto è sulla stessa rete, VM comprese. Il WR802N estende solo il Wi-Fi e non crea una rete sua: all'inizio pensavo il contrario e mi è costato parecchio tempo (lo racconto in [02 — Proxmox irraggiungibile](docs/02-rete.md)).

### Dove vorrei arrivare

```text
Internet
→ Vodafone Station
→ Futro con pfSense (firewall, DHCP, DNS)
→ Switch TL-SG108E (VLAN)
→ Proxmox e VM
```

Bozza delle VLAN future:

| VLAN | Uso | Rete |
| --- | --- | --- |
| 10 | Gestione (Proxmox, switch, pfSense) | `192.168.10.0/24` |
| 20 | Server | `192.168.20.0/24` |
| 30 | Client | `192.168.30.0/24` |

---

## La VM SRV01

Ho creato una VM con Windows Server 2022 con interfaccia grafica :

| Componente | Specifiche |
|---|---|
| CPU | 2 core |
| RAM | 5 GB |
| Disco | 70 GB, SATA |
| Scheda di rete | Intel E1000, collegata a `vmbr0` |

Per il disco e la scheda di rete ho usato SATA ed E1000 perché Windows li riconosce subito. Ho letto che i driver VirtIO sono più veloci, ma vanno caricati a parte durante l'installazione e per la prima volta ho preferito evitare complicazioni. Studiare VirtIO è uno dei prossimi passi.

---

## 1. La rete delle VM in Proxmox

La prima cosa che non mi era chiara era come facesse una VM ad andare in rete, visto che il PC ha una sola scheda di rete.

In pratica Proxmox crea uno switch virtuale, il bridge `vmbr0`. La scheda fisica è collegata a questo switch e porta il traffico al router, mentre ogni VM accesa si "attacca" allo stesso switch come se fosse un altro PC con il suo cavo. Per il router quindi la VM è un dispositivo come gli altri: ha il suo MAC e riceve un IP dal DHCP.

Ho capito anche che si può creare un bridge senza scheda fisica (per esempio `vmbr1`): le VM collegate lì si vedono tra loro ma non escono su internet. Mi servirà quando proverò a fare un server DHCP, così non va in conflitto con quello del router di casa.

Come verifica, dal PC il ping verso la VM va a buon fine.

---

## 2. Collegamento alla VM con RDP

All'inizio lavoravo sulla VM dalla console web di Proxmox, ma da lì non potevo copiare i comandi per inserirli qui e dovevo riscriverli a mano, con il risultato di sbagliare spesso. Ho quindi attivato il Desktop Remoto e mi collego direttamente nel mio PC.

- Sulla VM ho attivato il Desktop Remoto con `sconfig`.
- Su Linux Mint ho configurato una connessione RDP con Remmina, con l'utente Administrator e la sicurezza lasciata su "Automatico".
- Al primo collegamento ho accettato il certificato del server.

Ha funzionato al primo tentativo, e ora posso copiare e incollare tra il mio host e la VM.

Se un giorno non dovesse collegarsi, il primo controllo che farei è vedere se la porta xxxx risponde, con `nc -zv <IP> xxxx`.

---

## 3. Utenti e gruppi

Dopo il modulo "Users and Permissions" del corso ho provato a creare un piccolo scenario: due utenti e un gruppo per il reparto contabilità.

- `mrossi`, creato dall'interfaccia grafica (`lusrmgr.msc`)
- `lbianchi`, creato da PowerShell
- il gruppo `Contabilità`, con dentro solo `mrossi`

```powershell
$pw = Read-Host -AsSecureString "Password"
New-LocalUser -Name "lbianchi" -Password $pw -FullName "Luca Bianchi"
New-LocalGroup -Name "Contabilità"
Add-LocalGroupMember -Group "Contabilità" -Member "mrossi"
```

L'idea è dare i permessi al gruppo e non al singolo utente, così se arriva un nuovo collega basta aggiungerlo al gruppo.

Verifica:
```powershell
Get-LocalUser
Get-LocalGroupMember -Group "Contabilità"
```


Una cosa che farei diversamente: ho messo l'accento nel nome del gruppo. Funziona, ma mi è stato fatto notare che con script e sistemi diversi gli accenti possono dare problemi, quindi d'ora in poi li eviterò.

---

## 4. Permessi sulla cartella della contabilità

Obiettivo: chi è nel gruppo Contabilità può modificare i file della cartella, gli altri utenti possono solo leggerli.

```powershell
New-Item -Path "C:\Dati\Contabilita" -ItemType Directory
icacls "C:\Dati\Contabilita" /inheritance:r
icacls "C:\Dati\Contabilita" /grant "Administrators:(OI)(CI)F" "Contabilità:(OI)(CI)M" "Users:(OI)(CI)RX"
icacls "C:\Dati\Contabilita" /grant "SYSTEM:(OI)(CI)F"
```

Con `/inheritance:r` ho tolto i permessi che la cartella prendeva da `C:\`, e poi ho assegnato i miei. Togliendo l'ereditarietà era sparito anche SYSTEM, che ho dovuto rimettere.

Risultato di `icacls`:
```text
BUILTIN\Users:(OI)(CI)(RX)
SRV01\Contabilità:(OI)(CI)(M)
BUILTIN\Administrators:(OI)(CI)(F)
NT AUTHORITY\SYSTEM:(OI)(CI)(F)

<!-- NOTA PERSONALE:
-(OI)(CI): il permesso si applica anche a file e sottocartelle.
-F = controllo completo, M = modifica, RX = lettura ed esecuzione.
- Nessuna riga riporta (I): l'ereditarietà dalla cartella superiore è stata rimossa.
- SYSTEM va riassegnato dopo aver rimosso l'ereditarietà, perché servizi e backup operano con quell'account.-->
```

Il gruppo Contabilità ha la modifica (M), gli utenti normali la sola lettura (RX), amministratori e SYSTEM il controllo completo (F). Non compare nessun `(I)`, quindi nessun permesso è più ereditato.

**Prova con mrossi** (è nel gruppo, quindi deve poter scrivere):
```powershell
runas /user:mrossi powershell
whoami
"test" | Out-File C:\Dati\Contabilita\prova.txt
Get-Content C:\Dati\Contabilita\prova.txt
```
```text
srv01\mrossi
test
```
Ha funzionato.

**Prova con lbianchi** (non è nel gruppo, quindi deve poter leggere ma non scrivere):

<!-- DA COMPLETARE: -->

---

## 5. Accesso da fuori casa con Tailscale

Volevo poter lavorare sul laboratorio anche quando non sono a casa, senza aprire porte sul router (anche se poi ho dovuto aggiungere l'IP del server in whitelist perchè il router lo bloccava). 
Ho installato Tailscale sull'host Proxmox: crea una rete privata tra i miei dispositivi, e da fuori raggiungo l'interfaccia web di Proxmox con l'indirizzo che assegna Tailscale. Da lì apro anche la console delle VM.

Per ora Tailscale è solo sull'host. Per collegarmi in RDP direttamente alle VM da fuori dovrei configurarlo come "subnet router", ed è una delle prossime cose da studiare e provare.

---

## Problemi incontrati

I casi più lunghi li ho raccontati per intero nella cartella [`docs`](docs/).

| Problema | Causa | Come l'ho risolto |
| --- | --- | --- |
| Proxmox non raggiungibile in rete, nemmeno il ping al gateway | Avevo configurato un gateway su una rete che non esisteva (`192.168.1.x` invece di `192.168.0.x`) | Corretti IP e gateway, rigenerati i certificati. [Dettagli](docs/02-rete.md) |
| Dopo la reinstallazione su NVMe la password di root non veniva accettata | Il vecchio disco SATA era ancora collegato e il sistema leggeva i dati da quello | Scollegato il SATA. [Dettagli](docs/03-storage.md) |
| Windows non riusciva più a leggere né formattare la chiavetta di installazione | La ISO di Proxmox crea partizioni che Windows non riconosce | Chiavetta ripulita con `diskpart`. [Dettagli](docs/01-installazione.md) |
| Il BIOS HP non partiva da USB | Non l'ho scoperto | Non risolto: ho installato Proxmox sul disco collegandolo a un altro PC. [Dettagli](docs/01-installazione.md) |
| Login web di Proxmox rifiutato | Realm sbagliato | Scelto "Linux PAM" |
| Console di Proxmox scomoda per lavorare sulla VM | Non permette di copiare la shell | Collegamento in RDP con Remmina |

### Cosa ho imparato sul metodo

Il problema di rete mi ha fatto perdere molto tempo perché ho cercato la causa nel driver e nel kernel, quando era un semplice errore di indirizzo. Da allora, quando qualcosa non va in rete, parto sempre da questi controlli:

1. `ip a` — l'indirizzo IP è quello che mi aspetto?
2. `ip r` — il gateway è nella stessa rete?
3. `ip neigh` — il gateway risponde?
4. `ping` al gateway

Se già il ping al gateway non funziona, non ha senso guardare DNS, firewall o certificati. Un'altra cosa che ho imparato è cambiare una cosa alla volta: all'inizio modificavo più file insieme e poi non capivo cosa aveva fatto effetto.

---

## Roadmap

| Attività | Stato |
|---|---|
| Installazione di Proxmox | Fatto |
| Spostamento del sistema da SATA a NVMe | Fatto |
| Accesso da remoto con Tailscale | Fatto |
| VM Windows Server 2022 | Fatto |
| Collegamento RDP da Linux Mint | Fatto |
| Utenti e gruppi locali | Fatto |
| Permessi sulla cartella: configurazione e prova con mrossi | Fatto |
| Permessi sulla cartella: prova con lbianchi | Fatto |
| Ticket simulato "accesso negato" | Da fare |
| Snapshot e ripristino in Proxmox | Da fare |
| Backup delle VM con `vzdump` | Da fare |
| Driver VirtIO | Da fare |
| Active Directory, DNS, DHCP e Group Policy | Da fare |
| Client Windows 11 nel dominio | Da fare |
| Server Linux: servizi con `systemctl` e log con `journalctl` | Da fare |
| Accesso SSH con chiavi | Da fare |
| Tailscale come subnet router | Da fare |
| pfSense sul Futro | Da fare |
| VLAN sullo switch | Da fare |

---

## Documentazione

| Documento | Contenuto |
| --- | --- |
| [01 — L'inizio](docs/01-installazione.md) | Installazione di Proxmox: la chiavetta, il BIOS HP, il primo accesso |
| [02 — Proxmox irraggiungibile](docs/02-rete.md) | Il problema di rete più lungo che ho avuto |
| [03 — Dischi](docs/03-storage.md) | Passaggio da SATA a NVMe e il problema della password |
| 04 — Windows Server | In questo README |

---

## Il perché del mio progetto

Questo progetto nasce per mettere in pratica, a livello tecnico, quanto appreso durante lo studio autonomo del mondo dell'IT, supportato da corsi e materiale disponibile online.

Questa scelta parte dalla passione per le tecnologie e per la soddisfazione di risolvere i problemi, nonché dalla voglia di iniziare una carriera nel settore.

Il percorso non è stato semplice fino a ora, ma la passione e gli obiettivi alimentano la volontà e portano lontano.
Sono partito come un semplice utente qualche anno fa e oggi ho un background tecnico di cui vado fiero.

Con questo progetto ho imparato e sto continuando a imparare:
- La configurazione dell'hardware;
- L'installazione e la gestione di Windows Server, e a seguire di server Linux;
- Gli ambienti virtualizzati, partendo da zero per arrivare a una piccola infrastruttura;
- Come comunicano tra loro i vari livelli del modello TCP/IP;
- I principali protocolli di rete (IP, DNS, DHCP, HTTP, RDP, VPN ecc.).
