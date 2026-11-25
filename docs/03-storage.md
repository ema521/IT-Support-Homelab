# 03 — Dischi: passaggio da SATA a NVMe

[← Torna al README](../README.md)

All'inizio Proxmox era installato su un SSD SATA. Poi ho aggiunto un SSD NVMe, più veloce, e ho deciso di spostare lì il sistema.

---

## Come ho spostato il sistema

Invece di provare a clonare il disco, ho preferito reinstallare Proxmox da zero sul NVMe. Mi sembrava la strada più semplice e con meno rischi di ritrovarmi con un sistema che non si avvia.

Durante l'installazione però ho lasciato collegato anche il SATA, con sopra la vecchia installazione. Ed è da lì che è nato il problema.

---

## La password di root che non funzionava

Finita l'installazione sul NVMe, il login di `root` non funzionava con nessuna password: né con quella appena impostata, né con quelle che usavo prima sul SATA.

Ho ragionato così: fino a quando c'era un solo disco, tutto funzionava. Il problema è comparso solo con due dischi collegati, e sul SATA c'era ancora tutta la vecchia configurazione. Ho quindi provato a scollegare proprio il SATA e a riavviare.

Con il solo NVMe collegato, la password nuova ha funzionato al primo tentativo.

Dopo, cercando di capire il perché, ho scoperto che il problema nasce da come Proxmox organizza i dischi. Ogni installazione dà lo stesso nome alla parte del disco che contiene il sistema. Con due installazioni collegate insieme, il sistema si confonde e può finire per leggere le password dal disco sbagliato. In pratica partiva dal NVMe ma usava i dati del vecchio SATA.

---

## Cosa farei diversamente

Quando reinstallo un sistema su un disco nuovo, scollego prima tutti gli altri dischi che contengono una vecchia installazione. Mi sarei risparmiato parecchio tempo.

Il SATA adesso è scollegato e non lo uso. Lo tengo da parte per usarlo all'occorrenza come disco esterno, collegato tramite USB.
