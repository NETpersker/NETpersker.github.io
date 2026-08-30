---
title: Guida | Correggere le voci assenti in STEINS;GATE RE:BOOT su Mac con CrossOver
date: 2026-08-30 20:02:56
tags: [Guida, macOS, CrossOver, Steam, Videogiochi]
lang: it
translation_key: crossover-steins-gate-reboot-voice-fix
permalink: 2026/08/30/crossover-steins-gate-reboot-correzione-voci/
ai_translation: true
---

Qualche tempo fa, eseguendo l'edizione Steam di *STEINS;GATE RE:BOOT* (AppID 4012810) tramite CrossOver 26.3.0 su un Mac Apple Silicon, mi sono imbattuto in un problema piuttosto singolare. Il gioco si avviava normalmente e sia la musica di sottofondo sia gli effetti dell'interfaccia funzionavano, ma i personaggi restavano completamente muti quando parlavano.

In un primo momento era naturale sospettare che i file delle voci fossero danneggiati. I log diagnostici hanno infine indicato un'altra causa: le voci dei personaggi usano **Windows Media Audio 2 (WMA v2)**, mentre in questa installazione di CrossOver mancava il plugin GStreamer libav necessario a decodificare quella parte della catena audio. In questo articolo annoto il percorso seguito per individuare il problema e la soluzione che ho scelto, così da poterlo consultare di nuovo dopo futuri aggiornamenti di CrossOver.

> L'ambiente descritto è un Mac Apple Silicon con CrossOver 26.3.0 e l'edizione Steam di *STEINS;GATE RE:BOOT*. Un aggiornamento di CrossOver può cambiare la versione di GStreamer inclusa: le librerie dinamiche descritte qui non vanno quindi copiate alla cieca in un'altra versione.

## 1. Individuare il problema

Per riprodurre le voci attraverso CrossOver, i dati devono attraversare all'incirca questa catena:

```text
Voci WMA v2 contenute nel gioco
        ↓
wmadmod e winegstreamer di Wine / CrossOver
        ↓
GStreamer
        ↓
Decoder avdec_wmav2 di libav (FFmpeg)
        ↓
Voci dei personaggi
```

CrossOver include già Wine, il nucleo di GStreamer e l'uscita audio di macOS. Per questo la BGM e i normali effetti sonori continuano a funzionare. Ciò non significa però che ogni formato audio possa essere decodificato. In questo caso i dati vocali arrivavano alla catena multimediale, ma non era disponibile un componente `libgstlibav` capace di decodificarli. Il difetto compariva quindi soltanto quando un personaggio iniziava a parlare.

CodeWeavers documenta anche il problema più generale [Missing GStreamer 1.0 libav](https://support.codeweavers.com/en_US/missing-libraries/missinggstreamer1libav). Non riguarda esclusivamente *STEINS;GATE RE:BOOT*: in questo gioco si manifesta semplicemente come assenza delle voci mentre il resto dell'audio rimane intatto.

## 2. Duplicare prima il bottle

Prima di provare qualsiasi componente multimediale, il passaggio più importante non è scaricare una libreria, ma copiare il bottle Steam che già funziona.

1. Apri CrossOver.
2. Fai clic destro sul bottle in cui Steam si avvia correttamente.
3. Seleziona **Duplicate Bottle**.
4. Assegna alla copia il nome `Steam-SGRE-Voice-Test`.
5. Lascia intatto il bottle principale ed esegui tutte le prove successive nella copia.

Ho mantenuto la combinazione grafica già stabile: Graphics Backend su Auto e MSync attivo. Finché il bottle principale non viene modificato, una prova fallita non compromette l'installazione Steam usata normalmente.

## 3. Tentativi da evitare

### 3.1 Non modificare la directory condivisa di CrossOver

Non copiare direttamente i file in:

```text
/Applications/CrossOver.app/Contents/SharedSupport/CrossOver/
```

Questo ambiente è condiviso da tutti i bottle. Una sola libreria dinamica incompatibile può influire su Steam e sugli altri bottle; inoltre un aggiornamento di CrossOver potrebbe sovrascriverla.

### 3.2 Non mescolare le DLL multimediali di Windows

Ho provato il `wmadmod.dll` nativo di Windows 7. Chiamava il `mfplat.dll` incluso in CrossOver e provocava un arresto per scrittura su puntatore nullo appena un personaggio parlava. Sostituire anche quello con il `mfplat.dll` di Windows 7 non risolveva il problema: il gioco non si avviava perché quella versione non contiene l'interfaccia più recente `MFLockSharedWorkQueue`.

Le DLL multimediali provenienti da versioni diverse di Windows, Wine e CrossOver non sono pezzi intercambiabili.

### 3.3 Non usare il plugin ARM di Homebrew

Su Apple Silicon, GStreamer installato tramite Homebrew è normalmente ARM64, mentre il processo multimediale usato da CrossOver per questo gioco Windows è x86_64. Le due architetture non possono essere mescolate direttamente.

### 3.4 Non applicare l'ambiente del decoder privato a tutto Steam

Se variabili come `GST_PLUGIN_PATH` vengono applicate a Steam, anche Steam e i suoi componenti web esaminano il bundle libav privato. Il programma può quindi restare bloccato durante l'avvio. L'ambiente del decoder deve essere passato soltanto a `sgre_steam.exe`.

## 4. La soluzione adottata

Invece di sostituire DLL di sistema di Windows, ho collocato nel bottle di prova un bundle GStreamer libav privato, disponibile unicamente per SGRE:

```text
~/Library/Application Support/CrossOver/Bottles/
└── Steam-SGRE-Voice-Test/
    └── cx_gstreamer_libav/
        └── lib/
            ├── gstreamer-1.0/
            │   └── libgstlibav.dylib
            ├── libgstpbutils-1.0.0.dylib
            ├── libavcodec.60.dylib
            ├── libavformat.60.dylib
            ├── libavfilter.9.dylib
            ├── libavutil.58.dylib
            ├── libswresample.4.dylib
            ├── libz.1.dylib
            └── libbz2.1.dylib
```

I componenti provengono dal [runtime ufficiale GStreamer macOS Universal 1.24.13](https://gstreamer.freedesktop.org/download/). Ho scelto la serie 1.24 perché questa versione di CrossOver include GStreamer 1.24.4: rimanere nella stessa serie stabile rende più probabile la compatibilità binaria. Le informazioni upstream corrispondenti sono disponibili nelle [note di rilascio di GStreamer 1.24](https://gstreamer.freedesktop.org/releases/1.24/).

C'è tuttavia un limite importante. Il plugin ufficiale 1.24.13 dichiara come minimo una versione compatibile delle librerie al livello 1.24.14, mentre CrossOver fornisce la 1.24.4. Copiare semplicemente i file non basta. Ho adattato la versione minima compatibile e i percorsi delle dipendenze, poi ho applicato una firma ad-hoc alle librerie dinamiche modificate.

Gli SHA-256 dei due file essenziali che hanno funzionato in questo ambiente sono:

```text
libgstlibav.dylib
5133e1d0ef42d81f1e39f87dd4618c6f04679fadf2c6fa844a3c185440a00ff0

libgstpbutils-1.0.0.dylib
5a3f007aabde95632acc35ac16c4902b0060883e063e60e1bb32a1175cf3b6b4
```

> Non consiglio agli utenti meno esperti di modificare file `.dylib` con un editor esadecimale. È più sicuro usare componenti adatti alla propria versione di CrossOver, provenienti da una fonte nota e verificabili tramite checksum. Dopo un aggiornamento di CrossOver o GStreamer, la compatibilità va controllata di nuovo.

## 5. Caricare il plugin soltanto all'avvio del gioco

Anche dopo aver inserito i file nel bottle, usando il pulsante **Gioca** di Steam SGRE continua a vedere soltanto la directory predefinita dei plugin CrossOver. Ho quindi creato un launcher dedicato che imposta l'ambiente del decoder solo quando viene avviato `sgre_steam.exe`:

```zsh
#!/bin/zsh
set -eu

bottle_name='Steam-SGRE-Voice-Test'
bottle_root="$HOME/Library/Application Support/CrossOver/Bottles/Steam-SGRE-Voice-Test"
plugin_root="$bottle_root/cx_gstreamer_libav"
wine_bin='/Applications/CrossOver.app/Contents/SharedSupport/CrossOver/bin/wine'

export SteamAppId='4012810'
export SteamGameId='4012810'
export GST_PLUGIN_PATH="$plugin_root/lib/gstreamer-1.0"
export GST_PLUGIN_PATH_1_0="$plugin_root/lib/gstreamer-1.0"
export GST_PLUGIN_SYSTEM_PATH='/Applications/CrossOver.app/Contents/SharedSupport/CrossOver/lib64/gstreamer-1.0'
export GST_PLUGIN_SYSTEM_PATH_1_0="$GST_PLUGIN_SYSTEM_PATH"
export GST_REGISTRY="$plugin_root/registry.bin"
export GST_REGISTRY_FORK='no'
export GST_PLUGIN_FEATURE_RANK='avdec_wmav2:MAX'

exec "$wine_bin" \
  --bottle "$bottle_name" \
  --no-wait \
  --cx-app 'C:\Program Files (x86)\Steam\steamapps\common\SGRE\sgre_steam.exe'
```

Se il bottle ha un nome diverso, bisogna modificare sia `bottle_name` sia `bottle_root`. Per non dover aprire ogni volta il Terminale, ho impacchettato e firmato lo script come una normale applicazione macOS:

```text
STEINS;GATE REBOOT Voice Fix.app
```

Nell'uso quotidiano avvio prima Steam nel bottle `Steam-SGRE-Voice-Test`, senza premere **Gioca**, e poi apro il launcher dedicato. In questo modo le variabili restano limitate a SGRE e non collocano l'intero Steam nell'ambiente dei plugin privati.

## 6. Verificare che la correzione sia effettiva

Il fatto che il gioco non vada più in crash non è sufficiente. Ho controllato tutti questi punti:

- la finestra del gioco compare normalmente;
- BGM, suoni dell'interfaccia ed effetti ambientali funzionano;
- diverse battute consecutive contengono le voci;
- le voci rimangono dopo il caricamento di un salvataggio o un cambio di scena;
- il gioco può essere chiuso e riavviato tramite il launcher.

Nel log diagnostico comparivano esplicitamente:

```text
avdec_wmav2
Decoded data
return flow ok
```

Il processo del gioco caricava inoltre `libgstlibav.dylib`, `libavcodec.60.dylib` e gli altri componenti dalla directory `cx_gstreamer_libav` del bottle di prova. Le voci non erano tornate per caso: la catena di decodifica WMA v2 era realmente collegata.

## 7. Risoluzione dei problemi e ripristino

### Le voci sono ancora assenti

- Assicurati di aver avviato il gioco con il launcher dedicato e non con il pulsante **Gioca** di Steam.
- Verifica che il nome del bottle nello script corrisponda esattamente a quello mostrato da CrossOver.
- Controlla che `cx_gstreamer_libav` sia ancora nel bottle di prova.
- Verifica che `libgstlibav.dylib` sia x86_64 o Universal, non soltanto ARM64.

### Il gioco va in crash quando un personaggio parla

Controlla se `wmadmod.dll`, `mfplat.dll` nativi o altre DLL multimediali siano stati copiati in `drive_c/windows/system32`. In tal caso, interrompi la mescolanza dei componenti e ripristina il backup del bottle precedente alle prove.

### Steam resta bloccato durante l'avvio

Di solito significa che `GST_PLUGIN_PATH` e le variabili correlate sono state passate a Steam. Esci dal bottle di prova, ripristina il normale metodo di avvio di Steam e mantieni le variabili soltanto nel launcher SGRE.

### Le voci scompaiono dopo un aggiornamento di CrossOver

Un aggiornamento può cambiare la versione di GStreamer inclusa. Non copiare il vecchio plugin nella nuova directory condivisa. Duplica prima il bottle, quindi controlla versione e architettura del nuovo GStreamer prima di adattare di nuovo il bundle.

Poiché tutte le modifiche sono confinate nel bottle di prova e nel launcher separato, tornare indietro è semplice: smetti di usare `STEINS;GATE REBOOT Voice Fix.app`, sposta fuori dal bottle la directory `cx_gstreamer_libav` oppure ripristina la copia pulita. Non è necessario eliminare il bottle Steam principale né reinstallare CrossOver.

## 8. Breve riepilogo

Le voci mancanti erano dovute all'audio WMA v2 dei personaggi, incontrando una catena multimediale CrossOver priva di un decoder GStreamer libav utilizzabile. Il metodo finale può essere riassunto così:

```text
Duplicare il bottle di prova
→ inserire un plugin libav privato compatibile per versione e architettura
→ esporre il percorso del plugin soltanto a SGRE
→ avviare il gioco con un launcher separato
→ verificare sia i dialoghi sia i log del decoder
```

Richiede più lavoro rispetto a copiare alcune DLL nel bottle, ma mantiene l'intera modifica dentro una copia sacrificabile. L'ambiente Steam principale e la directory condivisa di CrossOver restano intatti.
