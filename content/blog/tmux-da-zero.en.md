+++
title = "Tmux: Minimal and Productive Configuration"
date = 2026-02-18
description = "Three windows. No panes. Zero plugins. One for the editor. One for continuous compilation. One for execution and tests. This is my daily setup with tmux. In this article I'll show you how to build a minimal, portable, and productive configuration from scratch."

[taxonomies]
tags = ["tmux"]

[extra]
comments = true
+++

Three windows.

No panes.

Zero plugins.

One for the editor.

One for continuous compilation.

One for execution and tests.

This is my daily setup with tmux.

In this article I'll show you how to build a minimal, portable, and productive configuration from scratch.

## Introduction

### What is Tmux?

`tmux` is a **terminal multiplexer** that lets you manage multiple sessions, windows, and panes in a single shell — even remotely — without losing anything when you disconnect.

Imagine being able to:

- **Split** a single terminal window into multiple panes
- **Manage** multiple projects simultaneously
- **Disconnect** from a server without losing your work in progress
- **Speed up** your workflow with custom keybindings

### Why a Minimal Configuration?

In this guide we adopt a **plugin-free** approach for several reasons:

**Speed**: No plugin loading overhead
**Portability**: Works on any system without dependencies
**Maintainability**: Easy to understand
**Learning**: You know exactly what every line does

## Prerequisites and Installation

### Installing Tmux

**Ubuntu/Debian:**

```bash
sudo apt update
sudo apt install tmux
```

**macOS (with Homebrew):**

```bash
brew install tmux
```

### Verify Installation

```bash
tmux -V
# Expected output: tmux 3.5a or higher
```


## Some Fundamental Concepts

Before getting started, it's essential to understand the **hierarchy** of tmux:

```
┌─────────────────────────────────────────────────┐
│  SESSION: "web-project"                         │
│  ┌───────────────────────────────────────────┐  │
│  │  WINDOW 1: "editor"                       │  │
│  │  ┌─────────────┬──────────────────────┐   │  │
│  │  │   PANE 1    │     PANE 2           │   │  │
│  │  │   (vim)     │   (terminal)         │   │  │
│  │  └─────────────┴──────────────────────┘   │  │
│  └───────────────────────────────────────────┘  │
│  ┌───────────────────────────────────────────┐  │
│  │  WINDOW 2: "server"                       │  │
│  │  ┌──────────────────────────────────────┐ │  │
│  │  │            PANE 1                    │ │  │
│  │  │          (server logs)               │ │  │
│  │  └──────────────────────────────────────┘ │  │
│  └───────────────────────────────────────────┘  │
└─────────────────────────────────────────────────┘
```

A *Session* in tmux is a work context — it could be an entire project, for example. You might have a session for the program you're working on, one for updating your blog, or another for configuring your NeoVim.

A *window* is, within a session, a virtual screen — like a tab in your favorite browser.

A *pane* is a division within a window.

| Term | Description | Analogy |
|---|---|---|
| **Session** | Complete work context | An entire project |
| **Window** | Virtual screen within a session | A browser tab |
| **Pane** | Division of a window | Split screen |

### The Prefix Key

The **prefix** is the "magic key" that activates tmux commands. By default it's `Ctrl+b`, but in our configuration we'll change it to `Ctrl+Space` for ergonomics.

**How it works:**

1. Press `Ctrl+Space` (the prefix)
2. Release
3. Press the command (e.g., `c` for new window)

⚠️ **Note:** in some terminals or keyboard layouts `Ctrl+Space` might not work correctly. In that case `Ctrl+a` is an excellent alternative with universal support.

## Initial Setup

First we need to create our configuration file:

```bash
touch ~/.tmux.conf
```

then, using our favorite editor (I don't think I need to tell you which one is mine 😀), let's add configurations step by step — I'll explain each one line by line.

### General Settings

#### Changing the Prefix Key

```bash
unbind C-b
set -g prefix C-Space
bind C-Space send-prefix
```

**What it does:**

- `unbind C-b`: Removes the default `Ctrl+b` binding
- `set -g prefix C-Space`: Sets `Ctrl+Space` as the new prefix
- `bind C-Space send-prefix`: Allows sending `Ctrl+Space` to the underlying application (by pressing it twice)

**Why:**

- `Ctrl+Space` is **more ergonomic**: you press it with your thumb instead of your pinky
- It doesn't interfere with common commands like `Ctrl+b` in Vim (page up)
- It's a less commonly used combination than `Ctrl+b`

	Some people prefer `Ctrl+a` — I prefer space, but it's just a matter of finding the most practical combination for you that doesn't cause conflicts with the software you typically use in your terminal workflow.

**Practical use:**

```bash
# Create a new window:
Ctrl+Space, then c

# If an application needs Ctrl+Space:
Ctrl+Space, Ctrl+Space
```

---

#### Reload Configuration

```bash
bind r source-file ~/.tmux.conf \; display "Configuration reloaded!"
```

**What it does:** Creates a keybinding to reload the configuration file without leaving tmux

**Why:** When you modify `~/.tmux.conf`, instead of closing and reopening tmux, you can reload changes on the fly

**Practical use:**

```bash
# 1. Edit ~/.tmux.conf with your editor
vim ~/.tmux.conf

# 2. Save changes

# 3. In tmux, press:
Ctrl+Space, then r

# You'll see: "Configuration reloaded!"
```

---

#### Mouse Support

```bash
set -g mouse on
```

**What it does:** Fully enables mouse support in tmux

**Enabled features:**

- **Click** on a pane to activate it
- **Drag** borders between panes to resize
- **Click** on a window in the status bar to select it
- **Scroll** with the wheel to navigate history
- **Select** text with the mouse and copy

**Note:** With the mouse active, to select text and copy it to the system clipboard, you may need to hold `Shift` while selecting

**Practical use:**

```bash
# Scenario: you have 4 panes open
# Instead of using Ctrl+hjkl to navigate,
# simply click on the desired pane
```

---

#### History Limit

```bash
set -g history-limit 20000
```

**What it does:** Sets the number of output lines stored per pane to 20,000

**Why:**

- **Default**: only 2,000 lines
- **20,000**: lets you scroll back to much earlier output
- Useful for analyzing long logs or finding past output

**Practical use:**

```bash
# After running commands with a lot of output:
Ctrl+Space, then [        # Enter copy mode
# Now you can scroll back through 20,000 lines
# Press q to exit
```

**📝 Memory usage:** 20,000 lines is roughly 2-3 MB per pane (depends on line length)

---

#### Numbering from 1

```bash
set -g base-index 1
setw -g pane-base-index 1
```

**What it does:** Makes window and pane numbering start from **1** instead of **0** as tmux defaults

**Why:**

```
Keyboard:  [1] [2] [3] [4] [5] [6] [7] [8] [9] [0]
With 0:     ^                               first window (awkward!)
With 1:     ^                               first window (convenient!)
```

Keys 1-9 are easier to reach than having to go all the way to 0 for the "first" window

**Practical use:**

```bash
Ctrl+Space, then 1    # Go to the FIRST window (not the second!)
```

---

#### Automatic Renumbering

```bash
set -g renumber-windows on
```

**What it does:** When you close a window, the subsequent ones are renumbered to maintain the sequence

**Example:**

```
Initial situation:  [1:vim] [2:server] [3:logs] [4:test]

Close window 2 (server)

❌ WITHOUT renumber:  [1:vim] [3:logs] [4:test]     # Numbering with gaps!
✅ WITH renumber:     [1:vim] [2:logs] [3:test]     # Always sequential!
```

**Advantage:** You don't have to "skip" numbers when switching between windows — you always maintain a 1-2-3-4 sequence

---

#### True Color Support

```bash
set -ga terminal-overrides ",xterm-256color:Tc"
```

**What it does:** Enables **true color** support (16 million colors) instead of just 256 colors

**Why:**

- Modern Vim/Neovim themes require true color
- Gradients and color shadings
- Richer syntax highlighting

**Technical details:**

- `-ga`: append (adds instead of overwriting)
- `Tc`: true color capability
- `xterm-256color`: terminal type

**Prerequisite:** Your terminal must support true color (iTerm2, Alacritty, Kitty, modern terminals)

---

### Window Splitting

#### Horizontal and Vertical Split

```bash
bind | split-window -h -c "#{pane_current_path}"
bind - split-window -v -c "#{pane_current_path}"
```

**What it does:**

- `|` (pipe): splits the window **horizontally** (creates columns)
- `-` (dash): splits the window **vertically** (creates rows)
- `-c "#{pane_current_path}"`: the new pane starts from the same directory

**Visualization:**

```
HORIZONTAL split (|):        VERTICAL split (-):
┌──────────┬──────────┐      ┌─────────────────────┐
│          │          │      │                     │
│  Pane 1  │  Pane 2  │      │      Pane 1         │
│  (vim)   │ (shell)  │      │      (vim)          │
│          │          │      ├─────────────────────┤
└──────────┴──────────┘      │      Pane 2         │
                             │     (shell)         │
                             └─────────────────────┘
```

**Practical use:**

```bash
# You're editing code and want a terminal on the right
Ctrl+Space, then |

# You're editing and want to see logs below
Ctrl+Space, then -
```

**Mnemonic:**

- `|` looks like a vertical line → horizontal split
- `-` looks like a horizontal line → vertical split

**Without `-c "#{pane_current_path}"`:** The new pane would start from the home directory, forcing you to navigate to the current directory

📝 By default tmux allows splitting using `%` and `"` — this configuration doesn't override them, they'll still be usable

---

#### New Window in Current Directory

```bash
bind c new-window -c "#{pane_current_path}"
```

**What it does:** When you create a new window with `c`, it starts from the same directory as the current pane

**Example:**

```bash
# You're in: /home/user/projects/app/src
Ctrl+Space, then c
# The new window starts in: /home/user/projects/app/src
# (not in /home/user as it would by default)
```

**Benefit:** You don't have to `cd` every time you open a new window to work on the same project

---

### Navigation

#### Pane Navigation (Vim Style)

```bash
bind -n C-h select-pane -L
bind -n C-l select-pane -R
bind -n C-k select-pane -U
bind -n C-j select-pane -D
```

**What it does:** Navigates between panes using `Ctrl+hjkl` **without pressing the prefix**

**Vim Layout:**

```
        k (up)
        ↑
h (left) ←+→ l (right)
        ↓
        j (down)
```

**Practical use:**

```bash
# You have 4 panes open
┌─────┬─────┐
│  1  │  2  │
├─────┼─────┤
│  3  │  4  │
└─────┴─────┘

# You're in pane 1, want to go right (2):
Ctrl+l    # No prefix!

# From pane 2, want to go down (4):
Ctrl+j
```

**⚠️ Potential Conflict:**

These bindings override some standard commands:

- `Ctrl+l`: clear screen in bash/zsh
- `Ctrl+h`: backspace in some terminals

**If you don't want conflicts:**

```bash
# Version with prefix (requires Ctrl+Space first)
bind h select-pane -L
bind l select-pane -R
bind k select-pane -U
bind j select-pane -D
```

**Alternative for clear:** If you lose `Ctrl+l` for clear, you can:

- Use the `clear` or `reset` command
- Create an alias: `alias cls='clear'`
- Use `Ctrl+Space` then `Ctrl+l` (sends the command to the application)

---

#### Window Navigation

```bash
bind -n S-Left previous-window
bind -n S-Right next-window
```

**What it does:**

- `Shift+Left Arrow`: go to the previous window
- `Shift+Right Arrow`: go to the next window

**Practical use:**

```bash
# You have 3 windows: [1:vim] [2:server] [3:logs]
# You're in window 2 (server)

Shift+Left Arrow    # Go to [1:vim]
Shift+Right Arrow   # Back to [2:server]
Shift+Right Arrow   # Go to [3:logs]
```

**Built-in alternatives:**

```bash
Ctrl+Space, then n     # next window
Ctrl+Space, then p     # previous window
Ctrl+Space, then 1     # go to window 1
Ctrl+Space, then 2     # go to window 2
# ... etc
```

---

### Pane Resizing

```bash
bind -n M-Left resize-pane -L 3
bind -n M-Right resize-pane -R 3
bind -n M-Up resize-pane -U 3
bind -n M-Down resize-pane -D 3
```

**What it does:** Resizes the current pane using `Alt+Arrows`, 3 cells at a time

**Note:** `M` = Meta (typically the `Alt` key or `Option` on Mac)

**Practical use:**

```bash
# Scenario: you have an editor on the left and a terminal on the right
┌────────────┬──────┐
│            │      │
│   Editor   │ Term │
│            │      │
└────────────┴──────┘

# You want to give more space to the editor:
Alt+Right Arrow (multiple times)

┌──────────────────┬──┐
│                  │T │
│     Editor       │e │
│                  │r │
└──────────────────┴──┘
```

**Customization:** If 3 cells is too much/too little, change the number:

```bash
bind -n M-Left resize-pane -L 5   # 5 cells at a time
```

**Alternatives with prefix (and repeat):**

```bash
# With -r (repeat) you can hold the key
bind -r H resize-pane -L 5
bind -r J resize-pane -D 5
bind -r K resize-pane -U 5
bind -r L resize-pane -R 5
```

---

### Pane Synchronization

```bash
bind C-y set-window-option synchronize-panes\; display-message "Sync: #{?pane_synchronized,ON,OFF}"
```

**What it does:** Toggles input synchronization across all panes in the current window

**When it's useful:**
- Running the same command on multiple SSH servers simultaneously
- Updating multiple servers at the same time
- Configuring multiple machines simultaneously

**Practical use:**

```bash
# Scenario: multi-server administration

# 1. Create 3 panes with SSH to different servers
Ctrl+Space |         # Split
ssh user@server1.com
Ctrl+Space |         # Split again
ssh user@server2.com
Ctrl+Space -         # Vertical split
ssh user@server3.com

# 2. Activate synchronization Ctrl+Space Ctrl+y

# 3. Type a command - it runs on ALL servers!

sudo apt update
sudo apt upgrade -y

# 4. Deactivate synchronization when done Ctrl+Space Ctrl+y
```


### Copy Mode

#### Vi Mode

```bash
setw -g mode-keys vi
```

**What it does:** Enables Vi/Vim keybindings in copy mode (instead of the default Emacs bindings)

**Copy Mode Navigation:**

| Key | Action |
|---|---|
| `h` | left |
| `j` | down |
| `k` | up |
| `l` | right |
| `w` | next word |
| `b` | previous word |
| `0` | beginning of line |
| `$` | end of line |
| `g` | beginning of document |
| `G` | end of document |
| `/` | search |
| `n` | next occurrence |
| `N` | previous occurrence |

---

#### Selection and Copy

```bash
bind -T copy-mode-vi v send-keys -X begin-selection
bind -T copy-mode-vi y send-keys -X copy-selection-and-cancel
```

**What it does:**

- `v`: starts visual selection (like in Vim)
- `y`: "yank" (copy) the selection and exits copy mode

**Complete workflow:**

```bash
# STEP 1: Enter copy mode
Ctrl+Space, then [

# STEP 2: Navigate to the text you want to copy
# (use hjkl, or arrows, or search with /)

# STEP 3: Start selection
v

# STEP 4: Extend selection (move with hjkl)
# The selected text will be highlighted

# STEP 5: Copy
y

# STEP 6: Paste (in another pane)
Ctrl+Space, then ]
```

---

#### System Clipboard Integration

The base configuration copies to tmux's internal buffer. To copy to the system clipboard:

**Linux (with xclip):**

```bash
# Install xclip
sudo apt install xclip

# Add to ~/.tmux.conf
bind -T copy-mode-vi y send-keys -X copy-pipe-and-cancel "xclip -in -selection clipboard"
```

**macOS:**

```bash
# Use pbcopy (already present)
bind -T copy-mode-vi y send-keys -X copy-pipe-and-cancel "pbcopy"
```

**With mouse selection:**

```bash
# Linux
bind -T copy-mode-vi MouseDragEnd1Pane send-keys -X copy-pipe-and-cancel "xclip -in -selection clipboard"

# macOS
bind -T copy-mode-vi MouseDragEnd1Pane send-keys -X copy-pipe-and-cancel "pbcopy"
```

---

### Status Bar

#### Update Interval

```bash
set -g status-interval 5
```

**What it does:** Updates the status bar every 5 seconds

**Why:**

- Keeps the clock updated
- If you add custom scripts (e.g., battery, CPU), they get updated
- 5 seconds is a good compromise between freshness and performance

---

#### Bar Position

```bash
set -g status-position bottom
```

**Options:**

- `bottom`: at the bottom (default and most common)
- `top`: at the top

**When to use `top`:** If you use a dropdown terminal (e.g., Guake, Yakuake) that appears from the top, a status bar at the top feels more natural

---

#### Left Bar Content

```bash
set -g status-left '<#S> #[default] '
```

**What it shows:**

- `#S`: session name
- Example: `<web-project>`

**Useful variables:**

| Variable | Shows |
|---|---|
| `#S` | Session name |
| `#I` | Current window index |
| `#P` | Current pane index |
| `#W` | Current window name |
| `#H` | Full hostname |
| `#h` | Hostname (name only) |
| `#(command)` | Output of a command |

**🔧 Custom examples:**

```bash
# With hostname
set -g status-left '#h <#S> | '

# With icons (requires a font with symbols)
set -g status-left '  #S | '

# With user info
set -g status-left '#(whoami)@#h <#S> | '
```

---

#### Right Bar Content

```bash
set -g status-right '%d/%m/%Y %H:%M '
```

**What it shows:** Date and time in European format: `08/02/2026 15:30`

**Date/time formats:**

| Code | Output | Example |
|---|---|---|
| `%d` | Day | 08 |
| `%m` | Month | 02 |
| `%Y` | Year (4 digits) | 2026 |
| `%H` | Hour (24h) | 15 |
| `%I` | Hour (12h) | 03 |
| `%M` | Minutes | 30 |
| `%S` | Seconds | 45 |
| `%p` | AM/PM | PM |
| `%A` | Day of week | Sunday |
| `%B` | Month name | February |

**Custom examples:**

```bash
# US format
set -g status-right '%m/%d/%Y %I:%M %p'
# Output: 02/08/2026 03:30 PM

# With day of week
set -g status-right '%A %d/%m/%Y %H:%M'
# Output: Sunday 08/02/2026 15:30

# With hostname
set -g status-right '#H | %d/%m/%Y %H:%M'
# Output: my-server | 08/02/2026 15:30

# With battery (requires custom script)
set -g status-right '🔋 #(battery.sh) | %H:%M'
# Output: 🔋 87% | 15:30
```

---

#### Window Format

```bash
setw -g window-status-format ' #I:#W '
setw -g window-status-current-format ' [#I:#W] '
```

**What it shows:**

- Normal windows: `1:vim` `2:bash` `3:server`
- Current window: `[2:bash]` (with square brackets)

**Window variables:**

| Variable | Shows |
|---|---|
| `#I` | Window index (1, 2, 3...) |
| `#W` | Window name |
| `#F` | Window flag (* = current, - = last, Z = zoomed) |
| `#T` | Pane title |

**Customizations:**

```bash
# With symbols
setw -g window-status-format ' #I:#W '
setw -g window-status-current-format ' ► #I:#W ◄ '

# With flags
setw -g window-status-format ' #I:#W#F '
setw -g window-status-current-format ' #I:#W#F '

# Minimal
setw -g window-status-format '#I'
setw -g window-status-current-format '[#I]'
```

---

#### Status Bar Colors

```bash
set -g status-style "bg=#1d2021,fg=#d5c4a1"
```

**What it does:** Sets the background (`bg`) and text color (`fg`) of the status bar

**Supported color formats:**

**Hex:**

```bash
set -g status-style "bg=#1d2021,fg=#d5c4a1"
```

**Color names:**

```bash
set -g status-style "bg=black,fg=white"
# Colors: black, red, green, yellow, blue, magenta, cyan, white
```

**256 colors:**

```bash
set -g status-style "bg=colour235,fg=colour250"
# colour0 ... colour255
```

**RGB (true color):**

```bash
set -g status-style "bg=#1d2021,fg=#d5c4a1"
```

---

#### Current Window Color

```bash
setw -g window-status-current-style "bg=#458588,fg=#1d2021,bold"
```

**What it does:** The active window will have a blue background (`#458588`), dark text (`#1d2021`) in bold

**Text attributes:**

```bash
# Combination of attributes
setw -g window-status-current-style "bg=blue,fg=white,bold,underscore"

# Available attributes:
# - bold
# - dim
# - underscore (underlined)
# - blink
# - reverse (swaps fg and bg)
# - italics
```

---

### Performance

#### Escape Time

```bash
set -sg escape-time 10
```

**What it does:** Reduces to 10ms the time tmux waits after ESC to see if it's part of a sequence

**Why it matters:** tmux default: **500ms** (half a second!)

When you press ESC in Vim:

```
❌ WITH 500ms: Vim waits → noticeable delay → frustration
✅ WITH 10ms:  Vim responds → instant → happiness
```

**Impact:** Crucial for those using Vim/Neovim inside tmux. Switching from insert to normal mode becomes instant

**Don't set it to 0:** `0` might cause some valid escape sequences to be lost. `10` is the recommended minimum

---

#### Window Title

```bash
set -g set-titles on
set -g set-titles-string '#T'
```

**What it does:** Allows applications to change the terminal window title

---

## Essential Commands

### Session Management

```bash
# Creating sessions
tmux new -s session-name           # New session with name
tmux new -s project -n editor      # With initial window name

# Listing sessions
tmux ls                            # From outside tmux
Ctrl+Space s                       # Inside tmux (interactive)

# Attaching to sessions
tmux attach -t session-name        # Reconnect
tmux a -t name                     # Short form
tmux attach                        # To the last session

# Renaming session
Ctrl+Space $                       # Rename current session

# Detaching
Ctrl+Space d                       # Detach
Ctrl+Space D                       # Choose which session to detach

# Killing sessions
tmux kill-session -t name          # From outside
Ctrl+Space :                       # Then type: kill-session
exit                               # From the last window of the session
```

---

### Window Management

```bash
# Creating and navigating
Ctrl+Space c                 # New window
Ctrl+Space ,                 # Rename window
Ctrl+Space &                 # Close window (with confirmation)
Ctrl+Space w                 # List windows (interactive)

# Moving between windows
Ctrl+Space n                 # Next window
Ctrl+Space p                 # Previous window
Ctrl+Space [0-9]             # Go to window N
Ctrl+Space l                 # Last visited window
Ctrl+Space '                 # Prompt: window number
Ctrl+Space f                 # Search window by name

# With our config
Shift+Right                  # Next window
Shift+Left                   # Previous window

# Moving windows
Ctrl+Space :                 # Then: swap-window -t 2
                             # (swap with window 2)
Ctrl+Space .                 # Move and renumber
```

**🎯 Tip:** Give meaningful names to windows

```bash
Ctrl+Space ,    # Then type: "editor"
Ctrl+Space c    # New window
Ctrl+Space ,    # Type: "server"
Ctrl+Space c    # New window
Ctrl+Space ,    # Type: "logs"

# Result: [1:editor] [2:server] [3:logs]
```

---

### Pane Management

```bash
# Creating panes
Ctrl+Space |                 # Horizontal split (our config)
Ctrl+Space -                 # Vertical split (our config)
Ctrl+Space %                 # Horizontal split (default)
Ctrl+Space "                 # Vertical split (default)

# Navigating between panes
Ctrl+hjkl                    # With our config (no prefix!)
Ctrl+Space o                 # Cycle through panes
Ctrl+Space ;                 # Go to last active pane
Ctrl+Space q                 # Show pane numbers

# Resizing
Alt+Arrows                   # With our config
Ctrl+Space Ctrl+Arrows       # Default (hold to repeat)

# Preset layouts
Ctrl+Space Space             # Cycle through layouts
Ctrl+Space Alt+1             # even-horizontal layout
Ctrl+Space Alt+2             # even-vertical layout
Ctrl+Space Alt+3             # main-horizontal layout
Ctrl+Space Alt+4             # main-vertical layout
Ctrl+Space Alt+5             # tiled layout

# Pane operations
Ctrl+Space x                 # Close pane (with confirmation)
Ctrl+Space z                 # Toggle zoom (temporary fullscreen)
Ctrl+Space !                 # Convert pane to window
Ctrl+Space {                 # Move pane left
Ctrl+Space }                 # Move pane right
Ctrl+Space Ctrl+o            # Rotate pane positions
```

**Zoom tip:** `Ctrl+Space z` is fantastic for temporarily focusing on a pane:

```bash
# You have 4 panes, one with Vim
┌────┬────┐
│ V  │  T │
├────┼────┤
│ S  │  L │
└────┴────┘

# Press Ctrl+Space z in the Vim pane
┌────────────┐
│            │
│    VIM     │
│ (fullscr)  │
└────────────┘

# Press Ctrl+Space z again to return to the 4 panes
```

---

### Copy Mode

```bash
# Enter copy mode
Ctrl+Space [                 # Enter
q                            # Exit

# Navigation (with mode-keys vi)
hjkl                         # Up/down/left/right
w / b                        # Next/previous word
0 / $                        # Beginning/end of line
g / G                        # Beginning/end of document
Ctrl+u / Ctrl+d              # Half page up/down
Ctrl+b / Ctrl+f              # Full page up/down

# Search
/text                        # Search "text" forward
?text                        # Search "text" backward
n                            # Next occurrence
N                            # Previous occurrence

# Selection and copy (with our config)
v                            # Start selection
y                            # Copy and exit
Esc                          # Cancel selection

# Paste
Ctrl+Space ]                 # Paste from last buffer
Ctrl+Space =                 # Choose which buffer to paste from
```


## Tips

### Conditional Configuration

**For different OSes:**

```bash
# In ~/.tmux.conf

# OS-specific configurations
if-shell "uname | grep -q Darwin" \
    'bind -T copy-mode-vi y send-keys -X copy-pipe-and-cancel "pbcopy"' \
    'bind -T copy-mode-vi y send-keys -X copy-pipe-and-cancel "xclip -in -selection clipboard"'

# tmux version-specific configurations
if-shell "test $(tmux -V | cut -d' ' -f2 | tr -d '[:alpha:]') -ge 3.2" \
    'set -g allow-passthrough on'
```

### Problem: Tmux sessions are lost after reboot

**Symptoms:** After a server reboot, tmux sessions are gone

**Explanation:** Tmux saves sessions in RAM, not on disk

**Solution: tmux-resurrect**

Even though we prefer a plugin-free config, for this case it's worth it:

```bash
# 1. Install TPM (Tmux Plugin Manager)
git clone https://github.com/tmux-plugins/tpm ~/.tmux/plugins/tpm

# 2. Add to ~/.tmux.conf
# Plugin list
set -g @plugin 'tmux-plugins/tpm'
set -g @plugin 'tmux-plugins/tmux-resurrect'

# Initialize TPM (keep this line at the end of the file)
run '~/.tmux/plugins/tpm/tpm'

# 3. Reload config
Ctrl+Space r

# 4. Install plugin
Ctrl+Space I (capital I)

# 5. Usage:
# - Save session: Ctrl+Space, Ctrl+s
# - Restore: Ctrl+Space, Ctrl+r
```

---

## Useful Resources

### Official Documentation

- **Tmux manual:** `man tmux` in the terminal
- **Official guide:** https://github.com/tmux/tmux/wiki
- **FAQ:** https://github.com/tmux/tmux/wiki/FAQ

### Cheatsheet

```
SESSIONS
--------
tmux                           New session
tmux new -s name               New session with name
tmux ls                        List sessions
tmux attach -t name            Attach to session
Ctrl+Space d                   Detach
Ctrl+Space s                   List sessions (interactive)
Ctrl+Space $                   Rename session

WINDOWS
-------
Ctrl+Space c                   New window
Ctrl+Space ,                   Rename window
Ctrl+Space w                   List windows
Ctrl+Space n                   Next window
Ctrl+Space p                   Previous window
Ctrl+Space [0-9]               Go to window N
Ctrl+Space &                   Close window

PANES
-----
Ctrl+Space |                   Horizontal split
Ctrl+Space -                   Vertical split
Ctrl+hjkl                      Navigate panes
Ctrl+Space z                   Toggle zoom
Ctrl+Space x                   Close pane
Ctrl+Space !                   Pane → window
Alt+Arrows                     Resize

COPY MODE
---------
Ctrl+Space [                   Enter copy mode
hjkl                           Navigate
v                              Start selection
y                              Copy
q                              Exit
/text                          Search
n / N                          Next/previous match

OTHER
-----
Ctrl+Space ?                   Show all keybindings
Ctrl+Space :                   Command prompt
Ctrl+Space r                   Reload config
```


---

## Workflow Example: Rust Development 🦀

My approach to Rust development with tmux is minimalist: **one session per project, three dedicated windows, zero panes**. Each window takes up the entire screen, keeping the focus on a single task at a time.

### Setup: 3 Dedicated Windows
```bash
# Create a session dedicated to the project
tmux new -s rust-project
cd ~/projects/my-rust-app
```

#### **Window 1: Editor**
```bash
# tmux automatically creates the first window
Ctrl+Space ,               # Rename: "editor"
nvim src/main.rs

# The whole screen for code, no distractions
# Git workflow fully managed by fugitive:
# - :Git for status
# - :Git commit for commit
# - :Git push for push
# - etc.
```

#### **Window 2: Cargo Watch**
```bash
Ctrl+Space c               # New window
Ctrl+Space ,               # Rename: "watch"

# Automatic compilation on every save
cargo watch -x 'check --all-features' -x 'test --all-features'
```

**Tip:** Install `cargo-watch` if you don't have it:
```bash
cargo install cargo-watch
```

#### **Window 3: Run & Test**
```bash
Ctrl+Space c               # New window
Ctrl+Space ,               # Rename: "run"

# Here I manually run:
# - Specific tests
# - Cargo run
# - Other ad-hoc cargo commands
# - Complex git commands (only when fugitive isn't enough)
```

**Final result:**
```
[1:editor] [2:watch] [3:run]
```

### Typical Workflow
```bash
# 1. Start or reattach the session
tmux attach -t rust-project

# 2. "editor" window (Ctrl+Space 1):
#    - Write code
#    - Save
#    - Git workflow via fugitive:
:Git                       # Git status
:Git add %                 # Stage current file
:Git commit                # Commit with message
:Git push                  # Push

# 3. "watch" window (Ctrl+Space 2):
#    - Immediately see compilation errors/warnings
#    - Tests run automatically

# 4. "run" window (Ctrl+Space 3):
#    - Specific tests:
cargo test test_authentication -- --nocapture

#    - Launch the application:
cargo run --release

#    - Complex git (only if necessary):
git rebase -i HEAD~3
git cherry-pick abc123
```

### Fast Navigation

With our bindings, switching between windows is dead simple:
```bash
# Method 1: Direct number (the one I use most)
Ctrl+Space 1               # Editor
Ctrl+Space 2               # Watch
Ctrl+Space 3               # Run

# Method 2: Arrows (from our config)
Shift+Right                # Next window
Shift+Left                 # Previous window

# Method 3: Next/Previous
Ctrl+Space n               # Next
Ctrl+Space p               # Previous
```

**This is my daily workflow for Rust with tmux: simple, effective, and fully integrated with Neovim.** 🦀

---

## Complete Configuration File

```bash
# ==============================================
# MINIMAL AND PRODUCTIVE TMUX CONFIGURATION
# ==============================================

# --- GENERAL SETTINGS ---
unbind C-b
set -g prefix C-Space
bind C-Space send-prefix

bind r source-file ~/.tmux.conf \; display "Configuration reloaded!"

set -g mouse on
set -g history-limit 20000
set -g base-index 1
setw -g pane-base-index 1
set -g renumber-windows on
set -ga terminal-overrides ",xterm-256color:Tc"

# --- WINDOW SPLITTING ---
bind | split-window -h -c "#{pane_current_path}"
bind - split-window -v -c "#{pane_current_path}"
bind c new-window -c "#{pane_current_path}"

# --- PANE NAVIGATION ---
bind -n C-h select-pane -L
bind -n C-l select-pane -R
bind -n C-k select-pane -U
bind -n C-j select-pane -D

# --- WINDOW NAVIGATION ---
bind -n S-Left previous-window
bind -n S-Right next-window

# --- PANE RESIZING ---
bind -n M-Left resize-pane -L 3
bind -n M-Right resize-pane -R 3
bind -n M-Up resize-pane -U 3
bind -n M-Down resize-pane -D 3

# --- PANE SYNC ---
bind C-y set-window-option synchronize-panes\; display-message "Sync: #{?pane_synchronized,ON,OFF}"

# --- COPY MODE ---
setw -g mode-keys vi
bind -T copy-mode-vi v send-keys -X begin-selection
bind -T copy-mode-vi y send-keys -X copy-selection-and-cancel

# --- STATUS BAR ---
set -g status-interval 5
set -g status-position bottom
set -g status-left '<#S> #[default] '
set -g status-right '%d/%m/%Y %H:%M '
setw -g window-status-format ' #I:#W '
setw -g window-status-current-format ' [#I:#W] '

# --- THEME ---
set -g status-style "bg=#3c3836,fg=#eebd35"
setw -g window-status-current-style "bg=#458588,fg=#1d2021,bold"

# --- PERFORMANCE ---
set -sg escape-time 10
set -g set-titles on
set -g set-titles-string '#T'
```

> **Translation note:** This article was originally written in Italian and translated into English with the assistance of an AI language model. While every effort has been made to preserve the original meaning and technical accuracy, minor nuances may differ from the source text.

