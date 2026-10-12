+++
title = "Ratatui passo dopo passo - Un text input basilare: campo di testo e cursore fatti in casa"
date = 2026-10-12
description = "Costruiamo un campo di testo in Ratatui da zero: inserimento, Backspace, movimento del cursore e gestione UTF-8 in Rust. Poi scopriamo come semplificare tutto con ratiform."
[taxonomies]
tags = ["rust", "ratatui"]
[extra]
comments = true
+++

Ratatui **non include un widget per l'inserimento di testo**, nemmeno nella 0.30. Non è una svista né un limite temporaneo: `Paragraph` è pensato per *mostrare* testo, non per farlo modificare da chi lo guarda — non ha un'idea di "cursore", non intercetta i tasti, non sa cosa significa "cancella l'ultimo carattere". Se in un'app da terminale vuoi un campo dove l'utente scrive qualcosa, quel pezzo di comportamento non te lo dà nessuna libreria: te lo scrivi tu, sopra gli strumenti che già conosci — gestione dello stato e lettura degli eventi tastiera, viste negli articoli precedenti.

Il risultato di oggi: un campo di testo su una sola riga, dove puoi scrivere, cancellare con Backspace, muovere il cursore con le frecce, e confermare con Invio portandoti dietro quello che hai scritto.

```
Inserisci il tuo nome: Marco|
```

(il `|` sta per il cursore lampeggiante vero e proprio, quello del terminale)

Partiamo sempre dal codice minimo iniziale della serie: il progetto `hello-ratatui`, cioè con questo `main.rs`:

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

## Codice, passo per passo

### Dal loop minimo alla forma che già conosci

Il loop di partenza reagisce a "un tasto qualsiasi" con `is_key_press()` — va bene per uscire, non basta per distinguere cosa scrivere. Ci serve la forma a cui siamo già arrivati in precedenza: estrarre `Event::Key(key)`, filtrare solo le pressioni con `key.kind == KeyEventKind::Press`, e decidere cosa fare con un `match` su `key.code`. Non la ricostruiamo passo passo — l'abbiamo già fatto — la usiamo direttamente come punto di partenza per il `match` di oggi:

```rust
use ratatui::crossterm::event::{self, Event, KeyCode};

if let Event::Key(key) = event::read()?
    && key.kind == event::KeyEventKind::Press
{
    match key.code {
        _ => {}
    }
}
```

Anche lo stato lo trattiamo come già acquisito: una struct dichiarata fuori dal loop, aggiornata dentro il `match`, letta dalla closure di `draw`. Quello che cambia oggi non è la forma, ma cosa ci mettiamo dentro.

### Uno stato minimo per il campo di testo

Un campo di testo, ridotto all'osso, deve ricordarsi due cose: **cosa** c'è scritto, e **dove** si trova il cursore dentro quel testo.

```rust
struct App {
    testo: String,
    cursore: usize,
}
```

`cursore` non è una posizione a schermo (colonna, riga): è un indice **dentro `testo`**, espresso in caratteri — "il cursore sta prima del terzo carattere"

```rust
let mut app = App {
    testo: String::new(),
    cursore: 0,
};
```

### Etichetta e campo, uno accanto all'altro

Fin qui il testo lo abbiamo sempre disegnato con `Paragraph`. Oggi lo sostituiamo con qualcosa di più mirato: uno `Span` per l'etichetta ("Inserisci il tuo nome: "), e un `Block` colorato di grigio chiaro per il campo vero e proprio, che segnala visivamente quant'è lo spazio a disposizione per scrivere — un piccolo effetto "casella di testo", pur restando sempre e solo widget disegnati su un terminale.

`Layout::horizontal` divide solo la larghezza: usato direttamente su `frame.area()`, l'area del campo resterebbe alta quanto tutto lo schermo. Isoliamo prima una riga sola.

**Isolare una riga, poi dividerla in due.** Prima ritagliamo dall'area disponibile una singola riga alta una cella (`Constraint::Length(1)`, il resto lo scartiamo con `_resto` — ci serve solo la prima parte); poi, dentro quella riga, dividiamo in larghezza fissa per l'etichetta e spazio restante per il campo — `Constraint::Fill(1)`, come già sai:

```rust
let [riga, _resto] = Layout::vertical([Constraint::Length(1), Constraint::Fill(1)])
    .areas(frame.area());

let [area_etichetta, area_campo] = Layout::horizontal([
    Constraint::Length(etichetta.chars().count() as u16),
    Constraint::Fill(1),
])
.areas(riga);
```

Due `Layout` in sequenza, uno annidato nell'altro: il primo lavora sull'intero schermo e produce `riga`, il secondo lavora su `riga` (non più su `frame.area()`) e produce le due aree finali. `Constraint::Length(...)` fissa la larghezza dell'etichetta al numero esatto di caratteri che contiene — contati con `.chars().count()`, non `.len()`, per lo stesso motivo byte-vs-carattere che tra poco tornerà per il testo digitato. `Constraint::Fill(1)` prende tutto quello che resta, sia in altezza (nel primo `Layout`, per `_resto`) sia in larghezza (nel secondo, per `area_campo`).

**Disegnare l'etichetta.** Uno `Span` direttamente nella sua area, senza passare da `Paragraph`: puoi farlo, l'hai già visto.

```rust
frame.render_widget(Span::raw(etichetta), area_etichetta);
```

**Colorare il campo.** Un `Block` senza bordi né titolo, con solo lo stile impostato, colora da solo l'intera area — lo stesso widget del quarto articolo, usato oggi solo per il suo sfondo:

```rust
use ratatui::widgets::Block;

frame.render_widget(Block::new().on_gray(), area_campo);
```

**Disegnare il testo sopra.** Non serve riempire nulla a mano: ogni fotogramma riparte da un buffer vuoto (celle vuote, senza stile), quindi lo sfondo grigio appena disegnato resta visibile ovunque il testo non arrivi. Disegniamo il testo direttamente, senza passare né da `Span` né da `Paragraph` — anche una `&str` sa disegnarsi da sola:

```rust
frame.render_widget(app.testo.as_str(), area_campo);
```

L'ordine conta: prima il `Block`, poi il testo sopra — al contrario, il grigio ridipingerebbe sopra le lettere appena scritte.

**Posizionare il cursore.** Non più sull'intero schermo, ma dentro `area_campo`:

```rust
frame.set_cursor_position((area_campo.x + app.cursore as u16, area_campo.y));
```

Messi in fila, questi quattro pezzi completano il quadro di oggi. C'è però una cosa da sistemare: oggi il loop deve poter finire con un `break` semplice, e dopo la fine del loop vogliamo ancora usare `app` (per stampare il testo scritto). È lo stesso motivo per cui, quando abbiamo imparato a gestire i tasti, siamo passati da `ratatui::run` a `ratatui::init()`/`ratatui::restore()`: la closure generica di `run` si mette in mezzo proprio in questi casi, torniamo a quella forma anche qui.

Ecco il file intero a questo punto — con già dentro un modo per uscire (`Invio`), altrimenti non potresti chiudere il programma per provarlo:

```rust
use ratatui::crossterm::event::{self, Event, KeyCode};
use ratatui::layout::{Constraint, Layout};
use ratatui::style::Stylize;
use ratatui::text::Span;
use ratatui::widgets::Block;

struct App {
    testo: String,
    cursore: usize,
}

fn main() -> std::io::Result<()> {
    let mut app = App {
        testo: String::new(),
        cursore: 0,
    };

    let mut terminal = ratatui::init();

    loop {
        terminal.draw(|frame| {
            let etichetta = "Inserisci il tuo nome: ";

            let [riga, _resto] =
                Layout::vertical([Constraint::Length(1), Constraint::Fill(1)])
                    .areas(frame.area());

            let [area_etichetta, area_campo] = Layout::horizontal([
                Constraint::Length(etichetta.chars().count() as u16),
                Constraint::Fill(1),
            ])
            .areas(riga);

            frame.render_widget(Span::raw(etichetta), area_etichetta);

            frame.render_widget(Block::new().on_gray(), area_campo);
            frame.render_widget(app.testo.as_str(), area_campo);

            frame.set_cursor_position((area_campo.x + app.cursore as u16, area_campo.y));
        })?;

        if let Event::Key(key) = event::read()?
            && key.kind == event::KeyEventKind::Press
        {
            match key.code {
                KeyCode::Enter => break,
                _ => {}
            }
        }
    }

    ratatui::restore();
    Ok(())
}
```

A questo punto, se lanci `cargo run`, vedi l'etichetta a sinistra e subito dopo una fascia grigio chiaro che occupa tutto lo spazio restante della riga — il campo — con il cursore lampeggiante al suo interno. Non puoi ancora scriverci dentro (arriva nel prossimo passaggio): puoi solo premere Invio per uscire.

> **Per approfondire.** Se il testo scritto supera la larghezza di `area_campo`, il comportamento è lo stesso già visto per `Text`: il rendering si ferma ai bordi dell'area, il testo in eccesso viene tagliato, non va a capo.

### Scrivere caratteri

Il ramo che ci interessa è sempre `KeyCode::Char(c)`, come già visto in precedenza — ma stavolta, invece di limitarci a leggere `c`, lo inseriamo dentro `app.testo` nella posizione del cursore. Lo vediamo dentro il `match` per intero, così è chiaro dove agiamo:

```rust
if let Event::Key(key) = event::read()?
    && key.kind == event::KeyEventKind::Press
{
    match key.code {
        KeyCode::Enter => break,
        KeyCode::Char(c) => {
            let byte_idx = app
                .testo
                .char_indices()
                .nth(app.cursore)
                .map_or(app.testo.len(), |(i, _)| i);

            app.testo.insert(byte_idx, c);
            app.cursore += 1;
        }
        _ => {}
    }
}
```

Perché non basta `app.testo.insert(app.cursore, c)`? Perché `String::insert` vuole un indice in **byte**, non in caratteri, e i due non coincidono appena nel testo compare un accento o qualunque carattere che in UTF-8 occupa più di un byte (`è`, `à`, ma anche un'emoji). Se `cursore` conta i caratteri (cosa comoda per tutto il resto: muoverlo, confrontarlo con la lunghezza del testo), va convertito in un indice di byte prima di poterlo passare a `insert`. `char_indices()` ti dà esattamente questa mappatura: per ogni carattere, il suo indice di byte nella stringa. `.nth(app.cursore)` prende la coppia corrispondente al carattere in cui vuoi inserire; `map_or(default, f)` applica `f` al valore se c'è, altrimenti restituisce direttamente `default` — un solo passaggio invece di `.map(...).unwrap_or(...)`. Se il cursore è alla fine del testo, `nth` restituisce `None` e usiamo `app.testo.len()` — inserimento in coda.

A questo punto, se lanci `cargo run` e scrivi qualcosa, il testo compare a schermo e il cursore lampeggiante avanza carattere dopo carattere — prova anche con lettere accentate, per vedere che non si spezza nulla.

### Cancellare con Backspace

Simmetrico all'inserimento: se il cursore non è già a inizio testo, rimuovi il carattere **prima** di lui. La condizione la mettiamo direttamente nel pattern, come *match guard*, invece che in un `if` dentro il corpo:

```rust
KeyCode::Backspace if app.cursore > 0 => {
    let byte_idx = app
        .testo
        .char_indices()
        .nth(app.cursore - 1)
        .map(|(i, _)| i)
        .unwrap();

    app.testo.remove(byte_idx);
    app.cursore -= 1;
}
```

`if app.cursore > 0` dopo il pattern è una condizione in più perché questo ramo scatti: con `cursore` a zero, `KeyCode::Backspace` non entra proprio in questo braccio del `match` e cade nel `_ => {}` più sotto, senza fare nulla — stesso comportamento di prima, scritto in modo più diretto. E dentro il corpo, proprio perché il guard ha già verificato `cursore > 0`, sai per certo che `cursore - 1` è un indice valido: quel `.unwrap()` finale non può mai fallire.

A questo punto, se lanci `cargo run`, Backspace cancella l'ultimo carattere scritto e il cursore arretra di conseguenza. A inizio testo, Backspace non fa nulla.

### Muovere il cursore con le frecce

Qui non c'è testo da toccare, solo `cursore` da spostare — con un vincolo per lato: non può andare sotto zero, non può superare la lunghezza del testo.

```rust
KeyCode::Left => app.cursore = app.cursore.saturating_sub(1),
KeyCode::Right => {
    app.cursore = (app.cursore + 1).min(app.testo.chars().count());
}
```

`saturating_sub(1)` è il modo idiomatico per "sottrai 1, ma fermati a zero invece di andare in overflow" su un `usize` — utile qui perché `cursore` non può mai essere negativo. Sul lato destro, `app.testo.chars().count()` è il numero totale di caratteri nel testo (non byte): il cursore può arrivare fino a lì, cioè "dopo l'ultimo carattere", non oltre.

A questo punto, se lanci `cargo run`, puoi spostare il cursore avanti e indietro nel testo già scritto, senza modificarlo, e Backspace cancella dalla posizione in cui ti trovi — non necessariamente dalla fine.

### Uscire con Invio, e stampare il risultato

Non serve un valore di ritorno speciale per portarsi dietro il testo scritto: `app` è dichiarata fuori dal loop, quindi resta semplicemente lì dopo che il loop finisce. Ripristiniamo il terminale con `ratatui::restore()` e poi stampiamo — nello stesso ordine già visto quando abbiamo introdotto `init`/`restore`:

```rust
loop {
    // ...
}

ratatui::restore();
println!("Hai scritto: {}", app.testo);
Ok(())
```

A questo punto, se lanci `cargo run`, scrivi qualcosa e premi Invio, il terminale torna al prompt normale e vedi stampato quello che avevi scritto — coerente con l'ordine "prima ripristina il terminale, poi stampa" già visto in precedenza.

### Un `impl App` per la logica

Fin qui il `match` nel loop ha fatto due lavori insieme: capire quale tasto è stato premuto, e sapere *come* manipolare `app.testo`/`app.cursore` — comprese le conversioni byte-carattere. Con quattro rami "veri" (oltre a `Enter` e `_`), vale la pena separare le due responsabilità: spostiamo la logica di modifica dentro `App` stessa, con un `impl App`. In precedenza avevamo scelto di non farlo ancora, tenendo `App` come puro contenitore di dati; oggi il `match` è cresciuto abbastanza da giustificare il passo.

```rust
impl App {
    fn inserisci(&mut self, c: char) {
        let byte_idx = self
            .testo
            .char_indices()
            .nth(self.cursore)
            .map_or(self.testo.len(), |(i, _)| i);

        self.testo.insert(byte_idx, c);
        self.cursore += 1;
    }

    fn cancella(&mut self) {
        if self.cursore == 0 {
            return;
        }

        let byte_idx = self
            .testo
            .char_indices()
            .nth(self.cursore - 1)
            .map(|(i, _)| i)
            .unwrap();

        self.testo.remove(byte_idx);
        self.cursore -= 1;
    }

    fn cursore_sinistra(&mut self) {
        self.cursore = self.cursore.saturating_sub(1);
    }

    fn cursore_destra(&mut self) {
        self.cursore = (self.cursore + 1).min(self.testo.chars().count());
    }
}
```

Nota il cambio dentro `cancella`: il match guard `KeyCode::Backspace if app.cursore > 0` funzionava bene finché la logica stava nel `match` stesso — la condizione faceva parte del pattern. Spostata dentro un metodo, quella stessa condizione diventa naturalmente un controllo interno, con un ritorno anticipato se non c'è nulla da cancellare: stesso comportamento, forma diversa perché è cambiato *dove* vive il codice.

Il `match` nel loop si riduce a una riga per ramo:

```rust
match key.code {
    KeyCode::Enter => break,
    KeyCode::Char(c) => app.inserisci(c),
    KeyCode::Backspace => app.cancella(),
    KeyCode::Left => app.cursore_sinistra(),
    KeyCode::Right => app.cursore_destra(),
    _ => {}
}
```

Ora il `match` risponde a una sola domanda — quale tasto, quale azione — e ogni metodo di `App` risponde all'altra: come cambia lo stato. È la stessa disciplina "chi legge l'evento aggiorna lo stato" già vista, solo un gradino più in là: prima lo stato era passivo (il `match` lo scriveva direttamente), ora sa aggiornarsi da solo.

A questo punto, se lanci `cargo run`, il comportamento è identico a prima — è proprio il punto, come già visto quando `App` è comparsa per la prima volta: riorganizzare il codice non cambia cosa fa il programma.

C'è un vantaggio in più in questo spostamento, oltre alla leggibilità: prima, la logica di inserimento/cancellazione viveva dentro il `match` del loop, mescolata con la lettura degli eventi — per testarla avresti dovuto simulare anche quella. Ora che vive in metodi di `App` che lavorano solo su `String` e `usize`, senza toccare `ratatui`, è testabile al 100% senza aprire un terminale:

```rust
#[cfg(test)]
mod tests {
    use super::App;

    #[test]
    fn inserisci_a_metà_testo() {
        let mut app = App {
            testo: String::from("Rst"),
            cursore: 1,
        };

        app.inserisci('u');

        assert_eq!(app.testo, "Rust");
        assert_eq!(app.cursore, 2);
    }
}
```

Non è un test che aggiungiamo al progetto oggi, per lo stesso motivo già detto per `App` come contenitore di dati — ma il fatto che *potresti* scriverlo, e che funzionerebbe senza toccare `ratatui`, è la controprova concreta che la logica è davvero disaccoppiata dalla libreria grafica.

### Cosa manca (limiti di oggi)

Vale la pena essere chiari su cosa questo campo di testo *non* fa:

- **Una sola riga.** Non c'è modo di andare a capo dentro il testo: `Invio` conferma e chiude il programma, non inserisce una riga nuova.
- **Niente annullamento.** `Esc` oggi non fa nulla (cade nel ramo `_ => {}`): non c'è un modo per uscire senza confermare il testo scritto finora.
- **Niente selezione.** Non puoi selezionare una porzione di testo, solo spostare il cursore e cancellare un carattere alla volta.
- **Niente Canc (Delete).** Solo Backspace, che cancella all'indietro; cancellare in avanti richiederebbe un ramo `KeyCode::Delete` speculare a quello di Backspace.
- **Niente scroll orizzontale.** Se il testo supera la larghezza di `area_campo`, oggi non lo vedi scorrere: il rendering si limita all'area disegnata e taglia il resto, esattamente come già visto per il testo che eccede un'area senza `.wrap()`.
- **Caratteri a doppia larghezza non gestiti.** Il calcolo della posizione del cursore assume che ogni carattere occupi esattamente una cella — vero per lettere, cifre e accenti latini, falso per emoji o molti caratteri CJK, che sul terminale occupano due celle. Gestirli correttamente è un problema a parte.

Nessuno di questi è un errore: sono scelte per tenere il codice di oggi concentrato sull'idea di fondo — stato che rappresenta un testo modificabile, eventi che lo aggiornano un pezzo alla volta.

## Codice completo finale

```rust
use ratatui::crossterm::event::{self, Event, KeyCode};
use ratatui::layout::{Constraint, Layout};
use ratatui::style::Stylize;
use ratatui::text::Span;
use ratatui::widgets::Block;

struct App {
    testo: String,
    cursore: usize,
}

impl App {
    fn inserisci(&mut self, c: char) {
        let byte_idx = self
            .testo
            .char_indices()
            .nth(self.cursore)
            .map_or(self.testo.len(), |(i, _)| i);

        self.testo.insert(byte_idx, c);
        self.cursore += 1;
    }

    fn cancella(&mut self) {
        if self.cursore == 0 {
            return;
        }

        let byte_idx = self
            .testo
            .char_indices()
            .nth(self.cursore - 1)
            .map(|(i, _)| i)
            .unwrap();

        self.testo.remove(byte_idx);
        self.cursore -= 1;
    }

    fn cursore_sinistra(&mut self) {
        self.cursore = self.cursore.saturating_sub(1);
    }

    fn cursore_destra(&mut self) {
        self.cursore = (self.cursore + 1).min(self.testo.chars().count());
    }
}

fn main() -> std::io::Result<()> {
    let mut app = App {
        testo: String::new(),
        cursore: 0,
    };

    let mut terminal = ratatui::init();

    loop {
        terminal.draw(|frame| {
            let etichetta = "Inserisci il tuo nome: ";

            let [riga, _resto] =
                Layout::vertical([Constraint::Length(1), Constraint::Fill(1)])
                    .areas(frame.area());

            let [area_etichetta, area_campo] = Layout::horizontal([
                Constraint::Length(etichetta.chars().count() as u16),
                Constraint::Fill(1),
            ])
            .areas(riga);

            frame.render_widget(Span::raw(etichetta), area_etichetta);

            frame.render_widget(Block::new().on_gray(), area_campo);
            frame.render_widget(app.testo.as_str(), area_campo);

            frame.set_cursor_position((area_campo.x + app.cursore as u16, area_campo.y));
        })?;

        if let Event::Key(key) = event::read()?
            && key.kind == event::KeyEventKind::Press
        {
            match key.code {
                KeyCode::Enter => break,
                KeyCode::Char(c) => app.inserisci(c),
                KeyCode::Backspace => app.cancella(),
                KeyCode::Left => app.cursore_sinistra(),
                KeyCode::Right => app.cursore_destra(),
                _ => {}
            }
        }
    }

    ratatui::restore();
    println!("Hai scritto: {}", app.testo);
    Ok(())
}
```

## E se non volessi scriverlo da zero?

Se in un progetto vero ti serve un campo di testo con i limiti visti sopra già risolti, non sei costretto a riscriverlo ogni volta. Esistono crate pensati apposta: [`tui-input`](https://crates.io/crates/tui-input), per esempio, gestisce lo stato di un singolo input testuale e lascia a te il disegno — concettualmente molto vicino a quello che hai appena scritto, ma più completo.

Qui però devo fare una confessione: a un certo punto il problema me lo sono ritrovato davanti anch'io, in un'altra applicazione, e la mia soluzione è diventata una libreria. Si chiama [`ratiform`](https://github.com/marc0x71/ratiform) ed è [su crates.io](https://crates.io/crates/ratiform). Quindi sì, quello che segue è un po' di autopromozione — ma è anche il seguito naturale di questo articolo.

### Da un campo a un form

Ho cominciato proprio come oggi, con una `String` nello stato e i tasti gestiti a mano. Al terzo campo, però, il problema non era più *il campo* ma tutto quello che ci sta intorno: quale campo ha il fuoco, cosa fa `Tab`, come tenere in ordine più pezzi di stato. Le librerie che ho trovato risolvevano il singolo input, oppure restituivano i valori come JSON con chiavi stringa. Da qui l'idea di `ratiform`: un form completo in cui **l'identificatore di ogni campo è un tipo Rust tuo**, tipicamente un `enum`, dalla creazione del campo fino alla lettura del valore.

### Lo stesso esempio, con `ratiform`

Ecco l'esempio di oggi riscritto: stessa etichetta, stesso Invio per confermare, stessa stampa finale.

```rust
use ratatui::crossterm::event::{self, Event};
use ratiform::builder::FormBuilder;
use ratiform::{Form, FormResult};

#[derive(Debug, Hash, Eq, PartialEq)]
enum Campo {
    Nome,
}

fn main() -> std::io::Result<()> {
    let mut stato = FormBuilder::new()
        .single_line(Campo::Nome, "Inserisci il tuo nome")
        .placeholder("es. Marco")
        .required("Il nome è obbligatorio".to_owned())
        .build()
        .map_err(std::io::Error::other)?;

    let mut terminal = ratatui::init();

    loop {
        terminal.draw(|frame| {
            frame.render_stateful_widget(Form::default(), frame.area(), &mut stato);

            if let Some(posizione) = stato.cursor_position() {
                frame.set_cursor_position(posizione);
            }
        })?;

        if let Event::Key(key) = event::read()?
            && key.kind == event::KeyEventKind::Press
        {
            stato.handle_input(key);

            if !matches!(stato.result(), FormResult::Working) {
                break;
            }
        }
    }

    ratatui::restore();

    match stato.result() {
        FormResult::Submitted => {
            let nome = stato.value(&Campo::Nome).unwrap_or_default();
            println!("Hai scritto: {nome}");
        }
        _ => println!("Annullato."),
    }
    Ok(())
}
```

```toml
[dependencies]
ratatui = "0.30"
ratiform = "0.7"
```

**Lo stato non è più `App`, ma `FormState`.** Al posto della nostra struct con `testo` e `cursore` c'è `stato`, prodotto da `FormBuilder`: ogni campo si dichiara con il suo identificatore (`Campo::Nome`) e la sua etichetta, poi lo si configura con metodi concatenati. È ancora "stato raccolto fuori dal loop" — la forma che conosci — solo che stavolta il contenuto lo gestisce la libreria. `build()` restituisce un `Result`, perché alcuni errori di configurazione (per esempio due campi con lo stesso identificatore) emergono solo lì; lo convertiamo in `io::Error` e lo propaghiamo con `?` *prima* di `ratatui::init()`, così un errore non lascia il terminale nello stato alterato.

**Niente più `Layout`, `Span` e `Block` a mano.** `Form` è un `StatefulWidget`: si disegna con `render_stateful_widget`, passandogli lo stato, e pensa da solo a etichetta, campo e messaggi di errore. Il cursore funziona come nel nostro esempio — `frame.set_cursor_position(...)` — ma la posizione la calcola il form: `cursor_position()` restituisce un `Option` perché, in generale, non tutti i campi hanno un cursore di testo (una casella di spunta, per esempio, non ce l'ha).

**Il `match` su `key.code` sparisce.** Tutti i rami `Char`/`Backspace`/`Left`/`Right` diventano una chiamata sola, `stato.handle_input(key)`. Il loop non decide più *quale tasto fa cosa*: chiede al form *come è andata*, tramite `result()` — `Working` finché si sta ancora scrivendo, `Submitted` dopo Invio, `Cancelled` dopo Esc.

**Il valore si rilegge per identificatore.** `stato.value(&Campo::Nome)`, non `stato.value("nome")`: se sbagli a scrivere `Campo::Nome`, te lo dice il compilatore, non un errore a runtime.

### Cosa è cambiato rispetto ai limiti di oggi

Riprendendo l'elenco del passo 9:

- **Annullamento.** `Esc` ora annulla il form, e lo distingui dalla conferma guardando `result()`: è il motivo del `match` finale.
- **Canc, Inizio, Fine.** Oltre a Backspace e alle frecce, il campo gestisce `Delete`, `Home` ed `End`.
- **Più righe.** C'è un tipo di campo apposta, `text_area(...)`, in cui Invio va a capo invece di confermare (per confermare si usa `Ctrl+Invio`).
- **Validazione.** I campi sono obbligatori per impostazione predefinita: prova a premere Invio con il campo vuoto e vedrai comparire "Il nome è obbligatorio" sotto il campo, mentre il form resta aperto. Per controlli più specifici ci sono `validator(...)` e una serie di validatori già pronti in `ratiform::validators`.
- **Segnaposto.** `placeholder(...)` mostra un suggerimento finché il campo è vuoto — una cosa che nella versione fatta in casa non avevamo nemmeno messo in conto.

### Dove serve davvero: più di un campo

Con un solo campo la differenza è soprattutto di comodità. Il motivo per cui `ratiform` esiste, però, si vede quando i campi diventano due o più. Aggiungere un'email è una riga in più nell'`enum` e due nel builder:

```rust
#[derive(Debug, Hash, Eq, PartialEq)]
enum Campo {
    Nome,
    Email,
}

let mut stato = FormBuilder::new()
    .single_line(Campo::Nome, "Nome")
    .single_line(Campo::Email, "Email")
    .optional()
    .build()
    .map_err(std::io::Error::other)?;
```

Il loop resta identico. `Tab` e `Shift+Tab` spostano il fuoco da un campo all'altro, le etichette si allineano in una colonna comune, e alla fine puoi raccogliere tutto con `values()` — che consuma lo stato e restituisce coppie `(Campo, String)`, pronte da mettere in una `HashMap<Campo, String>`. Nel [repository](https://github.com/marc0x71/ratiform/tree/main/examples) trovi esempi completi con caselle di spunta, liste di selezione, campi mascherati per le password e temi personalizzati.

Una cosa, in ogni caso, vale per tutte queste librerie: scriverne uno tuo prima di usarle non è tempo perso. Quando chiami `handle_input(key)` ora sai cosa succede lì dentro — un `match` sul tasto, un indice in caratteri da convertire in byte, un cursore da tenere tra zero e la lunghezza del testo. Esattamente quello che hai scritto oggi.
