+++
title = "Tmux: Configurazione Minimale e Produttiva"
date = 2026-02-18
description = "Tre finestre. Nessun pannello. Zero plugin. Una per l’editor. Una per la compilazione continua. Una per l’esecuzione e i test. Questo è il mio setup quotidiano con tmux. In questo articolo ti mostro come costruire una configurazione minimale, portabile e produttiva partendo da zero."

draft=true

[taxonomies]
tags = ["tmux"]

[extra]
comments = true
+++

Tre finestre.

Nessun pannello.

Zero plugin.

Una per l’editor.

Una per la compilazione continua.

Una per l’esecuzione e i test.

Questo è il mio setup quotidiano con tmux.

In questo articolo ti mostro come costruire una configurazione minimale, portabile e produttiva partendo da zero.

## Introduzione

### Cos'è Tmux?

`tmux` è un **multiplexer di terminale** che ti permette di gestire più sessioni, finestre e pannelli in un’unica shell, anche da remoto, senza perdere nulla quando ti disconnetti

Immagina di poter:

- **Dividere** una singola finestra del terminale in più pannelli
- **Gestire** più progetti contemporaneamente
- **Disconnetterti** dal server senza perdere il lavoro in corso
- **Velocizzare** il tuo workflow con keybinding personalizzati

### Perché una Configurazione Minimale?

In questa guida adottiamo un approccio **plugin-free** per diversi motivi:

**Velocità**: Nessun overhead di caricamento plugin  
**Portabilità**: Funziona su qualsiasi sistema senza dipendenze  
**Manutenibilità**: Facile da comprendere  
**Apprendimento**: Capisci esattamente cosa fa ogni riga  

## Prerequisiti e Installazione

### Installazione Tmux

**Ubuntu/Debian:**

```bash
sudo apt update
sudo apt install tmux
```

**macOS (con Homebrew):**

```bash
brew install tmux
```

### Verifica Installazione

```bash
tmux -V
# Output atteso: tmux 3.5a o superiore
```


## Alcuni concetti fondamentali

Prima di iniziare, è fondamentale capire la **gerarchia** di tmux:

```
┌─────────────────────────────────────────────────┐
│  SESSIONE (session): "progetto-web"             │
│  ┌───────────────────────────────────────────┐  │
│  │  FINESTRA 1 (window): "editor"            │  │
│  │  ┌─────────────┬──────────────────────┐   │  │
│  │  │ PANNELLO 1  │   PANNELLO 2         │   │  │
│  │  │   (vim)     │   (terminale)        │   │  │
│  │  └─────────────┴──────────────────────┘   │  │
│  └───────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────┐  │
│  │  FINESTRA 2 (window): "server"            │  │
│  │  ┌──────────────────────────────────────┐ │  │
│  │  │          PANNELLO 1                  │ │  │
│  │  │          (server logs)               │ │  │
│  │  └──────────────────────────────────────┘ │  │
│  └───────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

Una *Sessione* in tmux è un contesto di lavoro, potrebbe ad esempio essere un intero progetto. Potremmo cioè avere una sessione per il nostro programma a cui stiamo lavorando, una usata per aggiornare il nostro blog o magari un'altra dove lavoriamo alla configurazione del nostro NeoVim.

Una *finestra* invece è, all'interno di una sessione, uno schermo virtuale, come una scheda del nostro browser preferito, un *tab* per intenderci.

Un *pannello* è una divisione all'interno di una finestra

|Termine|Descrizione|Analogia|
|---|---|---|
|**Session**|Contesto di lavoro completo|Un progetto intero|
|**Window**|Schermo virtuale all'interno di una sessione|Una scheda del browser|
|**Pane**|Divisione di una window|Split screen|

### Il Prefix Key

Il **prefix** è la "chiave magica" che attiva i comandi tmux. Di default è `Ctrl+b`, ma nella nostra configurazione lo cambieremo in `Ctrl+Space` per ergonomia.

**Come funziona:**

1. Premi `Ctrl+Space` (il prefix)
2. Rilascia
3. Premi il comando (es. `c` per new window)

⚠️ **Nota:** in alcuni terminali o layout di tastiera `Ctrl+Space` potrebbe non funzionare correttamente. In tal caso `Ctrl+a` è un’ottima alternativa con supporto universale.

## Setup Iniziale

Per prima cosa dobbiamo creare il nostro file di configurazione:

```bash
touch ~/.tmux.conf
```

quindi, usando il nostro editor preferito (inutile credo dirvi quale sia il mio 😀), andiamo ad aggiungere passo-passo delle configurazioni che dettaglierò riga per riga

### Impostazioni Generali

#### Cambio del Prefix Key

```bash
unbind C-b
set -g prefix C-Space
bind C-Space send-prefix
```

**Cosa fa:**

- `unbind C-b`: Rimuove il binding di default `Ctrl+b`
- `set -g prefix C-Space`: Imposta `Ctrl+Space` come nuovo prefix
- `bind C-Space send-prefix`: Permette di inviare `Ctrl+Space` all'applicazione sottostante (premendolo due volte)

**Perché:**

- `Ctrl+Space` è **più ergonomico**: si preme con il pollice invece del mignolo
- Non interferisce con comandi comuni come `Ctrl+b` in Vim (page up)
- È una combinazione meno usata rispetto a `Ctrl+b`

	Alcuni preferiscono `Ctrl+a`, io preferisco lo spazio, ma è solo questione di trovare la combinazione più pratica per noi e che non ci crei problemi coi software che solitamente utilizziamo nel nostro workflow da terminale.

**Uso pratico:**

```bash
# Creare una nuova finestra:
Ctrl+Space, poi c

# Se un'applicazione ha bisogno di Ctrl+Space:
Ctrl+Space, Ctrl+Space
```

---

#### Ricarica Configurazione

```bash
bind r source-file ~/.tmux.conf \; display "Configurazione ricaricata!"
```

**Cosa fa:** Crea un keybinding per ricaricare il file di configurazione senza uscire da tmux

**Perché:** Quando modifichi `~/.tmux.conf`, invece di chiudere e riaprire tmux, puoi ricaricare al volo le modifiche

**Uso pratico:**

```bash
# 1. Modifica ~/.tmux.conf con il tuo editor
vim ~/.tmux.conf

# 2. Salva le modifiche

# 3. In tmux, premi:
Ctrl+Space, poi r

# Vedrai: "Configurazione ricaricata!"
```

---

#### Supporto Mouse

```bash
set -g mouse on
```

**Cosa fa:** Abilita completamente il supporto del mouse in tmux

**Funzionalità abilitate:**

- **Clicca** su un pannello per attivarlo
- **Trascina** i bordi tra pannelli per ridimensionare
- **Clicca** su una finestra nella status bar per selezionarla
- **Scroll** con la rotella per navigare la history
- **Seleziona** testo con il mouse e copia

**Nota:** Con il mouse attivo, per selezionare testo e copiarlo nella clipboard di sistema, potrebbe essere necessario tenere premuto `Shift` mentre selezioni

**Uso pratico:**

```bash
# Scenario: hai 4 pannelli aperti
# Invece di usare Ctrl+hjkl per navigare, 
# semplicemente clicca sul pannello desiderato
```

---

#### History Limit

```bash
set -g history-limit 20000
```

**Cosa fa:** Imposta a 20.000 il numero di righe di output memorizzate per ogni pannello

**Perché:**

- **Default**: solo 2.000 righe
- **20.000**: permette di risalire a output molto precedenti
- Utile per analizzare log lunghi o ritrovare output passati

**Uso pratico:**

```bash
# Dopo aver eseguito comandi con molto output:
Ctrl+Space, poi [        # Entra in copy mode
# Ora puoi scorrere all'indietro per 20.000 righe
# Usa q per uscire
```

**📝 Consumo memoria:** 20.000 righe sono circa 2-3 MB per pannello (dipende dalla lunghezza delle righe)

---

#### Numerazione da 1

```bash
set -g base-index 1
setw -g pane-base-index 1
```

**Cosa fa:** Fa iniziare la numerazione di finestre e pannelli da **1** invece che da **0** come da impostazione predefinita di tmux

**Perché:**

```
Tastiera:  [1] [2] [3] [4] [5] [6] [7] [8] [9] [0]
Con 0:      ^                               prima finestra (scomodo!)
Con 1:      ^                               prima finestra (comodo!)
```

I tasti 1-9 sono più facili da raggiungere rispetto a dover arrivare a 0 per la "prima" finestra

**Uso pratico:**

```bash
Ctrl+Space, poi 1    # Vai alla PRIMA finestra (non alla seconda!)
```

---

#### Rinumerazione Automatica

```bash
set -g renumber-windows on
```

**Cosa fa:** Quando chiudi una finestra, le successive vengono rinumerate per mantenere la sequenza

**Esempio:**

```
Situazione iniziale:  [1:vim] [2:server] [3:logs] [4:test]

Chiudi finestra 2 (server)

❌ SENZA renumber:    [1:vim] [3:logs] [4:test]     # Numerazione con buchi!
✅ CON renumber:      [1:vim] [2:logs] [3:test]     # Sempre sequenziale!
```

**Vantaggio:** Non devi "saltare" numeri quando switchi tra finestre, mantieni sempre una sequenza 1-2-3-4

---

#### True Color Support

```bash
set -ga terminal-overrides ",xterm-256color:Tc"
```

**Cosa fa:** Abilita il supporto per **true color** (16 milioni di colori) invece dei soli 256 colori

**Perché:**

- Temi moderni di Vim/Neovim richiedono true color
- Gradienti e sfumature nei colori
- Sintassi highlighting più ricco

**Dettagli tecnici:**

- `-ga`: append (aggiunge invece di sovrascrivere)
- `Tc`: capability per true color
- `xterm-256color`: terminal type

**Prerequisito:** Il tuo terminale deve supportare true color (iTerm2, Alacritty, Kitty, terminali moderni)

---

### Divisione Finestre

#### Split Orizzontale e Verticale

```bash
bind | split-window -h -c "#{pane_current_path}"
bind - split-window -v -c "#{pane_current_path}"
```

**Cosa fa:**

- `|` (pipe): divide la finestra **orizzontalmente** (crea colonne)
- `-` (trattino): divide la finestra **verticalmente** (crea righe)
- `-c "#{pane_current_path}"`: il nuovo pannello parte dalla stessa directory

**Visualizzazione:**

```
Split ORIZZONTALE (|):       Split VERTICALE (-):
┌──────────┬──────────┐      ┌─────────────────────┐
│          │          │      │                     │
│  Pane 1  │  Pane 2  │      │      Pane 1         │
│  (vim)   │ (shell)  │      │      (vim)          │
│          │          │      ├─────────────────────┤
└──────────┴──────────┘      │      Pane 2         │
                             │     (shell)         │
                             └─────────────────────┘
```

**Uso pratico:**

```bash
# Stai editando codice e vuoi un terminale a destra
Ctrl+Space, poi |

# Stai editando e vuoi vedere i log sotto
Ctrl+Space, poi -
```

**Mnemonic:**

- `|` assomiglia a una linea verticale → split orizzontale
- `-` assomiglia a una linea orizzontale → split verticale

**Senza `-c "#{pane_current_path}"`:** Il nuovo pannello partirebbe dalla home directory, costringendoti a navigare alla directory corrente

📝 Di default tmux consente lo split usando `%` e `"`, questa configurazione non li sovrascrive, resteranno comunque utilizzabili

---

#### Nuova Finestra nella Directory Corrente

```bash
bind c new-window -c "#{pane_current_path}"
```

**Cosa fa:** Quando crei una nuova finestra con `c`, parte dalla stessa directory del pannello corrente

**Esempio:**

```bash
# Sei in: /home/user/progetti/app/src
Ctrl+Space, poi c
# La nuova finestra inizia in: /home/user/progetti/app/src
# (non in /home/user come farebbe di default)
```

**Beneficio:** Non devi fare `cd` ogni volta che apri una nuova finestra per lavorare sullo stesso progetto

---

### Navigazione

#### Navigazione tra Pannelli (Vim Style)

```bash
bind -n C-h select-pane -L
bind -n C-l select-pane -R
bind -n C-k select-pane -U
bind -n C-j select-pane -D
```

**Cosa fa:** Naviga tra i pannelli usando `Ctrl+hjkl` **senza premere il prefix**

**Layout Vim:**

```
        k (su)
        ↑
h (sx) ←+→ l (dx)
        ↓
        j (giù)
```

**Uso pratico:**

```bash
# Hai 4 pannelli aperti
┌─────┬─────┐
│  1  │  2  │
├─────┼─────┤
│  3  │  4  │
└─────┴─────┘

# Sei nel pannello 1, vuoi andare a destra (2):
Ctrl+l    # Nessun prefix!

# Dal pannello 2, vuoi andare giù (4):
Ctrl+j
```

**⚠️ Conflitto Potenziale:**

Questi binding sovrascrivono alcuni comandi standard:

- `Ctrl+l`: clear screen in bash/zsh
- `Ctrl+h`: backspace in alcuni terminali

**Se non vuoi avere conflitti:**

```bash
# Versione con prefix (richiede Ctrl+Space prima)
bind h select-pane -L
bind l select-pane -R
bind k select-pane -U
bind j select-pane -D
```

**Alternativa per clear:** Se perdi `Ctrl+l` per clear, puoi:

- Usare il comando `clear` o `reset`
- Creare un alias: `alias cls='clear'`
- Usare `Ctrl+Space` poi `Ctrl+l` (invia il comando all'applicazione)

---

#### Navigazione tra Finestre

```bash
bind -n S-Left previous-window
bind -n S-Right next-window
```

**Cosa fa:**

- `Shift+Freccia Sinistra`: vai alla finestra precedente
- `Shift+Freccia Destra`: vai alla finestra successiva

**Uso pratico:**

```bash
# Hai 3 finestre: [1:vim] [2:server] [3:logs]
# Sei nella finestra 2 (server)

Shift+Freccia Sinistra   # Vai a [1:vim]
Shift+Freccia Destra     # Torni a [2:server]
Shift+Freccia Destra     # Vai a [3:logs]
```

**Alternative built-in:**

```bash
Ctrl+Space, poi n     # next window
Ctrl+Space, poi p     # previous window
Ctrl+Space, poi 1     # vai alla finestra 1
Ctrl+Space, poi 2     # vai alla finestra 2
# ... etc
```

---

### Ridimensionamento Pannelli

```bash
bind -n M-Left resize-pane -L 3
bind -n M-Right resize-pane -R 3
bind -n M-Up resize-pane -U 3
bind -n M-Down resize-pane -D 3
```

**Cosa fa:** Ridimensiona il pannello corrente usando `Alt+Frecce`, 3 celle alla volta

**Nota:** `M` = Meta (tipicamente il tasto `Alt` o `Option` su Mac)

**Uso pratico:**

```bash
# Scenario: hai un editor a sinistra e un terminale a destra
┌────────────┬──────┐
│            │      │
│   Editor   │ Term │
│            │      │
└────────────┴──────┘

# Vuoi dare più spazio all'editor:
Alt+Freccia Destra (più volte)

┌──────────────────┬──┐
│                  │T │
│     Editor       │e │
│                  │r │
└──────────────────┴──┘
```

**Personalizzazione:** Se 3 celle sono troppe/poche, modifica il numero:

```bash
bind -n M-Left resize-pane -L 5   # 5 celle alla volta
```

**Alternative con prefix (e repeat):**

```bash
# Con -r (repeat) puoi tenere premuto il tasto
bind -r H resize-pane -L 5
bind -r J resize-pane -D 5
bind -r K resize-pane -U 5
bind -r L resize-pane -R 5
```

---
### Sincronizzazione Pannelli 

```bash
bind C-y set-window-option synchronize-panes\; display-message "Sync: #{?pane_synchronized,ON,OFF}"
``` 

**Cosa fa:** Attiva/disattiva la sincronizzazione dell'input su tutti i pannelli della finestra corrente 

**Quando è utile:** 
- Eseguire lo stesso comando su più server SSH contemporaneamente 
- Aggiornare contemporaneamente più server
- Configurare simultaneamente più macchine

**Uso pratico:** 

```bash
# Scenario: amministrazione multi-server 

# 1. Crea 3 pannelli con SSH a server diversi 
Ctrl+Space |         # Split 
ssh user@server1.com 
Ctrl+Space |         # Split ancora
ssh user@server2.com 
Ctrl+Space -         # Split verticale 
ssh user@server3.com 

# 2. Attiva sincronizzazione Ctrl+Space Ctrl+y 

# 3. Digita un comando - viene eseguito su TUTTI i server! 

sudo apt update 
sudo apt upgrade -y 

# 4. Disattiva sincronizzazione quando hai finito Ctrl+Space Ctrl+y 
```


### Copy Mode

#### Modalità Vi

```bash
setw -g mode-keys vi
```

**Cosa fa:** Abilita i keybinding di Vi/Vim nella copy mode (invece dei binding di default Emacs)

**Navigazione in Copy Mode:**

|Tasto|Azione|
|---|---|
|`h`|sinistra|
|`j`|giù|
|`k`|su|
|`l`|destra|
|`w`|parola successiva|
|`b`|parola precedente|
|`0`|inizio riga|
|`$`|fine riga|
|`g`|inizio documento|
|`G`|fine documento|
|`/`|ricerca|
|`n`|prossima occorrenza|
|`N`|occorrenza precedente|

---

#### Selezione e Copia

```bash
bind -T copy-mode-vi v send-keys -X begin-selection
bind -T copy-mode-vi y send-keys -X copy-selection-and-cancel
```

**Cosa fa:**

- `v`: inizia selezione visuale (come in Vim)
- `y`: "yank" (copia) la selezione e esce dalla copy mode

**Workflow completo:**

```bash
# PASSO 1: Entra in copy mode
Ctrl+Space, poi [

# PASSO 2: Naviga fino al testo che vuoi copiare
# (usa hjkl, o frecce, o ricerca con /)

# PASSO 3: Inizia selezione
v

# PASSO 4: Estendi selezione (muoviti con hjkl)
# Il testo selezionato sarà evidenziato

# PASSO 5: Copia
y

# PASSO 6: Incolla (in un altro pannello)
Ctrl+Space, poi ]
```

---

#### Integrazione con Clipboard di Sistema

La configurazione base copia nel buffer interno di tmux. Per copiare nella clipboard di sistema:

**Linux (con xclip):**

```bash
# Installa xclip
sudo apt install xclip

# Aggiungi a ~/.tmux.conf
bind -T copy-mode-vi y send-keys -X copy-pipe-and-cancel "xclip -in -selection clipboard"
```

**macOS:**

```bash
# Usa pbcopy (già presente)
bind -T copy-mode-vi y send-keys -X copy-pipe-and-cancel "pbcopy"
```

**Con selezione mouse:**

```bash
# Linux
bind -T copy-mode-vi MouseDragEnd1Pane send-keys -X copy-pipe-and-cancel "xclip -in -selection clipboard"

# macOS
bind -T copy-mode-vi MouseDragEnd1Pane send-keys -X copy-pipe-and-cancel "pbcopy"
```

---

### Barra di Stato

#### Intervallo di Aggiornamento

```bash
set -g status-interval 5
```

**Cosa fa:** Aggiorna la barra di stato ogni 5 secondi

**Perché:**

- Mantiene l'orologio aggiornato
- Se aggiungi script custom (es. battery, CPU), vengono aggiornati
- 5 secondi è un buon compromesso tra aggiornamento e performance

---

#### Posizione Barra

```bash
set -g status-position bottom
```

**Opzioni:**

- `bottom`: in basso (default e più comune)
- `top`: in alto

**Quando usare `top`:** Se usi un terminale dropdown (es. Guake, Yakuake) che appare dall'alto, la status bar in alto è più naturale

---

#### Contenuto Barra Sinistra

```bash
set -g status-left '<#S> #[default] '
```

**Cosa mostra:**

- `#S`: nome della sessione
- Esempio: `<progetto-web>`

**Variabili utili:**

|Variabile|Mostra|
|---|---|
|`#S`|Nome sessione|
|`#I`|Indice finestra corrente|
|`#P`|Indice pannello corrente|
|`#W`|Nome finestra corrente|
|`#H`|Hostname completo|
|`#h`|Hostname (solo nome)|
|`#(comando)`|Output di un comando|

**🔧 Esempi personalizzati:**

```bash
# Con hostname
set -g status-left '#h <#S> | '

# Con icone (richiede font con simboli)
set -g status-left '  #S | '

# Con info utente
set -g status-left '#(whoami)@#h <#S> | '
```

---

#### Contenuto Barra Destra

```bash
set -g status-right '%d/%m/%Y %H:%M '
```

**Cosa mostra:** Data e ora in formato italiano: `08/02/2026 15:30`

**Formati data/ora:**

|Codice|Output|Esempio|
|---|---|---|
|`%d`|Giorno|08|
|`%m`|Mese|02|
|`%Y`|Anno (4 cifre)|2026|
|`%H`|Ora (24h)|15|
|`%I`|Ora (12h)|03|
|`%M`|Minuti|30|
|`%S`|Secondi|45|
|`%p`|AM/PM|PM|
|`%A`|Giorno settimana|Domenica|
|`%B`|Nome mese|Febbraio|

**Esempi personalizzati:**

```bash
# Formato USA
set -g status-right '%m/%d/%Y %I:%M %p'
# Output: 02/08/2026 03:30 PM

# Con giorno della settimana
set -g status-right '%A %d/%m/%Y %H:%M'
# Output: Domenica 08/02/2026 15:30

# Con hostname
set -g status-right '#H | %d/%m/%Y %H:%M'
# Output: mio-server | 08/02/2026 15:30

# Con batteria (richiede script personalizzato)
set -g status-right '🔋 #(battery.sh) | %H:%M'
# Output: 🔋 87% | 15:30
```

---

#### Formato Finestre

```bash
setw -g window-status-format ' #I:#W '
setw -g window-status-current-format ' [#I:#W] '
```

**Cosa mostra:**

- Finestre normali: `1:vim` `2:bash` `3:server`
- Finestra corrente: `[2:bash]` (con parentesi quadre)

**Variabili finestre:**

|Variabile|Mostra|
|---|---|
|`#I`|Indice finestra (1, 2, 3...)|
|`#W`|Nome finestra|
|`#F`|Flag finestra (* = corrente, - = ultima, Z = zoomed)|
|`#T`|Titolo pannello|

**Personalizzazioni:**

```bash
# Con simboli
setw -g window-status-format ' #I:#W '
setw -g window-status-current-format ' ► #I:#W ◄ '

# Con flag
setw -g window-status-format ' #I:#W#F '
setw -g window-status-current-format ' #I:#W#F '

# Minimale
setw -g window-status-format '#I'
setw -g window-status-current-format '[#I]'
```

---

#### Colori Barra di Stato

```bash
set -g status-style "bg=#1d2021,fg=#d5c4a1"
```

**Cosa fa:** Imposta sfondo (`bg`) e colore testo (`fg`) della barra di stato

**Formati colori supportati:**

**Hex:**

```bash
set -g status-style "bg=#1d2021,fg=#d5c4a1"
```

**Nomi colori:**

```bash
set -g status-style "bg=black,fg=white"
# Colori: black, red, green, yellow, blue, magenta, cyan, white
```

**Colori 256:**

```bash
set -g status-style "bg=colour235,fg=colour250"
# colour0 ... colour255
```

**RGB (true color):**

```bash
set -g status-style "bg=#1d2021,fg=#d5c4a1"
```

---

#### Colore Finestra Corrente

```bash
setw -g window-status-current-style "bg=#458588,fg=#1d2021,bold"
```

**Cosa fa:** La finestra attiva avrà sfondo blu (`#458588`), testo scuro (`#1d2021`) in grassetto

**Attributi testo:**

```bash
# Combinazione di attributi
setw -g window-status-current-style "bg=blue,fg=white,bold,underscore"

# Attributi disponibili:
# - bold
# - dim
# - underscore (sottolineato)
# - blink
# - reverse (inverte fg e bg)
# - italics
```

---

### Performance

#### Escape Time

```bash
set -sg escape-time 10
```

**Cosa fa:** Riduce a 10ms il tempo che tmux aspetta dopo ESC per vedere se fa parte di una sequenza

**Perché è importante:** Default di tmux: **500ms** (mezzo secondo!)

Quando premi ESC in Vim:

```
❌ CON 500ms: Vim aspetta → ritardo percepibile → frustrazione
✅ CON 10ms:  Vim risponde → istantaneo → felicità
```

**Impatto:** Cruciale per chi usa Vim/Neovim dentro tmux. Il passaggio da insert a normal mode diventa istantaneo

**Non mettere 0:** `0` potrebbe far perdere alcune sequenze di escape valide. `10` è il minimo raccomandato

---

#### Titolo Finestra

```bash
set -g set-titles on
set -g set-titles-string '#T'
```

**Cosa fa:** Permette alle applicazioni di cambiare il titolo della finestra del terminale

---

## Comandi Essenziali

### Gestione Sessioni

```bash
# Creare sessioni
tmux new -s nome-sessione          # Nuova sessione con nome
tmux new -s progetto -n editor     # Con nome finestra iniziale

# Listare sessioni
tmux ls                            # Da fuori tmux
Ctrl+Space s                       # Dentro tmux (interattivo)

# Attaccarsi a sessioni
tmux attach -t nome-sessione       # Riconnettiti
tmux a -t nome                     # Forma abbreviata
tmux attach                        # Alla ultima sessione

# Rinominare sessione
Ctrl+Space $                       # Rinomina sessione corrente

# Staccarsi (detach)
Ctrl+Space d                       # Detach
Ctrl+Space D                       # Scegli quale sessione detach

# Eliminare sessioni
tmux kill-session -t nome          # Da fuori
Ctrl+Space :                       # Poi digita: kill-session
exit                               # Dall'ultima finestra della sessione
```

---

### Gestione Finestre

```bash
# Creare e navigare
Ctrl+Space c                 # Nuova finestra
Ctrl+Space ,                 # Rinomina finestra
Ctrl+Space &                 # Chiudi finestra (con conferma)
Ctrl+Space w                 # Lista finestre (interattivo)

# Muoversi tra finestre
Ctrl+Space n                 # Finestra successiva (next)
Ctrl+Space p                 # Finestra precedente (previous)
Ctrl+Space [0-9]             # Vai alla finestra N
Ctrl+Space l                 # Ultima finestra visitata
Ctrl+Space '                 # Prompt: numero finestra
Ctrl+Space f                 # Cerca finestra per nome

# Con la nostra config
Shift+Destra                 # Finestra successiva
Shift+Sinistra               # Finestra precedente

# Spostare finestre
Ctrl+Space :                 # Poi: swap-window -t 2
                             # (scambia con finestra 2)
Ctrl+Space .                 # Sposta e rinumera
```

**🎯 Tip:** Dare nomi significativi alle finestre

```bash
Ctrl+Space ,    # Poi digita: "editor"
Ctrl+Space c    # Nuova finestra
Ctrl+Space ,    # Digita: "server"
Ctrl+Space c    # Nuova finestra
Ctrl+Space ,    # Digita: "logs"

# Risultato: [1:editor] [2:server] [3:logs]
```

---

### Gestione Pannelli

```bash
# Creare pannelli
Ctrl+Space |                 # Split orizzontale (nostra config)
Ctrl+Space -                 # Split verticale (nostra config)
Ctrl+Space %                 # Split orizzontale (default)
Ctrl+Space "                 # Split verticale (default)

# Navigare tra pannelli
Ctrl+hjkl                    # Con nostra config (no prefix!)
Ctrl+Space o                 # Cicla tra pannelli
Ctrl+Space ;                 # Vai all'ultimo pannello attivo
Ctrl+Space q                 # Mostra numeri pannelli

# Ridimensionare
Alt+Frecce                   # Con nostra config
Ctrl+Space Ctrl+Frecce       # Default (tieni premuto per ripetere)

# Layout predefiniti
Ctrl+Space Spazio            # Cicla tra layout
Ctrl+Space Alt+1             # Layout even-horizontal
Ctrl+Space Alt+2             # Layout even-vertical
Ctrl+Space Alt+3             # Layout main-horizontal
Ctrl+Space Alt+4             # Layout main-vertical
Ctrl+Space Alt+5             # Layout tiled

# Operazioni pannelli
Ctrl+Space x                 # Chiudi pannello (con conferma)
Ctrl+Space z                 # Toggle zoom (fullscreen temporaneo)
Ctrl+Space !                 # Converti pannello in finestra
Ctrl+Space {                 # Sposta pannello a sinistra
Ctrl+Space }                 # Sposta pannello a destra
Ctrl+Space Ctrl+o            # Ruota posizione pannelli
```

**Zoom tip:** `Ctrl+Space z` è fantastico per concentrarti temporaneamente su un pannello:

```bash
# Hai 4 pannelli, uno con Vim
┌────┬────┐
│ V  │  T │
├────┼────┤
│ S  │  L │
└────┴────┘

# Premi Ctrl+Space z nel pannello Vim
┌────────────┐
│            │
│    VIM     │
│ (fullscr)  │
└────────────┘

# Premi di nuovo Ctrl+Space z per tornare ai 4 pannelli
```

---

### Copy Mode

```bash
# Entrare in copy mode
Ctrl+Space [                 # Entra
q                            # Esci

# Navigazione (con mode-keys vi)
hjkl                         # Su/giù/sinistra/destra
w / b                        # Parola successiva/precedente
0 / $                        # Inizio/fine riga
g / G                        # Inizio/fine documento
Ctrl+u / Ctrl+d              # Mezza pagina su/giù
Ctrl+b / Ctrl+f              # Pagina completa su/giù

# Ricerca
/testo                       # Cerca "testo" in avanti
?testo                       # Cerca "testo" indietro
n                            # Prossima occorrenza
N                            # Occorrenza precedente

# Selezione e copia (con nostra config)
v                            # Inizia selezione
y                            # Copia e esci
Esc                          # Annulla selezione

# Incolla
Ctrl+Space ]                 # Incolla dall'ultimo buffer
Ctrl+Space =                 # Scegli da quale buffer incollare
```


## Tips

### Configurazione Condizionale

**Per diversi OS:**

```bash
# In ~/.tmux.conf

# Configurazioni specifiche per OS
if-shell "uname | grep -q Darwin" \
    'bind -T copy-mode-vi y send-keys -X copy-pipe-and-cancel "pbcopy"' \
    'bind -T copy-mode-vi y send-keys -X copy-pipe-and-cancel "xclip -in -selection clipboard"'

# Configurazioni specifiche per versione tmux
if-shell "test $(tmux -V | cut -d' ' -f2 | tr -d '[:alpha:]') -ge 3.2" \
    'set -g allow-passthrough on'
```

### Problema: Sessioni tmux si perdono dopo riavvio

**Sintomi:** Dopo il riavvio del server, le sessioni tmux non ci sono più

**Spiegazione:** Tmux salva le sessioni in RAM, non su disco

**Soluzione: tmux-resurrect**

Anche se preferiamo config senza plugin, per questo caso vale la pena:

```bash
# 1. Installa TPM (Tmux Plugin Manager)
git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm

# 2. Aggiungi a ~/.tmux.conf
# Lista plugins
set -g @plugin 'tmux-plugins/tpm'
set -g @plugin 'tmux-plugins/tmux-resurrect'

# Inizializza TPM (tieni questa riga alla fine del file)
run '~/.tmux/plugins/tpm/tpm'

# 3. Ricarica config
Ctrl+Space r

# 4. Installa plugin
Ctrl+Space I (I maiuscola)

# 5. Uso:
# - Salva sessione: Ctrl+Space, Ctrl+s
# - Ripristina: Ctrl+Space, Ctrl+r
```

---

## Risorse Utili

### Documentazione Ufficiale

- **Manual tmux:** `man tmux` nel terminale
- **Guida ufficiale:** https://github.com/tmux/tmux/wiki
- **FAQ:** https://github.com/tmux/tmux/wiki/FAQ

### Cheatsheet

```
SESSIONI
--------
tmux                           Nuova sessione
tmux new -s nome               Nuova sessione con nome
tmux ls                        Lista sessioni
tmux attach -t nome            Attacca a sessione
Ctrl+Space d                   Detach
Ctrl+Space s                   Lista sessioni (interattivo)
Ctrl+Space $                   Rinomina sessione

FINESTRE
--------
Ctrl+Space c                   Nuova finestra
Ctrl+Space ,                   Rinomina finestra
Ctrl+Space w                   Lista finestre
Ctrl+Space n                   Next window
Ctrl+Space p                   Previous window
Ctrl+Space [0-9]               Vai a finestra N
Ctrl+Space &                   Chiudi finestra

PANNELLI
--------
Ctrl+Space |                   Split orizzontale
Ctrl+Space -                   Split verticale
Ctrl+hjkl                      Naviga pannelli
Ctrl+Space z                   Toggle zoom
Ctrl+Space x                   Chiudi pannello
Ctrl+Space !                   Pannello → finestra
Alt+Frecce                     Ridimensiona

COPY MODE
---------
Ctrl+Space [                   Entra copy mode
hjkl                           Naviga
v                              Inizia selezione
y                              Copia
q                              Esci
/testo                         Cerca
n / N                          Next/previous match

ALTRO
-----
Ctrl+Space ?                   Mostra tutti i keybinding
Ctrl+Space :                   Prompt comandi
Ctrl+Space r                   Ricarica config
```


---

## Esempio di Workflow: Sviluppo Rust 🦀

Il mio approccio allo sviluppo Rust con tmux è minimalista: **una sessione per progetto, tre finestre dedicate, zero pannelli**. Ogni finestra occupa l'intero schermo, mantenendo il focus su un singolo task alla volta.

### Setup: 3 Finestre Dedicate
```bash
# Creo una sessione dedicata al progetto
tmux new -s rust-project
cd ~/progetti/my-rust-app
```

#### **Finestra 1: Editor**
```bash
# tmux crea automaticamente la prima finestra
Ctrl+Space ,               # Rinomino: "editor"
nvim src/main.rs

# Tutto lo schermo per il codice, nessuna distrazione
# Git workflow completamente gestito da fugitive:
# - :Git per status
# - :Git commit per commit
# - :Git push per push
# - etc.
```

#### **Finestra 2: Cargo Watch** 
```bash
Ctrl+Space c               # Nuova finestra
Ctrl+Space ,               # Rinomino: "watch"

# Compilazione automatica ad ogni salvataggio
cargo watch -x 'check --all-features' -x 'test --all-features'
```

**Tip:** Installa `cargo-watch` se non l'hai:
```bash
cargo install cargo-watch
```

#### **Finestra 3: Run & Test**
```bash
Ctrl+Space c               # Nuova finestra
Ctrl+Space ,               # Rinomino: "run"

# Qui eseguo manualmente:
# - Test specifici
# - Cargo run
# - Altri comandi cargo ad-hoc
# - Comandi git complessi (solo quando fugitive non basta)
```

**Risultato finale:**
```
[1:editor] [2:watch] [3:run]
```

### Workflow Tipico
```bash
# 1. Avvio o riattacco la sessione
tmux attach -t rust-project

# 2. Finestra "editor" (Ctrl+Space 1):
#    - Scrivo codice
#    - Salvo
#    - Git workflow via fugitive:
:Git                       # Git status
:Git add %                 # Stage file corrente
:Git commit                # Commit con messaggio
:Git push                  # Push

# 3. Finestra "watch" (Ctrl+Space 2):
#    - Vedo immediatamente errori/warning di compilazione
#    - I test girano automaticamente

# 4. Finestra "run" (Ctrl+Space 3):
#    - Test specifici:
cargo test test_authentication -- --nocapture

#    - Lancio l'applicazione:
cargo run --release

#    - Git complesso (solo se necessario):
git rebase -i HEAD~3
git cherry-pick abc123
```

### Navigazione Veloce

Con i nostri binding è semplicissimo passare tra le finestre:
```bash
# Metodo 1: Numero diretto (quello che uso di più)
Ctrl+Space 1               # Editor
Ctrl+Space 2               # Watch  
Ctrl+Space 3               # Run

# Metodo 2: Frecce (dalla nostra config)
Shift+Destra               # Finestra successiva
Shift+Sinistra             # Finestra precedente

# Metodo 3: Next/Previous
Ctrl+Space n               # Next
Ctrl+Space p               # Previous
```

**Questo è il mio workflow quotidiano per Rust con tmux: semplice, efficace e completamente integrato con Neovim.** 🦀

---

## File Configurazione Completo

```bash
# ==============================================
# CONFIGURAZIONE TMUX MINIMALE E PRODUTTIVA
# ==============================================

# --- IMPOSTAZIONI GENERALI ---
unbind C-b
set -g prefix C-Space
bind C-Space send-prefix

bind r source-file ~/.tmux.conf \; display "Configurazione ricaricata!"

set -g mouse on
set -g history-limit 20000
set -g base-index 1
setw -g pane-base-index 1
set -g renumber-windows on
set -ga terminal-overrides ",xterm-256color:Tc"

# --- DIVISIONE FINESTRE ---
bind | split-window -h -c "#{pane_current_path}"
bind - split-window -v -c "#{pane_current_path}"
bind c new-window -c "#{pane_current_path}"

# --- NAVIGAZIONE TRA PANNELLI ---
bind -n C-h select-pane -L
bind -n C-l select-pane -R
bind -n C-k select-pane -U
bind -n C-j select-pane -D

# --- NAVIGAZIONE TRA FINESTRE ---
bind -n S-Left previous-window
bind -n S-Right next-window

# --- RIDIMENSIONAMENTO PANNELLI ---
bind -n M-Left resize-pane -L 3
bind -n M-Right resize-pane -R 3
bind -n M-Up resize-pane -U 3
bind -n M-Down resize-pane -D 3

# --- SYNCRO PANNELLI ---
bind C-y set-window-option synchronize-panes\; display-message "Sync: #{?pane_synchronized,ON,OFF}"

# --- COPY MODE ---
setw -g mode-keys vi
bind -T copy-mode-vi v send-keys -X begin-selection
bind -T copy-mode-vi y send-keys -X copy-selection-and-cancel

# --- BARRA DI STATO ---
set -g status-interval 5
set -g status-position bottom
set -g status-left '<#S> #[default] '
set -g status-right '%d/%m/%Y %H:%M '
setw -g window-status-format ' #I:#W '
setw -g window-status-current-format ' [#I:#W] '

# --- TEMA ---
set -g status-style "bg=#3c3836,fg=#eebd35"
setw -g window-status-current-style "bg=#458588,fg=#1d2021,bold"

# --- PERFORMANCE ---
set -sg escape-time 10
set -g set-titles on
set -g set-titles-string '#T'
```


