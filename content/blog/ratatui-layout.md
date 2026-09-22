+++
title = "Ratatui passo dopo passo - Layout: dividere lo schermo, e usare Flex per gestire lo spazio che avanza"
date = 2026-09-22
description = "Impariamo a usare Layout e Flex in Ratatui per dividere il terminale in aree, creare layout innestati e distribuire lo spazio tra gli elementi."
[taxonomies]
tags = ["rust", "ratatui"]
[extra]
comments = true
+++

## Cosa costruiamo oggi

Finora abbiamo sempre disegnato su `frame.area()`, cioè tutto lo schermo in un colpo solo. Oggi impariamo a spezzarlo in zone — intestazione, corpo, piè di pagina — e a controllare cosa succede allo spazio che avanza quando le zone non riempiono esattamente lo schermo. Il secondo problema è quello che risolve `Flex`.

Il risultato finale: una schermata con intestazione fissa in alto, un corpo che si allarga o si stringe insieme al terminale, e in basso tre "pulsanti" testuali che, a seconda del `Flex` scelto, si dispongono in modi diversi nello spazio disponibile.

```
┌──────────────────────────────────────────┐
│          La mia app da terminale         │  ← intestazione, altezza fissa
├──────────────────────────────────────────┤
│                                          │
│  Qui in mezzo, in futuro, ci finirà il   │  ← corpo, si allarga con lo schermo
│  contenuto vero della schermata.         │
│                                          │
├──────────────────────────────────────────┤
│ [ Nuovo ]        [ Salva ]      [ Esci ] │  ← piè di pagina, Flex::SpaceBetween
└──────────────────────────────────────────┘
```

## Prerequisiti

Partiamo dal codice del primo articolo: il progetto `hello-ratatui`, con `Cargo.toml` invariato e questo `main.rs` come punto di partenza (stessa struttura di sempre: `ratatui::run`, loop, uscita a qualsiasi tasto):

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

Come al solito tocchiamo solo l'interno di `terminal.draw(...)`.

## Codice, passo per passo

### Dividere lo schermo in due parti

`Layout` prende un'area (un `Rect`) e la spezza in più aree più piccole, secondo una lista di vincoli (`Constraint`). Il caso più semplice: un'intestazione alta 3 righe, e il resto dello schermo sotto.

```rust
use ratatui::layout::{Constraint, Layout};

terminal.draw(|frame| {
    let [header, body] = Layout::vertical([
        Constraint::Length(3),
        Constraint::Fill(1),
    ])
    .areas(frame.area());

    frame.render_widget(Paragraph::new("intestazione"), header);
    frame.render_widget(Paragraph::new("corpo"), body);
})?;
```

Due cose da notare:

- `Constraint::Length(3)` significa "esattamente 3 righe, punto". `Constraint::Fill(1)` significa "prendi tutto quello che resta". Se ridimensioni il terminale, l'intestazione resta sempre alta 3 righe e il corpo si allarga o si restringe di conseguenza — perché `terminal.draw` viene richiamato ad ogni giro del loop, `Layout` ricalcola le aree sulla dimensione attuale dello schermo ogni volta.
- `.areas(frame.area())` restituisce un array di `Rect` che puoi destrutturare direttamente con `let [header, body] = ...`, a patto che il numero di elementi a sinistra corrisponda al numero di vincoli. È la forma più comoda quando sai in anticipo quante aree ti servono.

A questo punto, se lanci `cargo run`, vedi due righe di testo: "intestazione" in alto, "corpo" subito sotto, che segue il resto dello schermo. Il confine tra le due aree, però, è visibile solo se guardi bene dove finisce una riga e comincia l'altra — non proprio immediato. Per rendertelo evidente a colpo d'occhio, puoi dare a ciascuna area uno sfondo colorato diverso:

```rust
use ratatui::style::{Color, Style};
use ratatui::widgets::Block;

terminal.draw(|frame| {
    let [header, body] = Layout::vertical([
        Constraint::Length(3),
        Constraint::Fill(1),
    ])
    .areas(frame.area());

    let block_green = Block::default().style(Style::default().bg(Color::LightGreen));
    let block_blue = Block::default().style(Style::default().bg(Color::LightBlue));

    frame.render_widget(Paragraph::new("intestazione").block(block_green), header);
    frame.render_widget(Paragraph::new("corpo").block(block_blue), body);
})?;
```

> **Anticipazione.** `Block` è il widget che vedremo per bene nei prossimi articoli — di solito serve a disegnare un bordo attorno a un'area, ma qui ci basta la sua forma più semplice: uno sfondo colorato, applicato con `.block(...)` a un `Paragraph` proprio come faresti con un bordo. Per ora usalo solo come "evidenziatore" da smontare in seguito: non serve altro per capire dove finisce un'area e comincia l'altra.

Da qui in avanti, ogni volta che introduco un nuovo esempio di `Layout`, lo mostro anche colorato — aiuta a vedere subito la disposizione reale delle aree invece di doverla immaginare dal codice.

`Length` e `Fill` non sono gli unici vincoli disponibili. Prima di andare avanti vale la pena vederli tutti in un colpo solo, così quando ne spunta uno nuovo più avanti nell'articolo sai già cosa fa:

| Vincolo | Cosa dice a `Layout` |
|---|---|
| `Constraint::Length(n)` | "Voglio esattamente `n` celle, non una di più né una di meno." Non si adatta allo schermo: un `Length(20)` resta largo 20 celle sia su un terminale minuscolo sia su uno enorme. |
| `Constraint::Percentage(n)` | "Voglio il `n`% dell'area del genitore" (l'area che stai dividendo, non l'intero schermo se sei dentro un layout innestato). Cresce e si restringe insieme al terminale. |
| `Constraint::Ratio(a, b)` | Come `Percentage`, ma espresso come frazione (`Ratio(1, 3)` = un terzo) invece che come percentuale: utile quando la percentuale esatta non è un numero intero rappresentabile (un terzo dello schermo non è una percentuale tonda). |
| `Constraint::Min(n)` | "Non voglio scendere sotto `n` celle" — ma, spazio permettendo, cresce oltre `n` per occupare l'eccesso, un po' come `Fill`. Utile insieme ad altri vincoli per garantire una soglia minima. |
| `Constraint::Max(n)` | "Non voglio superare `n` celle" — il contrario di `Min`: fa da tetto, non da soglia. |
| `Constraint::Fill(peso)` | "Prendi la tua fetta proporzionale di quello che resta", come visto sopra: non ha una dimensione propria, si adatta sempre a riempire lo spazio libero. |

Nella pratica, userai quasi sempre `Length` per le zone a dimensione fissa (intestazioni, piè di pagina, barre laterali strette) e `Fill` per la zona che deve occupare "tutto il resto" — è lo schema che stiamo seguendo in questo articolo. `Percentage`, `Ratio`, `Min` e `Max` tornano utili quando hai bisogno di un controllo più fine: ad esempio una barra laterale che deve restare almeno larga 20 celle ma può crescere fino al 30% dello schermo se c'è spazio (`Constraint::Min(20)` combinato a un secondo vincolo `Constraint::Percentage(30)` sull'altra area).

> **Per approfondire.** Non esiste, che io sappia, un `Constraint` o un modo "pulito" che unisca `Min` e `Max` in un solo intervallo ("tra 20 e 40 celle, non di più non di meno") — ogni voce dell'array ne accetta uno solo. Se ti serve davvero un intervallo su un'unica area, la strada pratica è lasciare che un vincolo elastico (`Fill` o `Percentage`) calcoli l'area come al solito, e poi limitarne tu la dimensione con `.clamp(...)` prima di usarla, visto che i campi di `Rect` sono pubblicamente accessibili:
>
> ```rust
> let [sidebar, main] = Layout::horizontal([
>     Constraint::Percentage(30),
>     Constraint::Fill(1),
> ])
> .areas(frame.area());
>
> // vincolo il risultato tra 20 e 40 celle
> let sidebar = Rect {
>     width: sidebar.width.clamp(20, 40),
>     ..sidebar
> };
> ```

### Un terzo pezzo: intestazione, corpo, piè di pagina

Aggiungere una terza zona è questione di aggiungere un vincolo e una variabile in più:

```rust
let [header, body, footer] = Layout::vertical([
    Constraint::Length(3),
    Constraint::Fill(1),
    Constraint::Length(3),
])
.areas(frame.area());

let block_green = Block::default().style(Style::default().bg(Color::LightGreen));
let block_blue = Block::default().style(Style::default().bg(Color::LightBlue));
let block_yellow = Block::default().style(Style::default().bg(Color::LightYellow));

frame.render_widget(
    Paragraph::new("La mia app da terminale").centered().block(block_green),
    header,
);
frame.render_widget(
    Paragraph::new("Qui in mezzo, in futuro, ci finirà il contenuto vero della schermata.").block(block_blue),
    body,
);
frame.render_widget(Paragraph::new("piè di pagina").block(block_yellow), footer);
```

A schermo: intestazione fissa sopra, piè di pagina fisso sotto, corpo che occupa tutto lo spazio in mezzo — qualunque sia la dimensione del terminale. Con i tre colori è immediato vedere che le tre aree, messe una sotto l'altra, riempiono esattamente lo schermo senza sovrapposizioni né buchi.

### Layout innestati: dividere anche il piè di pagina

`footer` è un `Rect` come un altro: niente vieta di passarlo a un secondo `Layout` e spezzarlo ulteriormente, stavolta in orizzontale. Proviamo a metterci tre colonne uguali — **al posto** del singolo `Paragraph::new("piè di pagina")` di prima: togli quella riga e sostituiscila con questa:

```rust
let [col1, col2, col3] = Layout::horizontal([
    Constraint::Fill(1),
    Constraint::Fill(1),
    Constraint::Fill(1),
])
.areas(footer);

let block_magenta = Block::default().style(Style::default().bg(Color::LightMagenta));
let block_cyan = Block::default().style(Style::default().bg(Color::LightCyan));
let block_gray = Block::default().style(Style::default().bg(Color::Gray));

frame.render_widget(Paragraph::new("Nuovo").centered().block(block_magenta), col1);
frame.render_widget(Paragraph::new("Salva").centered().block(block_cyan), col2);
frame.render_widget(Paragraph::new("Esci").centered().block(block_gray), col3);
```

Con tre `Fill(1)` e tre colori diversi, vedi subito che le colonne sono davvero uguali e coprono tutta la larghezza del piè di pagina, senza spazio vuoto tra l'una e l'altra.

Il numero dentro `Fill(1)` non è una larghezza in celle — è un **peso**, lo stesso concetto di `flex-grow` in CSS per chi lo conosce. `Fill` non ha una dimensione propria: prende semplicemente una fetta dello spazio in eccesso rimasto dopo aver soddisfatto tutti gli altri vincoli, proporzionale al proprio peso rispetto agli altri `Fill` presenti nello stesso `Layout`. Con tre `Fill(1)` i pesi sono uguali, quindi lo spazio si divide in tre parti uguali — è il caso più comune, ma non l'unico possibile.

Se invece vuoi che ad esempio la colonna sia più larga delle altre, alzi il suo peso:

```rust
let [col1, col2, col3] = Layout::horizontal([
    Constraint::Fill(1),
    Constraint::Fill(2),
    Constraint::Fill(1),
])
.areas(footer);
```

Qui `col2` riceve il doppio dello spazio di `col1` e `col3` (che restano uguali tra loro): su una riga di 40 celle, ad esempio, otterresti circa 10, 20 e 10 celle. Il peso conta solo *relativamente* agli altri `Fill` — `Fill(1)` e `Fill(2)` insieme si comportano esattamente come `Fill(10)` e `Fill(20)`, cambia solo la scala interna, non il risultato finale.

Torniamo al caso delle tre parti uguali. Cosa succede, invece, se vuoi che i tre pulsanti abbiano una larghezza fissa (diciamo, abbastanza per contenere il testo) e non si allarghino a riempire tutta la riga? Lì `Fill` non è più lo strumento giusto — serve un vincolo che non cresca, ed è qui che entra in gioco `Flex`.

### Quando i vincoli non riempiono tutto lo spazio: `Flex`

Cambiamo i vincoli dei tre pulsanti da `Fill(1)` a `Length(10)`: ognuno vuole essere largo esattamente 10 celle, non una in più.

```rust
let [b1, b2, b3] = Layout::horizontal([
    Constraint::Length(10),
    Constraint::Length(10),
    Constraint::Length(10),
])
.areas(footer);
```

(qui, come nei punti precedenti, uso `.areas(footer)` con la destrutturazione `let [b1, b2, b3] = ...`: sappiamo già che i pulsanti sono tre, quindi è la forma più comoda. Se invece il numero non fosse noto a priori — ad esempio pulsanti generati da una `Vec` costruita a runtime — `.split(footer)` resta l'alternativa: fa la stessa cosa ma restituisce uno slice indicizzabile, ad esempio `buttons[0]`, `buttons[1]`, ..., senza bisogno di sapere in anticipo quanti elementi aspettarti.)

Se lo schermo è più largo di 30 celle, adesso c'è dello spazio "avanzato" che nessun vincolo reclama. Di default, `Layout` usa `Flex::Start`: lo spazio in eccesso resta tutto alla fine, quindi i tre pulsanti restano appiccicati a sinistra.

`Flex` è il metodo che controlla proprio questo comportamento — dove va a finire lo spazio che i vincoli non hanno consumato. Si applica con `.flex(...)` sul `Layout`, prima di `.areas(...)` o `.split(...)`:

```rust
use ratatui::layout::Flex;

let [b1, b2, b3] = Layout::horizontal([
    Constraint::Length(10),
    Constraint::Length(10),
    Constraint::Length(10),
])
.flex(Flex::SpaceBetween)
.areas(footer);

frame.render_widget(Paragraph::new("Nuovo").centered().block(block_magenta), b1);
frame.render_widget(Paragraph::new("Salva").centered().block(block_cyan), b2);
frame.render_widget(Paragraph::new("Esci").centered().block(block_gray), b3);
```

Con `Flex::SpaceBetween` lo spazio in eccesso viene distribuito **tra** i tre pulsanti, che quindi si spargono su tutta la larghezza del piè di pagina invece di restare ammassati a sinistra — esattamente il risultato mostrato nel mockup di apertura. Con i pulsanti colorati, questo spazio "avanzato" non è più un concetto astratto: è letteralmente lo sfondo del terminale (senza colore) che si vede tra un blocco e l'altro. Prova a cambiare `Flex::SpaceBetween` con le altre varianti dell'elenco qui sotto — guardando dove si sposta il colore rispetto allo sfondo vuoto, la differenza tra `Start`, `Center`, `SpaceAround` e le altre si vede molto più chiaramente che leggendola a parole.

Le varianti disponibili in `Flex`:

| Variante | Dove va lo spazio in eccesso |
|---|---|
| `Flex::Start` (default) | tutto alla fine, gli elementi restano appiccicati all'inizio |
| `Flex::End` | tutto all'inizio, gli elementi restano appiccicati alla fine |
| `Flex::Center` | diviso equamente ai due lati, gli elementi si accentrano |
| `Flex::SpaceBetween` | distribuito tra un elemento e l'altro, niente ai bordi esterni |
| `Flex::SpaceAround` | distribuito attorno a ogni elemento; i bordi esterni ne ricevono la metà rispetto allo spazio tra due elementi adiacenti |
| `Flex::SpaceEvenly` | distribuito in parti identiche ovunque, bordi esterni compresi |
| `Flex::Legacy` | comportamento storico (versioni precedenti alla 0.26): tutto lo spazio finisce nell'ultimo elemento, che si allarga |

Il modo più diretto per farsene un'idea è provarle tutte sullo stesso `footer`: cambia solo l'argomento di `.flex(...)`, tutto il resto del codice resta identico. Ecco come si comporterebbero, applicate ai nostri tre pulsanti (`[Nuovo]`, `[Salva]`, `[Esci]`) dentro un piè di pagina largo, con lo spazio libero segnato da `·` (i conteggi sono indicativi, per dare l'idea delle proporzioni):

```
Start        [Nuovo][Salva][Esci]······························
End          ······························[Nuovo][Salva][Esci]
Center       ···············[Nuovo][Salva][Esci]···············
SpaceBetween [Nuovo]···············[Salva]···············[Esci]
SpaceAround  ·····[Nuovo]··········[Salva]··········[Esci]·····
SpaceEvenly  ·······[Nuovo]·······[Salva]·······[Esci]·········
Legacy       [Nuovo][Salva][Esci------------------------------]
```

Un paio di cose saltano all'occhio guardando i diagrammi fianco a fianco:

- `SpaceAround` lascia ai due pulsanti esterni **la metà** dello spazio che lascia tra un pulsante e l'altro — non è simmetrico come sembrerebbe dal nome.
- `SpaceEvenly` invece tratta tutti gli spazi allo stesso modo, bordi compresi: è la versione "veramente" equidistribuita.
- `Legacy` è l'unico caso in cui lo spazio non è un vuoto tra i pulsanti, ma finisce **dentro** l'ultimo elemento, allargandolo (per questo nel diagramma compare come parte del blocco `[Esci...]` e non come `·`): è il comportamento che avevano le versioni di Ratatui precedenti alla 0.26, mantenuto per compatibilità.

Da qui la scelta pratica: se ti serve un menu con i pulsanti sparsi su tutta la riga, `SpaceBetween` o `SpaceEvenly`; se ti serve un gruppo di pulsanti accentrato, `Center`; se non ti interessa la posizione dello spazio avanzato (ad esempio perché i vincoli riempiono già tutto con `Fill`), non serve nemmeno pensarci — è il caso del punto 3.

> **Attenzione: `Flex` e `Fill` non convivono bene.** Se anche uno solo dei vincoli è `Fill`, `Flex` smette di avere un effetto visibile: `Fill` si allarga a consumare tutto lo spazio libero *prima* che la logica di `Flex` entri in gioco, quindi non resta nulla da distribuire. `Flex` ha margine di manovra solo quando *tutti* i vincoli sono "rigidi" (`Length`, `Percentage`, `Fixed`...): è per questo che nell'esempio dei pulsanti sopra ho usato tre `Length(10)` e non un `Fill`.

### Cosa `Layout` (e `Flex`) non fanno

- **Non gestiscono la sovrapposizione.** Ogni `split`/`areas` produce aree che non si sovrappongono mai tra loro: se ti serve un elemento sopra un altro (un popup, un tooltip), `Layout` da solo non basta — serve calcolare a mano il `Rect` del popup, o usare `Rect::centered(...)` e simili, argomento per un articolo più avanti.
- **Non è "responsive" nel senso di adattare automaticamente la struttura.** Se lo schermo diventa troppo stretto per i tuoi `Length(10)`, `Layout` fa comunque del suo meglio a soddisfare i vincoli (magari sacrificando in parte alcuni di essi), ma non passa da solo, ad esempio, da "tre colonne" a "una colonna sola": quella logica, se ti serve, la scrivi tu leggendo la larghezza dell'area e scegliendo i vincoli di conseguenza — con la stessa logica dei breakpoint di un sito web quando passa da desktop a tablet a cellulare: leggi la larghezza dell'area che hai a disposizione (`frame.area().width`) e, con un normale `if`, scegli tu quale array di vincoli costruire — uno per lo schermo largo, uno per quello stretto. Non è automatico come nel web, ma il concetto è lo stesso: una soglia di larghezza sotto la quale cambi disposizione. Ci torneremo con un esempio vero più avanti nella serie.
- **`Flex::SpaceAround`, `SpaceBetween` e `SpaceEvenly` ignorano `.spacing(...)`.** Se hai bisogno di uno spazio minimo garantito tra gli elementi oltre a quello distribuito automaticamente, va aggiunto ai vincoli stessi (ad esempio inserendo un `Constraint::Length(n)` fittizio tra un elemento e l'altro), non tramite `.spacing(...)`.

> **Per approfondire.** `Layout` internamente usa un risolutore di vincoli (l'algoritmo Cassowary, tramite il crate `kasuari`), lo stesso genere di motore che sta dietro a molti sistemi di layout dichiarativi. Non serve saperlo per usare l'API pubblica, ma se l'argomento ti incuriosisce: [Cassowary constraint solver](https://en.wikipedia.org/wiki/Cassowary_(software)).

## Codice completo finale

I colori usati sopra erano solo un aiuto per vedere a schermo i confini delle aree mentre imparavi a usare `Layout` e `Flex` — non fanno parte dell'app "vera". Il codice completo qui sotto torna alla versione senza sfondi colorati, la stessa mostrata nel mockup di apertura; se in futuro ti serve rimettere i `Block` colorati per capire come si dispone un layout più complesso, il trucco resta lo stesso di sopra.

```rust
use ratatui::crossterm::event;
use ratatui::layout::{Constraint, Flex, Layout};
use ratatui::style::Stylize;
use ratatui::widgets::Paragraph;

fn main() -> std::io::Result<()> {
    ratatui::run(|terminal| {
        loop {
            terminal.draw(|frame| {
                let [header, body, footer] = Layout::vertical([
                    Constraint::Length(3),
                    Constraint::Fill(1),
                    Constraint::Length(3),
                ])
                .areas(frame.area());

                frame.render_widget(
                    Paragraph::new("La mia app da terminale").centered().bold(),
                    header,
                );

                frame.render_widget(
                    Paragraph::new(
                        "Qui in mezzo, in futuro, ci finirà il contenuto vero della schermata.",
                    ),
                    body,
                );

                let [b1, b2, b3] = Layout::horizontal([
                    Constraint::Length(10),
                    Constraint::Length(10),
                    Constraint::Length(10),
                ])
                .flex(Flex::SpaceBetween)
                .areas(footer);

                frame.render_widget(Paragraph::new("[ Nuovo ]").centered(), b1);
                frame.render_widget(Paragraph::new("[ Salva ]").centered(), b2);
                frame.render_widget(Paragraph::new("[ Esci ]").centered().yellow(), b3);
            })?;

            if event::read()?.is_key_press() {
                break Ok(());
            }
        }
    })
}
```

`Cargo.toml` invariato rispetto ai due articoli precedenti:

```toml
[package]
name = "hello-ratatui"
version = "0.1.0"
edition = "2024"

[dependencies]
ratatui = "0.30"
```

