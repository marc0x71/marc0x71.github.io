+++
title = "Ratatui passo dopo passo - Block: bordi, titoli e padding intorno ai widget"
date = 2026-09-28
description = "Impariamo a usare `Block` in Ratatui per aggiungere bordi, titoli, padding e stili ai widget, trasformando il layout in una vera interfaccia da terminale."
[taxonomies]
tags = ["rust", "ratatui"]
[extra]
comments = true
+++

## Cosa costruiamo oggi

Finora ogni area del layout era un rettangolo invisibile: sapevi dov'era solo per come vi era posizionato il testo dentro, o (nell'articolo scorso) per uno sfondo colorato messo lì apposta per debug. Oggi diamo a quelle aree un aspetto vero da applicazione da terminale: un bordo, un titolo, un po' di respiro tra il testo e il bordo. È lo stesso layout header/corpo/piè di pagina dell'articolo precedente, ma questa volta si vede davvero.

```
╭ La mia app da terminale ──────────────────╮
│ Contenuto ─────────────────── F1: aiuto   │
│                                           │
│   Qui in mezzo, in futuro, ci finirà il   │
│   contenuto vero della schermata.         │
│                                           │
╰───────────────────────────────────────────╯
┌ [ Nuovo ] ┐     ┌ [ Salva ] ┐  ┌ [ Esci ] ┐
└───────────┘     └───────────┘  └──────────┘
```

## Prerequisiti

Partiamo dal codice completo del terzo articolo: il progetto `hello-ratatui`, `Cargo.toml` invariato e questo `main.rs`:

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

Da qui in avanti tocchiamo solo il contenuto di `terminal.draw(...)`.

## Codice, passo per passo

### Il bordo più semplice

`Block` è un widget come un altro — la differenza è che di solito non lo renderizzi da solo, ma lo passi a un altro widget con `.block(...)`, così come abbiamo già fatto nell'articolo scorso per dargli uno sfondo. Ora usiamolo per il suo scopo principale: il bordo. Applichiamolo al corpo, al posto del `Paragraph` senza bordo di prima:

```rust
use ratatui::widgets::{Block, Paragraph};

frame.render_widget(
    Paragraph::new("Qui in mezzo, in futuro, ci finirà il contenuto vero della schermata.")
        .block(Block::bordered()),
    body,
);
```

`Block::bordered()` è la scorciatoia per "tutti e quattro i lati" — equivale a `Block::new().borders(Borders::ALL)`. Se ti serve un bordo solo su alcuni lati, `Borders` è un insieme di flag che puoi combinare con `|`: 

```rust
Block::new().borders(Borders::TOP | Borders::LEFT)
```

Disegna solo il bordo superiore e quello sinistro, lasciando gli altri due lati aperti.

A questo punto, se lanci `cargo run`, il corpo ha un riquadro attorno, e il testo si è spostato di una cella verso l'interno su ogni lato: `Paragraph` calcola da solo lo spazio "utile" dentro al bordo (quello che l'API chiama l'area interna del `Block`) e ci disegna il contenuto, senza che tu debba fare i conti a mano.

> **Per approfondire.** Questo funziona in automatico perché `Paragraph` — come la maggior parte dei widget di Ratatui — accetta un `Block` opzionale e sa da solo come renderizzarsi nell'area che resta dopo aver disegnato il bordo. Se in futuro ti capiterà di creare un widget tuo che non ha questa comodità integrata, il metodo da usare a mano è `Block::inner(area)`, che restituisce esattamente quel rettangolo interno, ma ne riparleremo.

### Un titolo

Aggiungere un titolo è un'unica chiamata in più:

```rust
frame.render_widget(
    Paragraph::new("Qui in mezzo, in futuro, ci finirà il contenuto vero della schermata.")
        .block(Block::bordered().title(" Contenuto ")),
    body,
);
```

Il titolo si aggancia al bordo superiore, allineato a sinistra di default. Nota gli spazi dentro le virgolette (`" Contenuto "`, non `"Contenuto"`): senza, il testo del titolo toccherebbe direttamente gli angoli del bordo — è solo una convenzione estetica diffusa nei progetti Ratatui, non un obbligo dell'API.

### Cambiare la forma del bordo

Il bordo che hai visto finora (`┌─┐`, angoli squadrati) è solo uno dei tipi disponibili, ed è quello di default. Si cambia con `.border_type(...)`:

```rust
use ratatui::widgets::BorderType;

Block::bordered()
    .border_type(BorderType::Rounded)
    .title(" Contenuto ");
```

I quattro che userai più spesso:

| `BorderType` | Aspetto |
|---|---|
| `Plain` (default) | `┌───┐` `│   │` `└───┘` |
| `Rounded` | `╭───╮` `│   │` `╰───╯` |
| `Double` | `╔═══╗` `║   ║` `╚═══╝` |
| `Thick` | `┏━━━┓` `┃   ┃` `┗━━━┛` |

Ne esistono altri più particolari (bordi tratteggiati, o disegnati con i caratteri "a quadrante" invece delle linee) — se ti capita di averne bisogno, sono elencati nella documentazione di [`BorderType`](https://docs.rs/ratatui-widgets/latest/ratatui_widgets/borders/enum.BorderType.html), ma per il 99% dei casi questi quattro bastano. `Rounded` è probabilmente quello che vedrai più spesso: è diventato una specie di standard estetico (non ufficiale) nelle app Ratatui.

### Più titoli, posizionati dove vuoi

Un `Block` può avere più di un titolo, ciascuno posizionato in un punto diverso — utile per un pattern molto comune: titolo dell'area in alto a sinistra, un suggerimento o una scorciatoia in basso a destra. In Ratatui si fa con `.title_top(...)` e `.title_bottom(...)`, passando una `Line` (con l'allineamento che già conosci da due articoli fa) invece di una semplice stringa:

```rust
use ratatui::text::Line;

Block::bordered()
    .title_top(Line::from(" Contenuto ").left_aligned())
    .title_bottom(Line::from(" F1: aiuto ").right_aligned())
```

Nota che `.title(...)` da solo (quello del punto precedente) resta valido e in pratica equivale a `.title_top(...)` — è la forma comoda per il caso più comune di un titolo solo, in alto. Appena ti servono più titoli, o uno in basso, passi a `title_top`/`title_bottom` con una `Line` esplicita.

### Un po' di respiro: il padding

Il testo del corpo, per ora, tocca il bordo sinistro appena entra nell'area interna. Un po' di margine aiuta la leggibilità:

```rust
use ratatui::widgets::Padding;

Block::bordered()
    .title(" Contenuto ")
    .padding(Padding::uniform(1))
```

`Padding::uniform(1)` aggiunge una cella di margine su tutti e quattro i lati, dentro al bordo. Se ti serve solo in orizzontale (lasciando il testo attaccato in verticale) c'è `Padding::horizontal(n)`, e il simmetrico `Padding::vertical(n)`; per un controllo completamente indipendente su ogni lato, `Padding::new(left, right, top, bottom)`.

### Colorare bordo e titolo separatamente dal contenuto

`.style(...)` che già conosci si applica anche a `Block` — ma colora *tutto* quello che sta dentro, bordo compreso, perché lo stile del blocco viene applicato per primo e poi ereditato da quello che ci disegni sopra. Se vuoi che il bordo abbia un colore diverso dal testo, usa `.border_style(...)`:

```rust
use ratatui::style::{Color, Style};

Block::bordered()
    .title(" Contenuto ")
    .border_style(Style::default().fg(Color::DarkGray))
```

Qui il bordo diventa grigio scuro, mentre il testo dentro resta del colore di default — un accorgimento che rende le aree meno invadenti visivamente senza nasconderne i confini. Spesso, assegnare un colore diverso ad un bordo di un block, è usato per indicare il "blocco" attualmente in focus.

> **Per approfondire.** `Style::default().fg(Color::DarkGray)` è il caso più semplice di `Style`: `.fg(...)` imposta il colore del testo (*foreground*), `.bg(...)` quello dello sfondo, e si combinano liberamente — `Style::default().fg(Color::Red).bg(Color::Yellow)` dà testo rosso su sfondo giallo. È lo stesso `Style` che hai già incrociato per lo sfondo dei `Block` di debug nell'articolo scorso, e le scorciatoie `.yellow()`, `.bold()` di `Stylize` viste nel secondo articolo non sono altro che `Style` precompilati dietro le quinte. C'è di più — come si comportano i modificatori (`.bold()`, `.italic()`, ecc.) e cosa succede quando più `Style` si sovrappongono sullo stesso testo — ma è materiale per l'articolo dedicato a stili e temi, più avanti nella serie.

## Codice completo finale

Mettendo insieme tutti i pezzi — bordo arrotondato ovunque, titolo in alto a sinistra e un suggerimento in basso a destra sul corpo, padding, bordo in un grigio discreto con un tocco di colore sul titolo dell'intestazione e su "Contenuto", e un piccolo bordo anche sui tre pulsanti del piè di pagina:

```rust
use ratatui::crossterm::event;
use ratatui::layout::{Constraint, Flex, Layout};
use ratatui::style::{Color, Style, Stylize};
use ratatui::text::Line;
use ratatui::widgets::{Block, BorderType, Padding, Paragraph};

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

                let border_style = Style::default().fg(Color::DarkGray);

                frame.render_widget(
                    Paragraph::new("La mia app da terminale")
                        .centered()
                        .bold()
                        .yellow()
                        .block(
                            Block::bordered()
                                .border_type(BorderType::Rounded)
                                .border_style(border_style),
                        ),
                    header,
                );

                frame.render_widget(
                    Paragraph::new(
                        "Qui in mezzo, in futuro, ci finirà il contenuto vero della schermata.",
                    )
                    .block(
                        Block::bordered()
                            .border_type(BorderType::Rounded)
                            .border_style(border_style)
                            .title_top(Line::from(" Contenuto ").left_aligned().light_cyan())
                            .title_bottom(Line::from(" F1: aiuto ").right_aligned())
                            .padding(Padding::uniform(1)),
                    ),
                    body,
                );

                let [b1, b2, b3] = Layout::horizontal([
                    Constraint::Length(11),
                    Constraint::Length(11),
                    Constraint::Length(11),
                ])
                .flex(Flex::SpaceBetween)
                .areas(footer);

                frame.render_widget(
                    Paragraph::new("Nuovo").centered().block(Block::bordered()),
                    b1,
                );
                frame.render_widget(
                    Paragraph::new("Salva").centered().block(Block::bordered()),
                    b2,
                );
                frame.render_widget(
                    Paragraph::new("Esci")
                        .centered()
                        .yellow()
                        .block(Block::bordered()),
                    b3,
                );
            })?;

            if event::read()?.is_key_press() {
                break Ok(());
            }
        }
    })
}
```

Ora sai dare a qualunque area un bordo (`Block::bordered()`, o `Borders` scelti a mano per bordi parziali), sceglierne la forma con `.border_type(...)`, aggiungere uno o più titoli posizionati con `.title_top(...)`/`.title_bottom(...)` e allineati con la stessa sintassi di `Line` che già conoscevi, aggiungere margine interno con `.padding(...)`, e colorare bordo e contenuto in modo indipendente con `.border_style(...)` contro `.style(...)`. Con questo, il layout che dividevi "sulla carta" da due articoli comincia finalmente ad avere l'aspetto di un'app da terminale vera.
