+++
title = "Ratatui passo dopo passo - Input da tastiera: distinguere i tasti con `KeyCode`"
date = 2026-10-05
description = "Impariamo a gestire l'input da tastiera in Ratatui con `KeyCode`: distinguiamo frecce, Invio, caratteri ed Esc, aggiorniamo lo stato dell'app e trasformiamo il loop in un vero ciclo di gestione degli eventi."
[taxonomies]
tags = ["rust", "ratatui"]
[extra]
comments = true
+++

## Cosa costruiamo oggi

Finora il programma reagiva a "un tasto qualsiasi" — sempre lo stesso comportamento, chiudere l'app. Oggi lo facciamo reagire in modo diverso a tasti diversi: le frecce cambiano il testo a schermo, Invio conferma qualcosa, `q` o `Esc` chiudono l'app, tutti gli altri tasti vengono ignorati senza interrompere il programma. Il punto centrale non è solo *quale* tasto è stato premuto, ma *come cambia il loop* quando smette di essere "leggi un evento, esci" e diventa "leggi un evento, decidi cosa fare, continua o esci".

```
Ultimo tasto premuto: Freccia su
```

Riprendiamo il codice completo del primo articolo: il progetto `hello-ratatui`

```rust
use ratatui::{crossterm::event, widgets::Paragraph};

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

`ratatui::run(closure)` è una scorciatoia comoda: prepara il terminale (modalità raw, schermo alternato), ti passa un `Terminal` dentro la closure, e quando la closure finisce — anche per un panico — ripristina tutto. Comoda, ma per oggi ci mette in mezzo un confine scomodo: vogliamo uscire dal loop portandoci dietro un valore (quale tasto è stato premuto), e farlo attraverso il tipo di ritorno di una closure generica è più complicato del necessario.

Ratatui espone lo stesso lavoro di preparazione/ripristino anche come due funzioni libere, che chiami tu direttamente, senza closure di mezzo: `ratatui::init()` prepara il terminale e ti restituisce un `Terminal` che possiedi tu; `ratatui::restore()` lo rimette a posto quando hai finito. Guardiamo prima il cambio più semplice possibile: togliere `ratatui::run` e scrivere `init`/`restore` a mano, senza toccare nient'altro nel loop:

```rust
use ratatui::crossterm::event;
use ratatui::widgets::Paragraph;

fn main() -> std::io::Result<()> {
    let mut terminal = ratatui::init();

    loop {
        terminal.draw(|frame| {
            let paragraph = Paragraph::new("Ciao, Ratatui! Premi un tasto per uscire.");
            frame.render_widget(paragraph, frame.area());
        })?;

        if event::read()?.is_key_press() {
            break;
        }
    }

    ratatui::restore();
    Ok(())
}
```

A questo punto, se lanci `cargo run`, il comportamento è identico a prima — un tasto qualsiasi chiude il programma — ma guarda cosa è cambiato nella forma del codice. Non c'è più una closure: `loop { ... }` ora è un'istruzione al livello di `main`, non qualcosa che passi a `ratatui::run(...)`. Di conseguenza `break` non porta più `Ok(())`: prima quel valore doveva diventare il risultato della closure, ora il `loop` non produce alcun valore che usiamo, quindi `break` da solo basta per uscirne. Ripristini il terminale esplicitamente con `ratatui::restore()` subito dopo il loop, e chiudi con `Ok(())` come `main` si aspetta.

Con il loop finalmente a vista, senza un confine di closure di mezzo, possiamo modificarlo per portarci dietro il tasto premuto invece di limitarci a uscire. C'è anche un altro dettaglio che finora avevi delegato a `is_key_press()`: `event::read()?` non restituisce direttamente un tasto, ma un `Event`, un enum con più varianti possibili — pressione di tasto, ma anche ridimensionamento del terminale, movimento del mouse (se abilitato), incolla da clipboard. Per arrivare al tasto vero e proprio serve estrarre la variante `Event::Key` con un `if let`, che ti dà accesso a un `KeyEvent` — ed è proprio `key.code`, dentro quel `KeyEvent`, il `KeyCode` che ti dice quale tasto è stato premuto:

```rust
use ratatui::crossterm::event::{self, Event};
use ratatui::widgets::Paragraph;

fn main() -> std::io::Result<()> {
    let mut terminal = ratatui::init();

    let key = loop {
        terminal.draw(|frame| {
            let paragraph = Paragraph::new("Ciao, Ratatui! Premi un tasto per uscire.");
            frame.render_widget(paragraph, frame.area());
        })?;

        if let Event::Key(key) = event::read()? {
            break key;
        }
    };

    ratatui::restore();
    println!("{key:?}");
    Ok(())
}
```

Nota quanto è diretto rispetto a prima: `let key = loop { ... break key; };` è puro Rust — il valore passato a `break` diventa il valore dell'intera espressione `loop`, senza bisogno di avvolgerlo in `Ok(...)`. Anche i `?` dentro il loop (su `terminal.draw(...)` e `event::read()`) funzionano senza complicazioni, perché propagano direttamente verso il tipo di ritorno di `main`, non verso quello di una closure intermedia. E l'ordine tra `ratatui::restore()` e `println!("{key:?}")` non è casuale: prima ripristini il terminale normale, *poi* stampi — coerente con quello che abbiamo visto sul perché stampare mentre sei ancora nello schermo alternato dà risultati disordinati.

A questo punto, se lanci `cargo run`, il programma disegna il messaggio come sempre, esce al primo tasto qualsiasi (ancora nessuna distinzione tra i tasti), e appena tornato al prompt normale stampa qualcosa come `KeyEvent { code: Char('a'), kind: Press, ... }`. Da lì vedi già `key.code`, che è il pezzo che ci interessa.

### Il loop, prima e dopo: da "un ramo" a "più rami"

Guarda la differenza di *forma* tra il loop che avevi e quello che ti serve ora — è il cuore di questo articolo, più ancora di `KeyCode` in sé.

**Prima**, il corpo del loop dopo `draw` era un singolo `if`, con una sola conseguenza possibile: uscire.

```rust
if event::read()?.is_key_press() {
    break Ok(());
}
```

**Ora**, la stessa posizione nel codice deve poter fare cose diverse a seconda del tasto: alcuni tasti escono, altri modificano qualcosa e fanno ricominciare il loop, altri ancora vengono ignorati del tutto. Un singolo `if` non basta più — ti serve un `match` su `key.code`, con tanti rami quanti sono i comportamenti che vuoi distinguere:

```rust
if let Event::Key(key) = event::read()? {
    match key.code {
        KeyCode::Esc | KeyCode::Char('q') => break Ok(()),
        KeyCode::Up => { ... }
        KeyCode::Down => { ... }
        _ => {}
    }
}
```

Come sai già, `match` in Rust deve essere esaustivo: senza il ramo `_ => {}` il codice non compilerebbe nemmeno, perché `KeyCode` ha molte più varianti (tasti funzione, modificatori, tasti multimediali...) di quelle che elenchiamo esplicitamente qui.

Riempiamo ora i rami rimasti vuoti. Ci serve una variabile che tenga "l'ultimo tasto premuto" tra un giro di loop e l'altro — dichiarata **fuori** dal loop, altrimenti verrebbe ricreata da zero a ogni giro — e passata al `Paragraph` al posto della stringa fissa:

```rust
let mut ultimo_tasto = String::from("nessuno");

// ...

match key.code {
    KeyCode::Esc | KeyCode::Char('q') => break Ok(()),
    KeyCode::Up => ultimo_tasto = String::from("Freccia su"),
    KeyCode::Down => ultimo_tasto = String::from("Freccia giù"),
    KeyCode::Left => ultimo_tasto = String::from("Freccia sinistra"),
    KeyCode::Right => ultimo_tasto = String::from("Freccia destra"),
    KeyCode::Enter => ultimo_tasto = String::from("Invio"),
    KeyCode::Char(c) => ultimo_tasto = format!("Lettera '{c}'"),
    _ => {}
}

// ...

let paragraph = Paragraph::new(format!("Ultimo tasto premuto: {ultimo_tasto}"));
```

Il penultimo ramo, `KeyCode::Char(c)`, merita un chiarimento: `KeyCode::Char` non è un singolo tasto, ma una famiglia — porta con sé il carattere effettivo premuto (`a`, `Z`, `5`, `!`...) come dato agganciato alla variante. `c` qui è quel carattere, e lo puoi usare come qualunque altro `char` di Rust — in questo caso lo interpoli nella stringa con `format!`. Nota anche che `KeyCode::Char('q')` nel primo ramo è un caso *specifico* di questa stessa famiglia: Rust prova i rami del `match` in ordine dall'alto in basso, quindi mettendo `KeyCode::Esc | KeyCode::Char('q')` prima del generico `KeyCode::Char(c)`, la lettera `q` viene intercettata lì ed esce dal programma, mentre tutte le altre lettere arrivano al ramo generico più sotto.

A questo punto, se lanci `cargo run`, il testo a schermo cambia a ogni pressione di freccia, lettera o Invio, mentre `q` o `Esc` chiudono il programma come ti aspetti. Prova anche un tasto non gestito, come `F1` o Tab: il testo resta quello di prima, il programma non si blocca né esce — è il ramo `_ => {}` in azione.

Magari non salta subito all'occhio, ma `ultimo_tasto` è già una piccola anteprima di un concetto centrale in Ratatui: lo stato. Ne parleremo esplicitamente — con una struct dedicata al posto di una variabile sciolta — nel prossimo articolo.

### Un dettaglio a cui prestare attenzione: pressione vs rilascio

`key` porta con sé un campo `kind`, di tipo `KeyEventKind`, che distingue proprio questo: `KeyEventKind::Press`, `KeyEventKind::Release`, `KeyEventKind::Repeat` (quando tieni premuto un tasto a lungo). Per essere sicuri di reagire una sola volta a pressione, aggiungi la condizione al tuo `if let`:

```rust
if let Event::Key(key) = event::read()?
    && key.kind == event::KeyEventKind::Press
{
    match key.code {
        // ...
    }
}
```

### Il loop non va mai via: conseguenza dell'*immediate mode*

Fai un passo indietro e guarda cosa non è cambiato in tutti questi articoli: c'è sempre stato un loop attorno a `draw` + lettura evento, dal primissimo "Ciao, Ratatui!" fino a qui. E' conseguenza diretta di come funziona Ratatui, l'*immediate mode* di cui avevamo accennato nel primo articolo. Non esiste un albero di widget persistente che si aggiorna da solo quando cambia un dato, ogni volta che vuoi che qualcosa di nuovo appaia a schermo (un tasto premuto, un valore cambiato, il tempo che passa), devi richiamare `terminal.draw(...)`, ridisegnando concettualmente tutto. 

Quello che cambia da un articolo all'altro non è *se* c'è un loop, ma cosa succede dentro un suo giro: oggi è un `match` con sei rami invece di un `if` con uno solo, ma la forma resta "disegna, aspetta un evento, decidi, ricomincia". Man mano avremo uno stato più ricco, più tipi di widget, aggiornamenti che arrivano anche da un thread in background, quella forma di fondo resta la stessa: cambierà solo cosa succede dentro al giro, e da dove arrivano gli eventi a cui reagisci.

## Codice completo finale

```rust
use ratatui::crossterm::event::{self, Event, KeyCode};
use ratatui::widgets::Paragraph;

fn main() -> std::io::Result<()> {
    let mut ultimo_tasto = String::from("nessuno");

    ratatui::run(|terminal| {
        loop {
            terminal.draw(|frame| {
                let paragraph =
                    Paragraph::new(format!("Ultimo tasto premuto: {ultimo_tasto}"));
                frame.render_widget(paragraph, frame.area());
            })?;

            if let Event::Key(key) = event::read()?
                && key.kind == event::KeyEventKind::Press
            {
                match key.code {
                    KeyCode::Esc | KeyCode::Char('q') => break Ok(()),
                    KeyCode::Up => ultimo_tasto = String::from("Freccia su"),
                    KeyCode::Down => ultimo_tasto = String::from("Freccia giù"),
                    KeyCode::Left => ultimo_tasto = String::from("Freccia sinistra"),
                    KeyCode::Right => ultimo_tasto = String::from("Freccia destra"),
                    KeyCode::Enter => ultimo_tasto = String::from("Invio"),
                    KeyCode::Char(c) => ultimo_tasto = format!("Lettera '{c}'"),
                    _ => {}
                }
            }
        }
    })
}
```

Ora sai estrarre il dettaglio di un evento tastiera con `if let Event::Key(key) = ...`, distinguere i tasti con un `match` su `key.code` (incluso il caso speciale `KeyCode::Char(c)`, che porta con sé il carattere premuto), ed evitare doppie letture dello stesso tasto controllando `key.kind == KeyEventKind::Press`. Soprattutto, hai visto la vera differenza rispetto a prima: il loop non ha più un solo esito possibile a ogni giro, ma più rami — alcuni escono, altri aggiornano una variabile esterna al loop e lo fanno ripartire, uno ignora tutto il resto senza fare nulla. E hai visto perché quel loop, in una forma o nell'altra, non sparisce mai in nessuna app Ratatui: è la diretta conseguenza dell'immediate mode, non un dettaglio implementativo di questo tutorial.
