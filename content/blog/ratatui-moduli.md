+++
title = "Ratatui passo dopo passo - Organizzare il progetto su più file, con un enum `Message`"
date = 2026-10-19
description = "Organizziamo un progetto Ratatui in moduli separati e introduciamo l'enum `Message` per gestire gli eventi, separare le responsabilità e rendere il codice più facile da testare."
[taxonomies]
tags = ["rust", "ratatui"]
[extra]
comments = true
+++

## Cosa costruiamo oggi

Stessa app della scorsa volta — la `List` e la `Table` con focus dell'articolo precedente — ma il codice cambia forma, non comportamento: da un unico `main.rs` a quattro moduli con un ruolo ciascuno, `app.rs` per lo stato, `update.rs` per la logica che lo modifica, `ui.rs` per il disegno, `event.rs` per la lettura degli eventi. Nel farlo generalizziamo anche `handle_event(state, event: Event)` in `update(state, message: Message)`, dove `Message` elenca i significati che contano per l'app — non più un tasto travestito, ma è pensato apposta per accogliere in futuro eventi che non arrivano dalla tastiera.

A schermo non cambia niente rispetto all'articolo scorso:

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

Ovviamente partiamo dal codice completo dell'articolo su `List` e `Table`: 

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

## Codice, passo per passo

### `app.rs` — solo lo stato, niente logica

Spostiamo `Focus` e `AppState`, senza portarci dietro `handle_event`: questo file oggi contiene solo *cosa l'app ricorda*, non *come reagisce*.

```rust
// src/app.rs
use ratatui::widgets::{ListState, TableState};

#[derive(PartialEq)]
pub enum Focus {
    List,
    Table,
}

pub struct AppState {
    focus: Focus,
    list_state: ListState,
    table_state: TableState,
}

impl AppState {
    pub fn toggle_focus(&mut self) {
        self.focus = match self.focus {
            Focus::List => Focus::Table,
            Focus::Table => Focus::List,
        };
    }

    pub fn select_previous(&mut self) {
        match self.focus {
            Focus::List => self.list_state.select_previous(),
            Focus::Table => self.table_state.select_previous(),
        }
    }

    pub fn select_next(&mut self) {
        match self.focus {
            Focus::List => self.list_state.select_next(),
            Focus::Table => self.table_state.select_next(),
        }
    }

    pub fn is_list_focused(&self) -> bool {
        self.focus == Focus::List
    }

    pub fn is_table_focused(&self) -> bool {
        self.focus == Focus::Table
    }

    pub fn list_state_mut(&mut self) -> &mut ListState {
        &mut self.list_state
    }

    pub fn table_state_mut(&mut self) -> &mut TableState {
        &mut self.table_state
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

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn toggle_focus_alterna_tra_lista_e_tabella() {
        let mut state = AppState::default();
        state.toggle_focus();
        assert!(state.is_table_focused());
        state.toggle_focus();
        assert!(state.is_list_focused());
    }

    #[test]
    fn select_next_muove_la_selezione_del_pannello_con_focus() {
        let mut state = AppState::default();

        state.select_next();
        assert_eq!(state.list_state_mut().selected(), Some(1));
        assert_eq!(state.table_state_mut().selected(), Some(0));

        state.toggle_focus();
        state.select_next();
        assert_eq!(state.table_state_mut().selected(), Some(1));
    }
}
```

`list_state` e `table_state` restano privati, come `focus`: nessun altro modulo li tocca direttamente. Per disegnarli in `ui.rs` serve però un modo di ottenerne un riferimento mutabile dall'esterno — da qui `list_state_mut()`/`table_state_mut()`, due semplici getter mutabili accanto alle azioni di alto livello e alle domande già presenti.

Nota anche cosa compare per la prima volta in questo file: un `#[cfg(test)] mod tests` in fondo, con due test che chiamano solo metodi di `AppState` — niente terminale, niente `ratatui::run`. E' possibile proprio perché `app.rs`, appena separato, non dipende più da nient'altro che da `ratatui::widgets`. Finché stato, logica di aggiornamento e disegno vivevano tutti nello stesso `main.rs`, agganciati a `ratatui::run` e quindi a un terminale vero, l'unico modo per verificare che il programma facesse la cosa giusta era lanciarlo e guardare lo schermo. Il secondo test è quello più interessante dei due: verifica proprio il comportamento introdotto nell'articolo sul focus — `select_next()` sposta la selezione del pannello giusto, e lascia l'altro invariato.

> **Per approfondire.** Il nome `app.rs`/`AppState` non è l'unica convenzione in giro: chi preferisce seguire più da vicino la terminologia dell'architettura Elm-style verso cui questi ultimi articoli stanno convergendo (model → update → view) chiama spesso questo modulo `model.rs` e la struct `Model`. Qui restiamo su `app.rs`/`AppState` per continuità con gli articoli precedenti della serie, ma è una scelta di nome, non di struttura: se preferisci `model.rs`, il contenuto di questo file non cambia di una riga.

### `update.rs` — l'enum `Message` e chi decide cosa cambia

Qui va tutto quello che nell'articolo scorso era dentro `handle_event`: la traduzione da "cosa è successo" a "cosa cambia nello stato". Ma questa volta non ci limitiamo a spostare il `match` su `KeyCode` avvolgendolo in una variante — diamo a `Message` un significato per ognuna delle quattro cose che possono succedere:

```rust
// src/update.rs
use crate::app::AppState;

pub enum Message {
    Quit,
    ToggleFocus,
    SelectPrevious,
    SelectNext,
}

pub fn update(state: &mut AppState, message: Message) {
    match message {
        Message::ToggleFocus => state.toggle_focus(),
        Message::SelectPrevious => state.select_previous(),
        Message::SelectNext => state.select_next(),
        _ => {}
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn message_toggle_focus_cambia_il_focus() {
        let mut state = AppState::default();
        update(&mut state, Message::ToggleFocus);
        assert!(state.is_table_focused());
    }

    #[test]
    fn message_select_next_muove_la_selezione() {
        let mut state = AppState::default();
        update(&mut state, Message::SelectNext);
        assert_eq!(state.list_state_mut().selected(), Some(1));
    }
}
```

Nota cosa `update.rs` non importa: niente `KeyCode`, niente `crossterm`. Non sa cosa sia un tasto, sa solo che esistono quattro cose che possono succedere allo stato, e come reagire a ciascuna — `SelectPrevious`/`SelectNext` ricalcano di proposito i nomi dei metodi già `pub` su `AppState`, così la corrispondenza è diretta, un ramo del `match` per un metodo.

Il ramo `_ => {}` copre `Message::Quit`, di proposito. La decisione "quando uscire dal programma" non è più compito di `update` — vive in `main.rs`, come vedremo tra un momento, che intercetta `Message::Quit` prima ancora di chiamare `update`. `update` non restituisce più `bool`: prima quel valore serviva solo a comunicare "bisogna uscire", ora che quella decisione non passa più di qui, non c'è più niente da restituire. In pratica `update` gestisce solo i messaggi che comportano davvero un cambiamento nello stato dell'applicazione: possono esistere, come in questo caso `Quit`, messaggi che invece non toccano lo stato per niente, e per quelli basta il ramo jolly. E' una scelta consapevole che ho fatto, spesso quello che vedrete è una nuova enum (solitamente `Action`) restituita da questa funzione, che identifica eventuali azioni da eseguire, cioè `Message` identifica ciò che è accaduto e `Action` l'azione da intraprendere (ad esempio `Action::Quit`).

Quel "altrove" a cui accennavo per la traduzione tasto → `Message` è `event.rs`: è lì che un `KeyCode` come `KeyCode::F(5)` o `KeyCode::Home`, che non corrisponde a nessuna delle quattro varianti, smette semplicemente di produrre un `Message` — non arriva mai fino a `update`. Il confine tra "come arrivano gli input" e "come lo stato reagisce a un messaggio" è ora un confine di modulo netto: `update.rs` conosce solo `app.rs` (gli serve `AppState` e i tre metodi pubblici), non saprà mai cosa sia un `KeyCode`.

Anche qui, come in `app.rs`, i due test in fondo al file non aprono nessun terminale — ma stavolta testiamo `update` invece dei metodi di `AppState` direttamente: la differenza è sottile ma reale. Questi test verificano che il collegamento tra un `Message` e l'azione giusta su `AppState` sia quello corretto, non solo che `AppState` sappia fare quell'azione — quello è già coperto dai test in `app.rs`.

Aggiorniamo `main.rs` per usare `app.rs` e `update.rs`:

```rust
// src/main.rs
mod app;
mod update;

use app::AppState;
use update::Message;
use ratatui::Frame;
use ratatui::crossterm::event::{self, Event, KeyCode};
use ratatui::layout::{Constraint, Layout};
use ratatui::style::Style;
use ratatui::widgets::{Block, List, Row, Table};

fn main() -> std::io::Result<()> {
    let mut state = AppState::default();

    ratatui::run(|terminal| {
        loop {
            terminal.draw(|frame| render(frame, &mut state))?;

            if let Event::Key(key) = event::read()?
                && key.kind == event::KeyEventKind::Press
            {
                let message = match key.code {
                    KeyCode::Esc | KeyCode::Char('q') => Some(Message::Quit),
                    KeyCode::Tab => Some(Message::ToggleFocus),
                    KeyCode::Up => Some(Message::SelectPrevious),
                    KeyCode::Down => Some(Message::SelectNext),
                    _ => None,
                };

                if let Some(message) = message {
                    match message {
                        Message::Quit => break Ok(()),
                        _ => update::update(&mut state, message),
                    };
                }
            }
        }
    })
}

fn render(frame: &mut Frame, state: &mut AppState) {
    // invariata rispetto all'articolo scorso: qui vive ancora in main.rs,
    // ...
}
```

### `ui.rs` — solo disegno, niente logica

Qui ci spostiamo `render` così com'è e passiamo agli stateful widget il riferimento mutabile ottenuto con `list_state_mut()`/`table_state_mut()` invece che con `&mut state.list_state`.

Approfittiamone anche per dividere `render` in due funzioni più piccole, una per pannello — non cambia niente a schermo, solo la leggibilità del file quando i widget aumentano:

```rust
// src/ui.rs
use ratatui::Frame;
use ratatui::layout::{Constraint, Layout, Rect};
use ratatui::style::Style;
use ratatui::widgets::{Block, List, Row, Table};

use crate::app::AppState;

pub fn render(frame: &mut Frame, state: &mut AppState) {
    let [area_lista, area_tabella] =
        Layout::vertical([Constraint::Percentage(50), Constraint::Percentage(50)])
            .areas(frame.area());

    render_list(frame, area_lista, state);
    render_table(frame, area_tabella, state);
}

fn render_list(frame: &mut Frame, area: Rect, state: &mut AppState) {
    let items = [
        "Scrivere l'articolo su List e Table",
        "Rispondere alle email",
        "Fare la spesa",
        "Chiamare il dentista",
        "Rivedere il PR di Marco",
    ];

    let blocco = Block::bordered().title(" Da fare ").border_style(
        if state.is_list_focused() {
            Style::new().yellow()
        } else {
            Style::new()
        },
    );

    let lista = List::new(items)
        .block(blocco)
        .highlight_style(Style::new().reversed())
        .highlight_symbol("> ");

    frame.render_stateful_widget(lista, area, state.list_state_mut());
}

fn render_table(frame: &mut Frame, area: Rect, state: &mut AppState) {
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

    let blocco = Block::bordered().title(" Completate di recente ").border_style(
        if state.is_table_focused() {
            Style::new().yellow()
        } else {
            Style::new()
        },
    );

    let tabella = Table::new(righe, larghezze)
        .header(Row::new(["Data", "Attività", "Durata"]).style(Style::new().bold()))
        .block(blocco)
        .row_highlight_style(Style::new().reversed());

    frame.render_stateful_widget(tabella, area, state.table_state_mut());
}
```

`render` ora legge come un indice: divide lo schermo, poi chiama una funzione per pannello. `render_list` e `render_table` prendono ciascuna la propria `Rect` — non calcolano più il `Layout` per conto proprio, lo ricevono già pronto da `render` — così, se domani aggiungi un terzo pannello, tocchi solo `render`, non le funzioni che disegnano i singoli widget.

In `main.rs` aggiungiamo `mod ui;` e sostituiamo la funzione locale con la chiamata a `ui::render`, lasciando per ora al suo posto il blocco che traduce `key.code` in `Message`:

```rust
// src/main.rs, secondo stadio
mod app;
mod ui;
mod update;

use app::AppState;
use update::Message;
use ratatui::crossterm::event::{self, Event, KeyCode};

fn main() -> std::io::Result<()> {
    let mut state = AppState::default();

    ratatui::run(|terminal| {
        loop {
            terminal.draw(|frame| ui::render(frame, &mut state))?;

            if let Event::Key(key) = event::read()?
                && key.kind == event::KeyEventKind::Press
            {
                let message = match key.code {
                    KeyCode::Esc | KeyCode::Char('q') => Some(Message::Quit),
                    KeyCode::Tab => Some(Message::ToggleFocus),
                    KeyCode::Up => Some(Message::SelectPrevious),
                    KeyCode::Down => Some(Message::SelectNext),
                    _ => None,
                };

                if let Some(message) = message {
                    match message {
                        Message::Quit => break Ok(()),
                        _ => update::update(&mut state, message),
                    };
                }
            }
        }
    })
}
```

### `event.rs` — da tasto a `Message`, in un posto solo

Ultimo spostamento: la parte che legge un evento e decide se e come trasformarlo in un `Message`.

```rust
// src/event.rs
use ratatui::crossterm::event::{self, Event, KeyCode};

use crate::update::Message;

pub fn read_message() -> std::io::Result<Option<Message>> {
    let event = event::read()?;

    if let Event::Key(key) = event
        && key.kind == event::KeyEventKind::Press
    {
        return Ok(to_message(key.code));
    }

    Ok(None)
}

fn to_message(code: KeyCode) -> Option<Message> {
    match code {
        KeyCode::Esc | KeyCode::Char('q') => Some(Message::Quit),
        KeyCode::Tab => Some(Message::ToggleFocus),
        KeyCode::Up => Some(Message::SelectPrevious),
        KeyCode::Down => Some(Message::SelectNext),
        _ => None,
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn q_produce_un_messaggio_di_uscita() {
        assert!(matches!(to_message(KeyCode::Char('q')), Some(Message::Quit)));
    }

    #[test]
    fn un_tasto_non_gestito_non_produce_alcun_messaggio() {
        assert!(to_message(KeyCode::Char('x')).is_none());
    }
}
```

Il blocco che nell'articolo scorso stava scomodo in mezzo al loop di `main.rs` ora ha un nome e un posto suo: `to_message`, una funzione privata a questo modulo (niente `pub`, non serve fuori da qui) che fa esattamente il lavoro che faceva prima — decidere quali tasti contano e cosa significano — ma isolato, testabile in teoria da solo (prende un `KeyCode`, restituisce un `Option<Message>`, niente terminale di mezzo), e con un ramo `_ => None` che rende esplicito "questo tasto non produce niente", lo stesso ruolo che aveva il `_ => {}` di `handle_event` nell'articolo scorso — solo che ora vive nel posto giusto: dove si guarda un tasto grezzo, non dove si aggiorna lo stato.

E infatti i due test qui sopra chiamano proprio `to_message`, non `read_message`: `read_message` dipende da `event::read()`, che si blocca aspettando un evento vero dal terminale — testarla richiederebbe di simulare l'input, qualcosa che non affrontiamo in questo articolo. `to_message` invece è pura logica, niente input reale di mezzo: perfetta da testare.

`read_message` resta la funzione pubblica che il resto del programma chiama: legge un evento, e se è una pressione di tasto la fa passare per `to_message`. Il tipo di ritorno, `Option<Message>`, dice tutto quello che serve sapere su cosa può succedere: forse arriva un messaggio, forse no.

`main.rs` si riduce alla sua forma finale — e qui la forma del loop cambia una volta di più rispetto agli stadi intermedi. Non chiamiamo più `update::update(...)` dentro una condizione che decide se uscire: matchiamo direttamente su `message`, e la decisione "usciamo o no" sta tutta in quel `match`, non nel valore di ritorno di `update`.

```rust
// src/main.rs
mod app;
mod event;
mod ui;
mod update;

use app::AppState;
use update::Message;

fn main() -> std::io::Result<()> {
    let mut state = AppState::default();

    ratatui::run(|terminal| {
        loop {
            terminal.draw(|frame| ui::render(frame, &mut state))?;

            if let Some(message) = event::read_message()? {
                match message {
                    Message::Quit => break,
                    _ => update::update(&mut state, message),
                };
            }
        }

        Ok(())
    })
}
```

A questo punto, se lanci `cargo run`, il comportamento è identico a com'era all'inizio dell'articolo — è lo stesso principio già visto negli scorsi refactor: riorganizzare il codice non cambia cosa fa il programma, cambia solo come è distribuito il lavoro tra i pezzi. Guarda però cosa dice ora `main.rs`, letto per intero: quattro `mod`, uno stato di partenza, e un loop che dice "disegna, leggi un messaggio, se è `Quit` esci, altrimenti aggiorna lo stato". Ogni parola di quella frase corrisponde a una chiamata verso un modulo con un solo compito, e nessuno di quei moduli — a parte `event.rs` — sa cosa sia un `KeyCode`.

E se lanci `cargo test` invece di `cargo run`, trovi sei test — due per `app.rs`, due per `update.rs`, due per `event.rs` — che passano tutti senza aprire nessun terminale. È la dimostrazione concreta di quello che la separazione in moduli di questo articolo rende possibile: non solo un codice più facile da leggere, ma un codice che si può verificare pezzo per pezzo.

## Codice completo finale

`src/main.rs`:

```rust
mod app;
mod event;
mod ui;
mod update;

use app::AppState;
use update::Message;

fn main() -> std::io::Result<()> {
    let mut state = AppState::default();

    ratatui::run(|terminal| {
        loop {
            terminal.draw(|frame| ui::render(frame, &mut state))?;

            if let Some(message) = event::read_message()? {
                match message {
                    Message::Quit => break,
                    _ => update::update(&mut state, message),
                };
            }
        }

        Ok(())
    })
}
```

`src/app.rs`:

```rust
use ratatui::widgets::{ListState, TableState};

#[derive(PartialEq)]
pub enum Focus {
    List,
    Table,
}

pub struct AppState {
    focus: Focus,
    list_state: ListState,
    table_state: TableState,
}

impl AppState {
    pub fn toggle_focus(&mut self) {
        self.focus = match self.focus {
            Focus::List => Focus::Table,
            Focus::Table => Focus::List,
        };
    }

    pub fn select_previous(&mut self) {
        match self.focus {
            Focus::List => self.list_state.select_previous(),
            Focus::Table => self.table_state.select_previous(),
        }
    }

    pub fn select_next(&mut self) {
        match self.focus {
            Focus::List => self.list_state.select_next(),
            Focus::Table => self.table_state.select_next(),
        }
    }

    pub fn is_list_focused(&self) -> bool {
        self.focus == Focus::List
    }

    pub fn is_table_focused(&self) -> bool {
        self.focus == Focus::Table
    }

    pub fn list_state_mut(&mut self) -> &mut ListState {
        &mut self.list_state
    }

    pub fn table_state_mut(&mut self) -> &mut TableState {
        &mut self.table_state
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

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn toggle_focus_alterna_tra_lista_e_tabella() {
        let mut state = AppState::default();
        state.toggle_focus();
        assert!(state.is_table_focused());
        state.toggle_focus();
        assert!(state.is_list_focused());
    }

    #[test]
    fn select_next_muove_la_selezione_del_pannello_con_focus() {
        let mut state = AppState::default();

        state.select_next();
        assert_eq!(state.list_state_mut().selected(), Some(1));
        assert_eq!(state.table_state_mut().selected(), Some(0));

        state.toggle_focus();
        state.select_next();
        assert_eq!(state.table_state_mut().selected(), Some(1));
    }
}
```

`src/update.rs`:

```rust
use crate::app::AppState;

pub enum Message {
    Quit,
    ToggleFocus,
    SelectPrevious,
    SelectNext,
}

pub fn update(state: &mut AppState, message: Message) {
    match message {
        Message::ToggleFocus => state.toggle_focus(),
        Message::SelectPrevious => state.select_previous(),
        Message::SelectNext => state.select_next(),
        _ => {}
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn message_toggle_focus_cambia_il_focus() {
        let mut state = AppState::default();
        update(&mut state, Message::ToggleFocus);
        assert!(state.is_table_focused());
    }

    #[test]
    fn message_select_next_muove_la_selezione() {
        let mut state = AppState::default();
        update(&mut state, Message::SelectNext);
        assert_eq!(state.list_state_mut().selected(), Some(1));
    }
}
```

`src/event.rs`:

```rust
use ratatui::crossterm::event::{self, Event, KeyCode};

use crate::update::Message;

pub fn read_message() -> std::io::Result<Option<Message>> {
    let event = event::read()?;

    if let Event::Key(key) = event
        && key.kind == event::KeyEventKind::Press
    {
        return Ok(to_message(key.code));
    }

    Ok(None)
}

fn to_message(code: KeyCode) -> Option<Message> {
    match code {
        KeyCode::Esc | KeyCode::Char('q') => Some(Message::Quit),
        KeyCode::Tab => Some(Message::ToggleFocus),
        KeyCode::Up => Some(Message::SelectPrevious),
        KeyCode::Down => Some(Message::SelectNext),
        _ => None,
    }
}

#[cfg(test)]
mod tests {
    use super::*;

    #[test]
    fn q_produce_un_messaggio_di_uscita() {
        assert!(matches!(to_message(KeyCode::Char('q')), Some(Message::Quit)));
    }

    #[test]
    fn un_tasto_non_gestito_non_produce_alcun_messaggio() {
        assert!(to_message(KeyCode::Char('x')).is_none());
    }
}
```

`src/ui.rs`:

```rust
use ratatui::Frame;
use ratatui::layout::{Constraint, Layout, Rect};
use ratatui::style::Style;
use ratatui::widgets::{Block, List, Row, Table};

use crate::app::AppState;

pub fn render(frame: &mut Frame, state: &mut AppState) {
    let [area_lista, area_tabella] =
        Layout::vertical([Constraint::Percentage(50), Constraint::Percentage(50)])
            .areas(frame.area());

    render_list(frame, area_lista, state);
    render_table(frame, area_tabella, state);
}

fn render_list(frame: &mut Frame, area: Rect, state: &mut AppState) {
    let items = [
        "Scrivere l'articolo su List e Table",
        "Rispondere alle email",
        "Fare la spesa",
        "Chiamare il dentista",
        "Rivedere il PR di Marco",
    ];

    let blocco = Block::bordered().title(" Da fare ").border_style(
        if state.is_list_focused() {
            Style::new().yellow()
        } else {
            Style::new()
        },
    );

    let lista = List::new(items)
        .block(blocco)
        .highlight_style(Style::new().reversed())
        .highlight_symbol("> ");

    frame.render_stateful_widget(lista, area, state.list_state_mut());
}

fn render_table(frame: &mut Frame, area: Rect, state: &mut AppState) {
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

    let blocco = Block::bordered().title(" Completate di recente ").border_style(
        if state.is_table_focused() {
            Style::new().yellow()
        } else {
            Style::new()
        },
    );

    let tabella = Table::new(righe, larghezze)
        .header(Row::new(["Data", "Attività", "Durata"]).style(Style::new().bold()))
        .block(blocco)
        .row_highlight_style(Style::new().reversed());

    frame.render_stateful_widget(tabella, area, state.table_state_mut());
}
```

Il progetto è passato da un file a cinque, ciascuno con una domanda a cui risponde: `app.rs` sa cosa l'app ricorda, `update.rs` sa come reagisce a un messaggio, `ui.rs` sa come disegnarlo, `event.rs` sa come trasformare un tasto premuto in un `Message`; `main.rs` orchestra tutti e quattro senza contenere logica propria.

Soprattutto, `handle_event(state, event: Event)` è diventato `update(state, message: Message)`, con `Message` che non avvolge più semplicemente un `KeyCode`, ma elenca i quattro *significati* che contano per questa app — `Quit`, `ToggleFocus`, `SelectPrevious`, `SelectNext`. La differenza non è cosmetica: `update.rs` non importa più niente di `crossterm`, e potresti collegare `Message::ToggleFocus` a un tasto diverso, a una voce di menu, o — quando arriveremo ai thread — a un evento che non ha niente a che fare con la tastiera, senza toccare una riga di `update.rs`. La traduzione da "che tasto era" a "cosa significa" ha ora una casa precisa, `to_message` dentro `event.rs`.

Hai anche visto una scelta di design in più rispetto a prima: la decisione "quando uscire dal programma" si è spostata da `update` (che restituiva `bool`) a `main.rs`, che matcha su `Message::Quit` prima ancora di chiamare `update`. `update` non ha più bisogno di restituire nulla — fa solo quello che il nome promette, aggiorna lo stato — e il `match` in `main.rs` rende esplicito, leggendolo, cosa fa uscire dal loop e cosa no.

Due cose in più rispetto al programma dell'articolo scorso, entrambe piccole: `render` in `ui.rs` si è divisa in `render_list`/`render_table`, una funzione per pannello — non cambia niente a schermo, ma se domani arriva un terzo widget tocchi solo `render`. E, soprattutto, sono comparsi i primi `#[cfg(test)] mod tests` della serie: sei test in tutto, tre file, senza aprire un terminale. Non è un caso che sia potuto succedere solo ora — è la conseguenza diretta di aver separato lo stato (`app.rs`) e la logica di aggiornamento (`update.rs`, e la parte di `event.rs` che traduce un tasto) da tutto ciò che tocca `ratatui::run` e il terminale vero.
