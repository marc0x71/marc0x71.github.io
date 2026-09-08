+++
title = "Ratatui passo dopo passo - Dal testo semplice a Paragraph"
date = 2026-09-08
description = "Una guida pratica a Span, Line, Text e Paragraph in Ratatui: testo stilizzato, allineamento, wrapping e scroll."
[taxonomies]
tags = ["rust", "ratatui"]
[extra]
comments = true
+++

## Cosa costruiamo oggi

Partiamo dalla finestra minima del primo articolo e la costruiamo un pezzo alla volta, dal componente più piccolo al più capace: prima uno `Span` da solo, poi più `Span` messi in fila in una `Line`, poi più `Line` impilate in un `Text`, e infine lo stesso contenuto passato a `Paragraph`. Ogni passaggio ti mostra concretamente cosa il livello precedente non riusciva a fare. Il risultato finale è lo stesso di prima: più righe colorate, allineate in modi diversi, con un blocco di testo che va a capo automaticamente. Premi un tasto qualsiasi per uscire, come già sapevi fare.

```
┌─────────────────────────────────────────────┐
│ Stato: OK   Avvisi: 2                       │
│                                             │
│ a sinistra                                  │
│                 al centro                   │
│                                    a destra │
│                                             │
│ Questo testo è abbastanza lungo da andare a │
│ capo automaticamente quando supera la       │
│ larghezza dell'area che lo contiene.        │
└─────────────────────────────────────────────┘
```

## Prerequisiti

Partiamo esattamente da dove eravamo rimasti alla fine del precedente articolo: il progetto `hello-ratatui`:

```rust
use ratatui::crossterm::event;
use ratatui::widgets::Paragraph;

fn main() -> std::io::Result<()> {
    ratatui::run(|terminal| {
        loop {
            terminal.draw(|frame| {
                let paragraph = Paragraph::new("Ciao, Ratatui! Premi un tasto per uscire.");
                frame.render_widget(paragraph, frame.area());
            })?;

            if event::read()?.is_key_press() {
                break Ok(());
            }
        }
    })
}
```

Da qui in avanti tocchiamo solo il contenuto della closure passata a `terminal.draw(...)`: il resto — `ratatui::run`, il loop, l'uscita alla pressione di un tasto qualsiasi — resta identico. Nota anche che l'unica dipendenza è `ratatui`: la tastiera si legge tramite `ratatui::crossterm::event`, il re-export della versione di `crossterm` con cui Ratatui stesso è stato compilato, proprio per evitare di finire con due versioni diverse della stessa libreria nel progetto.

## Codice, passo per passo

### `Span` — il pezzo minimo di testo stilizzato

Uno `Span` è testo + stile, e nient'altro: non ha un'idea di "riga", non ha allineamento, non va a capo. È il mattone più piccolo con cui lavorerai.

Due modi per crearne uno, più una scorciatoia:

```rust
use ratatui::style::{Style, Stylize};
use ratatui::text::Span;

let a = Span::raw("testo senza stile");
let b = Span::styled("testo verde e corsivo", Style::new().green().italic());
let c = "testo giallo".yellow(); // scorciatoia via Stylize
```
 
Le scorciatoie di `Stylize` evitano di costruire esplicitamente uno `Style` nei casi più semplici.

`Span` sa disegnarsi da solo, senza bisogno di `Paragraph`: implementa `Widget` proprio come i widget "veri". Prova a sostituire il contenuto di `terminal.draw(|frame| { ... })?;` nel `main.rs` di partenza con questo:

```rust
use ratatui::style::Stylize;
use ratatui::text::Span;

terminal.draw(|frame| {
    let span = Span::raw("Ciao, Ratatui!").yellow();
    frame.render_widget(span, frame.area());
})?;
```

A schermo vedi la stessa identica riga di prima, semplicemente gialla — nessun `Paragraph` di mezzo:

```
Ciao, Ratatui!
```

**Cosa non puoi fare con un solo `Span`:** mescolare più colori nella stessa riga. Se provi a scrivere qualcosa tipo "Stato: **OK** Avvisi: **2**", con `OK` verde e il resto normale, un singolo `Span` non basta — ha un solo stile per l'intero contenuto. Ti serve un contenitore che metta in fila più `Span`: `Line`.

### `Line` — più `Span` sulla stessa riga

Una `Line` è una sequenza di `Span`, disegnati uno dopo l'altro sulla stessa riga di terminale:

```rust
use ratatui::style::{Style, Stylize};
use ratatui::text::{Line, Span};

terminal.draw(|frame| {
    let riga = Line::from(vec![
        Span::raw("Stato: "),
        Span::styled("OK", Style::new().green().bold()),
        Span::raw("   Avvisi: "),
        Span::raw("2").yellow(),
    ]);

    frame.render_widget(riga, frame.area());
})?;
```

Anche `Line`, come `Span`, sa disegnarsi da sola: `frame.render_widget(riga, frame.area())` funziona senza `Paragraph`. A schermo vedi tutti e quattro i pezzi affiancati sulla stessa riga, ognuno con il proprio colore.

`Line` ti dà anche qualcosa che `Span` non ha: l'allineamento.

```rust
use ratatui::text::Line;

let riga = Line::from("al centro").centered();
// scorciatoie equivalenti: .left_aligned(), .right_aligned(), o più direttamente .alignment(Alignment::Center)
```

Da precisare: `.centered()` centra la riga **nello spazio che le è stato assegnato**, non nell'intero terminale. Qui coincidono perché stiamo ancora disegnando su `frame.area()`, cioè tutto lo schermo — ma quando nel prossimo articolo dividerai lo schermo in aree più piccole con `Layout`, vedrai che l'allineamento centra sempre rispetto all'area passata a `render_widget`, qualunque essa sia, ma di questo parleremo in un prossimo articolo.

> **`Line` non va mai a capo da sola.** Per quanto sia lungo il contenuto che ci metti dentro, una `Line` resta sempre una singola riga di terminale: se non ci sta, viene **tagliata**, non spezzata su più righe. Prova a restringere il terminale finché una riga lunga non ci sta più, e guarda la parte finale sparire invece di scendere a capo. Se il testo sorgente contiene `\n` i newline vengono rimossi durante la costruzione.

**Cosa non puoi fare con una sola `Line`:** più righe di testo. `Line` è, appunto, una riga sola — per definizione. Se vuoi un blocco di testo su più righe (magari con allineamenti diversi riga per riga, come nel mockup di apertura), ti serve un contenitore che raccolga più `Line`: `Text`.

### `Text` — più `Line`, il blocco di testo completo

`Text` è proprio questo: una raccolta di `Line`, con in più un proprio stile e un proprio allineamento complessivi. È il livello a cui finalmente hai un blocco di testo multi-riga vero e proprio:

```rust
use ratatui::text::{Line, Text};

terminal.draw(|frame| {
    let testo = Text::from(vec![
        Line::from("a sinistra").left_aligned(),
        Line::from("al centro").centered(),
        Line::from("a destra").right_aligned(),
    ]);

    frame.render_widget(testo, frame.area());
})?;
```

A schermo vedi tre righe, ognuna allineata come indicato:

```
a sinistra
                al centro
                                a destra
```

(Vale anche il contrario di quanto visto finora: puoi costruire un `Text` con un solo colore/stile per l'intero blocco usando `Text::styled(contenuto, stile)`, invece di stilizzare ogni singola `Line`.)

Verrebbe da pensare che un `Vec<Line>` basti già per avere un blocco di testo su più righe — dopotutto è già una collezione di `Line`. Non è così: un `Vec<Line>` da solo *non* implementa `Widget`, quindi non puoi passarlo direttamente a `render_widget`. Serve un contenitore che lo impacchetti e che sappia disegnarsi da sé: `Text`.

**Cosa non puoi fare con un `Text` renderizzato direttamente:** andare a capo automaticamente, scorrere il contenuto, o avvolgerlo in un bordo. Se aggiungi al `Vec` di prima questa riga

```rust
Line::from(
    "Questo testo è abbastanza lungo da andare a capo automaticamente \
     quando supera la larghezza dell'area che lo contiene.",
),
```

E la finestra del terminale non è abbastanza larga, la vedi tagliata — esattamente come una `Line` singola, perché `Text` eredita lo stesso comportamento. Per il wrapping, per lo scroll, e per un bordo attorno al tutto, serve l'ultimo livello: `Paragraph`.

### `Paragraph` — wrapping e scroll, quello che `Text` non fa

`Paragraph::new(...)` accetta qualunque cosa sappia trasformarsi in un `Text` — quindi, in pratica, tutto quello che hai appena imparato a costruire: una `&str`, una `String`, una `Line`, un `Vec<Line>`, o un `Text` già pronto. Prendi lo stesso identico contenuto di prima e passalo a `Paragraph` invece di renderizzarlo direttamente:

```rust
use ratatui::text::Line;
use ratatui::widgets::{Paragraph, Wrap};

terminal.draw(|frame| {
    let testo = vec![
        Line::from("a sinistra").left_aligned(),
        Line::from("al centro").centered(),
        Line::from("a destra").right_aligned(),
        Line::from(
            "Questo testo è abbastanza lungo da andare a capo automaticamente \
             quando supera la larghezza dell'area che lo contiene.",
        ),
    ];

    let paragraph = Paragraph::new(testo).wrap(Wrap { trim: true });

    frame.render_widget(paragraph, frame.area());
})?;
```

Stessa struttura dati di prima (nemmeno serve costruire esplicitamente un `Text`: il `Vec<Line>` va bene così, `Paragraph::new` fa la conversione da solo), ma ora, se restringi il terminale, l'ultima riga va a capo invece di tagliarsi — è la differenza concreta tra `Text` e `Paragraph`. `trim: true` toglie gli spazi bianchi all'inizio di ogni riga spezzata; con `trim: false` li mantiene.

Oltre al wrapping, `Paragraph` aggiunge lo scroll, con `.scroll((y, x))`: sposta la "finestra" di testo visibile, anche se — come vedremo — sei tu a dover tenere da qualche parte il valore di `y` e aggiornarlo, ad esempio quando l'utente preme una freccia.

### Cosa `Paragraph` non fa

- **Lo scroll non è automatico.** `.scroll((y, x))` esiste, ma sei tu a dover gestire il valore di `y` e aggiornarlo — questo lo vedremo quando parleremo di gestione dello stato.
- **Non gestisce input o selezione del testo.** Non c'è un cursore, non puoi selezionare o copiare porzioni di testo: è puro output.
- **Non interpreta markup.** Niente Markdown, niente HTML, niente sequenze tipo `**grassetto**`: ogni pezzo stilizzato va costruito esplicitamente come `Span`.
- **Il wrapping è intenzionalmente semplice.** Va a capo tra una parola e l'altra, non fa sillabazione né wrapping "intelligente" tipografico . 

> **Per approfondire.** Questi limiti non sono un difetto di progettazione: `Paragraph` è pensato per il caso "voglio mostrare del testo, eventualmente stilizzato", non per essere un editor o un motore di layout testuale. Per liste selezionabili c'è `List`, per tabelle c'è `Table` — li vedremo più avanti nella serie.

### Codice completo

```rust
use ratatui::crossterm::event;
use ratatui::style::{Style, Stylize};
use ratatui::text::{Line, Span};
use ratatui::widgets::{Paragraph, Wrap};

fn main() -> std::io::Result<()> {
    ratatui::run(|terminal| {
        loop {
            terminal.draw(|frame| {
                let testo = vec![
                    Line::from(vec![
                        Span::raw("Stato: "),
                        Span::styled("OK", Style::new().green().bold()),
                        Span::raw("   Avvisi: "),
                        Span::raw("2").yellow(),
                    ]),
                    Line::default(),
                    Line::from("a sinistra").left_aligned(),
                    Line::from("al centro").centered(),
                    Line::from("a destra").right_aligned(),
                    Line::default(),
                    Line::from(
                        "Questo testo è abbastanza lungo da andare a capo automaticamente \
             quando supera la larghezza dell'area che lo contiene.",
                    ),
                ];

                let paragraph = Paragraph::new(testo).wrap(Wrap { trim: true });

                frame.render_widget(paragraph, frame.area());
            })?;
            if event::read()?.is_key_press() {
                break Ok(());
            }
        }
    })
}
```

### Conclusioni

Riassumendo: `Span`, `Line` e `Text` sanno disegnarsi da soli, senza `Paragraph`. La scelta pratica è semplice: usa `Paragraph` quando ti serve almeno una di queste tre cose in più, wrapping, scroll o (dal prossimo argomento della serie) un bordo con `Block`; altrimenti renderizzare direttamente il livello più basso che ti basta è perfettamente legittimo, ed è la stessa documentazione ufficiale di Ratatui a suggerirlo.

Rispetto al precedente articolo, ora conosci i quattro livelli con cui Ratatui rappresenta il testo — `Span` (contenuto + stile), `Line` (più `Span` sulla stessa riga, con allineamento), `Text` (più `Line` impilate) e `Paragraph` (wrapping e scroll in più) — e sai che i primi tre sanno disegnarsi da soli, senza passare da `Paragraph`. Sai costruire `Span` con `Span::raw`/`Span::styled`/le scorciatoie di `Stylize`, comporli in `Line`, impilare più `Line` in un `Text`, allineare per riga, e attivare il wrapping con `.wrap()`. E sai anche, cosa altrettanto utile, cosa `Paragraph` deliberatamente non fa.

