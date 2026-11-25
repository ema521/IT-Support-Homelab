# 01 — L'inizio: installazione di Proxmox

[← Torna al README](../README.md)

Pensavo fosse la parte più semplice e invece è stata quella che mi ha fatto perdere più tempo, tra la chiavetta e il BIOS dell'HP.

---

## La chiavetta di installazione

Ho scritto la ISO di Proxmox sulla chiavetta con Rufus. Dopo, Windows la vedeva come un disco da circa 220 KB e non riusciva più a formattarla. Lì per lì ho pensato che si fosse rotta.

In realtà non era guasta: la ISO di Proxmox crea delle partizioni che Windows non sa leggere, per questo la mostra in quel modo. L'ho recuperata ripulendola con `diskpart`, da prompt dei comandi come amministratore:

```text
diskpart
list disk
select disk N
clean
create partition primary
format fs=fat32 quick
assign
```

Su `select disk` bisogna fare molta attenzione a scegliere il numero giusto, perché `clean` cancella tutto il disco selezionato.

Per la prossima volta: Rufus, quando si scrive questa ISO, chiede se usare la modalità ISO o DD. Per Proxmox va scelta la DD, e dopo è normale che Windows non legga più la chiavetta.

---

## Il BIOS HP che non parte da USB

Il secondo problema: all'avvio l'HP mi dava sempre "Boot device not found. Please install an operating system.", come se la chiavetta non ci fosse, anche mettendola prima nella priorità di boot.

Ho provato in tanti modi:
- disattivato Secure Boot;
- disattivato HP Sure Start;
- messo il TPM su "Hidden";
- disattivato Fast Boot;
- attivato il boot da USB;
- cambiato porte USB, sia 2.0 che 3.0;
- provato ad avviare direttamente il file `EFI/boot/bootx64.efi` dal menu F9.

Nessuno ha funzionato. Strada facendo ho scoperto un paio di cose su questo BIOS: alcune voci non compaiono finché non si imposta una password di amministratore, e dopo aver cambiato Secure Boot al riavvio chiede di digitare un codice numerico per confermare.

Alla fine il problema **non l'ho risolto**. Ho tolto il disco dall'HP, l'ho collegato a un altro PC con un adattatore SATA-USB, ci ho installato Proxmox e poi l'ho rimontato nell'HP. Ha funzionato al primo avvio.

---

## Il primo accesso all'interfaccia web

Al primo login il mio utente veniva rifiutato. Il motivo era il campo "Realm": per entrare con `root` va scelto "Linux PAM". L'altra opzione, "Proxmox VE authentication server", usa gli utenti interni di Proxmox, dove `root` non esiste.
