+++
title = "Complete Guide: Neovim 0.11 IDE for Rust on Ubuntu/MacOS"
date = 2025-10-15
description = "If you're a Rust developer looking for a text editor that's lightweight, fast, and incredibly customizable, Neovim is the perfect choice. Forget heavy traditional IDEs: with this guide, we'll transform Neovim into a modern and productive Rust development environment, tailored to your needs."

[taxonomies]
tags = ["rust", "neovim"]

[extra]
comments = true
+++

Welcome to this complete guide to installing and configuring Neovim 0.11 for Rust development! 🦀

![neovim-rust-configuration](https://github.com/user-attachments/assets/1bcf3da2-0c38-48cd-a1e6-faa84c78460d)

If you're a Rust developer looking for a text editor that's lightweight, fast, and incredibly customizable, Neovim is the perfect choice. Forget heavy traditional IDEs: with this guide, we'll transform Neovim into a modern and productive Rust development environment, tailored to your needs.

In this guide, we'll see how to:

Install the latest version of Neovim (0.11).

Configure from scratch a minimal development environment using Lua.

Integrate rust-analyzer through Neovim's native Language Server Protocol (LSP) to get IDE features like autocompletion, real-time diagnostics, and code navigation.

Set up syntax highlighting with Tree-sitter for more precise and faster code analysis.

By the end of this journey, you'll have a functional and performant Neovim configuration, ready to tackle any Rust project. Let's begin!


## Prerequisites and System Dependencies

### Installation on GNU/Linux (Ubuntu)
Before we start, let's install all necessary dependencies:

```bash
# Update the system
sudo apt update && sudo apt upgrade -y

# Plugin dependencies
sudo apt install -y ripgrep fd-find

# Build essentials (required to compile Treesitter parsers and some Rust extensions)
sudo apt install -y build-essential git curl

# Node.js (optional, only needed for some non-Rust LSPs)
sudo apt install -y nodejs

# Create a symlink for fd (on Ubuntu it's called fdfind)
sudo ln -s $(which fdfind) /usr/local/bin/fd 2>/dev/null || true
```

### MacOS

On macOS we'll use Homebrew as the package manager:

```bash
# Install Homebrew if not already installed
/bin/bash -c "$(curl -fsSL https://raw.githubusercontent.com/Homebrew/install/HEAD/install.sh)"

# Install Xcode Command Line Tools (contains gcc, git, etc.)
xcode-select --install

# Plugin dependencies
brew install ripgrep fd

# Node.js (optional, useful for other languages)
brew install node
```
**Dependency explanation:**
- **ripgrep**: extremely fast file search, used by Telescope
- **fd**: modern alternative to `find`, used by Telescope
- **build-essential**: gcc and g++ needed to compile Treesitter parsers
- **Node.js**: optional, but useful if you'll use other languages besides Rust

---

## Installing Neovim 0.11

### Installation on MacOS

To install Neovim, simply run:

```bash
brew install neovim
```

### Installation on GNU/Linux (Ubuntu)

For this guide, we'll use the Ubuntu GNU/Linux distribution as reference.

Let's install Neovim via snap:

```bash
# Install Neovim 0.11 via snap (--classic flag for full system access)
sudo snap install nvim --classic

# Verify the installation
nvim --version
```

You should see output similar to:
```
NVIM v0.11.4
Build type: Release
...
```

**Why use snap?**
- ✅ Quick installation (few seconds vs 10-15 minutes of compilation)
- ✅ Always updated to latest stable
- ✅ Automatic updates managed by snap
- ✅ No build dependencies needed

**Note**: The `--classic` flag is necessary because Neovim needs to freely access system files to function as an editor.

### Alternative method: Compiling from source

If you prefer to compile from source to have complete control, on Ubuntu you'll need:

```bash
# Install build dependencies
sudo apt install -y ninja-build gettext cmake unzip
```
while on MacOS:

```bash
# Install build dependencies
brew install ninja cmake gettext
```
We can now compile Neovim for our system:

```bash
# Download and compile
mkdir -p ~/build && cd ~/build
git clone https://github.com/neovim/neovim.git
cd neovim
git checkout v0.11.4
make CMAKE_BUILD_TYPE=Release
sudo make install
```

Pro: Complete control over version and build options  
Con: Requires 10-15 minutes and more dependencies

---

## Installing Rust Toolchain

Let's install Rust and the components necessary for development:

```bash
# Install rustup (Rust toolchain manager)
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Reload the PATH
source "$HOME/.cargo/env"

# Install necessary components if not previously installed
rustup component add rust-analyzer  # LSP for Rust
rustup component add rustfmt         # Formatter
rustup component add clippy          # Linter

# Verify the installation
rustc --version
cargo --version
rust-analyzer --version
```

**Important note**: 
- `rust-analyzer` is the language server that provides autocompletion, diagnostics, and other IDE features (LSP)
- **codelldb** (debugger) will be automatically installed via Mason (Neovim plugin) in the next section

---

## Configuration Structure

Let's create the directory structure for Neovim configuration:

```bash
mkdir -p ~/.config/nvim/lua/plugins
```

The final structure will be:
```
~/.config/nvim/
├── init.lua                 # Main file
└── lua/
    └── plugins/
        ├── mason.lua        # Mason config (package manager)
        ├── cmp.lua          # nvim-cmp config (autocompletion)
        ├── treesitter.lua   # Treesitter config
        ├── telescope.lua    # Telescope config
        ├── oil.lua          # Oil config
        ├── rustacean.lua    # Rustaceanvim config
        ├── gitsigns.lua     # Gitsigns config (Git integration)
        ├── fugitive.lua     # Fugitive config (Git commands)
        ├── which-key.lua    # Which-Key config (keybinding hints)
        ├── trouble.lua      # Trouble config (diagnostics list)
        ├── fidget.lua       # Fidget config (LSP notifications)
        ├── lualine.lua      # Lualine config (statusline)
        └── colorscheme.lua  # Gruvbox Material config
```

---

## Lazy.nvim Configuration

Let's create the `~/.config/nvim/init.lua` file:

```lua
-- init.lua
-- Neovim base configuration

-- General options
vim.g.mapleader = " "  -- Space as leader key
vim.g.maplocalleader = " "

-- UI settings
vim.opt.number = true          -- Line numbers
vim.opt.relativenumber = true  -- Relative numbers
vim.opt.mouse = 'a'            -- Enable mouse
vim.opt.ignorecase = true      -- Ignore case in search
vim.opt.smartcase = true       -- Case-sensitive if uppercase present
vim.opt.hlsearch = false       -- Don't highlight searches
vim.opt.wrap = false           -- Don't wrap lines
vim.opt.breakindent = true     -- Keep indentation when wrapped
vim.opt.tabstop = 4            -- Tab = 4 spaces
vim.opt.shiftwidth = 4         -- Indentation = 4 spaces
vim.opt.expandtab = true       -- Use spaces instead of tabs
vim.opt.termguicolors = true   -- Enable 24-bit colors

-- System clipboard
vim.opt.clipboard = 'unnamedplus'

-- Automatic undo save
vim.opt.undofile = true

-- Reduced update time
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

-- Setup lazy.nvim with plugins
require("lazy").setup({
  -- Import all files from plugins directory
  { import = "plugins" },
}, {
  checker = {
    enabled = true,  -- Automatically check for updates
    notify = false,  -- Don't notify
  },
  change_detection = {
    notify = false,  -- Don't notify config changes
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

-- General keymaps
vim.keymap.set('n', '<leader>w', '<cmd>write<cr>', { desc = 'Save' })
vim.keymap.set('n', '<leader>q', '<cmd>quit<cr>', { desc = 'Quit' })
vim.keymap.set("n", "<M-n>", "<cmd>bnext<CR>", {})
vim.keymap.set("n", "<M-p>", "<cmd>bprev<CR>", {})
vim.keymap.set("n", "<Esc>", "<cmd>nohlsearch<CR>")
vim.keymap.set("t", "<esc><esc>", "<c-\\><c-n>", {})

vim.keymap.set("n", "<leader>tt", function()
  open_terminal()
end, { desc = "[T]erminal " })


-- Keymaps for diagnostics
vim.keymap.set('n', '[d', vim.diagnostic.goto_prev, { desc = 'Previous diagnostic' })
vim.keymap.set('n', ']d', vim.diagnostic.goto_next, { desc = 'Next diagnostic' })
vim.keymap.set('n', '<leader>e', vim.diagnostic.open_float, { desc = 'Show diagnostic error' })
vim.keymap.set("n", "<leader>xi", toggle_diagnostics, { desc = "Toggle [i]nline diagnostic" })
```

**Diagnostics keymaps explanation:**
- `<leader>e` (Space + e) shows popup with error/warning of current line
- `<leader>xi` (Space + xi) enables/disables error/warning display
- `]d` jumps to next diagnostic (error/warning)
- `[d` jumps to previous diagnostic

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
      -- This list is empty because for Rust we use rustaceanvim
      -- which manages rust-analyzer automatically
      ensure_installed = {},
    })
    
    -- Setup Mason DAP - automatically installs codelldb
    mason_nvim_dap.setup({
      ensure_installed = { "codelldb" },
      automatic_installation = true,
    })
    
    -- Keymap to open Mason
    vim.keymap.set('n', '<leader>m', '<cmd>Mason<cr>', { desc = 'Open Mason' })
  end,
}
```

**Explanation**:
- Mason is a package manager for Neovim that handles LSP servers, DAP adapters, linters and formatters
- `mason-nvim-dap` **automatically** installs codelldb on first startup
- We don't include rust-analyzer here because rustaceanvim uses it directly from the system

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
    
    -- Load snippets from friendly-snippets
    require("luasnip.loaders.from_vscode").lazy_load()
    
    cmp.setup({
      snippet = {
        expand = function(args)
          luasnip.lsp_expand(args.body)
        end,
      },
      
      mapping = cmp.mapping.preset.insert({
        ["<C-k>"] = cmp.mapping.select_prev_item(), -- Select previous
        ["<C-j>"] = cmp.mapping.select_next_item(), -- Select next
        ["<C-b>"] = cmp.mapping.scroll_docs(-4),
        ["<C-f>"] = cmp.mapping.scroll_docs(4),
        ["<C-Space>"] = cmp.mapping.complete(),     -- Show completion
        ["<C-e>"] = cmp.mapping.abort(),            -- Close
        ["<CR>"] = cmp.mapping.confirm({ select = false }), -- Confirm selection
        
        -- Tab to navigate snippets
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
      
      -- Sources for autocompletion (order = priority)
      sources = cmp.config.sources({
        { name = "nvim_lsp" },   -- From LSP
        { name = "luasnip" },    -- From snippets
        { name = "buffer" },     -- From current buffer
        { name = "path" },       -- From filesystem
        { name = "nvim_lua" },   -- From Neovim Lua API
      }),
      
      -- Completion entries formatting
      formatting = {
        format = function(entry, item)
          -- Show which source it comes from
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

**Explanation**:
- nvim-cmp provides intelligent autocompletion with multiple sources
- LuaSnip is the snippet engine
- Configured to integrate perfectly with rust-analyzer LSP

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
        
        -- Navigate between hunks
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

**Explanation**:
- Shows Git changes in the sign column (left bar)
- Quick navigation between changes with `]c` and `[c`
- Stage/reset hunks directly from Neovim
- Inline preview of changes

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

**Explanation**:
- Complete Git interface directly from Neovim
- Native Git commands (commit, push, pull, blame, diff)
- Integrates perfectly with Gitsigns

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
    
    -- Register keybinding groups
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

**Explanation**:
- Automatically shows popup with available keybindings
- Helps discover and remember commands

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
    -- Optional configuration
  },
}
```

**Explanation**:
- Shows organized list of all project errors/warnings
- Better visualization than default quickfix list
- LSP integration for diagnostics, references, definitions

### Fidget (`~/.config/nvim/lua/plugins/fidget.lua`)

```lua
return {
  "j-hui/fidget.nvim",
  opts = {
    notification = {
      window = {
        winblend = 0,  -- Window transparency (0-100)
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

**Explanation**:
- Shows LSP notifications in bottom right (e.g., "rust-analyzer: indexing...")
- Progress indicator when rust-analyzer is analyzing code
- Clean and non-invasive interface

### Treesitter (`~/.config/nvim/lua/plugins/treesitter.lua`)

```lua
return {
  "nvim-treesitter/nvim-treesitter",
  build = ":TSUpdate",
  event = { "BufReadPost", "BufNewFile" },
  dependencies = {
    "nvim-treesitter/nvim-treesitter-textobjects",  -- Advanced text objects
  },
  config = function()
    require("nvim-treesitter.configs").setup({
      -- Languages to install automatically
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
      
      -- Install parsers synchronously
      sync_install = false,
      
      -- Automatic installation of missing parsers
      auto_install = true,
      
      -- Syntax highlighting
      highlight = {
        enable = true,
        additional_vim_regex_highlighting = false,
      },
      
      -- Treesitter-based indentation
      indent = {
        enable = true,
      },
      
      -- Incremental selection
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
            build = "make",  -- Compilation for better performance
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
                    hidden = true,  -- Show hidden files
                },
            },
        })

        -- Load fzf extension
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
      -- Columns to show
      columns = {
        "icon",
        "permissions",
        "size",
        "mtime",
      },
      
      -- Keymaps inside oil
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
      
      -- Show hidden files by default
      view_options = {
        show_hidden = true,
      },
    })
    
    -- Open oil with -
    vim.keymap.set("n", "-", "<CMD>Oil<CR>", { desc = "Open parent directory" })
  end,
}
```

### Rustaceanvim (`~/.config/nvim/lua/plugins/rustacean.lua`)

```lua
return {
  "mrcjkb/rustaceanvim",
  version = "^5",
  ft = { "rust" },  -- Load only for Rust files
  config = function()
    vim.g.rustaceanvim = {
      -- LSP settings
      server = {
        on_attach = function(client, bufnr)
          -- LSP keymaps
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
            -- Use 'check' instead of 'checkOnSave'
            check = {
              command = "clippy",  -- Use clippy instead of check
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
    -- Debugging plugin
    {
      "mfussenegger/nvim-dap",
      config = function()
        local dap = require("dap")
        
        -- Debugging keymaps
        vim.keymap.set('n', '<F5>', dap.continue, { desc = 'Debug: Start/Continue' })
        vim.keymap.set('n', '<F10>', dap.step_over, { desc = 'Debug: Step Over' })
        vim.keymap.set('n', '<F11>', dap.step_into, { desc = 'Debug: Step Into' })
        vim.keymap.set('n', '<F12>', dap.step_out, { desc = 'Debug: Step Out' })
        vim.keymap.set('n', '<leader>b', dap.toggle_breakpoint, { desc = 'Debug: Toggle Breakpoint' })
      end,
    },
    
    -- Debugger UI
    {
      "rcarriga/nvim-dap-ui",
      dependencies = { "nvim-neotest/nvim-nio" },
      config = function()
        local dap, dapui = require("dap"), require("dapui")
        
        dapui.setup()
        
        -- Automatically open/close UI
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
        globalstatus = true,  -- One statusline for all windows
      },
      sections = {
        lualine_a = { 'mode' },
        lualine_b = { 'branch', 'diff' },
        lualine_c = { 
          {
            'filename',
            path = 1,  -- 0 = name only, 1 = relative, 2 = absolute
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
  lazy = false,     -- Load immediately
  priority = 1000,  -- Load before other plugins
  config = function()
    -- Configure the colorscheme
    vim.g.gruvbox_material_background = 'medium'  -- 'soft', 'medium', 'hard'
    vim.g.gruvbox_material_better_performance = 1
    vim.g.gruvbox_material_enable_italic = 1
    
    -- Apply the colorscheme
    vim.cmd.colorscheme('gruvbox-material')
  end,
}
```

## Testing and Verification

### First Neovim launch

```bash
nvim
```

On first startup, Lazy.nvim will automatically install all plugins. You'll see a window with the installation progress. Mason will also install codelldb automatically in the background.

### Verify plugin installation

Inside Neovim, type:
```vim
:Lazy
```

You should see all plugins installed with a green checkmark.

To verify that codelldb was installed:
```vim
:Mason
```

You should see `codelldb` with the ✓ (installed) sign.

### Verify Treesitter

```vim
:checkhealth nvim-treesitter
```

### Test with Rust project

Create a test project:
```bash
cargo new test_project
cd test_project
nvim src/main.rs
```

You should see:
- ✅ Colored syntax highlighting
- ✅ Autocompletion (press `Ctrl+Space`)
- ✅ Inline diagnostics (errors/warnings underlined)
- ✅ Hover documentation (press `K` on a function)

### Test debugging

In the `src/main.rs` file, add:
```rust
fn main() {
    let x = 5;
    let y = 10;
    println!("Sum: {}", x + y);  // Put breakpoint here
}
```

- Press `<leader>b` on the println line to add a breakpoint
- Press `<leader>rd` to see debug options
- Select "Debug" and press Enter
- Use `F10` for step over, `F11` for step into

---

## Useful Commands

### Plugin Management (Lazy.nvim)
- `:Lazy` - Open Lazy interface
- `:Lazy update` - Update all plugins
- `:Lazy sync` - Install/update/remove plugins

### Mason (Package Manager)
- `:Mason` - Open Mason interface
- `:MasonInstall <package>` - Install a package
- `:MasonUninstall <package>` - Remove a package
- `<leader>m` - Open Mason

### Autocompletion (nvim-cmp)
- `Ctrl+Space` - Trigger completion manually
- `Ctrl+j/k` - Navigate suggestions
- `Tab/Shift+Tab` - Navigate snippets
- `Enter` - Confirm selection
- `Ctrl+e` - Close completion menu

### Telescope
- `<leader>ff` - Find files
- `<leader>fg` - Live grep (search in all files)
- `<leader>fb` - List open buffers
- `<leader>fr` - Find LSP references
- `<leader>fs` - Document symbols

### Oil.nvim
- `-` - Open current directory
- `<CR>` - Enter directory/open file
- `<C-s>` - Open in horizontal split
- `<C-v>` - Open in vertical split
- `g.` - Toggle hidden files

### LSP
- `gd` - Go to definition
- `gr` - Go to references
- `K` - Hover documentation
- `<leader>rn` - Rename symbol
- `<leader>ca` - Code actions
- `<leader>f` - Format document

### Diagnostics
- `<leader>e` - Show diagnostic popup (error/warning on current line)
- `]d` - Next diagnostic
- `[d` - Previous diagnostic

### Rust Specific
- `<leader>rr` - Rust runnables (run main, examples, etc.)
- `<leader>rt` - Rust testables (run tests)
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
- `<leader>` - Show all leader keybindings

### Trouble (Diagnostics)
- `<leader>xx` - Toggle diagnostics
- `<leader>xX` - Toggle buffer diagnostics
- `<leader>xs` - Toggle symbols
- `<leader>xl` - Toggle LSP info
- `<leader>xQ` - Toggle quickfix

### Treesitter
- `:TSUpdate` - Update parsers
- `:TSInstall <lang>` - Install parser for language
- `<CR>` in visual mode - Incremental selection

---

## Troubleshooting

### Problem: Treesitter doesn't compile parsers
```bash
# Install C dependencies
sudo apt install -y gcc g++

# In Neovim
:TSInstall rust
```

### Warning: tree-sitter executable not found
This is a common and **completely normal** warning:

```
⚠️ WARNING tree-sitter executable not found (parser generator, only needed for :TSInstallFromGrammar)
```

**To eliminate the warning (optional):**
```bash
# Install tree-sitter CLI
cargo install tree-sitter-cli
```

After installation, restart Neovim and the warning will disappear. But I repeat: it's purely cosmetic, the functionality is already complete!

### Warning: mini.icons is not installed
```
⚠️ WARNING mini.icons is not installed
✅ OK nvim-web-devicons is installed
```

**What it means:**
- which-key can use `mini.icons` OR `nvim-web-devicons` for icons
- Our config uses `nvim-web-devicons` (already installed with Oil and Trouble)
- The warning is only informational, icons work perfectly

### Problem: Telescope doesn't find files
```bash
# Verify ripgrep and fd
which rg
which fd

# Ubuntu: If missing, reinstall
sudo apt install ripgrep fd-find

# macOS: If missing, reinstall
brew install ripgrep fd
```

## Next Steps and Customizations

Now you have a **complete and professional** Rust IDE! 🎉

**What you've achieved:**
- ✅ Intelligent autocompletion with nvim-cmp
- ✅ Complete Git integration (Gitsigns + Fugitive)
- ✅ Organized diagnostics (Trouble)
- ✅ Elegant LSP notifications (Fidget)
- ✅ Keybinding guide (Which-Key)
- ✅ Complete visual debugging
- ✅ Advanced syntax highlighting
- ✅ Modern file explorer (Oil)
- ✅ Powerful search (Telescope)
- ✅ Informative statusline (Lualine)

**Consider adding (optional):**

1. **toggleterm.nvim** - Integrated and toggleable terminal
2. **nvim-autopairs** - Automatic closure of parentheses/quotes
3. **Comment.nvim** - Comment code easily with `gc`
4. **indent-blankline.nvim** - Show indentation guides
5. **nvim-colorizer.lua** - Highlight color codes in code
6. **rust-tools.nvim** - Extra Rust tools (hover actions, etc.)

**Useful resources:**
- Neovim documentation: `:help` or https://neovim.io/doc/
- rust-analyzer docs: https://rust-analyzer.github.io/
- Awesome Neovim: https://github.com/rockerBOO/awesome-neovim

Happy coding! 🦀

---

**Note:** This article was translated using AI assistance.
