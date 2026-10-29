+++
title = "Moving from NvChad to Native Neovim 0.12"
+++

> Part of my [dotfiles](https://github.com/j-dumas/dotfiles)

## Summary
I've wanted for a while now to move from [NvChad](https://nvchad.com/) to a more simple and personalized Neovim config. I thus decided to redo all my config in the new Neovim 0.12 from scratch by readding only what I need and throwing out the rest, things like `mason` and `mason-lspconfig`, `lazy`, etc.

## The Config
### What's Needed?
To transform Neovim into the experience I had with NvChad, i.e. close to a full IDE, I'll need

- [x] Colorscheme
- [x] Statusline
- [x] Tree
- [x] Highlight
- [x] Formatter
- [x] LSP
- [x] File picker/live grep
- [x] Autocompletion
- [x] A Git client to not rely on switching tmux panes for lazygit
- [ ] Linter
- [ ] Debugger

### A Clean Structure
Let's start with the structure of the config directory:
```txt
~/.config/nvim/
├── init.lua
└── lua/
	├── plugins/
	|   ├── init.lua
	|   ├── cmp.lua
	|   ├── format.lua
	|   ├── fzf-lua.lua
	|   ├── lsp.lua
	|   ├── qol.lua
	|   ├── tree.lua
	|   ├── treesitter.lua
	|   └── ui.lua
	├── autocmds.lua
	├── keymaps.lua
	└── options.lua
	
```

### Common Options
Here are the options I used. These were taken from the NvChad config to have a similar feel and also from [a list of the most common options in Neovim](https://dotfiles.substack.com/p/neovim-options-the-most-common-ones), like tab size, splitting directions, clipboard and rounded borders, etc.
```lua
require('vim._core.ui2').enable()

vim.g.netrw_banner = 0
vim.g.mapleader = ' '

vim.o.nu = true
vim.o.relativenumber = true

vim.o.tabstop = 4
vim.o.softtabstop = 4
vim.o.shiftwidth = 4
vim.o.expandtab = true
vim.o.smartindent = true

vim.o.inccommand = "split"
vim.o.splitbelow = true
vim.o.splitright = true

vim.o.ignorecase = true
vim.o.smartcase = true
vim.o.laststatus = 3

vim.o.swapfile = false
vim.o.backup = false
vim.o.undodir = vim.fn.stdpath("data") .. "/undodir"
vim.o.undofile = true

vim.o.clipboard = "unnamedplus"
vim.o.cursorline = true
vim.o.cursorlineopt = "number"
vim.o.scrolloff = 8
vim.o.signcolumn = "yes"
vim.o.termguicolors = true
vim.opt.whichwrap:append "<>[]hl"
vim.o.winborder = "rounded"
```

### UI: Colorscheme and Statusline
Here are the three plugins I use for UI: [`catppuccin-nvim`](https://github.com/catppuccin/nvim), [`lualine`](https://github.com/nvim-lualine/lualine.nvim) and some icons with [`nvim-web-devicons`](https://github.com/nvim-tree/nvim-web-devicons).
```lua
vim.pack.add { 
    { 
        src = "https://github.com/catppuccin/nvim",
        name = "catppuccin" 
    } ,
    'https://github.com/nvim-tree/nvim-web-devicons',
    'https://github.com/nvim-lualine/lualine.nvim'
}
```

Since I already use Catppuccin Mocha nearly everywhere, I installed it for Neovim.  For a terminal, I use kitty with a transparent background, so I opted to have Neovim transparent as well. I also modified the colors of the sections to fit my tmux look (which is kind of a bastard between the dracula tmux plugin and catppuccin colors). I changed some highlights to fit my taste, like the `MatchParen` to remove colors of matching brackets and only keep the background highlight.
```lua
require("catppuccin").setup({
    flavour = "mocha",
    transparent_background = true,
    custom_highlights = function(colors)
        return {
            MatchParen = { fg = colors.none },
            IncSearch = { bg = colors.green },
            NormalFloat = { bg = colors.none },
            FloatBorder = { bg = colors.none },
        }
    end,
    integrations = {
        lualine = {
            mocha = function(colors)
                return {
                    normal = {
                        a = { bg = "#8bc2f0" },
                        b = { fg = colors.text },
                        c = { fg = colors.surface2 },
                    },
                    insert = {
                        b = { fg = colors.text },
                    },
                    visual = {
                        b = { fg = colors.text },
                    },
                    command = {
                        b = { fg = colors.text },
                    },
                }
            end
        }
    }
})
vim.cmd.colorscheme "catppuccin-nvim"
```

And here is lualine as I like:
```lua
require('lualine').setup({
    options = {
        component_separators = {left = '', right = ''},
        section_separators = {left = '◤', right = ''}
    },
    sections = {
        lualine_a = {'mode'},
        lualine_b = {{'filename', path = 1}},
        lualine_c = {'branch', 'diff'},
        lualine_x = {'diagnostics', 'lsp_status'},
        lualine_y = {'location'},
        lualine_z = {'progress'}
    },
    extensions = {'nvim-tree'}
})
```

### NeoTree: A File Viewer
I'm used to having a file viewer in the side bar and did not really want to use `netrw`, so I decided to try [`neo-tree`](https://github.com/nvim-neo-tree/neo-tree.nvim). I added  `nvim-window-picker` for picking a specific window when opening a file instead of opening in the last one. I added nicer colors for the window picker because the default was bright green. It's now red with a transparent background. Finally I changed the mappings to always ask which window to open a file when there are splits.
```lua
vim.pack.add({
  {
    src = 'https://github.com/nvim-neo-tree/neo-tree.nvim',
    version = vim.version.range('3')
  },
  -- Dependencies
  "https://github.com/nvim-lua/plenary.nvim",
  "https://github.com/MunifTanjim/nui.nvim",
  {
      src = 'https://github.com/s1n7ax/nvim-window-picker', 
      version = vim.version.range('2.*') 
  },
})

require 'window-picker'.setup({
    hint = 'statusline-winbar',
    selection_chars = 'ABCDEFGHIJKLMNOP',
    highlights = {
        enabled = true,
        statusline = {
            unfocused = {
                fg = '#f38ba8',
                bg = 'NONE',
                bold = true,
            },
        },
        winbar = {
            unfocused = {
                fg = '#f38ba8',
                bg = 'NONE',
                bold = true,
            },
        },
    },
})

require("neo-tree").setup({
  window = {
    mappings = {
      ["<cr>"] = "open_with_window_picker",
	  ["<2-LeftMouse>"] = "open_with_window_picker",
      ["s"]    = "split_with_window_picker",
      ["v"]    = "vsplit_with_window_picker",
      ["S"]    = "noop",
      ["w"]    = "noop",
    },
  },
})

local opts = function(desc)
            return { desc = "Neotree: " .. desc }
        end
local map = vim.keymap.set

map("n", "<C-n>", "<cmd>Neotree toggle reveal=true<cr>", opts("Toggle Explorer"))
map("n", "<leader>e", "<cmd>Neotree focus", opts("Focus Explorer"))
map("n", "<leader>b", "<cmd>Neotree focus source=buffers<cr>", opts("Focus Buffers"))
```

### Treesitter: Better Highlighting and Parsers
Neovim 0.12+ now bundles a couple languages for highlights, but it's not enough. Since [`nvim-treesitter`](https://github.com/nvim-treesitter/nvim-treesitter) is now archived, I installed [`tree-sitter-manager`](https://github.com/romus204/tree-sitter-manager.nvim) and installed a couple languages like `c`, `cpp`, `latex`, `lua` and `python`.
```lua
vim.pack.add {
  "https://github.com/romus204/tree-sitter-manager.nvim"
}

require("tree-sitter-manager").setup()
```

### Conform: A Formatter
I decided to use [`conform`](https://github.com/stevearc/conform.nvim) to invoke formatters in my code and set their settings. I started by configuring `stylua` to then format my config files correctly and thus test if everything works.
```lua
vim.pack.add({
    "https://github.com/stevearc/conform.nvim",
})

require("conform").setup({
    formatters_by_ft = {
        lua = { "stylua" },
    },
    default_format_opts = {
        lsp_format = "fallback",
    },
    formatters = {
        stylua = {
            prepend_args = { "--indent-type", "Spaces" },
        },
    },
})

vim.keymap.set("n", "<leader>fm", function()
    require("conform").format()
end, { desc = "Format file" })
```


> [!IMPORTANT] Installing tools
> I mentioned I wanted to get away from `mason` to install tools like LSPs and formatters. I landed on [`mise`](https://mise.jdx.dev/) to install the tools I need on my system and get them in the `$PATH` so that Neovim can find them. It works just like using `npm` or `pip`, but with the recent security track record of the former, I did not want to install my tools with it.
> 
> You can install tools like `stylua` with `mise use -g stylua` and `lua_ls` with `muse use -g lua-language-server`.

### LSPs
Neovim's native lsp is powerful, but I still like to have default configs since they are not yet in nvim. So I installed [`nvim-lsp-config`](https://github.com/neovim/nvim-lspconfig) for defaults that I can extend.

I also decided to change the keymaps for something more intuitive to me, like goto definition (`gd`), or code actions (`ca`) or rename (`rn`) like it was used in NvChad.

Then, I enabled the LSPs I wanted after having installed them with either `mise`, `pip` or `dnf`. I decided to let `pylsp` manage the linters and formatters since it seemed to simpler to let them work together as intended by the devs, as opposed to C where I'll add `clang-format` and `clang-tidy`. Later I'll add `ltex_plus` to edit Latex files.

I also added virtual lines to show error diagnostic under the errors. We'll see if I keep it or if it is too cluttering.

Finally, I added [`nvim-file-operations`](https://github.com/Crysthamus/nvim-file-operations) to implement the events emitted by the file manager so that server that can will update import statements and paths.
```lua
vim.pack.add({
    "https://github.com/neovim/nvim-lspconfig",
})

local map = vim.keymap.set

vim.api.nvim_create_autocmd("LspAttach", {
    callback = function(args)
        local opts = function(desc)
            return { buffer = args.buf, desc = "LSP: " .. desc }
        end

        -- Navigation
        map("n", "gd", vim.lsp.buf.definition, opts("Go to Definition"))
        map("n", "gD", vim.lsp.buf.declaration, opts("Go to Declaration"))
        map("n", "gi", vim.lsp.buf.implementation, opts("Go to Implementation"))
        map("n", "gr", vim.lsp.buf.references, opts("Go to References"))

        -- Actions
        map("n", "<leader>rn", vim.lsp.buf.rename, opts("Rename"))
        map({ "n", "v" }, "<leader>ca", vim.lsp.buf.code_action, opts("Code Action"))

        -- Diagnostics
        map("n", "<leader>df", vim.diagnostic.open_float, opts("Show Diagnostic"))
        map("n", "<leader>dn", function()
            vim.diagnostic.jump({ count = 1, float = true })
        end, opts("Next Diagnostic"))
        map("n", "<leader>dp", function()
            vim.diagnostic.jump({ count = -1, float = true })
        end, opts("Previous Diagnostic"))
        map("n", "<leader>dl", vim.diagnostic.setloclist, opts("Diagnostics List"))
    end,
})

local lsp = vim.lsp

lsp.config("lua_ls", {
    settings = {
        Lua = { diagnostics = { globals = { "vim" } } },
    },
})

vim.lsp.config("clangd", {
    cmd = { "clangd", "--enable-config" },
})

-- pylsp comes with linter and formatter, so simpler to set here together
vim.lsp.config("pylsp", {
    settings = {
        pylsp = {
            plugins = {
                pycodestyle = {
                    ignore = { "E203" },
                    maxLineLength = 88, -- Black formatter
                },
                black = { enabled = true },
            },
        },
    },
})

lsp.enable({ "lua_ls", "clangd", "pylsp" })

vim.diagnostic.config({
    virtual_lines = {
        format = function(diagnostic)
            -- Only show the message, hide the code
            return diagnostic.message
        end,
    },
})

require("nvim-file-operations").setup()
vim.lsp.config("*", {
    capabilities = require("nvim-file-operations.config").default_capabilities(),
})

```

### Fzf-Lua: A File Picker
I installed [`fzf-lua`](https://github.com/ibhagwan/fzf-lua) instead of Telescope for the speed. I then set it up with the commands I used the most in NvChad, namely finding files, finding in all files starting from home, finding recent files and `live_grep` to find words in files in the project.
```lua
vim.pack.add({ "https://github.com/ibhagwan/fzf-lua" })

local fzf = require("fzf-lua")
fzf.setup({})

local opts = function(desc)
    return { desc = "FZF: " .. desc }
end

local map = vim.keymap.set
map("n", "<leader>ff", fzf.files, opts("Find files"))
map("n", "<leader>fa", function()
    fzf.files({ cwd = "~" })
end, opts("Find all files"))
map("n", "<leader>fo", fzf.oldfiles, opts("Find recent files"))
map("n", "<leader>fw", fzf.live_grep, opts("Live grep"))
map("n", "<leader>fk", fzf.keymaps, opts("Find keymaps"))
map("n", "<leader>fb", fzf.buffers, opts("Find buffers"))
```

### Blink: Autocompletion
I used [`blink`](https://github.com/Saghen/blink.cmp), the newer, faster completion plugin, to have completion and snippets sourced from LSPs. I modified a bit the keymaps to my liking using enter to accept a completion and tab to move inside the list. Finally, I added the capabilities of blink to the LSPs.
```lua
vim.pack.add({ { src = "https://github.com/Saghen/blink.cmp", version = "v1" } })

local blink = require("blink.cmp")
blink.setup({
    keymap = {
        preset = "enter",

        ["<Tab>"] = { "select_next", "snippet_forward", "fallback" },
        ["<S-Tab>"] = { "select_prev", "snippet_backward", "fallback" },

        ["<Up>"] = false,
        ["<Down>"] = false,
        ["<C-p>"] = false,
        ["<C-n>"] = false,
    },
    completion = {
        list = {
            selection = { preselect = false, auto_insert = true },
        },
    },
})

vim.lsp.config("*", {
  capabilities = blink.get_lsp_capabilities(),
})
```

### Quality of Life
For now, I added [`which-key`](https://github.com/folke/which-key.nvim) as a bonus on top of the keymaps listing of `fzf-lua` to help me if I forget a combination. I then added [`nvim-autopairs`](https://github.com/windwp/nvim-autopairs) to automatically close brackets and parentheses.
```lua
vim.pack.add({
    "https://github.com/folke/which-key.nvim",
    "https://github.com/windwp/nvim-autopairs",
})

require("which-key").setup({ preset = "helix" })
require("nvim-autopairs").setup({})
```

### Git Client
I added [`gitsigns`](https://github.com/lewis6991/gitsigns.nvim) to show lines that have been changes in the buffers and added the [`lazygit` nvim plugin](https://github.com/kdheepak/lazygit.nvim) to toggle a floating Lazygit terminal. I'm already used to Lazygit, but this plugins makes it so I can 1) use it in the same window and 2) open the files in conflicts inside the same Neovim instance.
```lua
vim.pack.add({
    "https://github.com/lewis6991/gitsigns.nvim",
    "https://github.com/kdheepak/lazygit.nvim",
})

vim.keymap.set("n", "<leader>g", "<cmd>LazyGit<cr>", { desc = "Open Lazygit" })
```

### Keymaps
Finally, here are some general keymaps I like. They are taken directly from NvChad.
```lua
local map = vim.keymap.set

map("n", "<Esc>", "<cmd>noh<CR>", { silent = true, desc = "Clear highlights" })

-- Moving in insert mode
map("i", "<C-b>", "<ESC>^i", { desc = "Move beginning of line" })
map("i", "<C-e>", "<End>", { desc = "Move end of line" })
map("i", "<C-h>", "<Left>", { desc = "Move left" })
map("i", "<C-l>", "<Right>", { desc = "Move right" })
map("i", "<C-j>", "<Down>", { desc = "Move down" })
map("i", "<C-k>", "<Up>", { desc = "Move up" })

-- Moving in panes
map("n", "<C-h>", "<C-w>h", { desc = "Switch pane left" })
map("n", "<C-l>", "<C-w>l", { desc = "Switch pane right" })
map("n", "<C-j>", "<C-w>j", { desc = "Switch pane down" })
map("n", "<C-k>", "<C-w>k", { desc = "Switch pane up" })

-- Comments
map("n", "<leader>/", "gcc", { desc = "toggle comment", remap = true })
map("v", "<leader>/", "gc", { desc = "toggle comment", remap = true })
```

# Conclusion
This rewrite of my config allowed me to have a clean and minimal configuration of my Neovim and learn a bit more about the new features of the 0.12+ version. Everything else I'll add will be in my dotfiles but not detailed here, things like vimtex and other plugins I land on in my researchs.
