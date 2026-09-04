+++
title = "Ratatui passo dopo passo - Introduzione"
date = 2026-09-04
description = "Introduzione a Ratatui, la libreria Rust per costruire interfacce testuali nel terminale, e al percorso che seguiremo passo dopo passo nella serie."
[taxonomies]
tags = ["rust", "ratatui"]
[extra]
comments = true
+++

## Una nota personale

Nel 2000 lavoravo principalmente su Windows NT — stavo pure prendendo una certificazione — quando mi hanno assegnato a un nuovo progetto che doveva collegarsi a server Unix. Il mio collega, in affiancamento, mi dice: "apri il terminale, PuTTY". Io, tranquillo, doppio click sull'icona. Poi: "seleziona questo server". Doppio click. Poi: "metti la finestra a tutto schermo". Doppio click, tutto contento, sapevo esattamente cosa fare.

E poi lui: "ora quel mouse non ti serve più, puoi anche staccarlo".

Panico. Avevo iniziato col Commodore 64, altro che Windows, poi MS-DOS — la tastiera non mi era certo estranea. Ma erano passati anni, e ritrovarmi lì, su un sistema Unix, senza più il mouse a cui appoggiarmi, mi ha fatto pensare per un attimo: *ora mi cacciano*.

Non mi hanno cacciato. È stata semplicemente la fine del mio attaccamento al mouse. Oggi lavoro quasi esclusivamente da tastiera — Sway come window manager, tmux, nvim — e quando devo aprire Firefox mi scoccia quasi cercarlo. Questa serie nasce anche da lì: dal fatto che il terminale, una volta che smetti di temerlo, diventa lo strumento più diretto che hai.

## Cos'è Ratatui

Ratatui è una libreria Rust per costruire interfacce interattive che vivono interamente nel terminale: niente finestre, niente mouse (a meno che tu non lo voglia), solo caratteri, colori e un layout che tu controlli riga per riga. Se hai mai usato `htop` o `lazygit`, hai già usato il tipo di applicazione che Ratatui ti mette in condizione di costruire: quelle hanno una TUI, *terminal user interface* (o qualcuno preferisce *text user interface*), e sono l'equivalente testuale di una GUI — pannelli, liste selezionabili, barre di stato, il tutto renderizzato con caratteri invece che con pixel.

Non è un framework che nasconde il terminale dietro un'astrazione: è una libreria pensata per dare controllo diretto su cosa appare a schermo.

Oggi trovi TUI scritte in Rust ovunque: client Git, monitor di sistema, gestori di database, dashboard di infrastruttura. Ratatui è uno dei nomi che incontrerai più spesso nell’ecosistema Rust quando si parla di TUI.

## A chi si rivolge questa serie

Questa serie parte dal presupposto che tu conosca già Rust — ownership, trait, `match`, closure — ma non abbia mai toccato Ratatui. Non troverai spiegazioni di sintassi di base del linguaggio, ma nemmeno voli pindarici sull'architettura interna della libreria: l'obiettivo di ogni articolo è farti scrivere codice che produce qualcosa di visibile sul terminale, un tassello alla volta, riusando sempre quello scritto nella puntata precedente.

Serve solo Rust installato e un terminale. Nient'altro.

## Come è strutturata la serie

Si parte dal minimo indispensabile — una schermata vuota con una riga di testo — e si arriva, passo dopo passo, a un'applicazione con stato, più widget ed eventi che arrivano anche da un thread in background. Ogni tappa aggiunge un solo concetto nuovo rispetto alla precedente.

Questa è la traccia che ho in mente al momento: potrebbe cambiare strada facendo, soprattutto se qualche argomento si rivelerà abbastanza interessante da meritarsi una puntata tutta sua.

1. **Hello Ratatui** — una schermata minima e un `Paragraph`.
2. **Testo** — `Span`, `Line`, `Text` e `Paragraph`, dal componente più piccolo al più capace.
3. **Layout base** — dividere lo schermo in aree con `Layout` e `Constraint`.
4. **`Block`** — bordi, titoli e padding.
5. **Input da tastiera** — leggere tasti specifici con `KeyCode`, non solo "un tasto qualsiasi".
6. **Gestire lo stato dell'app** — una struct `App` e il ciclo evento → aggiornamento → ridisegno.
7. **Un text input basilare** — costruirsi da soli un campo di testo, cursore compreso.
8. **Un widget alla volta: `List` e `Table`** — la selezione come stato, con `ListState` e `TableState`.
9. **Organizzare il progetto su più file** — moduli separati e un enum per generalizzare gli eventi.
10. **Aggiornamenti in background con i thread** — un thread separato che produce dati e li invia con `std::sync::mpsc`.
11. **Async e aggiornamenti in background** — la stessa idea, ma con un runtime asincrono.

E probabilmente non finirà qui.

Detto questo: perdersi in chiacchiere ogni tanto fa bene, ma direi che per oggi abbiamo esagerato. Si comincia dal primo passo: una schermata vuota, e la prima riga di testo che ci scriviamo dentro.

