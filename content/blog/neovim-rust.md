+++
title = "Guida Completa: Neovim 0.11 IDE per Rust su Ubuntu/MacOS"
date = 2025-10-15
description = "Riflessioni sul perché ho scelto Rust come linguaggio principale"

[taxonomies]
tags = ["rust", "neovim"]

[extra]
comments = true
+++

Benvenuto in questa guida completa all'installazione e alla configurazione di Neovim 0.11 per lo sviluppo in Rust! 🦀

![configurazione-neovim-rust](https://github.com/user-attachments/assets/1bcf3da2-0c38-48cd-a1e6-faa84c78460d)

Se sei uno sviluppatore Rust alla ricerca di un editor di testo che sia allo stesso tempo leggero, veloce e incredibilmente personalizzabile, Neovim è la scelta perfetta. Dimentica i pesanti IDE tradizionali: con questa guida, trasformeremo Neovim in un ambiente di sviluppo Rust moderno e produttivo, su misura per le tue esigenze.

In questa guida vedremo come:

Installare l'ultima versione di Neovim (0.11).

Configurare da zero un ambiente di sviluppo minimale utilizzando Lua.

Integrare rust-analyzer tramite il Language Server Protocol (LSP) nativo di Neovim per ottenere funzionalità da IDE come l'autocompletamento, la diagnostica in tempo reale e la navigazione del codice.

Impostare il syntax highlighting con Tree-sitter per un'analisi del codice più precisa e veloce.

Alla fine di questo percorso, avrai una configurazione Neovim funzionale e performante, pronta per affrontare qualsiasi progetto Rust. Iniziamo!


## Prerequisiti e Dipendenze di Sistema

### Installazione su GNU/Linux (Ubuntu)
Prima di iniziare, installiamo tutte le dipendenze necessarie:

```bash
# Aggiorna il sistema
sudo apt update && sudo apt upgrade -y

# Dipendenze per i plugin
sudo apt install -y ripgrep fd-find

# Build essentials (necessari per compilare parser Treesitter e alcune estensioni Rust)
sudo apt install -y build-essential git curl

# Node.js (opzionale, necessario solo per alcuni LSP non-Rust)
sudo apt install -y nodejs

# Creiamo un symlink per fd (su Ubuntu si chiama fdfind)
sudo ln -s $(which fdfind) /usr/local/bin/fd 2>/dev/null || true
```

### MacOS

Su macOS useremo Homebrew come package manager:

```bash
# Installa Homebrew se non già installato
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Installa Xcode Command Line Tools (contiene gcc, git, etc.)
xcode-select --install

# Dipendenze per i plugin
brew install ripgrep fd

# Node.js (opzionale, utile per altri linguaggi)
brew install node
```
**Spiegazione delle dipendenze:**
- **ripgrep**: ricerca velocissima nei file, usata da Telescope
- **fd**: alternativa moderna a `find`, usata da Telescope
- **build-essential**: gcc e g++ necessari per compilare i parser Treesitter
- **Node.js**: opzionale, ma utile se userai altri linguaggi oltre a Rust

---

## Installazione di Neovim 0.11

### Installazione su MacOS

Per eseguire l'installazione di NeoVim, basterà:

```bash
brew install neovim
```

### Installazione su GNU/Linux (Ubuntu)

Per questa guida prendiamo come riferimento la distribuzione GNU/Linux Ubuntu.

Installiamo Neovim tramite snap:

```bash
# Installa Neovim 0.11 tramite snap (flag --classic per accesso completo al sistema)
sudo snap install nvim --classic

# Verifichiamo l'installazione
nvim --version
```

Dovresti vedere output simile a:
```
NVIM v0.11.4
Build type: Release
...
```

**Perché usare snap?**
- ✅ Installazione rapida (pochi secondi vs 10-15 minuti di compilazione)
- ✅ Sempre aggiornato al latest stable
- ✅ Aggiornamenti automatici gestiti da snap
- ✅ Nessuna dipendenza di build necessaria

**Nota**: Il flag `--classic` è necessario perché Neovim deve accedere liberamente ai file del sistema per funzionare come editor.

### Metodo alternativo: Compilazione dai sorgenti

Se preferisci compilare dai sorgenti per avere il controllo completo, su Ubuntu ti servirà:

```bash
# Installa dipendenze di build
sudo apt install -y ninja-build gettext cmake unzip
```
mentre su MacOS:

```bash
# Installa dipendenze di build
brew install ninja cmake gettext
```
Possiamo ora compilare NeoVim per il nostro sistema:

```bash
# Scarica e compila
mkdir -p ~/build && cd ~/build
git clone https://github.com/neovim/neovim.git
cd neovim
git checkout v0.11.4
make CMAKE_BUILD_TYPE=Release
sudo make install
```

Pro: Controllo totale sulla versione e opzioni di build  
Contro: Richiede 10-15 minuti e più dipendenze

---

## Installazione Toolchain Rust

Installiamo Rust e i componenti necessari per lo sviluppo:

```bash
# Installa rustup (gestore della toolchain Rust)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Ricarichiamo il PATH
source "$HOME/.cargo/env"

# Installiamo i componenti necessari se non precedentemente installati
rustup component add rust-analyzer  # LSP per Rust
rustup component add rustfmt         # Formatter
rustup component add clippy          # Linter

# Verifichiamo l'installazione
rustc --version
cargo --version
rust-analyzer --version
```

**Nota importante**: 
- `rust-analyzer` è il language server che fornisce autocompletamento, diagnostica e altre funzionalità IDE (LSP)
- **codelldb** (debugger) verrà installato automaticamente tramite Mason (plugin di Neovim) nella sezione successiva

---

## Struttura della Configurazione

Creiamo la struttura delle directory per la configurazione di Neovim:

```bash
mkdir -p ~/.config/nvim/lua/plugins
```

La struttura finale sarà:
```
~/.config/nvim/
├── init.lua                 # File principale
└── lua/
    └── plugins/
        ├── mason.lua        # Config Mason (package manager)
        ├── cmp.lua          # Config nvim-cmp (autocompletamento)
        ├── treesitter.lua   # Config Treesitter
        ├── telescope.lua    # Config Telescope
        ├── oil.lua          # Config Oil
        ├── rustacean.lua    # Config Rustaceanvim
        ├── gitsigns.lua     # Config Gitsigns (Git integration)
        ├── fugitive.lua     # Config Fugitive (Git commands)
        ├── which-key.lua    # Config Which-Key (keybinding hints)
        ├── trouble.lua      # Config Trouble (diagnostics list)
        ├── fidget.lua       # Config Fidget (LSP notifications)
        ├── lualine.lua      # Config Lualine (statusline)
        └── colorscheme.lua  # Config Gruvbox Material
```

---

## Configurazione Lazy.nvim

Creiamo il file `~/.config/nvim/init.lua`:

```lua
-- init.lua
-- Configurazione base di Neovim

-- Opzioni generali
vim.g.mapleader = " "  -- Space come leader key
vim.g.maplocalleader = " "

-- Impostazioni UI
vim.opt.number = true          -- Numeri di riga
vim.opt.relativenumber = true  -- Numeri relativi
vim.opt.mouse = 'a'            -- Abilita mouse
vim.opt.ignorecase = true      -- Ignora case nella ricerca
vim.opt.smartcase = true       -- Case-sensitive se maiuscole presenti
vim.opt.hlsearch = false       -- Non evidenziare ricerche
vim.opt.wrap = false           -- Non wrappare le righe
vim.opt.breakindent = true     -- Mantieni indentazione quando wrapped
vim.opt.tabstop = 4            -- Tab = 4 spazi
vim.opt.shiftwidth = 4         -- Indentazione = 4 spazi
vim.opt.expandtab = true       -- Usa spazi invece di tab
vim.opt.termguicolors = true   -- Abilita colori 24-bit

-- Clipboard di sistema
vim.opt.clipboard = 'unnamedplus'

-- Salvataggio automatico undo
vim.opt.undofile = true

-- Tempo di aggiornamento ridotto
vim.opt.updatetime = 250
vim.opt.signcolumn = 'yes'

-- Bootstrap lazy.nvim
local lazypath = vim.fn.stdpath("data") .. "/lazy/lazy.nvim"
if not vim.loop.fs_stat(lazypath) then
  vim.fn.system({
    "git",
    "clone",
    "--filter=blob:none",
    "https://github.com/folke/lazy.nvim.git",
    "--branch=stable",
    lazypath,
  })
end
vim.opt.rtp:prepend(lazypath)

-- Setup lazy.nvim con i plugin
require("lazy").setup({
  -- Importa tutti i file dalla directory plugins
  { import = "plugins" },
}, {
  checker = {
    enabled = true,  -- Controlla automaticamente aggiornamenti
    notify = false,  -- Non notificare
  },
  change_detection = {
    notify = false,  -- Non notificare cambiamenti config
  },
})

-- Toggle diagnostics
local diagnostics_active = true
local toggle_diagnostics = function()
  diagnostics_active = not diagnostics_active
  if diagnostics_active then
    vim.api.nvim_echo({ { "Show diagnostics" } }, false, {})
    vim.diagnostic.enable()
  else
    vim.api.nvim_echo({ { "Disable diagnostics" } }, false, {})
    vim.diagnostic.enable(false)
  end
end

local function open_terminal(command)
  command = command or os.getenv("SHELL")
  if os.getenv("TMUX") then
    if os.getenv("POETRY_ACTIVE") then
      command = "poetry run " .. command
    end
    --  TMUX session active
    print("silent !tmux split-window -l 10 " .. command .. "<cr>")
    vim.cmd("silent !tmux split-window -l 10 " .. command)
  else
    vim.cmd.vnew()
    vim.cmd.term(command)
    vim.cmd.wincmd("J")
    vim.api.nvim_win_set_height(0, 15)
  end
end

-- Keymaps generali
vim.keymap.set('n', '<leader>w', '<cmd>write<cr>', { desc = 'Save' })
vim.keymap.set('n', '<leader>q', '<cmd>quit<cr>', { desc = 'Quit' })
vim.keymap.set("n", "<M-n>", "<cmd>bnext<CR>", {})
vim.keymap.set("n", "<M-p>", "<cmd>bprev<CR>", {})
vim.keymap.set("n", "<Esc>", "<cmd>nohlsearch<CR>")
vim.keymap.set("t", "<esc><esc>", "<c-\\><c-n>", {})

vim.keymap.set("n", "<leader>tt", function()
  open_terminal()
end, { desc = "[T]erminal " })


-- Keymaps per diagnostics
vim.keymap.set('n', '[d', vim.diagnostic.goto_prev, { desc = 'Previous diagnostic' })
vim.keymap.set('n', ']d', vim.diagnostic.goto_next, { desc = 'Next diagnostic' })
vim.keymap.set('n', '<leader>e', vim.diagnostic.open_float, { desc = 'Show diagnostic error' })
vim.keymap.set("n", "<leader>xi", toggle_diagnostics, { desc = "Toggle [i]nline diagnostic" })
```

**Spiegazione keymaps diagnostics:**
- `<leader>e` (Space + e) mostra popup con errore/warning della riga corrente
- `<leader>xi` (Space + xi) abilita/disabilita la visualizzazione degli errori/warning
- `]d` salta al prossimo diagnostic (errore/warning)
- `[d` salta al diagnostic precedente

---

### Mason (`~/.config/nvim/lua/plugins/mason.lua`)

```lua
return {
  "williamboman/mason.nvim",
  dependencies = {
    "williamboman/mason-lspconfig.nvim",
    "jay-babu/mason-nvim-dap.nvim",
  },
  config = function()
    local mason = require("mason")
    local mason_lspconfig = require("mason-lspconfig")
    local mason_nvim_dap = require("mason-nvim-dap")
    
    -- Setup Mason
    mason.setup({
      ui = {
        icons = {
          package_installed = "✓",
          package_pending = "➜",
          package_uninstalled = "✗"
        }
      }
    })
    
    -- Setup Mason LSPConfig
    mason_lspconfig.setup({
      -- Questa lista è vuota perché per Rust usiamo rustaceanvim
      -- che gestisce rust-analyzer automaticamente
      ensure_installed = {},
    })
    
    -- Setup Mason DAP - installa automaticamente codelldb
    mason_nvim_dap.setup({
      ensure_installed = { "codelldb" },
      automatic_installation = true,
    })
    
    -- Keymap per aprire Mason
    vim.keymap.set('n', '<leader>m', '<cmd>Mason<cr>', { desc = 'Open Mason' })
  end,
}
```

**Spiegazione**:
- Mason è un package manager per Neovim che gestisce LSP servers, DAP adapters, linters e formatters
- `mason-nvim-dap` installa **automaticamente** codelldb al primo avvio
- Non includiamo rust-analyzer qui perché rustaceanvim lo usa direttamente dal sistema

### nvim-cmp (`~/.config/nvim/lua/plugins/cmp.lua`)

```lua
return {
  "hrsh7th/nvim-cmp",
  event = "InsertEnter",
  dependencies = {
    -- Snippet Engine
    {
      "L3MON4D3/LuaSnip",
      version = "v2.*",
      build = "make install_jsregexp",
    },
    
    -- Autocompletion sources
    "saadparwaiz1/cmp_luasnip",     -- Snippet completions
    "hrsh7th/cmp-nvim-lsp",         -- LSP completions
    "hrsh7th/cmp-buffer",           -- Buffer completions
    "hrsh7th/cmp-path",             -- Path completions
    "hrsh7th/cmp-nvim-lua",         -- Neovim Lua API completions
    
    -- Snippet collection (optional)
    "rafamadriz/friendly-snippets",
  },
  config = function()
    local cmp = require("cmp")
    local luasnip = require("luasnip")
    
    -- Carica snippet da friendly-snippets
    require("luasnip.loaders.from_vscode").lazy_load()
    
    cmp.setup({
      snippet = {
        expand = function(args)
          luasnip.lsp_expand(args.body)
        end,
      },
      
      mapping = cmp.mapping.preset.insert({
        ["<C-k>"] = cmp.mapping.select_prev_item(), -- Seleziona precedente
        ["<C-j>"] = cmp.mapping.select_next_item(), -- Seleziona successivo
        ["<C-b>"] = cmp.mapping.scroll_docs(-4),
        ["<C-f>"] = cmp.mapping.scroll_docs(4),
        ["<C-Space>"] = cmp.mapping.complete(),     -- Mostra completamento
        ["<C-e>"] = cmp.mapping.abort(),            -- Chiudi
        ["<CR>"] = cmp.mapping.confirm({ select = false }), -- Conferma selezione
        
        -- Tab per navigare snippet
        ["<Tab>"] = cmp.mapping(function(fallback)
          if cmp.visible() then
            cmp.select_next_item()
          elseif luasnip.expand_or_jumpable() then
            luasnip.expand_or_jump()
          else
            fallback()
          end
        end, { "i", "s" }),
        
        ["<S-Tab>"] = cmp.mapping(function(fallback)
          if cmp.visible() then
            cmp.select_prev_item()
          elseif luasnip.jumpable(-1) then
            luasnip.jump(-1)
          else
            fallback()
          end
        end, { "i", "s" }),
      }),
      
      -- Sources per autocompletamento (ordine = priorità)
      sources = cmp.config.sources({
        { name = "nvim_lsp" },   -- Da LSP
        { name = "luasnip" },    -- Da snippet
        { name = "buffer" },     -- Da buffer corrente
        { name = "path" },       -- Da filesystem
        { name = "nvim_lua" },   -- Da Neovim Lua API
      }),
      
      -- Formattazione voci completamento
      formatting = {
        format = function(entry, item)
          -- Mostra da quale source proviene
          item.menu = ({
            nvim_lsp = "[LSP]",
            luasnip = "[Snippet]",
            buffer = "[Buffer]",
            path = "[Path]",
            nvim_lua = "[Lua]",
          })[entry.source.name]
          return item
        end,
      },
    })
  end,
}
```

**Spiegazione**:
- nvim-cmp fornisce autocompletamento intelligente con multiple sources
- LuaSnip è il motore per gli snippet
- Configurato per integrarsi perfettamente con rust-analyzer LSP

### Gitsigns (`~/.config/nvim/lua/plugins/gitsigns.lua`)

```lua
return {
  "lewis6991/gitsigns.nvim",
  event = { "BufReadPre", "BufNewFile" },
  config = function()
    require("gitsigns").setup({
      signs = {
        add          = { text = '┃' },
        change       = { text = '┃' },
        delete       = { text = '_' },
        topdelete    = { text = '‾' },
        changedelete = { text = '~' },
        untracked    = { text = '┆' },
      },
      
      on_attach = function(bufnr)
        local gs = package.loaded.gitsigns
        
        -- Keymaps
        local function map(mode, l, r, opts)
          opts = opts or {}
          opts.buffer = bufnr
          vim.keymap.set(mode, l, r, opts)
        end
        
        -- Navigazione tra hunks
        map('n', ']c', function()
          if vim.wo.diff then return ']c' end
          vim.schedule(function() gs.next_hunk() end)
          return '<Ignore>'
        end, { expr = true, desc = 'Next Git hunk' })
        
        map('n', '[c', function()
          if vim.wo.diff then return '[c' end
          vim.schedule(function() gs.prev_hunk() end)
          return '<Ignore>'
        end, { expr = true, desc = 'Previous Git hunk' })
        
        -- Actions
        map('n', '<leader>hs', gs.stage_hunk, { desc = 'Stage hunk' })
        map('n', '<leader>hr', gs.reset_hunk, { desc = 'Reset hunk' })
        map('v', '<leader>hs', function() gs.stage_hunk {vim.fn.line('.'), vim.fn.line('v')} end, { desc = 'Stage hunk' })
        map('v', '<leader>hr', function() gs.reset_hunk {vim.fn.line('.'), vim.fn.line('v')} end, { desc = 'Reset hunk' })
        map('n', '<leader>hS', gs.stage_buffer, { desc = 'Stage buffer' })
        map('n', '<leader>hu', gs.undo_stage_hunk, { desc = 'Undo stage hunk' })
        map('n', '<leader>hR', gs.reset_buffer, { desc = 'Reset buffer' })
        map('n', '<leader>hp', gs.preview_hunk, { desc = 'Preview hunk' })
        map('n', '<leader>hb', function() gs.blame_line{full=true} end, { desc = 'Blame line' })
        map('n', '<leader>hd', gs.diffthis, { desc = 'Diff this' })
        map('n', '<leader>hD', function() gs.diffthis('~') end, { desc = 'Diff this ~' })
        
        -- Text object
        map({'o', 'x'}, 'ih', ':<C-U>Gitsigns select_hunk<CR>', { desc = 'Select hunk' })
      end,
    })
  end,
}
```

**Spiegazione**:
- Mostra modifiche Git nella sign column (barra a sinistra)
- Navigazione rapida tra modifiche con `]c` e `[c`
- Stage/reset hunks direttamente da Neovim
- Preview delle modifiche inline

### Fugitive (`~/.config/nvim/lua/plugins/fugitive.lua`)

```lua
return {
  "tpope/vim-fugitive",
  cmd = { "Git", "G", "Gdiffsplit", "Gread", "Gwrite", "Ggrep", "GMove", "GRename", "GDelete", "GBrowse" },
  keys = {
    { "<leader>gs", "<cmd>Git<cr>", desc = "Git status" },
    { "<leader>gc", "<cmd>Git commit<cr>", desc = "Git commit" },
    { "<leader>gp", "<cmd>Git push<cr>", desc = "Git push" },
    { "<leader>gl", "<cmd>Git pull<cr>", desc = "Git pull" },
    { "<leader>gb", "<cmd>Git blame<cr>", desc = "Git blame" },
    { "<leader>gd", "<cmd>Gdiffsplit<cr>", desc = "Git diff split" },
  },
}
```

**Spiegazione**:
- Interfaccia completa a Git direttamente da Neovim
- Comandi Git nativi (commit, push, pull, blame, diff)
- Si integra perfettamente con Gitsigns

### Which-Key (`~/.config/nvim/lua/plugins/which-key.lua`)

```lua
return {
  "folke/which-key.nvim",
  event = "VeryLazy",
  init = function()
    vim.o.timeout = true
    vim.o.timeoutlen = 300
  end,
  config = function()
    local wk = require("which-key")
    
    wk.setup({
      preset = "modern",
      win = {
        border = "rounded",
      },
    })
    
    -- Registra gruppi di keybindings
    wk.add({
      { "<leader>f", group = "find" },
      { "<leader>r", group = "rust" },
      { "<leader>g", group = "git" },
      { "<leader>h", group = "hunk" },
      { "<leader>d", group = "debug" },
      { "<leader>x", group = "diagnostics" },
    })
  end,
}
```

**Spiegazione**:
- Mostra automaticamente popup con keybindings disponibili
- Aiuta a scoprire e ricordare i comandi

### Trouble (`~/.config/nvim/lua/plugins/trouble.lua`)

```lua
return {
  "folke/trouble.nvim",
  dependencies = { "nvim-tree/nvim-web-devicons" },
  cmd = { "Trouble" },
  keys = {
    { "<leader>xx", "<cmd>Trouble diagnostics toggle<cr>", desc = "Diagnostics (Trouble)" },
    { "<leader>xX", "<cmd>Trouble diagnostics toggle filter.buf=0<cr>", desc = "Buffer Diagnostics (Trouble)" },
    { "<leader>xs", "<cmd>Trouble symbols toggle focus=false<cr>", desc = "Symbols (Trouble)" },
    { "<leader>xl", "<cmd>Trouble lsp toggle focus=false win.position=right<cr>", desc = "LSP Definitions / references / ... (Trouble)" },
    { "<leader>xL", "<cmd>Trouble loclist toggle<cr>", desc = "Location List (Trouble)" },
    { "<leader>xQ", "<cmd>Trouble qflist toggle<cr>", desc = "Quickfix List (Trouble)" },
  },
  opts = {
    -- Configurazione opzionale
  },
}
```

**Spiegazione**:
- Mostra lista organizzata di tutti gli errori/warning del progetto
- Visualizzazione migliore della quickfix list di default
- Integrazione con LSP per diagnostics, references, definitions

### Fidget (`~/.config/nvim/lua/plugins/fidget.lua`)

```lua
return {
  "j-hui/fidget.nvim",
  opts = {
    notification = {
      window = {
        winblend = 0,  -- Trasparenza finestra (0-100)
        border = "none",
      },
    },
    progress = {
      display = {
        done_icon = "✓",
      },
    },
  },
}
```

**Spiegazione**:
- Mostra notifiche LSP in basso a destra (es. "rust-analyzer: indexing...")
- Progress indicator quando rust-analyzer sta analizzando il codice
- Interfaccia pulita e non invasiva

### Treesitter (`~/.config/nvim/lua/plugins/treesitter.lua`)

```lua
return {
  "nvim-treesitter/nvim-treesitter",
  build = ":TSUpdate",
  event = { "BufReadPost", "BufNewFile" },
  dependencies = {
    "nvim-treesitter/nvim-treesitter-textobjects",  -- Text objects avanzati
  },
  config = function()
    require("nvim-treesitter.configs").setup({
      -- Linguaggi da installare automaticamente
      ensure_installed = {
        "rust",
        "toml",
        "lua",
        "vim",
        "vimdoc",
        "markdown",
        "markdown_inline",
        "json",
        "yaml",
      },
      
      -- Installa parser in modo sincrono
      sync_install = false,
      
      -- Installazione automatica parser mancanti
      auto_install = true,
      
      -- Syntax highlighting
      highlight = {
        enable = true,
        additional_vim_regex_highlighting = false,
      },
      
      -- Indentazione basata su treesitter
      indent = {
        enable = true,
      },
      
      -- Selezione incrementale
      incremental_selection = {
        enable = true,
        keymaps = {
          init_selection = "<CR>",
          node_incremental = "<CR>",
          node_decremental = "<BS>",
          scope_incremental = "<TAB>",
        },
      },
      
      -- Text objects
      textobjects = {
        select = {
          enable = true,
          lookahead = true,
          keymaps = {
            ["af"] = "@function.outer",
            ["if"] = "@function.inner",
            ["ac"] = "@class.outer",
            ["ic"] = "@class.inner",
          },
        },
      },
    })
  end,
}
```

### Telescope (`~/.config/nvim/lua/plugins/telescope.lua`)

```lua
return {
    "nvim-telescope/telescope.nvim",
    branch = "0.1.x",
    dependencies = {
        "nvim-lua/plenary.nvim",
        {
            "nvim-telescope/telescope-fzf-native.nvim",
            build = "make",  -- Compilazione per performance migliori
        },
    },
    config = function()
        local telescope = require("telescope")
        local actions = require("telescope.actions")

        telescope.setup({
            defaults = {
                path_display = { "truncate" },
                mappings = {
                    i = {
                        ["<C-k>"] = actions.move_selection_previous,
                        ["<C-j>"] = actions.move_selection_next,
                        ["<C-q>"] = actions.send_selected_to_qflist + actions.open_qflist,
                    },
                },
                file_ignore_patterns = {
                    "^.git/",
                    "node_modules",
                    ".cache"
                }
            },
            pickers = {
                find_files = {
                    hidden = true,  -- Mostra file nascosti
                },
            },
        })

        -- Carica estensione fzf
        telescope.load_extension("fzf")

        -- Keymaps
        local builtin = require("telescope.builtin")
        vim.keymap.set('n', '<leader>ff', builtin.find_files, { desc = 'Find Files' })
        vim.keymap.set('n', '<leader>fg', builtin.live_grep, { desc = 'Live Grep' })
        vim.keymap.set('n', '<leader>fb', builtin.buffers, { desc = 'Buffers' })
        vim.keymap.set('n', '<leader>fh', builtin.help_tags, { desc = 'Help Tags' })
        vim.keymap.set('n', '<leader>fr', builtin.lsp_references, { desc = 'LSP References' })
        vim.keymap.set('n', '<leader>fs', builtin.lsp_document_symbols, { desc = 'Document Symbols' })
    end,
}
```

### Oil.nvim (`~/.config/nvim/lua/plugins/oil.lua`)

```lua
return {
  "stevearc/oil.nvim",
  dependencies = { "nvim-tree/nvim-web-devicons" },
  config = function()
    require("oil").setup({
      -- Colonne da mostrare
      columns = {
        "icon",
        "permissions",
        "size",
        "mtime",
      },
      
      -- Keymaps dentro oil
      keymaps = {
        ["g?"] = "actions.show_help",
        ["<CR>"] = "actions.select",
        ["<C-v>"] = "actions.select_vsplit",
        ["<C-s>"] = "actions.select_split",
        ["<C-t>"] = "actions.select_tab",
        ["<C-p>"] = "actions.preview",
        ["<C-c>"] = "actions.close",
        ["<C-r>"] = "actions.refresh",
        ["-"] = "actions.parent",
        ["_"] = "actions.open_cwd",
        ["`"] = "actions.cd",
        ["~"] = "actions.tcd",
        ["gs"] = "actions.change_sort",
        ["gx"] = "actions.open_external",
        ["g."] = "actions.toggle_hidden",
      },
      
      -- Mostra file nascosti di default
      view_options = {
        show_hidden = true,
      },
    })
    
    -- Apri oil con -
    vim.keymap.set("n", "-", "<CMD>Oil<CR>", { desc = "Open parent directory" })
  end,
}
```

### Rustaceanvim (`~/.config/nvim/lua/plugins/rustacean.lua`)

```lua
return {
  "mrcjkb/rustaceanvim",
  version = "^5",
  ft = { "rust" },  -- Carica solo per file Rust
  config = function()
    vim.g.rustaceanvim = {
      -- LSP settings
      server = {
        on_attach = function(client, bufnr)
          -- Keymaps per LSP
          local opts = { buffer = bufnr, noremap = true, silent = true }
          
          vim.keymap.set('n', 'gd', vim.lsp.buf.definition, opts)
          vim.keymap.set('n', 'gD', vim.lsp.buf.declaration, opts)
          vim.keymap.set('n', 'gi', vim.lsp.buf.implementation, opts)
          vim.keymap.set('n', 'gr', vim.lsp.buf.references, opts)
          vim.keymap.set('n', 'K', vim.lsp.buf.hover, opts)
          vim.keymap.set('n', '<leader>rn', vim.lsp.buf.rename, opts)
          vim.keymap.set('n', '<leader>ca', vim.lsp.buf.code_action, opts)
          vim.keymap.set('n', '<leader>f', vim.lsp.buf.format, opts)
          
          -- Rustacean specific
          vim.keymap.set('n', '<leader>rd', '<cmd>RustLsp debuggables<cr>', opts)
          vim.keymap.set('n', '<leader>rr', '<cmd>RustLsp runnables<cr>', opts)
          vim.keymap.set('n', '<leader>rt', '<cmd>RustLsp testables<cr>', opts)
          vim.keymap.set('n', '<leader>re', '<cmd>RustLsp expandMacro<cr>', opts)
        end,
        
        default_settings = {
          ['rust-analyzer'] = {
            cargo = {
              allFeatures = true,
              loadOutDirsFromCheck = true,
              buildScripts = {
                enable = true,
              },
            },
            -- Usa 'check' invece di 'checkOnSave'
            check = {
              command = "clippy",  -- Usa clippy invece di check
            },
            procMacro = {
              enable = true,
            },
            diagnostics = {
              enable = true,
              experimental = {
                enable = true,
              },
            },
          },
        },
      },
      
      -- DAP (Debug Adapter Protocol)
      dap = {
        adapter = {
          type = "server",
          port = "${port}",
          executable = {
            command = vim.fn.stdpath("data") .. "/mason/bin/codelldb",
            args = { "--port", "${port}" },
          },
        },
      },
    }
  end,
  dependencies = {
    -- Plugin per debugging
    {
      "mfussenegger/nvim-dap",
      config = function()
        local dap = require("dap")
        
        -- Keymaps per debugging
        vim.keymap.set('n', '<F5>', dap.continue, { desc = 'Debug: Start/Continue' })
        vim.keymap.set('n', '<F10>', dap.step_over, { desc = 'Debug: Step Over' })
        vim.keymap.set('n', '<F11>', dap.step_into, { desc = 'Debug: Step Into' })
        vim.keymap.set('n', '<F12>', dap.step_out, { desc = 'Debug: Step Out' })
        vim.keymap.set('n', '<leader>b', dap.toggle_breakpoint, { desc = 'Debug: Toggle Breakpoint' })
      end,
    },
    
    -- UI per il debugger
    {
      "rcarriga/nvim-dap-ui",
      dependencies = { "nvim-neotest/nvim-nio" },
      config = function()
        local dap, dapui = require("dap"), require("dapui")
        
        dapui.setup()
        
        -- Apri/chiudi automaticamente UI
        dap.listeners.after.event_initialized["dapui_config"] = function()
          dapui.open()
        end
        dap.listeners.before.event_terminated["dapui_config"] = function()
          dapui.close()
        end
        dap.listeners.before.event_exited["dapui_config"] = function()
          dapui.close()
        end
        
        vim.keymap.set('n', '<leader>du', dapui.toggle, { desc = 'Debug: Toggle UI' })
      end,
    },
  },
}
```

### 6.9 Lualine (`~/.config/nvim/lua/plugins/lualine.lua`)

```lua
return {
  "nvim-lualine/lualine.nvim",
  dependencies = { "nvim-tree/nvim-web-devicons" },
  config = function()
    require("lualine").setup({
      options = {
        theme = "gruvbox-material",
        component_separators = { left = '|', right = '|' },
        section_separators = { left = '', right = '' },
        globalstatus = true,  -- Una statusline per tutte le finestre
      },
      sections = {
        lualine_a = { 'mode' },
        lualine_b = { 'branch', 'diff' },
        lualine_c = { 
          {
            'filename',
            path = 1,  -- 0 = solo nome, 1 = relativo, 2 = assoluto
          }
        },
        lualine_x = {
          {
            'diagnostics',
            sources = { 'nvim_diagnostic' },
            symbols = { error = ' ', warn = ' ', info = ' ', hint = ' ' },
          },
          'encoding',
          'fileformat',
          'filetype',
        },
        lualine_y = { 'progress' },
        lualine_z = { 'location' },
      },
      inactive_sections = {
        lualine_a = {},
        lualine_b = {},
        lualine_c = { 'filename' },
        lualine_x = { 'location' },
        lualine_y = {},
        lualine_z = {},
      },
      extensions = { 'fugitive', 'oil', 'trouble' },
    })
  end,
}
```

### Gruvbox Material (`~/.config/nvim/lua/plugins/colorscheme.lua`)

```lua
return {
  "sainnhe/gruvbox-material",
  lazy = false,     -- Carica immediatamente
  priority = 1000,  -- Carica prima di altri plugin
  config = function()
    -- Configura il colorscheme
    vim.g.gruvbox_material_background = 'medium'  -- 'soft', 'medium', 'hard'
    vim.g.gruvbox_material_better_performance = 1
    vim.g.gruvbox_material_enable_italic = 1
    
    -- Applica il colorscheme
    vim.cmd.colorscheme('gruvbox-material')
  end,
}
```

## Test e Verifica

### Prima apertura di Neovim

```bash
nvim
```

Al primo avvio, Lazy.nvim installerà automaticamente tutti i plugin. Vedrai una finestra con il progresso dell'installazione. Mason installerà anche automaticamente codelldb in background.

### Verifica installazione plugin

Dentro Neovim, digita:
```vim
:Lazy
```

Dovresti vedere tutti i plugin installati con segno di spunta verde.

Per verificare che codelldb sia stato installato:
```vim
:Mason
```

Dovresti vedere `codelldb` con il segno ✓ (installato).

### Verifica Treesitter

```vim
:checkhealth nvim-treesitter
```

### Test con progetto Rust

Crea un progetto di test:
```bash
cargo new test_project
cd test_project
nvim src/main.rs
```

Dovresti vedere:
- ✅ Syntax highlighting colorato
- ✅ Autocompletamento (premi `Ctrl+Space`)
- ✅ Diagnostica inline (errori/warning sottolineati)
- ✅ Hover documentation (premi `K` su una funzione)

### Test debugging

Nel file `src/main.rs`, aggiungi:
```rust
fn main() {
    let x = 5;
    let y = 10;
    println!("Sum: {}", x + y);  // Metti breakpoint qui
}
```

- Premi `<leader>b` sulla riga del println per aggiungere breakpoint
- Premi `<leader>rd` per vedere opzioni di debug
- Seleziona "Debug" e premi Enter
- Usa `F10` per step over, `F11` per step into

---

## Comandi Utili

### Plugin Management (Lazy.nvim)
- `:Lazy` - Apri interfaccia Lazy
- `:Lazy update` - Aggiorna tutti i plugin
- `:Lazy sync` - Installa/aggiorna/rimuovi plugin

### Mason (Package Manager)
- `:Mason` - Apri interfaccia Mason
- `:MasonInstall <package>` - Installa un package
- `:MasonUninstall <package>` - Rimuovi un package
- `<leader>m` - Apri Mason

### Autocompletamento (nvim-cmp)
- `Ctrl+Space` - Trigger completamento manualmente
- `Ctrl+j/k` - Naviga suggerimenti
- `Tab/Shift+Tab` - Naviga snippet
- `Enter` - Conferma selezione
- `Ctrl+e` - Chiudi menu completamento

### Telescope
- `<leader>ff` - Find files
- `<leader>fg` - Live grep (ricerca in tutti i file)
- `<leader>fb` - Lista buffer aperti
- `<leader>fr` - Trova riferimenti LSP
- `<leader>fs` - Simboli del documento

### Oil.nvim
- `-` - Apri directory corrente
- `<CR>` - Entra in directory/apri file
- `<C-s>` - Apri in split orizzontale
- `<C-v>` - Apri in split verticale
- `g.` - Toggle file nascosti

### LSP
- `gd` - Go to definition
- `gr` - Go to references
- `K` - Hover documentation
- `<leader>rn` - Rename symbol
- `<leader>ca` - Code actions
- `<leader>f` - Format document

### Diagnostics
- `<leader>e` - Show diagnostic popup (errore/warning riga corrente)
- `]d` - Next diagnostic
- `[d` - Previous diagnostic

### Rust Specific
- `<leader>rr` - Rust runnables (esegui main, esempi, etc.)
- `<leader>rt` - Rust testables (esegui test)
- `<leader>rd` - Rust debuggables (debug)
- `<leader>re` - Expand macro

### Debugging
- `<F5>` - Start/Continue debug
- `<F10>` - Step over
- `<F11>` - Step into
- `<F12>` - Step out
- `<leader>b` - Toggle breakpoint
- `<leader>du` - Toggle debug UI

### Git (Gitsigns)
- `]c` / `[c` - Next/Previous hunk
- `<leader>hs` - Stage hunk
- `<leader>hr` - Reset hunk
- `<leader>hS` - Stage buffer
- `<leader>hp` - Preview hunk
- `<leader>hb` - Blame line
- `<leader>hd` - Diff this
- `ih` - Select hunk (text object)

### Git (Fugitive)
- `<leader>gs` - Git status
- `<leader>gc` - Git commit
- `<leader>gp` - Git push
- `<leader>gl` - Git pull
- `<leader>gb` - Git blame
- `<leader>gd` - Git diff split

### Which-Key
- `<leader>` - Mostra tutti i keybindings del leader

### Trouble (Diagnostics)
- `<leader>xx` - Toggle diagnostics
- `<leader>xX` - Toggle buffer diagnostics
- `<leader>xs` - Toggle symbols
- `<leader>xl` - Toggle LSP info
- `<leader>xQ` - Toggle quickfix

### Treesitter
- `:TSUpdate` - Aggiorna parser
- `:TSInstall <lang>` - Installa parser per linguaggio
- `<CR>` in visual mode - Selezione incrementale

---

## Troubleshooting

### Problema: Treesitter non compila parser
```bash
# Installa dipendenze C
sudo apt install -y gcc g++

# In Neovim
:TSInstall rust
```

### Warning: tree-sitter executable not found
Questo è un warning comune e **completamente normale**:

```
⚠️ WARNING tree-sitter executable not found (parser generator, only needed for :TSInstallFromGrammar)
```

**Per eliminare il warning (opzionale):**
```bash
# Installa tree-sitter CLI
cargo install tree-sitter-cli
```

Dopo l'installazione, riavvia Neovim e il warning sparirà. Ma ripeto: è puramente estetico, la funzionalità è già completa!

### Warning: mini.icons is not installed
```
⚠️ WARNING mini.icons is not installed
✅ OK nvim-web-devicons is installed
```

**Cosa significa:**
- which-key può usare `mini.icons` OPPURE `nvim-web-devicons` per le icone
- La nostra config usa `nvim-web-devicons` (già installato con Oil e Trouble)
- Il warning è solo informativo, le icone funzionano perfettamente

### Problema: Telescope non trova file
```bash
# Verifica ripgrep e fd
which rg
which fd

# Ubuntu: Se mancano, reinstalla
sudo apt install ripgrep fd-find

# macOS: Se mancano, reinstalla
brew install ripgrep fd
```

## Prossimi Passi e Personalizzazioni

Ora hai un IDE Rust **completo e professionale**! 🎉

**Cosa hai ottenuto:**
- ✅ Autocompletamento intelligente con nvim-cmp
- ✅ Integrazione Git completa (Gitsigns + Fugitive)
- ✅ Diagnostics organizzate (Trouble)
- ✅ Notifiche LSP eleganti (Fidget)
- ✅ Guida ai keybindings (Which-Key)
- ✅ Debugging visuale completo
- ✅ Syntax highlighting avanzato
- ✅ File explorer moderno (Oil)
- ✅ Ricerca potente (Telescope)
- ✅ Statusline informativa (Lualine)

**Considera di aggiungere (opzionale):**

1. **toggleterm.nvim** - Terminale integrato e toggleable
2. **nvim-autopairs** - Chiusura automatica di parentesi/virgolette
3. **Comment.nvim** - Commenta codice facilmente con `gc`
4. **indent-blankline.nvim** - Mostra guide di indentazione
5. **nvim-colorizer.lua** - Evidenzia codici colore nel codice
6. **rust-tools.nvim** - Tools extra per Rust (hover actions, ecc.)

**Risorse utili:**
- Documentazione Neovim: `:help` o https://neovim.io/doc/
- rust-analyzer docs: https://rust-analyzer.github.io/
- Awesome Neovim: https://github.com/rockerBOO/awesome-neovim

Buon coding! 🦀
