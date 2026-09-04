+++
title = "Ratatui passo dopo passo - la prima finestra sul terminale"
date = 2026-09-04
description = "Iniziamo a usare Ratatui partendo dal minimo indispensabile: un'app Rust, l'alternate screen e il primo testo visualizzato nel terminale."
[taxonomies]
tags = ["rust", "ratatui"]
[extra]
comments = true
+++

Questo è il primo pezzo di una serie che ti porta dentro [Ratatui](https://ratatui.rs/) partendo da zero, un tassello alla volta. Il punto di partenza è che conosci già Rust, ma probabilmente non hai mai scritto un'interfaccia che vive dentro il terminale invece che in una finestra grafica o nel browser — ed è un mondo con le sue regole, diverso da quello a cui sei abituato, ma non per questo meno interessante. 

Oggi costruiamo un piccolo programma che prende il controllo del terminale, ci scrive dentro una riga di testo, e torna al prompt normale alla pressione di un tasto qualsiasi.

Il risultato, semplicemente questo:

```
Ciao, Ratatui! Premi un tasto per uscire.
```

Poca roba, ma è il primo mattone: da qui in avanti ogni articolo aggiunge un pezzo senza toccare quello che hai già scritto.

## Codice, passo per passo

### Creiamo il progetto e aggiungiamo le dipendenze

Per prima cosa creiamo il progetto e aggiungiamo l'unica dipendenza che ci serve per ora: Ratatui. Al momento in cui scrivo la versione corrente è la `0.30.2`; nel `Cargo.toml` useremo `0.30`, così Cargo potrà adottare automaticamente le successive release compatibili della serie 0.30.

```bash
cargo new hello-ratatui
cd hello-ratatui
cargo add ratatui@0.30
```

`ratatui` è la libreria che disegna l'interfaccia. Per leggere gli eventi della tastiera ci serve anche `crossterm`, la libreria che Ratatui usa come backend predefinito per interagire con il terminale. Non dobbiamo però aggiungerla come dipendenza separata: Ratatui la ri-esporta come `ratatui::crossterm`.

Usare questo re-export ha anche un vantaggio: siamo sicuri di utilizzare la stessa versione di Crossterm scelta da Ratatui, evitando possibili incompatibilità tra i tipi delle due librerie.

Il tuo `Cargo.toml` dopo questo passaggio avrà una sezione simile a questa:

```toml
[dependencies]
ratatui = "0.30"
```

### Lo scheletro dell'app

Sostituisci il contenuto di `src/main.rs` con questo:

```rust
use ratatui::crossterm::event;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    ratatui::run(|terminal| {
        loop {
            terminal.draw(|_frame| {})?;

            if event::read()?.is_key_press() {
                break Ok(());
            }
        }
    })
}
```

Cosa fa, riga per riga a livello di comportamento (non di meccanismi interni):

- `ratatui::run(...)` prende il controllo del terminale (lo mette in modalità "schermo alternato", quella usata da programmi come `vim` o `htop`) e lo restituisce automaticamente quando la closure finisce, anche se nel frattempo va in panic.
- Dentro, come vedi, c'è un ciclo infinito: ad ogni giro disegni un fotogramma con `terminal.draw(...)`, poi aspetti un evento con `event::read()`.
- `is_key_press()` è vero quando l'evento arrivato è la pressione di un tasto (e non, per esempio, il rilascio, o un ridimensionamento del terminale). Appena succede, esci dal ciclo con `break` e il programma finisce, restituendo il controllo del terminale.

A questo punto, se lanci `cargo run`, vedi lo schermo diventare nero e vuoto (sei nella modalità alternata di Ratatui, ma non hai ancora disegnato nulla). Premendo un tasto qualsiasi torni al prompt normale.

L'app "funziona", ma semplicemente non mostra ancora niente.

### Scriviamoci qualcosa

Aggiungiamo un widget `Paragraph`, che è il modo più comune per mettere del testo a schermo. Prima l'import:

```rust
use ratatui::widgets::Paragraph;
```

Poi riempiamo la closure di `draw` che avevamo lasciato vuota:

```rust
terminal.draw(|frame| {
    let paragraph = Paragraph::new("Ciao, Ratatui! Premi un tasto per uscire.");
    frame.render_widget(paragraph, frame.area());
})?;
```

`Paragraph::new(...)` crea il widget a partire da una stringa, `frame.render_widget(widget, area)` lo disegna nell'area che gli indichi — qui `frame.area()`, cioè tutto lo schermo disponibile.

A questo punto, se lanci `cargo run`, vedi la scritta "Ciao, Ratatui! Premi un tasto per uscire." comparire in alto a sinistra dello schermo. Premi un tasto qualsiasi per uscire.

> **Per approfondire**: ad ogni chiamata di `draw` descriviamo nuovamente ciò che vogliamo vedere sullo schermo. È un approccio di tipo *immediate mode*: non modifichiamo direttamente i widget già visualizzati, ma produciamo il nuovo frame a partire dallo stato corrente dell'applicazione. Se vuoi capire meglio il concetto in generale (non è specifico di Ratatui o di Rust): [Immediate mode su Wikipedia](https://en.wikipedia.org/wiki/Immediate_mode_(computer_graphics)).
>
> Questo non significa però che Ratatui riscriva fisicamente tutto il terminale a ogni giro: confronta il nuovo buffer con quello precedente e aggiorna solo le celle che sono cambiate.

## Codice completo finale

Vediamo nella pratica cosa abbiamo ottenuto:

```rust
use ratatui::crossterm::event;
use ratatui::widgets::Paragraph;

fn main() -> Result<(), Box<dyn std::error::Error>> {
    ratatui::run(|terminal| {
        // Siamo nello "schermo-alternato" o "alternate screen"
        loop {
            terminal.draw(|frame| {
                // Crea il widget Paragraph
                let paragraph = Paragraph::new("Ciao, Ratatui! Premi un tasto per uscire.");
                // Disegna il widget
                frame.render_widget(paragraph, frame.area());
            })?;

            // Attendiamo la pressione di un tasto
            if event::read()?.is_key_press() {
                break Ok(());
            }
        }
    })
    // Qui siamo usciti da "alternate screen"
}
```

Nei tutorial su Ratatui incontrerai spesso `color_eyre::Result` o `anyhow::Result` al posto di `Result<(), Box<dyn std::error::Error>>`. Sono soluzioni più comode per la gestione degli errori nelle applicazioni, ma per il momento possiamo tranquillamente farne a meno.

`color-eyre`, in particolare, è molto usato negli esempi Ratatui e produce anche report degli errori più leggibili. Ci torneremo quando ne avremo davvero bisogno.

E per oggi è tutto, la prossima volta vediamo che non c'è solo un modo che possiamo usare per mostrare del testo, ma ce ne sono diversi.

