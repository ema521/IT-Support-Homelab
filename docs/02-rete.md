# 02 — Proxmox irraggiungibile in rete

[← Torna al README](../README.md)

È il problema su cui ho perso più tempo, e alla fine la causa era banale. Lo racconto per intero perché è quello da cui ho imparato di più.

---

## La situazione

Dopo l'installazione, l'interfaccia web di Proxmox (`https://<ip>:8006`) non si apriva. Dal server:
- i servizi di Proxmox (`pveproxy`, `pvedaemon`, `pve-cluster`) risultavano attivi;
- la scheda di rete era su UP;
- il ping verso il gateway non funzionava;
- l'orologio (`chrony`) non si sincronizzava;
- nei log c'erano errori `RRDC update error`.

Avevo configurato IP `192.168.1.15` e gateway `192.168.1.1`.

---

## Le strade sbagliate

Siccome la scheda di rete era su UP, ho dato per scontato che la rete fosse a posto e ho cominciato a cercare problemi software:

- **il driver della scheda di rete**: una delle ipotesi era un problema del driver `e1000e` con i kernel recenti, quindi ho provato a disattivare alcune funzioni della scheda con `ethtool` e ad aggiungere parametri al kernel (`pcie_aspm=off`, `e1000e.SmartPowerDownEnable=0`). Nessun cambiamento;
- **il firewall di Proxmox**: l'ho fermato con `pve-firewall stop`. Niente;
- **il DNS**: il file `/etc/resolv.conf` aveva un errore, l'ho corretto ma non c'entrava;
- **l'orologio**: ho pensato che il problema fosse `chrony`, ma era il contrario: non si sincronizzava proprio perché il server non aveva rete.

---

## Cosa ho trovato

Alla fine ho guardato con calma la configurazione di rete:

```bash
ip a
ip r
ip neigh show
bridge link show
```

Le cose importanti erano queste:
```text
# ip r
default via 192.168.1.1 dev vmbr0 proto kernel onlink
192.168.1.0/24 dev vmbr0 proto kernel scope link src 192.168.1.15

# ip neigh show
192.168.1.1 e 192.168.0.1 con lo stesso MAC, stato STALE

# bridge link show
nic0 ... master vmbr0 state forwarding
```

`bridge link` diceva che la scheda fisica era collegata al bridge, quindi il cavo e la scheda funzionavano. `ip neigh` mostrava il MAC del router, quindi il server il router lo vedeva. Il problema quindi non era né il cavo né il driver, ma gli indirizzi.

E infatti: ero convinto che l'access point TP-Link WR802N creasse una sua rete `192.168.1.x`. Invece tutti i dispositivi di casa erano sulla rete del router Vodafone, `192.168.0.x`. Il server aveva come gateway un indirizzo che in casa non esisteva.

---

## La soluzione

1. In `/etc/network/interfaces` ho messo IP e gateway della rete giusta, `192.168.0.x`.
2. In `/etc/hosts` ho aggiornato l'IP associato al nome del server.
3. Siccome i certificati di Proxmox sono legati all'IP, li ho rigenerati:
   ```bash
   pvecm updatecerts --force
   systemctl restart pveproxy
   ```
4. Ho riavviato.

Dopo il riavvio l'interfaccia web si apriva, l'orologio si è sincronizzato e gli errori nei log sono spariti.

---

## Cosa ho imparato

Il mio errore è stato dare per scontato come fosse fatta la rete di casa, senza controllarlo, e poi cercare la causa in posti sempre più complicati. Ho anche modificato più file nello stesso momento, e così non capivo quale modifica servisse e quale no.

Adesso, quando un server non va in rete, parto da qui:

1. `ip a` — l'IP è quello che mi aspetto?
2. `ip r` — il gateway è nella stessa rete dell'IP?
3. `ip neigh` — il gateway risponde?
4. `ping` al gateway

Solo se questi controlli vanno bene passo a guardare DNS, firewall e il resto.
