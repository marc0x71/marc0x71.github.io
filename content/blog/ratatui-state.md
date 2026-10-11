+++
title = "Ratatui passo dopo passo - Gestire lo stato dell'app: una struct `App` al posto delle variabili sciolte"
date = 2026-10-11
description = "Facciamo crescere la nostra TUI con Ratatui: raccogliamo lo stato in una struct App, contiamo i tasti premuti e impariamo a riconoscere i modificatori Ctrl, Alt e Shift."
[taxonomies]
tags = ["rust", "ratatui"]
[extra]
comments = true
+++


Riprendiamo il programma dell'articolo scorso e lo facciamo crescere: non più solo l'ultimo tasto premuto, ma anche quante volte hai premuto un tasto e se c'era un modificatore (Ctrl, Alt, Shift) attivo. Con tre cose da tenere a mente invece di una, le variabili sciolte iniziano a starci strette — è il momento giusto per raccoglierle in un'unica struct `App`.

```
Ultimo tasto premuto: Freccia su
Tasti premuti finora: 7
Ultimo modificatore: Ctrl
```

Ma partiamo da zero, quindi riprendiamo il codice completo dell'articolo precedente: 

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

E proviamo quindi ad aggiungere un contatore, con lo stesso stile di prima, quindi un'altra variabile mutabile, dichiarata fuori dal loop accanto a `ultimo_tasto`.

```rust
let mut ultimo_tasto = String::from("nessuno");
let mut contatore_tasti: u32 = 0;
```

La incrementiamo a ogni pressione di tasto, prima ancora del `match` (perché vogliamo contare ogni tasto, compresi quelli che il `match` ignora):

```rust
if let Event::Key(key) = event::read()?
    && key.kind == event::KeyEventKind::Press
{
    contatore_tasti += 1;

    match key.code {
        // ... invariato
    }
}
```

E la aggiungiamo al testo mostrato a schermo:

```rust
let paragraph = Paragraph::new(format!(
    "Ultimo tasto premuto: {ultimo_tasto}\nTasti premuti finora: {contatore_tasti}"
));
```

A questo punto, se lanci `cargo run`, il contatore sale a ogni tasto premuto, compresi quelli che il match ignora, come Tab o Backspace. Funziona — ma guarda cosa è già successo: due `let mut` dichiarate nello stesso punto per nessun altro motivo se non perché "vivono insieme", aggiornate in due punti diversi del codice (una dentro il `match`, l'altra prima), passate entrambe a `Paragraph` a mano, una per una. Con due variabili è solo scomodo. Con cinque o sei diventa facile dimenticarsene una da qualche parte.

### Raccogliere lo stato in una struct App

Il rimedio è raccogliere queste variabili in un'unica struct, con un nome che dice cosa rappresentano davvero: non "due variabili che capitano nello stesso scope", ma "lo stato dell'applicazione".

```rust
struct App {
    ultimo_tasto: String,
    contatore_tasti: u32,
}
```

E un solo `let mut` fuori dal loop, al posto dei due di prima:

```rust
let mut app = App {
    ultimo_tasto: String::from("nessuno"),
    contatore_tasti: 0,
};
```

Il resto del codice cambia in modo meccanico: ogni `ultimo_tasto` diventa `app.ultimo_tasto`, ogni `contatore_tasti` diventa `app.contatore_tasti`.

```rust
if let Event::Key(key) = event::read()?
    && key.kind == event::KeyEventKind::Press
{
    app.contatore_tasti += 1;

    match key.code {
        KeyCode::Esc | KeyCode::Char('q') => break Ok(()),
        KeyCode::Up => app.ultimo_tasto = String::from("Freccia su"),
        KeyCode::Down => app.ultimo_tasto = String::from("Freccia giù"),
        KeyCode::Left => app.ultimo_tasto = String::from("Freccia sinistra"),
        KeyCode::Right => app.ultimo_tasto = String::from("Freccia destra"),
        KeyCode::Enter => app.ultimo_tasto = String::from("Invio"),
        KeyCode::Char(c) => app.ultimo_tasto = format!("Lettera '{c}'"),
        _ => {}
    }
}
```

```rust
let paragraph = Paragraph::new(format!(
    "Ultimo tasto premuto: {}\nTasti premuti finora: {}",
    app.ultimo_tasto, app.contatore_tasti
));
```

A questo punto, se lanci `cargo run`, il comportamento a schermo è identico a prima del refactoring — è proprio il punto: raccogliere lo stato in una struct non cambia cosa fa il programma, cambia solo quanto è facile ragionarci sopra e farlo crescere.

**Una nota per il futuro**: con uno stato tutto raccolto in `app` potresti prima o poi scontrarti con il borrow checker. Non è questo il caso, ma succede facilmente con struct più complesse e annidate, o quando inizi ad aggiungere metodi che prendono `&mut self`. In quei casi torna utile il destructuring, che scompone la struct nei suoi campi e ti dà un prestito indipendente per ciascuno:

```rust
let App { ultimo_tasto, contatore_tasti } = &mut app;
```

Magari ci torneremo quando servirà davvero.

### Novità: `key.modifiers`, per sapere se c'era Ctrl/Alt/Shift

Finora da `KeyEvent` avevi usato solo `key.code` (quale tasto) e `key.kind` (pressione o rilascio). C'è un terzo campo che non abbiamo ancora toccato, `key.modifiers`, che ti dice se insieme al tasto erano premuti anche Ctrl, Alt o Shift. Si comporta come un piccolo insieme di modificatori attivi, e lo interroghi con `.contains(...)`:

```rust
use ratatui::crossterm::event::KeyModifiers;

if key.modifiers.contains(KeyModifiers::CONTROL) {
    // Ctrl era premuto insieme al tasto
}
```

Aggiungiamo un terzo campo alla struct per tenerne traccia:

```rust
struct App {
    ultimo_tasto: String,
    contatore_tasti: u32,
    ultimo_modificatore: String,
}
```

```rust
let mut app = App {
    ultimo_tasto: String::from("nessuno"),
    contatore_tasti: 0,
    ultimo_modificatore: String::from("nessuno"),
};
```

E lo aggiorniamo insieme al contatore, prima del `match` (un modificatore può accompagnare qualunque tasto, non solo alcuni):

```rust
app.contatore_tasti += 1;
app.ultimo_modificatore = if key.modifiers.contains(KeyModifiers::CONTROL) {
    String::from("Ctrl")
} else if key.modifiers.contains(KeyModifiers::ALT) {
    String::from("Alt")
} else if key.modifiers.contains(KeyModifiers::SHIFT) {
    String::from("Shift")
} else {
    String::from("nessuno")
};
```

A questo punto, se lanci `cargo run` e premi ad esempio Ctrl insieme a una lettera, vedi "Ultimo modificatore: Ctrl" comparire a schermo. Su tasti senza modificatore torna a "nessuno" al giro successivo.

> **Per approfondire.** Con i tasti-carattere c'è una sottigliezza da tenere a mente: premendo Shift+A, `key.code` arriva già come `Char('A')` invece di `Char('a')` — il maiuscolo lo vedi comunque nel carattere, a prescindere da `key.modifiers`. Se in più anche `key.modifiers` segnali `SHIFT` insieme a un carattere già maiuscolo dipende dal terminale e dal backend in uso, e non è un comportamento garantito in modo uniforme ovunque. Per tasti senza un "maiuscolo" — frecce, Invio, tasti funzione — `key.modifiers` resta l'unico modo per sapere se Shift era premuto; per le lettere, la fonte più affidabile è il carattere che ricevi in `key.code`, non il modificatore.

### Regole per uno stato pulito

C'è una disciplina che vale la pena rendere esplicita, ora che `App` esiste: lo stato non deve sapere come è stato prodotto. `App` rappresenta "cosa è vero adesso" — l'ultimo tasto, quante volte è stato premuto, quale modificatore è attivo — non "cosa è appena successo sulla tastiera". Guarda dove finisce `key` nel codice: entra ed esce dal blocco `if let Event::Key(key) = ...`, e non va oltre. La closure di `terminal.draw(...)`, che disegna a schermo, non vede mai né `key` né `Event`: legge soltanto i campi di `app`.

Questa separazione non richiede una riga di codice nuova rispetto a quello che hai già scritto: il `draw` veniva prima della lettura dell'evento anche prima. È però il motivo per cui questa struttura regge bene man mano che l'app cresce. Il codice che disegna non deve chiedersi da dove arriva un dato: per lui la risposta è sempre la stessa, è già dentro `app`.

Rivediamo la forma del loop, è sempre la stessa da quando l'abbiamo introdotta:

```text
disegna → leggi un evento → aggiorna qualcosa → ricomincia
```

Questa forma non è una particolarità di Ratatui: è il classico event loop di un'applicazione interattiva. L'immediate mode, di cui parlavamo nel primo articolo, entra in gioco in un altro punto: spiega perché, dentro `terminal.draw(...)`, possiamo semplicemente leggere `app` e ridisegnare tutto da capo, senza dover aggiornare manualmente soltanto i pezzi dello schermo che sono cambiati.

In altre parole, ci sono due meccanismi che collaborano ma non vanno confusi. Il loop procura gli eventi e, a ogni giro, richiama il rendering; l'immediate mode fa sì che il rendering possa essere una funzione dello stato corrente, senza doversi ricordare cosa era disegnato prima. Possiamo pensarlo, schematicamente, così:

```text
UI = f(App)
```

Quello che è cambiato oggi, quindi, non è il loop: è ciò che succede nel passo "aggiorna qualcosa". Prima erano assegnazioni sparse a variabili diverse; ora sono tutte assegnazioni a campi dello stesso `App`. Il meccanismo che tiene in piedi tutto resta identico.

Ci sono altre due proprietà di `App`, nello stesso spirito, che vale la pena rendere volute e non casuali.

La prima: guarda i tipi dei campi di `App` — `String`, `u32`. Nessuno viene da `ratatui`. Questo significa che `App` non dipende in alcun modo dalla libreria che stai usando per disegnare: potresti, per dire, scrivere un test che costruisce e controlla un `App` senza avviare un terminale, senza toccare `ratatui::run`, senza nemmeno importare `ratatui` in quel test:

```rust
#[cfg(test)]
mod tests {
    use super::App;

    #[test]
    fn stato_iniziale_ha_contatore_a_zero() {
        let app = App {
            ultimo_tasto: String::from("nessuno"),
            contatore_tasti: 0,
            ultimo_modificatore: String::from("nessuno"),
        };

        assert_eq!(app.contatore_tasti, 0);
    }
}
```

Non è un test che aggiungiamo al progetto oggi — `hello-ratatui` resta un veicolo per imparare, non un'applicazione da coprire di test — ma il fatto che *potresti* scriverlo, e che funzionerebbe senza toccare `ratatui`, è la prova concreta che lo stato è davvero disaccoppiato dalla libreria grafica. Se un domani cambiassi libreria per disegnare l'interfaccia, questa definizione di `App` potrebbe rimanere invariata.

La seconda: `App` per ora contiene solo dati, nessun metodo — il `match` che decide *cosa* cambia resta nel `main`, fuori dalla struct. Non è un giudizio su come si scrive Rust in generale: un `impl App` con metodi propri è assolutamente idiomatico, spesso la scelta preferibile. Qui, semplicemente, scegliamo di non introdurli ancora, per tenere nette le due domande finché sono nuove per chi legge: cosa l'app ricorda, e come reagisce a un evento.

Questa separazione tra stato e interfaccia non è esclusiva di Ratatui. La ritrovi anche in altre librerie grafiche, come `egui`, sempre nell'ecosistema Rust: l'applicazione conserva i propri dati in una struct e l'interfaccia viene costruita leggendo lo stato corrente. Cambiano le API e il modo in cui vengono gestiti gli eventi, ma l'idea di fondo rimane la stessa: lo stato descrive cosa sa l'applicazione, il codice di rendering decide come mostrarlo.

## Codice completo finale

```rust
use ratatui::crossterm::event::{self, Event, KeyCode, KeyModifiers};
use ratatui::widgets::Paragraph;

struct App {
    ultimo_tasto: String,
    contatore_tasti: u32,
    ultimo_modificatore: String,
}

fn main() -> std::io::Result<()> {
    let mut app = App {
        ultimo_tasto: String::from("nessuno"),
        contatore_tasti: 0,
        ultimo_modificatore: String::from("nessuno"),
    };

    ratatui::run(|terminal| {
        loop {
            terminal.draw(|frame| {
                let paragraph = Paragraph::new(format!(
                    "Ultimo tasto premuto: {}\nTasti premuti finora: {}\nUltimo modificatore: {}",
                    app.ultimo_tasto, app.contatore_tasti, app.ultimo_modificatore
                ));
                frame.render_widget(paragraph, frame.area());
            })?;

            if let Event::Key(key) = event::read()?
                && key.kind == event::KeyEventKind::Press
            {
                app.contatore_tasti += 1;
                app.ultimo_modificatore = if key.modifiers.contains(KeyModifiers::CONTROL) {
                    String::from("Ctrl")
                } else if key.modifiers.contains(KeyModifiers::ALT) {
                    String::from("Alt")
                } else if key.modifiers.contains(KeyModifiers::SHIFT) {
                    String::from("Shift")
                } else {
                    String::from("nessuno")
                };

                match key.code {
                    KeyCode::Esc | KeyCode::Char('q') => break Ok(()),
                    KeyCode::Up => app.ultimo_tasto = String::from("Freccia su"),
                    KeyCode::Down => app.ultimo_tasto = String::from("Freccia giù"),
                    KeyCode::Left => app.ultimo_tasto = String::from("Freccia sinistra"),
                    KeyCode::Right => app.ultimo_tasto = String::from("Freccia destra"),
                    KeyCode::Enter => app.ultimo_tasto = String::from("Invio"),
                    KeyCode::Char(c) => app.ultimo_tasto = format!("Lettera '{c}'"),
                    _ => {}
                }
            }
        }
    })
}
```

