+++
title = "Ratatui passo dopo passo - `List` e `Table`: due widget selezionabili"
date = 2026-10-14
description = "Impariamo a usare List e Table in Ratatui con Rust: selezione, gestione del focus, navigazione da tastiera e organizzazione dello stato, passo dopo passo."
[taxonomies]
tags = ["rust", "ratatui"]
[extra]
comments = true
+++

Il protagonista di oggi sono `List` e `Table` che gestiscono da sole la propria selezione tramite `ListState`/`TableState`, e vederle isolate aiuta a concentrarsi sul meccanismo, useremo comunque `Layout` e `Block`, già visti negli articoli precedenti.

## Cosa costruiamo oggi

Due widget a schermo, uno sopra l'altro: in alto una `List` di cose da fare, in basso una `Table` con lo storico di cose già fatte. Solo uno dei due riceve le frecce alla volta — lo decide il **focus**, che passi da un widget all'altro con `Tab`. Il pannello attivo si riconoscerà dal bordo giallo.

```
┌ Da fare ─────────────────────────────────┐
│ > Scrivere l'articolo su List e Table    │
│   Rispondere alle email                  │
│   Fare la spesa                          │
│   Chiamare il dentista                   │
│   Rivedere il PR di Marco                │
└──────────────────────────────────────────┘
┌ Completate di recente ───────────────────┐
│ Data     Attività                  Durata│
│ 12/08    Setup del progetto        45 min│
│ 13/08    Bozza articolo #7       2h 10min│
│ 14/08    Fix bug su Layout         30 min│
└──────────────────────────────────────────┘
```

Partiamo come sempre dal codice completo del primo articolo che vi riporto per comodità:

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

### Dividere lo schermo in due: `Layout`

Prima ancora di occuparci di `List` e `Table`, prepariamo lo spazio per entrambe. Come già sai da un articolo precedente, `Layout` divide un'area in più aree secondo dei vincoli:

```rust
use ratatui::layout::{Constraint, Layout};

let [area_lista, area_tabella] =
    Layout::vertical([Constraint::Percentage(50), Constraint::Percentage(50)])
        .areas(frame.area());
```

`.areas(...)` restituisce direttamente un array di `Rect`, uno per ogni vincolo passato — comodo quando il numero di aree è fisso e noto in anticipo, come qui: metà schermo sopra per la lista, metà sotto per la tabella.

### `List` con selezione: `ListState`

`List::new(...)` da sola, come hai già visto per `Paragraph` e `Text`, è un widget "semplice": si disegna con `render_widget` e basta, senza ricordarsi niente tra un fotogramma e l'altro. Per farla diventare selezionabile — cioè per farle ricordare *quale* elemento è evidenziato — serve accoppiarla a uno stato esterno, `ListState`, e renderizzarla con `render_stateful_widget` invece che con `render_widget`:

```rust
use ratatui::widgets::{List, ListState};

let mut list_state = ListState::default().with_selected(Some(0));
```

`with_selected(Some(0))` seleziona il primo elemento fin da subito, così la lista non parte "vuota".

Per muovere la selezione, `ListState` offre dei metodi già pronti — non devi calcolarti tu l'indice successivo/precedente come avresti dovuto fare con una variabile sciolta:

```rust
list_state.select_next();     // sposta la selezione di un elemento in giù
list_state.select_previous(); // sposta la selezione di un elemento in su
```

Entrambi si fermano da soli ai bordi della lista (non serve gestire tu il caso "sono già sull'ultimo elemento").

C'è un altro aspetto che `ListState` gestisce da sola, nello stesso spirito: lo scroll. Se gli elementi sono più di quanti ne entrino nell'area disponibile, non devi calcolarti tu quale porzione della lista mostrare — è `List`, ad ogni render, ad aggiornare l'offset dentro `ListState` in modo che l'elemento selezionato resti sempre visibile. Prova tu stesso, quando arrivi al punto di controllo qui sotto: rimpicciolisci il terminale finché la lista non ci sta più tutta, e guarda cosa succede muovendo la selezione oltre il bordo visibile.

Per far vedere quale elemento è selezionato, usi `.highlight_style(...)` e opzionalmente `.highlight_symbol(...)` sul widget:

```rust
let items = [
    "Scrivere l'articolo su List e Table",
    "Rispondere alle email",
    "Fare la spesa",
    "Chiamare il dentista",
    "Rivedere il PR di Marco",
];

let lista = List::new(items)
    .highlight_style(Style::new().reversed())
    .highlight_symbol("> ");
```

E infine la disegni nell'area che le abbiamo riservato, passando anche lo stato:

```rust
frame.render_stateful_widget(lista, area_lista, &mut list_state);
```

Nota che `list_state` viene passata come riferimento mutabile: è `List` stessa, durante il rendering, a poter correggere lo stato se necessario (ad esempio l'offset di scroll, se la lista non entra tutta a schermo) — un dettaglio che, come per `line_count` di `Paragraph`, non serve gestire manualmente.

Prima di aggiungere la `Table`, vale la pena fermarsi: quello che hai in mano in questo momento è già un programma completo e compilabile. `area_tabella` esiste (l'abbiamo ricavata dal `Layout`), ma per ora non ci disegniamo dentro niente — resta vuota, e va bene così.

```rust
use ratatui::crossterm::event::{self, Event, KeyCode};
use ratatui::layout::{Constraint, Layout};
use ratatui::style::Style;
use ratatui::widgets::{List, ListState};

fn main() -> std::io::Result<()> {
    let mut list_state = ListState::default().with_selected(Some(0));

    ratatui::run(|terminal| {
        loop {
            terminal.draw(|frame| {
                let [area_lista, _area_tabella] =
                    Layout::vertical([Constraint::Percentage(50), Constraint::Percentage(50)])
                        .areas(frame.area());

                let items = [
                    "Scrivere l'articolo su List e Table",
                    "Rispondere alle email",
                    "Fare la spesa",
                    "Chiamare il dentista",
                    "Rivedere il PR di Marco",
                ];

                let lista = List::new(items)
                    .highlight_style(Style::new().reversed())
                    .highlight_symbol("> ");

                frame.render_stateful_widget(lista, area_lista, &mut list_state);
            })?;

            if let Event::Key(key) = event::read()?
                && key.kind == event::KeyEventKind::Press
            {
                match key.code {
                    KeyCode::Esc | KeyCode::Char('q') => break Ok(()),
                    KeyCode::Up => list_state.select_previous(),
                    KeyCode::Down => list_state.select_next(),
                    _ => {}
                }
            }
        }
    })
}
```

A questo punto, se lanci `cargo run`, vedi la lista occupare la metà superiore dello schermo, con il primo elemento già evidenziato. `Up` e `Down` muovono la selezione, `q` o `Esc` escono. La metà inferiore è semplicemente nera, spazio non ancora utilizzato. Da qui in avanti aggiungiamo la `Table` in quello spazio, e poi il focus per scegliere quale dei due widget muovere.

### `Table`, stesso pattern: `TableState`

`Table` segue esattamente lo stesso schema di `List`: un widget "semplice" di per sé, che diventa selezionabile accoppiandolo a `TableState` e renderizzandolo con `render_stateful_widget`.

La costruzione è un po' più articolata perché `Table` ha colonne, non solo righe:

```rust
use ratatui::widgets::{Row, Table, TableState};

let righe = [
    Row::new(["12/08", "Setup del progetto", "45 min"]),
    Row::new(["13/08", "Bozza articolo #7", "2h 10min"]),
    Row::new(["14/08", "Fix bug su Layout", "30 min"]),
];

let larghezze = [
    Constraint::Length(8),
    Constraint::Fill(1),
    Constraint::Length(10),
];

let tabella = Table::new(righe, larghezze)
    .header(Row::new(["Data", "Attività", "Durata"]).style(Style::new().bold()))
    .row_highlight_style(Style::new().reversed());
```

`larghezze` usa lo stesso `Constraint` che hai già incontrato con `Layout` — non è una API nuova per `Table`, è lo stesso strumento riusato per un compito diverso: lì spartiva lo schermo tra widget, qui spartisce la larghezza orizzontale tra le colonne dello stesso widget.

Lo stato si dichiara e si usa identico a `ListState`:

```rust
let mut table_state = TableState::default().with_selected(Some(0));

// ...

table_state.select_next();
table_state.select_previous();

// ...

frame.render_stateful_widget(tabella, area_tabella, &mut table_state);
```

A questo punto hai visto il pattern due volte: `List`+`ListState` e `Table`+`TableState`, sono la stessa idea (widget semplice + stato esterno + `render_stateful_widget`) applicata a due widget diversi. Se però provi a collegare anche `table_state.select_next()`/`select_previous()` agli stessi rami `KeyCode::Up`/`KeyCode::Down` di prima, ti accorgi subito del problema: le frecce muoverebbero *entrambe* le selezioni insieme, e non è quello che vogliamo. Serve un modo per dire "adesso le frecce sono per la lista" oppure "adesso sono per la tabella".

### Il focus: un pezzo di stato in più, e il match che lo usa

La soluzione più comune che vedrete spesso è rappresentare esplicitamente "chi riceve le frecce in questo momento" con un enum a due varianti:

```rust
#[derive(PartialEq)]
enum Focus {
    List,
    Table,
}
```

E una variabile che lo tiene, fuori dal loop come tutto lo stato che deve sopravvivere tra un giro e l'altro:

```rust
let mut focus = Focus::List;
```

Con `focus` a disposizione, il `match` sull'evento tastiera si allarga di un ramo (`Tab`) e i rami delle frecce diventano condizionali su `focus`:

```rust
if let Event::Key(key) = event::read()?
    && key.kind == event::KeyEventKind::Press
{
    match key.code {
        KeyCode::Esc | KeyCode::Char('q') => break Ok(()),
        KeyCode::Tab => {
            focus = match focus {
                Focus::List => Focus::Table,
                Focus::Table => Focus::List,
            };
        }
        KeyCode::Up => match focus {
            Focus::List => list_state.select_previous(),
            Focus::Table => table_state.select_previous(),
        },
        KeyCode::Down => match focus {
            Focus::List => list_state.select_next(),
            Focus::Table => table_state.select_next(),
        },
        _ => {}
    }
}
```

Nota la forma: `Up` e `Down` non decidono più da soli cosa fare, delegano la decisione a un secondo `match` su `focus`. È lo stesso `key.code` di sempre, ma la sua conseguenza dipende ora anche da un secondo pezzo di stato — è la prima volta nella serie che due eventi diversi (il tasto premuto, il focus corrente) concorrono a decidere cosa succede.

Anche qui vale la pena fermarsi: il programma è già completo e fa quello che deve, solo che non lo vedi ancora.

```rust
use ratatui::crossterm::event::{self, Event, KeyCode};
use ratatui::layout::{Constraint, Layout};
use ratatui::style::Style;
use ratatui::widgets::{List, ListState, Row, Table, TableState};

#[derive(PartialEq)]
enum Focus {
    List,
    Table,
}

fn main() -> std::io::Result<()> {
    let mut focus = Focus::List;
    let mut list_state = ListState::default().with_selected(Some(0));
    let mut table_state = TableState::default().with_selected(Some(0));

    ratatui::run(|terminal| {
        loop {
            terminal.draw(|frame| {
                let [area_lista, area_tabella] =
                    Layout::vertical([Constraint::Percentage(50), Constraint::Percentage(50)])
                        .areas(frame.area());

                let items = [
                    "Scrivere l'articolo su List e Table",
                    "Rispondere alle email",
                    "Fare la spesa",
                    "Chiamare il dentista",
                    "Rivedere il PR di Marco",
                ];

                let lista = List::new(items)
                    .highlight_style(Style::new().reversed())
                    .highlight_symbol("> ");

                frame.render_stateful_widget(lista, area_lista, &mut list_state);

                let righe = [
                    Row::new(["12/08", "Setup del progetto", "45 min"]),
                    Row::new(["13/08", "Bozza articolo #7", "2h 10min"]),
                    Row::new(["14/08", "Fix bug su Layout", "30 min"]),
                ];

                let larghezze = [
                    Constraint::Length(8),
                    Constraint::Fill(1),
                    Constraint::Length(10),
                ];

                let tabella = Table::new(righe, larghezze)
                    .header(Row::new(["Data", "Attività", "Durata"]).style(Style::new().bold()))
                    .row_highlight_style(Style::new().reversed());

                frame.render_stateful_widget(tabella, area_tabella, &mut table_state);
            })?;

            if let Event::Key(key) = event::read()?
                && key.kind == event::KeyEventKind::Press
            {
                match key.code {
                    KeyCode::Esc | KeyCode::Char('q') => break Ok(()),
                    KeyCode::Tab => {
                        focus = match focus {
                            Focus::List => Focus::Table,
                            Focus::Table => Focus::List,
                        };
                    }
                    KeyCode::Up => match focus {
                        Focus::List => list_state.select_previous(),
                        Focus::Table => table_state.select_previous(),
                    },
                    KeyCode::Down => match focus {
                        Focus::List => list_state.select_next(),
                        Focus::Table => table_state.select_next(),
                    },
                    _ => {}
                }
            }
        }
    })
}
```

A questo punto, se lanci `cargo run`, il comportamento è già tutto lì: `Tab` sposta il focus, le frecce muovono la selezione del pannello giusto. Prova a farlo senza guardare il codice: ti accorgi che non hai alcun modo di sapere, guardando lo schermo, se in questo momento le frecce controllano la lista o la tabella. La logica funziona, manca solo il riscontro visivo — è esattamente il problema che risolviamo nell'ultimo passo.

### Rendere visibile il focus: bordo diverso con `Block`

Un focus che non si vede è inutile quanto una selezione senza `highlight_style`. Avvolgiamo ciascun widget in un `Block` il cui colore del bordo dipende da `focus`:

```rust
use ratatui::widgets::Block;

let blocco_lista = Block::bordered()
    .title(" Da fare ")
    .border_style(if focus == Focus::List {
        Style::new().yellow()
    } else {
        Style::new()
    });

let blocco_tabella = Block::bordered()
    .title(" Completate di recente ")
    .border_style(if focus == Focus::Table {
        Style::new().yellow()
    } else {
        Style::new()
    });
```

E li agganci ai widget con `.block(...)`, prima di renderizzarli:

```rust
let lista = List::new(items)
    .block(blocco_lista)
    .highlight_style(Style::new().reversed())
    .highlight_symbol("> ");

let tabella = Table::new(righe, larghezze)
    .header(Row::new(["Data", "Attività", "Durata"]).style(Style::new().bold()))
    .block(blocco_tabella)
    .row_highlight_style(Style::new().reversed());
```

A questo punto, se lanci `cargo run`, vedi la `List` in alto con il bordo giallo (ha il focus di default) e la `Table` in basso con bordo normale. Le frecce muovono la selezione della lista, evidenziata con `> ` e sfondo invertito. Premi `Tab`: il bordo giallo passa alla tabella, e ora sono le frecce a muovere la riga selezionata lì. `q` o `Esc` chiudono il programma da qualunque pannello abbia il focus. Il programma fa già tutto quello che avevamo in mente per oggi — quello che segue è una revisione della *forma* del codice, non del suo comportamento.

### Un po' di pulizia: da closure a funzioni

Vale la pena fermarsi su una cosa che è successa senza che lo decidessimo esplicitamente: la closure passata a `terminal.draw(...)` e il blocco `if let Event::Key(...)` sono cresciuti parecchio, e sono entrambi incastonati dentro `ratatui::run(|terminal| { ... })`. Va bene per una demo di poche righe, meno bene quando, come oggi, dentro ci sono `Layout`, due widget con i rispettivi `Block`, un `match` a più livelli e uno stato che coinvolge tre variabili diverse (`focus`, `list_state`, `table_state`).

Il rimedio è lo stesso che avevi già visto nell'articolo sullo stato — raccogliere le variabili sciolte in una struct — spinto un passo più in là: estrarre anche il *comportamento* in funzioni con un nome e una firma proprie, fuori dalle closure.

Prima lo stato. Le tre variabili sciolte diventano una struct:

```rust
struct AppState {
    focus: Focus,
    list_state: ListState,
    table_state: TableState,
}
```

E, invece di modificarne i campi da fuori come facevamo con `App` nell'articolo sullo stato, le diamo dei metodi che esprimono le azioni possibili, non i singoli campi da toccare:

```rust
impl AppState {
    fn toggle_focus(&mut self) {
        self.focus = match self.focus {
            Focus::List => Focus::Table,
            Focus::Table => Focus::List,
        };
    }

    fn select_previous(&mut self) {
        match self.focus {
            Focus::List => self.list_state.select_previous(),
            Focus::Table => self.table_state.select_previous(),
        }
    }

    fn select_next(&mut self) {
        match self.focus {
            Focus::List => self.list_state.select_next(),
            Focus::Table => self.table_state.select_next(),
        }
    }

    fn is_list_focused(&self) -> bool {
        self.focus == Focus::List
    }

    fn is_table_focused(&self) -> bool {
        self.focus == Focus::Table
    }
}
```

La logica del doppio `match` — "quale stato muovere in base al focus" — non è sparita: si è solo spostata da dentro il loop di eventi a dentro `AppState`. Il vantaggio è che chi chiama `state.select_next()` non ha più bisogno di sapere che esiste un `focus`, né come funziona: gli basta sapere che esiste un'azione "seleziona il prossimo elemento", qualunque cosa significhi in questo momento.

`is_list_focused()` e `is_table_focused()` fanno lo stesso lavoro per la lettura invece che per la scrittura: sostituiscono i due confronti `focus == Focus::List` / `focus == Focus::Table` che hai appena scritto nel passo precedente per i bordi dei `Block`. Il confronto con l'enum non sparisce — è ancora lì, dentro il metodo — ma chi disegna il bordo non ha più bisogno di sapere cos'è `Focus` o come si confronta: gli basta una domanda con un nome chiaro.

Aggiungiamo anche un `Default`, per non dover ripetere i valori iniziali a mano ogni volta che serve un `AppState`:

```rust
impl Default for AppState {
    fn default() -> Self {
        Self {
            focus: Focus::List,
            list_state: ListState::default().with_selected(Some(0)),
            table_state: TableState::default().with_selected(Some(0)),
        }
    }
}
```

Ora il comportamento. Estraiamo il corpo del `match` sulla tastiera in una funzione a sé, che prende lo stato e l'evento, e restituisce `true` quando è il momento di uscire:

```rust
use ratatui::crossterm::event::Event;

fn handle_event(state: &mut AppState, event: Event) -> bool {
    if let Event::Key(key) = event
        && key.kind == event::KeyEventKind::Press
    {
        match key.code {
            KeyCode::Esc | KeyCode::Char('q') => return true,
            KeyCode::Tab => state.toggle_focus(),
            KeyCode::Up => state.select_previous(),
            KeyCode::Down => state.select_next(),
            _ => {}
        }
    }

    false
}
```

Nota due cose. Primo: `handle_event` non sa nulla di `terminal` o di `ratatui::run` — prende un `AppState` e un `Event` "nudi", quindi in teoria potresti chiamarla da un test, costruendoti a mano un `KeyEvent` con `KeyCode::Down`, senza aprire nessun terminale. Non lo facciamo oggi (vale lo stesso principio già visto nell'articolo sullo stato), ma è la prova che separare la logica dalla closure di `ratatui::run` non è solo estetica.

Secondo: il valore di ritorno `bool` sostituisce il `break Ok(())` di prima. Non possiamo più scrivere `break` dentro `handle_event`, perché non siamo più dentro il `loop` — quindi la funzione si limita a *dire* "bisogna uscire" restituendo `true`, ed è il chiamante, nel `loop`, a decidere cosa farne.

Stessa idea per il disegno: estraiamo il corpo di `terminal.draw(...)` in una funzione `render`. Il contenuto di ogni widget non cambia da come l'hai già visto, a parte i due confronti sul bordo che ora passano dai metodi appena aggiunti — qui mostro solo la forma della funzione, il dettaglio completo lo ritrovi nel codice finale:

```rust
use ratatui::Frame;

fn render(frame: &mut Frame, state: &mut AppState) {
    let [area_lista, area_tabella] =
        Layout::vertical([Constraint::Percentage(50), Constraint::Percentage(50)])
            .areas(frame.area());

    // ... costruzione di `lista` e del suo `Block`, come già visto —
    // solo che ora il bordo usa `state.is_list_focused()` al posto di `focus == Focus::List`

    frame.render_stateful_widget(lista, area_lista, &mut state.list_state);

    // ... costruzione di `tabella` e del suo `Block`, stesso discorso con `state.is_table_focused()`

    frame.render_stateful_widget(tabella, area_tabella, &mut state.table_state);
}
```

E il `main`, ridotto a orchestrare le due funzioni:

```rust
fn main() -> std::io::Result<()> {
    let mut state = AppState::default();

    ratatui::run(|terminal| {
        loop {
            terminal.draw(|frame| render(frame, &mut state))?;

            if handle_event(&mut state, event::read()?) {
                break Ok(());
            }
        }
    })
}
```

Confronta questo `main` con quello che avevamo all'inizio dell'articolo: il loop dice ancora "disegna, leggi un evento, decidi", ma ora ogni verbo di quella frase corrisponde a una chiamata di funzione, non a un blocco di codice incastonato in una closure. A questo punto, se lanci `cargo run`, il comportamento è identico a prima del refactoring — è lo stesso principio già visto nell'articolo sullo stato: riorganizzare lo stato e il comportamento non cambia cosa fa il programma, cambia solo quanto è facile leggerlo e farlo crescere.

## Codice completo finale

```rust
use ratatui::Frame;
use ratatui::crossterm::event::{self, Event, KeyCode};
use ratatui::layout::{Constraint, Layout};
use ratatui::style::Style;
use ratatui::widgets::{Block, List, ListState, Row, Table, TableState};

#[derive(PartialEq)]
enum Focus {
    List,
    Table,
}

struct AppState {
    focus: Focus,
    list_state: ListState,
    table_state: TableState,
}

impl AppState {
    fn toggle_focus(&mut self) {
        self.focus = match self.focus {
            Focus::List => Focus::Table,
            Focus::Table => Focus::List,
        };
    }

    fn select_previous(&mut self) {
        match self.focus {
            Focus::List => self.list_state.select_previous(),
            Focus::Table => self.table_state.select_previous(),
        }
    }

    fn select_next(&mut self) {
        match self.focus {
            Focus::List => self.list_state.select_next(),
            Focus::Table => self.table_state.select_next(),
        }
    }

    fn is_list_focused(&self) -> bool {
        self.focus == Focus::List
    }

    fn is_table_focused(&self) -> bool {
        self.focus == Focus::Table
    }
}

impl Default for AppState {
    fn default() -> Self {
        Self {
            focus: Focus::List,
            list_state: ListState::default().with_selected(Some(0)),
            table_state: TableState::default().with_selected(Some(0)),
        }
    }
}

fn main() -> std::io::Result<()> {
    let mut state = AppState::default();

    ratatui::run(|terminal| {
        loop {
            terminal.draw(|frame| render(frame, &mut state))?;

            if handle_event(&mut state, event::read()?) {
                break Ok(());
            }
        }
    })
}

fn handle_event(state: &mut AppState, event: Event) -> bool {
    if let Event::Key(key) = event
        && key.kind == event::KeyEventKind::Press
    {
        match key.code {
            KeyCode::Esc | KeyCode::Char('q') => return true,
            KeyCode::Tab => state.toggle_focus(),
            KeyCode::Up => state.select_previous(),
            KeyCode::Down => state.select_next(),
            _ => {}
        }
    }

    false
}

fn render(frame: &mut Frame, state: &mut AppState) {
    let [area_lista, area_tabella] =
        Layout::vertical([Constraint::Percentage(50), Constraint::Percentage(50)])
            .areas(frame.area());

    let items = [
        "Scrivere l'articolo su List e Table",
        "Rispondere alle email",
        "Fare la spesa",
        "Chiamare il dentista",
        "Rivedere il PR di Marco",
    ];

    let blocco_lista = Block::bordered().title(" Da fare ").border_style(
        if state.is_list_focused() {
            Style::new().yellow()
        } else {
            Style::new()
        },
    );

    let lista = List::new(items)
        .block(blocco_lista)
        .highlight_style(Style::new().reversed())
        .highlight_symbol("> ");

    frame.render_stateful_widget(lista, area_lista, &mut state.list_state);

    let righe = [
        Row::new(["12/08", "Setup del progetto", "45 min"]),
        Row::new(["13/08", "Bozza articolo #7", "2h 10min"]),
        Row::new(["14/08", "Fix bug su Layout", "30 min"]),
    ];

    let larghezze = [
        Constraint::Length(8),
        Constraint::Fill(1),
        Constraint::Length(10),
    ];

    let blocco_tabella = Block::bordered().title(" Completate di recente ").border_style(
        if state.is_table_focused() {
            Style::new().yellow()
        } else {
            Style::new()
        },
    );

    let tabella = Table::new(righe, larghezze)
        .header(Row::new(["Data", "Attività", "Durata"]).style(Style::new().bold()))
        .block(blocco_tabella)
        .row_highlight_style(Style::new().reversed());

    frame.render_stateful_widget(tabella, area_tabella, &mut state.table_state);
}
```

## Cosa abbiamo aggiunto rispetto a prima

Hai visto `List` e `Table` come `StatefulWidget`: entrambe, di per sé, sono widget "semplici" come `Paragraph`; diventano selezionabili solo quando le accoppi a uno stato esterno (`ListState`/`TableState`) e le disegni con `render_stateful_widget` invece di `render_widget`. Hai usato `select_next()`/`select_previous()` per muovere la selezione senza calcolarti tu gli indici. Hai introdotto il **focus**: quando più widget selezionabili condividono lo schermo, serve un pezzo di stato in più che decida quale di loro riceve gli eventi — qui un semplice enum a due varianti, cambiato con `Tab` e reso visibile con un bordo `Block` diverso a seconda di chi è attivo. E hai fatto un passo oltre quanto visto nell'articolo sullo stato: non solo raccogliere lo stato in una struct, ma dargli metodi che esprimono azioni (`toggle_focus`, `select_next`, `select_previous`) e domande (`is_list_focused`, `is_table_focused`), ed estrarre `render`/`handle_event` come funzioni a sé — più facili da leggere una alla volta, e in prospettiva testabili senza aprire un terminale.
