+++
title = "Ratatui passo dopo passo - Aggiornamenti in background: due thread, un canale `mpsc`, due vere chiamate di rete"
date = 2026-10-21
description = "Impariamo a gestire chiamate HTTP in background con Rust e Ratatui: thread, canali mpsc e messaggi per aggiornare l'interfaccia senza bloccarla."
[taxonomies]
tags = ["rust", "ratatui"]
[extra]
comments = true
+++


## Cosa costruiamo oggi

Riprendiamo l'app modulare dell'articolo scorso e le cambiamo il contenuto: la `List` non mostra più cinque cose da fare scritte a mano, ma gli utenti recuperati da [JSONPlaceholder](https://jsonplaceholder.typicode.com/) — una API REST pubblica pensata apposta per esercitarsi — caricati da un thread che parte da solo all'avvio. La `Table` resta vuota finché non selezioni un utente nella lista e premi **Invio**: a quel punto parte un *secondo* thread, che recupera solo i to-do di quella persona e li mostra. Due fonti di dati diverse, due momenti diversi in cui un thread si genera, un solo meccanismo che le fa arrivare entrambe fino allo schermo: un canale `mpsc`, sul quale oggi viaggia anche la tastiera — non solo i risultati di rete.

All'avvio:

```
┌ Utenti ──────────────────────────────────┐
│ / Caricamento utenti...                  │
└──────────────────────────────────────────┘
┌ To-do ───────────────────────────────────┐
│ Seleziona un utente e premi Invio.       │
└──────────────────────────────────────────┘
```

Un istante dopo, con la lista caricata:

```
┌ Utenti ─────────────────────────────────────────┐
│ > Leanne Graham                                 │
│   Ervin Howell                                  │
│   Clementine Bauch                              │
│   Patricia Lebsack                              │
│   Chelsey Dietrich                              │
└──────────── ↑↓ seleziona · Invio mostra i to-do ┘
┌ To-do ──────────────────────────────────────────┐
│ Seleziona un utente e premi Invio.              │
└─────────────────────────────────────────────────┘
```

E dopo aver premuto Invio su "Leanne Graham":

```
┌ To-do di Leanne Graham ─────────────────────────────┐
│ ID    Titolo                          Stato         │
│ 1     delectus aut autem              Da fare       │
│ 2     quis ut nam facilis...          Completato    │
│ 3     fugiat veniam minus             Da fare       │
│ ...                                                 │
└─────────────────────────────────────────────────────┘
```

(il bordo giallo del focus, che l'ASCII qui sopra non può rendere, è già passato da solo sul pannello "To-do": dopo la conferma puoi scorrere subito i to-do con le frecce, senza premere `Tab`).

## Codice, passo per passo

Prendiamo come base il codice completo dell'articolo sui moduli e su `Message`

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

### Le nuove dipendenze: `reqwest` in modalità `blocking`

Dalla cartella del progetto:

```bash
cargo add ratatui
cargo add reqwest --features=blocking,json
cargo add serde --features=derive
```

Il primo comando non è strettamente necessario — `ratatui` è già una dipendenza dai Prerequisiti — ma è innocuo rilanciarlo: `cargo add` su una dipendenza già presente si limita ad aggiornarne la voce nel `Cargo.toml` alla versione più recente compatibile, senza duplicarla. Lo elenco per completezza, così i tre comandi insieme bastano a ricreare lo stato delle dipendenze da zero.

Il `Cargo.toml` dopo questi tre comandi dovrebbe apparire come qui di seguito:

```toml
[dependencies]
ratatui = "0.30"
reqwest = { version = "0.13", features = ["blocking", "json"] }
serde = { version = "1", features = ["derive"] }
```

Due dettagli non ovvi in questa riga:

- **`features = ["blocking", "json"]` su `reqwest`.** Per default `reqwest` è una libreria *async*: le sue funzioni restituiscono `Future` e vanno eseguite dentro un runtime come `tokio`. Noi async non l'abbiamo ancora introdotto, quindi ci serve la variante sincrona che `reqwest` espone proprio per casi come questo: la feature `blocking` aggiunge un modulo `reqwest::blocking` con funzioni che restituiscono direttamente il risultato, bloccando il thread chiamante finché la risposta non arriva. È esattamente il comportamento che vogliamo per i nostri thread dedicati. La feature `json` aggiunge la possibilità di deserializzare automaticamente il corpo della risposta in un tipo Rust.
- **`serde` con `features = ["derive"]`.** Ci serve solo per poter scrivere `#[derive(Deserialize)]` sui tipi che rappresentano un utente e un to-do; non chiamiamo mai `serde_json` direttamente — ci pensa `reqwest` a usarlo sotto al suo `.json()`.

### `data.rs` — i tipi, separati da come li procuriamo

Prima ancora di scrivere una riga che parli di rete, isoliamo *cosa rappresenta un utente* e *cosa rappresenta un to-do* in un modulo tutto suo, senza nessuna dipendenza da `reqwest` o da `std::thread`:

```rust
// src/data.rs
#[derive(Debug, Clone, serde::Deserialize)]
pub struct User {
    pub id: u32,
    pub name: String,
}

#[derive(Debug, Clone, serde::Deserialize)]
pub struct Todo {
    pub id: u32,
    pub title: String,
    pub completed: bool,
}
```

Teniamo solo i campi che ci servono davvero — `serde` scarta dal JSON quello che non trova dichiarato nella struct, quindi `User` e `Todo` restano volutamente più piccoli di quello che JSONPlaceholder restituisce davvero (che include, ad esempio, indirizzo e azienda per ogni utente).

### `background.rs` — due funzioni, due chiamate a JSONPlaceholder

Ora il file che sa fare le richieste vere e proprie, importando i tipi appena definiti:

```rust
// src/background.rs
use std::sync::mpsc::Sender;
use std::thread;

use crate::data::{Todo, User};
use crate::update::Message;

const USERS_URL: &str = "https://jsonplaceholder.typicode.com/users";

pub fn spawn_fetch_users(tx: Sender<Message>) {
    thread::spawn(move || {
        let message = match fetch_users() {
            Ok(users) => Message::UsersLoaded(users),
            Err(errore) => Message::UsersLoadFailed(errore.to_string()),
        };

        // Se chi doveva ricevere non c'è più (es. il programma si è già chiuso),
        // send() fallisce: ignoriamo l'errore, non c'è nessuno a cui interessi più.
        let _ = tx.send(message);
    });
}

fn fetch_users() -> Result<Vec<User>, reqwest::Error> {
    reqwest::blocking::get(USERS_URL)?.json()
}
```

E la seconda coppia, per i to-do di un utente — l'unica differenza reale è che questa volta la funzione prende un parametro, e lo usa per costruire l'URL:

```rust
pub fn spawn_fetch_todos(tx: Sender<Message>, user_id: u32) {
    thread::spawn(move || {
        let message = match fetch_todos(user_id) {
            Ok(todos) => Message::TodosLoaded(todos),
            Err(errore) => Message::TodosLoadFailed(errore.to_string()),
        };

        let _ = tx.send(message);
    });
}

fn fetch_todos(user_id: u32) -> Result<Vec<Todo>, reqwest::Error> {
    let url = format!("https://jsonplaceholder.typicode.com/todos?userId={user_id}");
    reqwest::blocking::get(url)?.json()
}
```

`?userId={user_id}` è un parametro di query supportato direttamente da JSONPlaceholder: invece di scaricare tutti i 200 to-do e filtrarli lato client come avremmo dovuto fare altrimenti, chiediamo al server di darci solo quelli dell'utente che ci interessa — di norma una ventina.

Nota la forma comune alle due coppie: una funzione privata (`fetch_users`/`fetch_todos`) che sa fare *una* richiesta e basta, restituendo un `Result`; una funzione pubblica (`spawn_fetch_users`/`spawn_fetch_todos`) che genera il thread, chiama quella privata, e traduce comunque vada — successo o errore — in un `Message` da spedire sul canale. `reqwest::blocking::get(url)` accetta sia una `&str` sia una `String` posseduta (come quella che costruiamo con `format!` per i to-do): non serve nessuna conversione esplicita.

Come sappiamo, `thread::spawn` prende una `closure` senza argomenti e la esegue su un thread nuovo; il `move` sposta dentro la `closure` sia `tx` (e, per `spawn_fetch_todos`, anche `user_id`) sia tutto ciò che serve alla funzione di fetch. Da fuori, chiamare una di queste due funzioni non blocca: tornano subito, e la richiesta di rete gira per conto sua. Ed è questo l'unico modulo che può creare dei thread.

I quattro `Message` usati qui sopra non esistono ancora — li aggiungiamo.

### `Message` si allarga: i risultati di rete, la conferma, e un margine per l'input

```rust
// src/update.rs
use crate::app::AppState;
use crate::data::{Todo, User};

pub enum Message {
    Quit,
    ToggleFocus,
    SelectPrevious,
    SelectNext,
    Confirm,
    Tick,
    UsersLoaded(Vec<User>),
    UsersLoadFailed(String),
    TodosLoaded(Vec<Todo>),
    TodosLoadFailed(String),
    InputError(String),
}

pub fn update(state: &mut AppState, message: Message) {
    match message {
        Message::ToggleFocus => state.toggle_focus(),
        Message::SelectPrevious => state.select_previous(),
        Message::SelectNext => state.select_next(),
        Message::Tick => state.advance_spinner(),
        Message::Confirm => state.confirm_selection(),
        Message::UsersLoaded(users) => state.set_users(users),
        Message::UsersLoadFailed(errore) => state.set_users_error(errore),
        Message::TodosLoaded(todos) => state.set_todos(todos),
        Message::TodosLoadFailed(errore) => state.set_todos_error(errore),
        _ => {}
    }
}
```

Varianti nuove rispetto all'articolo scorso:

- **`Confirm`**: l'utente ha premuto Invio. `update` si limita a chiamare `state.confirm_selection()`, guarda quale utente è evidenziato nella lista e prepara lo stato per il caricamento dei suoi to-do. Non genera un thread: `update` non sa cosa sia un thread, esattamente come non lo sapeva nell'articolo scorso.
- **`Tick`**: come nell'idea originale, "è passato del tempo, non è successo nulla di specifico" — usato solo per far avanzare lo spinner.
- **`UsersLoaded`/`UsersLoadFailed`** e **`TodosLoaded`/`TodosLoadFailed`**: gli esiti delle due richieste di rete, portati dai thread di `background.rs` fino a qui.
- **`InputError`**: la aggiungiamo già ora, anche se il modulo che la genera arriva tra due passi. È un caso diverso dagli altri due `*Failed`: non è "una richiesta di rete non è andata a buon fine" (cosa mostrabile a schermo, l'app resta usabile), ma "non riusciamo più a leggere la tastiera" — un problema che rende l'app inutilizzabile, e che infatti tratteremo diversamente: non passa da `update`, la vedrai gestita a parte in `main.rs` al passo 7. Nota che non compare in nessun ramo esplicito del `match` qui sopra: il jolly `_ => {}` la copre, ma in pratica non ci arriverà mai — `main.rs` la intercetta prima.

### `app.rs` — due stati paralleli, `UsersStatus` e `TodosStatus`

Stesso pattern usato due volte, per due fonti di dati diverse: un piccolo enum privato per rappresentare "a che punto siamo", e metodi pubblici su `AppState` per farlo evolvere e per interrogarlo.

```rust
// src/app.rs
use crate::data::{Todo, User};

enum UsersStatus {
    Loading,
    Loaded(Vec<User>),
    Failed(String),
}

enum TodosStatus {
    Idle,
    Loading,
    Loaded(Vec<Todo>),
    Failed(String),
}

const SPINNER_FRAMES: [char; 4] = ['|', '/', '-', '\\'];
```

`TodosStatus` ha una variante in più rispetto a `UsersStatus`: `Idle`, "non è ancora successo nulla". Ha senso solo per i to-do — gli utenti iniziano subito a caricarsi all'avvio, ma i to-do restano fermi finché non premi Invio su qualcuno.

I campi nuovi sulla struct:

```rust
pub struct AppState {
    focus: Focus,
    list_state: ListState,
    table_state: TableState,
    users: UsersStatus,
    todos: TodosStatus,
    current_user_id: Option<u32>,
    current_user_name: Option<String>,
    spinner_frame: usize,
}
```

`current_user_id`/`current_user_name` meritano una parola: non sono ridondanti rispetto alla selezione della `List`. La selezione (`list_state.selected()`) può muoversi in ogni momento con le frecce, ma i to-do mostrati in tabella devono restare quelli dell'ultimo utente *confermato* con Invio, non di chi in questo istante è evidenziato. Questi due campi fotografano quella conferma nel momento in cui avviene, e restano stabili finché non arriva una conferma successiva.

I metodi che aggiornano lo stato:

```rust
impl AppState {
    // ... i metodi di focus/selezione restano identici a prima ...

    pub fn set_users(&mut self, users: Vec<User>) {
        let selezione = if users.is_empty() { None } else { Some(0) };
        self.list_state = ListState::default().with_selected(selezione);
        self.users = UsersStatus::Loaded(users);
    }

    pub fn set_users_error(&mut self, errore: String) {
        self.users = UsersStatus::Failed(errore);
    }

    pub fn confirm_selection(&mut self) {
        let indice = self.list_state.selected();

        let utente = match (self.users(), indice) {
            (Some(users), Some(indice)) => users.get(indice).map(|u| (u.id, u.name.clone())),
            _ => None,
        };

        if let Some((id, nome)) = utente {
            self.current_user_id = Some(id);
            self.current_user_name = Some(nome);
            self.todos = TodosStatus::Loading;
            self.focus = Focus::Table;
        }
    }

    pub fn set_todos(&mut self, todos: Vec<Todo>) {
        let selezione = if todos.is_empty() { None } else { Some(0) };
        self.table_state = TableState::default().with_selected(selezione);
        self.todos = TodosStatus::Loaded(todos);
    }

    pub fn set_todos_error(&mut self, errore: String) {
        self.todos = TodosStatus::Failed(errore);
    }

    pub fn advance_spinner(&mut self) {
        self.spinner_frame = self.spinner_frame.wrapping_add(1) % SPINNER_FRAMES.len();
    }
}
```

`confirm_selection` non fa nulla se non c'è ancora una lista di utenti caricata, o se per qualche motivo l'indice selezionato non corrisponde a nessun utente reale (`users.get(indice)` restituisce `None` in quel caso, e l'intera catena collassa a `None` senza bisogno di gestirlo esplicitamente). Se invece trova un utente valido, salva `id` e `name` come copie possedute (`.clone()` sul nome, `id` è un `u32` e si copia da solo), mette `todos` in `Loading` — pronto perché `main.rs`, tra un momento, faccia partire davvero il thread — e sposta il focus sulla `Table`: dopo aver confermato un utente ha senso continuare a scorrere i suoi to-do con le frecce, non restare fermi sulla lista. È anche il motivo per cui, in `main.rs`, il ramo vuoto `Message::Confirm => {}` visto al passo precedente smette di essere un caso puramente ipotetico: dopo la prima conferma il focus è sulla `Table`, quindi finché non premi di nuovo `Tab` un secondo Invio cade proprio in quel ramo, senza fare nulla — comportamento corretto, non un buco lasciato scoperto.

`set_todos`, come già visto nell'articolo precedente per un caso simile, azzera `table_state` selezionando la prima riga se ce n'è almeno una: senza, una selezione rimasta su un indice del caricamento precedente potrebbe puntare a una riga che nel nuovo elenco non esiste.

Per `ui.rs` quindi:

```rust
impl AppState {
    pub fn is_users_loading(&self) -> bool {
        matches!(self.users, UsersStatus::Loading)
    }

    pub fn users_error(&self) -> Option<&str> {
        match &self.users {
            UsersStatus::Failed(errore) => Some(errore.as_str()),
            _ => None,
        }
    }

    pub fn users(&self) -> Option<&[User]> {
        match &self.users {
            UsersStatus::Loaded(users) => Some(users.as_slice()),
            _ => None,
        }
    }

    pub fn is_todos_idle(&self) -> bool {
        matches!(self.todos, TodosStatus::Idle)
    }

    pub fn is_todos_loading(&self) -> bool {
        matches!(self.todos, TodosStatus::Loading)
    }

    pub fn todos_error(&self) -> Option<&str> {
        match &self.todos {
            TodosStatus::Failed(errore) => Some(errore.as_str()),
            _ => None,
        }
    }

    pub fn todos(&self) -> Option<&[Todo]> {
        match &self.todos {
            TodosStatus::Loaded(todos) => Some(todos.as_slice()),
            _ => None,
        }
    }

    pub fn current_user_id(&self) -> Option<u32> {
        self.current_user_id
    }

    pub fn current_user_name(&self) -> Option<&str> {
        self.current_user_name.as_deref()
    }

    pub fn spinner_symbol(&self) -> char {
        SPINNER_FRAMES[self.spinner_frame]
    }
}
```

E ovviamente `Default`, aggiornato per i campi nuovi:

```rust
impl Default for AppState {
    fn default() -> Self {
        Self {
            focus: Focus::List,
            list_state: ListState::default(),
            table_state: TableState::default(),
            users: UsersStatus::Loading,
            todos: TodosStatus::Idle,
            current_user_id: None,
            current_user_name: None,
            spinner_frame: 0,
        }
    }
}
```

Due differenze che vale la pena notare: `list_state` parte senza selezione (`ListState::default()`, non più `.with_selected(Some(0))`) perché all'avvio non c'è ancora nessun utente da selezionare — sarà `set_users` a scegliere la prima riga, quando i dati arrivano davvero. E `users` parte già in `Loading`: il primo caricamento scatta subito, prima ancora che arrivi il primo `Message::Tick`, quindi ha senso che lo stato iniziale lo rifletta fin da subito.

### `event.rs` — la tastiera diventa un thread come gli altri

Finora la lettura della tastiera viveva nello stesso loop che disegna lo schermo e controlla il canale di rete: `event::poll(timeout)` aspettava al più `timeout`, poi si tornava a guardare il canale. Funziona, ma nasconde un costo: se una risposta di rete arriva un istante dopo che `poll` ha iniziato ad aspettare, resta ferma lì finché quel timeout non scade — nel caso peggiore, fino a 100ms di ritardo per vedere a schermo qualcosa che in realtà era già pronto. Risolviamo alla radice: la tastiera diventa un thread a sé, con la stessa forma di `spawn_fetch_users`/`spawn_fetch_todos` — genera, traduce l'esito in un `Message`, lo spedisce sul canale — così il loop principale avrà un solo punto di attesa invece di due.

```rust
// src/event.rs
use std::sync::mpsc::Sender;
use std::thread;

use ratatui::crossterm::event::{self, Event, KeyCode, KeyEventKind};

use crate::update::Message;

pub fn spawn_input(tx: Sender<Message>) {
    thread::spawn(move || {
        loop {
            let message = match read_message() {
                Ok(message) => message,
                Err(errore) => Message::InputError(errore.to_string()),
            };

            let input_valido = !matches!(message, Message::InputError(_));

            if tx.send(message).is_err() || !input_valido {
                return;
            }
        }
    });
}

fn read_message() -> std::io::Result<Message> {
    loop {
        let event = event::read()?;

        if let Event::Key(key) = event
            && key.kind == KeyEventKind::Press
            && let Some(message) = to_message(key.code)
        {
            return Ok(message);
        }
    }
}

fn to_message(code: KeyCode) -> Option<Message> {
    match code {
        KeyCode::Esc | KeyCode::Char('q') => Some(Message::Quit),
        KeyCode::Enter => Some(Message::Confirm),
        KeyCode::Tab => Some(Message::ToggleFocus),
        KeyCode::Up => Some(Message::SelectPrevious),
        KeyCode::Down => Some(Message::SelectNext),
        _ => None,
    }
}
```

Da notare:

- **`spawn_input` ha la stessa forma di `spawn_fetch_users`**: una funzione pubblica che genera un thread, chiama una funzione privata che fa il lavoro vero, e traduce comunque vada in un `Message` da spedire. L'unica differenza è che qui il lavoro non finisce mai da solo — continua a girare finché non c'è un motivo per fermarsi.
- **`event::read()` torna a essere bloccante**, e va benissimo che lo sia: prima doveva convivere nello stesso loop col controllo del canale, da cui il bisogno di `poll` con un timeout; ora ha un thread tutto suo, e può aspettare un tasto quanto serve senza rubare tempo a nessun altro.
- `to_message` non cambia di una riga rispetto a prima; cambia solo chi la chiama e come: `read_message` ora è un `loop` che continua a leggere finché non trova un tasto che sappiamo tradurre in un `Message`, invece di restituire `Ok(None)` quando il tasto non corrisponde a nulla.
- **Gestione dell'errore, senza `.unwrap()`**: se `event::read()` fallisce (raro, ma possibile), `read_message` restituisce `Err`, e `spawn_input` lo trasforma in `Message::InputError`. `input_valido` fa sì che, dopo aver spedito quell'errore, il thread si fermi da solo con `return` invece di continuare a leggere tasti da un'app che sta per chiudersi. Lo stesso `return` scatta se `tx.send(...)` fallisce per qualunque messaggio: significa che non c'è più nessuno in ascolto, e continuare non avrebbe senso.

### `main.rs` — un canale solo, per tutto

```rust
// src/main.rs
mod app;
mod background;
mod data;
mod event;
mod ui;
mod update;

use std::sync::mpsc::{self, RecvTimeoutError};
use std::time::Duration;

use app::AppState;
use update::Message;

fn main() -> std::io::Result<()> {
    let mut state = AppState::default();
    let (tx, rx) = mpsc::channel::<Message>();

    background::spawn_fetch_users(tx.clone());
    event::spawn_input(tx.clone());

    ratatui::run(|terminal| {
        loop {
            terminal.draw(|frame| ui::render(frame, &mut state))?;

            let message = match rx.recv_timeout(Duration::from_millis(100)) {
                Ok(message) => message,
                Err(RecvTimeoutError::Timeout) => Message::Tick,
                Err(RecvTimeoutError::Disconnected) => break,
            };

            match message {
                Message::Quit => break,
                Message::InputError(errore) => return Err(std::io::Error::other(errore)),
                Message::Confirm if state.is_list_focused() => {
                    update::update(&mut state, message);

                    if let Some(user_id) = state.current_user_id() {
                        background::spawn_fetch_todos(tx.clone(), user_id);
                    }
                }
                Message::Confirm => {}
                _ => update::update(&mut state, message),
            }
        }

        Ok(())
    })
}
```

Rispetto a un `main.rs` "semplice" che smista tutto verso `update::update` allo stesso modo, oggi quattro casi restano intercettati qui, non dentro `update`:

- **`Quit`**, come sempre — decide di uscire dal loop, `update` non ha modo di farlo.
- **`InputError`**, nuovo: unico ramo che esce dalla closure con un `Err` invece che con `break`. È deliberatamente un errore "duro", diverso dai `UsersLoadFailed`/`TodosLoadFailed` che restano invece mostrati a schermo — se non riusciamo più a leggere la tastiera l'app è comunque inutilizzabile, non ha senso continuare a disegnarla.
- **`Message::Confirm if state.is_list_focused()`** è il pezzo che già conoscevi dall'articolo scorso, invariato.
- **`Message::Confirm` (senza guard, subito sotto)**: copre esattamente il caso in cui il guard sopra non è passato — Invio premuto con il focus sulla `Table`. Non fa nulla, di proposito.

> **Per approfondire.** `update` potrebbe ricevere anche `tx` e generare il thread da solo, direttamente nel ramo `Message::Confirm`. Non lo facciamo: perderebbe la proprietà che gli abbiamo costruito addosso fin dal passo 4, "non sa cosa sia un thread né un canale", e con quella la possibilità di testarlo con un semplice `AppState` e un `Message`, senza aprire un canale vero. Il prezzo è la piccola duplicazione che vedi qui: dopo `update()` reinterroghiamo `state` per sapere cosa fare. È lo stesso compromesso che Elm risolve con un tipo `Cmd` — `update` descrive l'effetto invece di eseguirlo — che qui non introduciamo per non aggiungere un concetto in più, ma è il nome giusto se `main.rs` iniziasse ad accumulare troppi casi come questo.

Il vero cambiamento rispetto a prima è la riga con `rx.recv_timeout(Duration::from_millis(100))`, che prende il posto sia del vecchio `event::read_message(timeout)` sia del vecchio `rx.try_recv()`: un solo punto di attesa, qualunque cosa arrivi — tasto, utenti, to-do — sblocca il loop nell'istante in cui è pronta, non più fino a 100ms dopo. `Err(RecvTimeoutError::Timeout)` prende esattamente il posto che aveva il `Message::Tick` generato da `event.rs`: stesso significato — "non è arrivato nulla entro 100ms" — prodotto da un punto diverso. `Err(RecvTimeoutError::Disconnected)` è un caso che il compilatore ci obbliga a coprire ma che qui non dovrebbe mai scattare, finché `main` tiene in vita il proprio `tx`: lo trattiamo come un normale `break`.

Il resto è la stessa struttura già vista: sia `spawn_fetch_users` sia `event::spawn_input` partono prima ancora di entrare nel loop, così l'app comincia a popolarsi da sola fin dal primo fotogramma, e la tastiera è pronta da subito.

### `ui.rs` — due pannelli, ciascuno con i propri stati da disegnare

`render_list` ora si comporta come `render_table` si comportava nell'articolo precedente: tre possibili esiti (caricamento, errore, dati), un `Block` costruito una sola volta e riusato in tutti e tre.

```rust
// src/ui.rs
use ratatui::Frame;
use ratatui::layout::{Constraint, Layout, Rect};
use ratatui::style::Style;
use ratatui::text::Line;
use ratatui::widgets::{Block, List, Paragraph, Row, Table};

use crate::app::AppState;

pub fn render(frame: &mut Frame, state: &mut AppState) {
    let [area_lista, area_tabella] =
        Layout::vertical([Constraint::Percentage(50), Constraint::Percentage(50)])
            .areas(frame.area());

    render_list(frame, area_lista, state);
    render_table(frame, area_tabella, state);
}

fn render_list(frame: &mut Frame, area: Rect, state: &mut AppState) {
    let blocco = Block::bordered().title(" Utenti ").border_style(
        if state.is_list_focused() {
            Style::new().yellow()
        } else {
            Style::new()
        },
    );

    if state.is_users_loading() {
        let testo = format!("{} Caricamento utenti...", state.spinner_symbol());
        frame.render_widget(Paragraph::new(testo).block(blocco), area);
        return;
    }

    if let Some(errore) = state.users_error() {
        let testo = format!("Errore nel caricamento: {errore}");
        frame.render_widget(Paragraph::new(testo).block(blocco), area);
        return;
    }

    let blocco =
        blocco.title_bottom(Line::from(" ↑↓ seleziona · Invio mostra i to-do ").right_aligned());

    let items: Vec<String> = state
        .users()
        .unwrap_or(&[])
        .iter()
        .map(|user| user.name.clone())
        .collect();

    let lista = List::new(items)
        .block(blocco)
        .highlight_style(Style::new().reversed())
        .highlight_symbol("> ");

    frame.render_stateful_widget(lista, area, state.list_state_mut());
}

fn render_table(frame: &mut Frame, area: Rect, state: &mut AppState) {
    let titolo = match state.current_user_name() {
        Some(nome) => format!(" To-do di {nome} "),
        None => " To-do ".to_string(),
    };

    let blocco = Block::bordered().title(titolo).border_style(
        if state.is_table_focused() {
            Style::new().yellow()
        } else {
            Style::new()
        },
    );

    if state.is_todos_idle() {
        let paragraph = Paragraph::new("Seleziona un utente e premi Invio.").block(blocco);
        frame.render_widget(paragraph, area);
        return;
    }

    if state.is_todos_loading() {
        let testo = format!("{} Caricamento to-do...", state.spinner_symbol());
        frame.render_widget(Paragraph::new(testo).block(blocco), area);
        return;
    }

    if let Some(errore) = state.todos_error() {
        let testo = format!("Errore nel caricamento: {errore}");
        frame.render_widget(Paragraph::new(testo).block(blocco), area);
        return;
    }

    let righe: Vec<Row> = state
        .todos()
        .unwrap_or(&[])
        .iter()
        .map(|todo| {
            let stato = if todo.completed { "Completato" } else { "Da fare" };
            Row::new([todo.id.to_string(), todo.title.clone(), stato.to_string()])
        })
        .collect();

    let larghezze = [
        Constraint::Length(5),
        Constraint::Fill(1),
        Constraint::Length(11),
    ];

    let tabella = Table::new(righe, larghezze)
        .header(Row::new(["ID", "Titolo", "Stato"]).style(Style::new().bold()))
        .block(blocco)
        .row_highlight_style(Style::new().reversed());

    frame.render_stateful_widget(tabella, area, state.table_state_mut());
}
```

In entrambe le funzioni, i dati che finiscono nel widget vero e proprio (`items: Vec<String>` per la lista, `righe: Vec<Row>` per la tabella) sono costruiti come copie possedute — `.clone()`/`.to_string()` — prima di essere passati al widget, non come riferimenti presi in prestito da `state`.

Nota anche `blocco.title_bottom(...)` in `render_list`: come `.title(...)` (che in Ratatui è in realtà una scorciatoia per `.title_top(...)`), ma ancorato al bordo inferiore invece che a quello superiore. `Line::from(...).right_aligned()` — lo stesso metodo già visto sulle `Line` nell'articolo sul testo — lo spinge verso l'angolo in basso a destra invece di lasciarlo di default a sinistra. La riga con l'hint sui tasti compare *solo* nel ramo con i dati veri: `blocco` viene ridefinito con `let blocco = blocco.title_bottom(...)` subito prima di costruire la `List`, dopo i due `return` anticipati di caricamento ed errore — mostrare "↑↓ seleziona · Invio mostra i to-do" mentre la lista è ancora vuota o in errore non avrebbe senso, non c'è ancora nulla da selezionare.

Il titolo dinamico della `Table` (`" To-do di {nome} "` invece del solito titolo fisso passato come stringa letterale) è l'unica vera novità rispetto a come avevamo usato `Block` finora: `.title(...)` accetta tanto una `&str` quanto una `String` costruita al volo, esattamente come già ti aspetteresti da `Paragraph::new(...)`.

A questo punto, se lanci `cargo run`, vedi prima lo spinner nel pannello "Utenti" e poco dopo l'elenco vero, con "Leanne Graham" già evidenziata. Il pannello "To-do" resta fermo su "Seleziona un utente e premi Invio." finché non lo fai davvero: a quel punto il titolo cambia a "To-do di Leanne Graham", il bordo giallo del focus salta da solo sul pannello "To-do", appare lo spinner per un istante, e poi la tabella con i suoi to-do — completati e non, già pronta da scorrere con le frecce senza dover premere `Tab`. Premi `Tab` per tornare alla lista, scegli un altro utente e premi di nuovo Invio: il pannello "To-do" torna a caricare, questa volta per la nuova persona, e il focus ci salta sopra un'altra volta.

## Codice completo finale

`Cargo.toml`:

```toml
[package]
name = "hello-ratatui"
version = "0.1.0"
edition = "2024"

[dependencies]
ratatui = "0.30"
reqwest = { version = "0.13", features = ["blocking", "json"] }
serde = { version = "1", features = ["derive"] }
```

`src/main.rs`:

```rust
mod app;
mod background;
mod data;
mod event;
mod ui;
mod update;

use std::sync::mpsc::{self, RecvTimeoutError};
use std::time::Duration;

use app::AppState;
use update::Message;

fn main() -> std::io::Result<()> {
    let mut state = AppState::default();
    let (tx, rx) = mpsc::channel::<Message>();

    background::spawn_fetch_users(tx.clone());
    event::spawn_input(tx.clone());

    ratatui::run(|terminal| {
        loop {
            terminal.draw(|frame| ui::render(frame, &mut state))?;

            let message = match rx.recv_timeout(Duration::from_millis(100)) {
                Ok(message) => message,
                Err(RecvTimeoutError::Timeout) => Message::Tick,
                Err(RecvTimeoutError::Disconnected) => break,
            };

            match message {
                Message::Quit => break,
                Message::InputError(errore) => return Err(std::io::Error::other(errore)),
                Message::Confirm if state.is_list_focused() => {
                    update::update(&mut state, message);

                    if let Some(user_id) = state.current_user_id() {
                        background::spawn_fetch_todos(tx.clone(), user_id);
                    }
                }
                Message::Confirm => {}
                _ => update::update(&mut state, message),
            }
        }

        Ok(())
    })
}
```

`src/data.rs`:

```rust
#[derive(Debug, Clone, serde::Deserialize)]
pub struct User {
    pub id: u32,
    pub name: String,
}

#[derive(Debug, Clone, serde::Deserialize)]
pub struct Todo {
    pub id: u32,
    pub title: String,
    pub completed: bool,
}
```

`src/background.rs`:

```rust
use std::sync::mpsc::Sender;
use std::thread;

use crate::data::{Todo, User};
use crate::update::Message;

const USERS_URL: &str = "https://jsonplaceholder.typicode.com/users";

pub fn spawn_fetch_users(tx: Sender<Message>) {
    thread::spawn(move || {
        let message = match fetch_users() {
            Ok(users) => Message::UsersLoaded(users),
            Err(errore) => Message::UsersLoadFailed(errore.to_string()),
        };

        let _ = tx.send(message);
    });
}

fn fetch_users() -> Result<Vec<User>, reqwest::Error> {
    reqwest::blocking::get(USERS_URL)?.json()
}

pub fn spawn_fetch_todos(tx: Sender<Message>, user_id: u32) {
    thread::spawn(move || {
        let message = match fetch_todos(user_id) {
            Ok(todos) => Message::TodosLoaded(todos),
            Err(errore) => Message::TodosLoadFailed(errore.to_string()),
        };

        let _ = tx.send(message);
    });
}

fn fetch_todos(user_id: u32) -> Result<Vec<Todo>, reqwest::Error> {
    let url = format!("https://jsonplaceholder.typicode.com/todos?userId={user_id}");
    reqwest::blocking::get(url)?.json()
}
```

`src/app.rs`:

```rust
use ratatui::widgets::{ListState, TableState};

use crate::data::{Todo, User};

#[derive(PartialEq)]
pub enum Focus {
    List,
    Table,
}

enum UsersStatus {
    Loading,
    Loaded(Vec<User>),
    Failed(String),
}

enum TodosStatus {
    Idle,
    Loading,
    Loaded(Vec<Todo>),
    Failed(String),
}

const SPINNER_FRAMES: [char; 4] = ['|', '/', '-', '\\'];

pub struct AppState {
    focus: Focus,
    list_state: ListState,
    table_state: TableState,
    users: UsersStatus,
    todos: TodosStatus,
    current_user_id: Option<u32>,
    current_user_name: Option<String>,
    spinner_frame: usize,
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

    pub fn set_users(&mut self, users: Vec<User>) {
        let selezione = if users.is_empty() { None } else { Some(0) };
        self.list_state = ListState::default().with_selected(selezione);
        self.users = UsersStatus::Loaded(users);
    }

    pub fn set_users_error(&mut self, errore: String) {
        self.users = UsersStatus::Failed(errore);
    }

    pub fn confirm_selection(&mut self) {
        let indice = self.list_state.selected();

        let utente = match (self.users(), indice) {
            (Some(users), Some(indice)) => users.get(indice).map(|u| (u.id, u.name.clone())),
            _ => None,
        };

        if let Some((id, nome)) = utente {
            self.current_user_id = Some(id);
            self.current_user_name = Some(nome);
            self.todos = TodosStatus::Loading;
            self.focus = Focus::Table;
        }
    }

    pub fn set_todos(&mut self, todos: Vec<Todo>) {
        let selezione = if todos.is_empty() { None } else { Some(0) };
        self.table_state = TableState::default().with_selected(selezione);
        self.todos = TodosStatus::Loaded(todos);
    }

    pub fn set_todos_error(&mut self, errore: String) {
        self.todos = TodosStatus::Failed(errore);
    }

    pub fn advance_spinner(&mut self) {
        self.spinner_frame = self.spinner_frame.wrapping_add(1) % SPINNER_FRAMES.len();
    }

    pub fn is_users_loading(&self) -> bool {
        matches!(self.users, UsersStatus::Loading)
    }

    pub fn users_error(&self) -> Option<&str> {
        match &self.users {
            UsersStatus::Failed(errore) => Some(errore.as_str()),
            _ => None,
        }
    }

    pub fn users(&self) -> Option<&[User]> {
        match &self.users {
            UsersStatus::Loaded(users) => Some(users.as_slice()),
            _ => None,
        }
    }

    pub fn is_todos_idle(&self) -> bool {
        matches!(self.todos, TodosStatus::Idle)
    }

    pub fn is_todos_loading(&self) -> bool {
        matches!(self.todos, TodosStatus::Loading)
    }

    pub fn todos_error(&self) -> Option<&str> {
        match &self.todos {
            TodosStatus::Failed(errore) => Some(errore.as_str()),
            _ => None,
        }
    }

    pub fn todos(&self) -> Option<&[Todo]> {
        match &self.todos {
            TodosStatus::Loaded(todos) => Some(todos.as_slice()),
            _ => None,
        }
    }

    pub fn current_user_id(&self) -> Option<u32> {
        self.current_user_id
    }

    pub fn current_user_name(&self) -> Option<&str> {
        self.current_user_name.as_deref()
    }

    pub fn spinner_symbol(&self) -> char {
        SPINNER_FRAMES[self.spinner_frame]
    }
}

impl Default for AppState {
    fn default() -> Self {
        Self {
            focus: Focus::List,
            list_state: ListState::default(),
            table_state: TableState::default(),
            users: UsersStatus::Loading,
            todos: TodosStatus::Idle,
            current_user_id: None,
            current_user_name: None,
            spinner_frame: 0,
        }
    }
}
```

`src/update.rs`:

```rust
use crate::app::AppState;
use crate::data::{Todo, User};

pub enum Message {
    Quit,
    ToggleFocus,
    SelectPrevious,
    SelectNext,
    Confirm,
    Tick,
    UsersLoaded(Vec<User>),
    UsersLoadFailed(String),
    TodosLoaded(Vec<Todo>),
    TodosLoadFailed(String),
    InputError(String),
}

pub fn update(state: &mut AppState, message: Message) {
    match message {
        Message::ToggleFocus => state.toggle_focus(),
        Message::SelectPrevious => state.select_previous(),
        Message::SelectNext => state.select_next(),
        Message::Tick => state.advance_spinner(),
        Message::Confirm => state.confirm_selection(),
        Message::UsersLoaded(users) => state.set_users(users),
        Message::UsersLoadFailed(errore) => state.set_users_error(errore),
        Message::TodosLoaded(todos) => state.set_todos(todos),
        Message::TodosLoadFailed(errore) => state.set_todos_error(errore),
        _ => {}
    }
}
```

`src/event.rs`:

```rust
use std::sync::mpsc::Sender;
use std::thread;

use ratatui::crossterm::event::{self, Event, KeyCode, KeyEventKind};

use crate::update::Message;

pub fn spawn_input(tx: Sender<Message>) {
    thread::spawn(move || {
        loop {
            let message = match read_message() {
                Ok(message) => message,
                Err(errore) => Message::InputError(errore.to_string()),
            };

            let input_valido = !matches!(message, Message::InputError(_));

            if tx.send(message).is_err() || !input_valido {
                return;
            }
        }
    });
}

fn read_message() -> std::io::Result<Message> {
    loop {
        let event = event::read()?;

        if let Event::Key(key) = event
            && key.kind == KeyEventKind::Press
            && let Some(message) = to_message(key.code)
        {
            return Ok(message);
        }
    }
}

fn to_message(code: KeyCode) -> Option<Message> {
    match code {
        KeyCode::Esc | KeyCode::Char('q') => Some(Message::Quit),
        KeyCode::Enter => Some(Message::Confirm),
        KeyCode::Tab => Some(Message::ToggleFocus),
        KeyCode::Up => Some(Message::SelectPrevious),
        KeyCode::Down => Some(Message::SelectNext),
        _ => None,
    }
}
```

`src/ui.rs`:

```rust
use ratatui::Frame;
use ratatui::layout::{Constraint, Layout, Rect};
use ratatui::style::Style;
use ratatui::text::Line;
use ratatui::widgets::{Block, List, Paragraph, Row, Table};

use crate::app::AppState;

pub fn render(frame: &mut Frame, state: &mut AppState) {
    let [area_lista, area_tabella] =
        Layout::vertical([Constraint::Percentage(50), Constraint::Percentage(50)])
            .areas(frame.area());

    render_list(frame, area_lista, state);
    render_table(frame, area_tabella, state);
}

fn render_list(frame: &mut Frame, area: Rect, state: &mut AppState) {
    let blocco = Block::bordered().title(" Utenti ").border_style(
        if state.is_list_focused() {
            Style::new().yellow()
        } else {
            Style::new()
        },
    );

    if state.is_users_loading() {
        let testo = format!("{} Caricamento utenti...", state.spinner_symbol());
        frame.render_widget(Paragraph::new(testo).block(blocco), area);
        return;
    }

    if let Some(errore) = state.users_error() {
        let testo = format!("Errore nel caricamento: {errore}");
        frame.render_widget(Paragraph::new(testo).block(blocco), area);
        return;
    }

    let blocco =
        blocco.title_bottom(Line::from(" ↑↓ seleziona · Invio mostra i to-do ").right_aligned());

    let items: Vec<String> = state
        .users()
        .unwrap_or(&[])
        .iter()
        .map(|user| user.name.clone())
        .collect();

    let lista = List::new(items)
        .block(blocco)
        .highlight_style(Style::new().reversed())
        .highlight_symbol("> ");

    frame.render_stateful_widget(lista, area, state.list_state_mut());
}

fn render_table(frame: &mut Frame, area: Rect, state: &mut AppState) {
    let titolo = match state.current_user_name() {
        Some(nome) => format!(" To-do di {nome} "),
        None => " To-do ".to_string(),
    };

    let blocco = Block::bordered().title(titolo).border_style(
        if state.is_table_focused() {
            Style::new().yellow()
        } else {
            Style::new()
        },
    );

    if state.is_todos_idle() {
        let paragraph = Paragraph::new("Seleziona un utente e premi Invio.").block(blocco);
        frame.render_widget(paragraph, area);
        return;
    }

    if state.is_todos_loading() {
        let testo = format!("{} Caricamento to-do...", state.spinner_symbol());
        frame.render_widget(Paragraph::new(testo).block(blocco), area);
        return;
    }

    if let Some(errore) = state.todos_error() {
        let testo = format!("Errore nel caricamento: {errore}");
        frame.render_widget(Paragraph::new(testo).block(blocco), area);
        return;
    }

    let righe: Vec<Row> = state
        .todos()
        .unwrap_or(&[])
        .iter()
        .map(|todo| {
            let stato = if todo.completed { "Completato" } else { "Da fare" };
            Row::new([todo.id.to_string(), todo.title.clone(), stato.to_string()])
        })
        .collect();

    let larghezze = [
        Constraint::Length(5),
        Constraint::Fill(1),
        Constraint::Length(11),
    ];

    let tabella = Table::new(righe, larghezze)
        .header(Row::new(["ID", "Titolo", "Stato"]).style(Style::new().bold()))
        .block(blocco)
        .row_highlight_style(Style::new().reversed());

    frame.render_stateful_widget(tabella, area, state.table_state_mut());
}
```

## Cosa abbiamo aggiunto rispetto a prima

Due fonti di dati remote invece di una, e soprattutto due *momenti* diversi in cui un thread nasce: `spawn_fetch_users` parte da solo all'avvio, `spawn_fetch_todos` parte in risposta a un'azione dell'utente (`Message::Confirm`, generato premendo Invio). Lo stesso identico meccanismo — canale `mpsc`, traduzione del risultato in un `Message` — regge entrambi i casi senza modifiche: è la prova che il lavoro fatto nell'articolo scorso per disaccoppiare `Message` dalla tastiera paga proprio in situazioni come questa.

La terza fonte, quella nuova rispetto a una prima versione più ingenua, è la tastiera stessa: anche `event::spawn_input` genera un thread e scrive sullo stesso canale, invece di essere letta con un `poll` a parte nel loop principale. Il vantaggio non è solo di uniformità — `main.rs` ha un solo punto di attesa, `rx.recv_timeout(...)`, invece di due meccanismi separati — è anche una latenza percepita più bassa: prima, un risultato di rete arrivato mentre `poll` era a metà della sua attesa restava fermo lì fino a 100ms anche se pronto subito; ora qualunque cosa arrivi, da qualunque delle tre fonti, sblocca il loop nell'istante in cui è pronta.

Il progetto guadagna anche un modulo in più rispetto al minimo indispensabile: `data.rs`, che tiene `User` e `Todo`.

Sul fronte dello stato, `AppState` ora tiene due enum paralleli (`UsersStatus`, `TodosStatus`) dietro la stessa interfaccia a metodi pubblici già vista, e un piccolo "fermo immagine" della conferma dell'utente (`current_user_id`/`current_user_name`) che tiene la tabella stabile anche se la selezione nella lista si sposta altrove nel frattempo. `ui.rs` è cresciuto in modo simmetrico: entrambi i pannelli ora sanno disegnare tre condizioni diverse — caricamento, errore, dati — con lo stesso schema (`Block` costruito una volta, `return` anticipato per i primi due casi, widget vero nel terzo).

L'unico punto davvero nuovo, mai incontrato prima nella serie, è il conflitto tra prestito immutabile e mutabile visto al passo 8: quando i dati da disegnare vengono letti da `state` e lo stesso `state` serve poco dopo, mutabile, per lo `stateful widget`, la soluzione è sempre la stessa — trasformare il prestito in dati posseduti (`.clone()`, `.to_string()`) prima che il secondo prestito venga richiesto.
