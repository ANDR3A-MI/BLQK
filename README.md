# Syntra BLQK

Sistema operativo leggerissimo per **Asus Transformer Book T100T** (Intel Atom Bay Trail, 2 GB di RAM, 32 GB eMMC), costruito su **antiX 26.1 core** (Debian 13, senza systemd) e ribrandizzato.

La ISO si crea **da sola su GitHub** (GitHub Actions): non serve Linux sul tuo PC.

| | |
|---|---|
| **App incluse** | Pale Moon · FreeLLMAPI · Arduino IDE · **Syntra Store** (installa altre app) · Impostazioni (Wi-Fi, Bluetooth, audio, luminosità, schermo, tastiera…) · File · Blocco note · Terminale |
| **Store** | Ufficio (LibreOffice/OnlyOffice) · Internet · Multimedia · Sviluppo (VS Code…) · **Sicurezza/Kali** · Utilità. Installa da apt, dal repository ufficiale Kali e da Flathub |
| **Interfaccia** | Openbox + tint2 + rofi in **blu/nero/bianco**, stile Ubuntu: barra in alto con "Attività", orologio e indicatori; dock a sinistra; griglia delle app. Sfondo personalizzato |
| **All'avvio** | video a schermo intero (al posto del testo del terminale), si salta con un tasto |
| **Tastiera** | tasti funzione/multimediali (volume, luminosità, ecc.) attivi; **tasto Windows** da solo apre la griglia delle app |
| **RAM a riposo** | circa 270 MB con il desktop avviato, misurati in macchina virtuale (di cui ~100 MB sono del server grafico senza accelerazione: sul tablet vero ne servono meno). Swap compresso in RAM (zram) attivo |
| **Avvio dalla chiavetta** | menu con **Avvia Live** · **Installa sul disco** · modalità compatibilità · solo terminale |
| **Boot** | BIOS, UEFI a 64 bit **e UEFI a 32 bit** (il T100T ha un firmware a 32 bit su CPU a 64 bit) |

---

## 1. Crea la ISO su GitHub

1. Crea un nuovo repository **pubblico** su GitHub, per esempio `syntra-os`. (Con l'account gratuito i repository privati hanno solo 500 MB di spazio per gli artefatti: la ISO è più grande e il caricamento finale fallirebbe. Nei repository pubblici GitHub Actions è gratuito e senza questo limite.)
2. Nel repository bastano **due file**:
   - `syntra-os.tar.gz` nella cartella principale (*Add file → Upload files*): contiene tutto questo progetto;
   - `.github/workflows/build-iso.yml` (*Add file → Create new file*, scrivi quel percorso come nome e incolla il contenuto del file).

   Il workflow estrae l'archivio da solo. In alternativa puoi caricare tutti i file sciolti mantenendo la struttura delle cartelle: funziona allo stesso modo.
3. Vai su **Actions → "Crea ISO Syntra" → Run workflow**. La build dura circa 30–60 minuti.
4. A fine build apri l'esecuzione e scarica l'artefatto **syntra-iso** (contiene `syntra-1.0-x64.iso`, il checksum e `build-report.txt`).
5. Per una release pubblica con la ISO allegata: crea un tag, es. `git tag v1.0 && git push --tags`.

> Leggi sempre `build-report.txt` (compare anche nel riepilogo dell'esecuzione): elenca eventuali pacchetti non trovati. Se la build si ferma, scarica l'artefatto **syntra-log**: `chroot.log` dice esattamente dove.

## 2. Scrivi la chiavetta

- **balenaEtcher** (consigliato), oppure
- **Rufus**: va bene sia "modalità immagine DD" sia "modalità ISO" (la ISO contiene i loader EFI anche come file), oppure
- da Linux: `sudo dd if=syntra-1.0-x64.iso of=/dev/sdX bs=4M status=progress oflag=sync`.

## 3. Avvia il T100T dalla chiavetta

1. Spegni il tablet e collega la chiavetta (alla dock con tastiera, o con un adattatore OTG).
2. Accendi tenendo premuto **F2** (o **Esc**) per entrare nel BIOS.
3. **Security → Secure Boot → Disabled** (il boot loader non è firmato).
4. **Save & Exit → Boot Override** → scegli la chiavetta USB.
5. Nel menu scegli **Avvia Syntra BLQK (Live)** per provarlo, oppure **Installa Syntra BLQK sul disco**.

Utente della sessione Live: `demo` / password `demo`.

## 4. Installazione

L'installer (a finestre di testo, in italiano) chiede disco, nome, utente, password e accesso automatico, poi:

- crea una tabella **GPT** con partizione EFI (512 MB) + sistema ext4;
- copia il sistema dalla chiavetta;
- installa **GRUB i386-efi** sul T100T, x86_64-efi sugli altri PC UEFI, i386-pc sui PC con BIOS;
- scrive il loader nel percorso standard `EFI/BOOT/BOOTIA32.EFI`, che il firmware del T100T trova sempre (anche senza voce NVRAM).

⚠️ **Il disco scelto viene cancellato completamente** (Windows compreso). Il dual-boot non è previsto in questa versione.

---

## Cosa è stato fatto per il T100T

| Componente | Soluzione |
|---|---|
| UEFI a 32 bit | ISO con `bootia32.efi`; l'installer riconosce il firmware (anche tramite il parametro `syntra.fw` passato da GRUB) e usa `--target=i386-efi` |
| CPU senza AVX | Pale Moon in versione **SSE2** (i pacchetti ufficiali richiedono AVX e sull'Atom non partirebbero) |
| Wi-Fi Broadcom 43241 (SDIO) | `firmware-brcm80211`; se serve, `syntra-hwsetup` estrae la NVRAM dalla variabile EFI e ricarica il driver |
| Audio (bytcr-rt5640) | `firmware-intel-sound` + `alsa-ucm-conf`; all'avvio `syntra-hwsetup` sceglie la scheda giusta (sul T100T la prima è l'HDMI), applica il profilo **HiFi/Speaker** e crea un volume generale software |
| 2 GB di RAM | zram (75% della RAM), nessun compositor, indicatori di rete e volume disegnati dalla barra stessa (nessuna applet residente), FreeLLMAPI e Arduino avviati solo su richiesta |
| Schermo 1366×768 + touch | tap-to-click, Pale Moon con scorrimento XInput2, pulsanti grandi |
| Blocchi casuali Bay Trail | voce di boot "modalità compatibilità" (`intel_idle.max_cstate=1`); se installi partendo da lì, l'opzione resta anche nel sistema installato |
| Luminosità | tasti Fn e cursore in Impostazioni (script proprio, nessuna dipendenza) |

**Bluetooth:** il chip del T100T (BCM4324B3) richiede un firmware che non è distribuibile liberamente. Se ti serve, copia `BCM4324B3.hcd` (si trova nella partizione Windows del tablet, in `C:\Windows\System32\drivers`, con un nome tipo `BCM4324B3_*.hcd`) in `overlay/usr/lib/firmware/brcm/BCM4324B3.hcd` prima di lanciare la build.

## Le app

- **Pale Moon** — build SSE2/GTK3 dal repository Debian indicato da palemoon.org tra le *contributed builds* (kannegieser.net). Si aggiorna con il resto del sistema.
- **FreeLLMAPI** — l'**app desktop ufficiale per Linux** (la stessa di `FreeLLMAPI.exe` per Windows), scaricata dalla pagina Releases del progetto e verificata con l'hash SHA-512 pubblicato. Clic sull'icona: avvia il router locale (`http://127.0.0.1:31415/v1`) e apre la dashboard; l'icona nella barra in alto resta finché non lo chiudi. Usa circa 220 MB di RAM quando è aperta: chiudila dall'icona (*Quit*) o con `syntra-freellmapi --stop` quando non serve. In `config.env` c'è anche la variante `FREELLMAPI_MODE="server"` (solo server, ~100 MB, dashboard nel browser).
- **Arduino IDE** — versione **1.8.19** (Java incluso, archivio ufficiale verificato con il suo checksum): è quella adatta a 2 GB di RAM. L'utente può già usare le porte seriali USB. Per la 2.x (molto più pesante) metti `ARDUINO_IDE="2"` in `config.env` o sceglila quando avvii il workflow.
- **Impostazioni** (Super+I) — Wi-Fi e rete, Bluetooth, audio, luminosità, schermo, aspetto, sfondo, tastiera, data e ora, batteria, aggiornamenti, informazioni.
- **Syntra Store** — installa altre applicazioni con pochi clic, dai gestori standard:
  - **Ufficio**: LibreOffice e OnlyOffice (aprono e salvano Word/Excel/PowerPoint), AbiWord+Gnumeric per la poca RAM.
  - **Internet, Multimedia, Sviluppo** (anche VS Code), **Utilità**.
  - **Sicurezza (Kali)**: aggiunge il repository ufficiale Kali (con priorità bassa, così **non** sostituisce il sistema) e installa gli strumenti scelti (nmap, Wireshark, Metasploit, Burp Suite, aircrack-ng…). Prima di installarli lo Store ricorda che vanno usati solo su sistemi e reti **tuoi** o con **autorizzazione scritta**.
  
  Lo Store usa `apt`, il repository Kali e **Flathub** (Flatpak): niente download manuali. Alcune categorie sono grandi, tienine conto con 32 GB di disco.
- **File** (pcmanfm, chiavette montate in automatico), **Blocco note** (mousepad), **Terminale** (lxterminal).

### Tasti e sfondo
- **Tasti funzione / multimediali**: volume, luminosità, muto, microfono, schermo, play/pausa, Wi-Fi funzionano (il T100T li invia come tasti multimediali; Syntra li associa alle azioni giuste).
- **Tasto Windows** da solo → apre la griglia delle app (tramite *ksuperkey*, che lo traduce in Alt+F1). Se sull'hardware non funzionasse, **Alt+F1** fa sempre la stessa cosa.
- **Sfondo**: l'immagine che hai fornito (adattata a 1366×768). Cambiabile da Impostazioni → Sfondo.
- **Video di avvio**: all'accensione parte il video che hai fornito, a schermo intero, al posto del testo. Si salta con un tasto o un clic. In `config.env` (`SPLASH_MODE`) o scrivendo `off`/`once` in `~/.config/syntra/splash` lo disattivi o lo limiti al primo avvio.

Scorciatoie: **Super** (da solo) o **Super+A** / **Super+Spazio** app · **Super+E** file · **Ctrl+Alt+T** terminale · **Super+←/→** affianca finestre · **Alt+F4** chiudi.

## Personalizzare

Tutto è in `config.env` (nome, versione, lingua, versione di antiX, Arduino, FreeLLMAPI). I file dell'interfaccia sono in `overlay/` (copiati così come sono nel sistema), la grafica in `branding/`.

```
.github/workflows/build-iso.yml   la build su GitHub
build.sh                          scarica antiX, personalizza, ricrea la ISO
scripts/chroot-setup.sh           pacchetti, app, rebranding (dentro il sistema)
config.env                        impostazioni della build
branding/                         logo, sfondo (wallpaper.jpg), tema del menu di avvio, icone
vendor/ksuperkey/                 sorgente di ksuperkey (compilato dalla build)
overlay/                          file copiati nel sistema
  usr/share/syntra/boot.mp4         video di avvio
  usr/local/sbin/syntra-installer   installer
  usr/local/sbin/syntra-hwsetup     zram, Wi-Fi T100T, audio, servizi
  usr/local/sbin/syntra-getty       accesso automatico (qualunque init)
  usr/local/bin/syntra-store        app store
  usr/local/bin/syntra-splash       video di avvio
  usr/local/bin/syntra-*            impostazioni, alimentazione, FreeLLMAPI…
  etc/skel/.config/                 Openbox, tint2, rofi, GTK, dunst
```

## Cosa è stato provato (e cosa no)

Provato in macchina virtuale (QEMU con firmware UEFI a 32 bit, CPU a 64 bit, 2 GB di RAM, disco eMMC emulato), usando il vero `build.sh` su una base di prova con la stessa struttura di una ISO antiX:

- la build completa, dall'estrazione della ISO alla ISO finale;
- avvio della chiavetta con BIOS, UEFI 64 bit, **UEFI 32 bit** (sia scritta in modalità DD, sia copiata su FAT32);
- accesso automatico, avvio del desktop senza systemd/logind, barra, dock, indicatore di rete;
- l'installer dall'inizio alla fine su disco eMMC e il riavvio del sistema installato con UEFI a 32 bit;
- l'app FreeLLMAPI lanciata dalla griglia delle app nel sistema installato (icona nella barra, dashboard in finestra) e, a parte, il server compilato dai sorgenti;
- indicatori di rete e volume, volume software, menu Impostazioni e spegnimento;
- i **colori blu/nero/bianco** e lo **sfondo** nuovo; la **griglia app**, la **finestra dello Store** e la generazione dei comandi di installazione (apt/Kali/Flathub); il **video di avvio** a schermo intero con salto da tastiera; la compilazione di **ksuperkey**.

**Non** è stato possibile provare (i siti non erano raggiungibili dall'ambiente di sviluppo, da GitHub lo sono):

- la build sulla **vera ISO antiX 26.1** e sui suoi repository: la prima esecuzione su GitHub può richiedere qualche ritocco (nomi di pacchetti, servizi). `build-report.txt` e `chroot.log` dicono cosa;
- **Pale Moon** e **Arduino IDE** veri (nelle prove sono stati sostituiti da segnaposto);
- l'**installazione vera** delle app dello Store (serve la rete e i repository Debian/Kali/Flathub): è stata verificata solo la generazione corretta dei comandi, non il download finale;
- il **tasto Windows fisico** e i tasti multimediali **sull'hardware reale** (provati solo come associazioni; ksuperkey genera comunque Alt+F1);
- l'**hardware reale** del T100T: Wi-Fi, audio, touchscreen, luminosità, sospensione.

## Limiti da conoscere

- **"Non riconoscibile" fin dove è possibile.** Tutto ciò che l'utente vede dice Syntra: menu di avvio, desktop, installer, `/etc/os-release`, nome del computer. Un occhio tecnico può comunque risalire alla base: nome del kernel (`uname -r`), messaggi dei primi secondi di avvio della chiavetta, cartella `/antiX` sulla chiavetta (il sistema live la cerca con quel nome), repository apt da cui arrivano gli aggiornamenti. Le note di licenza dei pacchetti (`/usr/share/doc/*/copyright`) restano al loro posto: le licenze del software libero lo richiedono.
- **Pale Moon e i siti moderni.** È leggero proprio perché usa un motore diverso da Chrome/Firefox: alcuni siti recenti possono non funzionare bene. Per questo FreeLLMAPI usa la propria finestra e non il browser.
- **Audio** tramite ALSA (niente PulseAudio/PipeWire, per risparmiare RAM): un programma alla volta usa la scheda audio.
- **Word ed Excel veri (Microsoft) non esistono per Linux.** Lo Store installa le alternative gratuite **LibreOffice** e **OnlyOffice**, che aprono e salvano i file `.docx` e `.xlsx` (OnlyOffice è il più fedele all'aspetto di Microsoft Office).
- **Gli strumenti di sicurezza (Kali) sono potenti e a doppio uso.** Lo Store li installa dal repository ufficiale, ma usarli su reti o sistemi non tuoi e senza permesso è illegale. Su 2 GB di RAM e 32 GB di disco il set completo di Kali è grande e pesante: meglio installare solo i singoli strumenti che ti servono.
- **Pesantezza delle app dello Store.** Molte (LibreOffice, VS Code, GIMP, i browser moderni) sono fatte per PC più potenti: sul T100T funzionano ma lente. Le alternative leggere sono segnalate nello Store.
- **Video di avvio.** Prima che parta la grafica c'è ancora un breve lampo di testo del sistema (un paio di secondi); il video copre la parte restante dell'avvio. Non è un vero "splash" dentro l'avvio del kernel (quello non può riprodurre un filmato).
- **Se all'avvio dalla chiavetta compare il login testuale**, entra con `demo` / `demo`: il desktop parte subito dopo (vuol dire che l'accesso automatico non ha riconosciuto il tipo di init: segnalamelo con il `build-report.txt`).
- **Se il sistema live non trova la chiavetta**, metti `ISO_LABEL="keep"` in `config.env` (riusa l'etichetta originale di antiX) e rifai la build.
