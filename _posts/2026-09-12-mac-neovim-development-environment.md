---
title: 'Mac에서 Neovim 개발환경 처음부터 구축하기'
date: 2026-09-12 12:00:00 +0900
permalink: /posts/2026/09/mac-neovim-development-environment/
categories:
  - Misc
tags:
  - Neovim
  - macOS
  - LSP
  - Tree-sitter
  - Claude Code
  - Dev Environment
---

이번 글에서는 Mac에서 거의 빈 상태의 Neovim부터 시작해 실제 개발에
사용할 수 있는 환경을 직접 구성합니다.

목표는 거대한 Neovim 배포판을 설치하는 것이 아닙니다. `lazy.nvim`을
기반으로 필요한 기능을 하나씩 추가하면서 각 구성 요소가 무엇을
담당하는지 이해할 수 있는 환경을 만드는 것입니다.

최종적으로 다음과 같은 구성을 만듭니다.

``` text
Neovim
├── lazy.nvim             Plugin Manager
├── Telescope             파일/텍스트 검색
├── Tree-sitter           Syntax Parsing
├── Mason                 LSP 도구 관리
├── LSP
│   ├── Pyright           Python
│   ├── gopls             Go
│   └── ts_ls             TypeScript/JavaScript
├── blink.cmp             자동완성
├── Diagnostics           오류/경고 표시
├── Conform
│   ├── Ruff              Python formatting
│   ├── gofmt             Go formatting
│   └── Prettier          JS/TS/JSON/YAML/Markdown
├── Oil.nvim              파일 탐색
├── Gitsigns              Git 변경사항
├── Claude Code           AI coding
├── which-key             단축키 안내
└── lualine               Status line
```

여기서는 **Neovim 0.12 계열**을 기준으로 합니다. 특히 LSP와 Tree-sitter
설정은 예전 블로그 글에서 흔히 볼 수 있는 방식과 다르므로 주의해야 합니다.

------------------------------------------------------------------------

## 1. Neovim 설치

Homebrew를 이용합니다.

``` bash
brew install neovim
```

확인합니다.

``` bash
nvim --version
```

설정 디렉터리를 만듭니다.

``` bash
mkdir -p ~/.config/nvim/lua/config
mkdir -p ~/.config/nvim/lua/plugins
```

최종 설정 구조는 다음처럼 가져갑니다.

``` text
~/.config/nvim/
├── init.lua
└── lua/
    ├── config/
    │   ├── options.lua
    │   └── keymaps.lua
    └── plugins/
        ├── telescope.lua
        ├── treesitter.lua
        ├── lsp.lua
        ├── completion.lua
        ├── oil.lua
        ├── git.lua
        ├── format.lua
        ├── which-key.lua
        └── statusline.lua
```

플러그인별 설정을 파일로 나누는 이유는 나중에 특정 기능을 수정하거나
제거하기 쉽기 때문입니다.

------------------------------------------------------------------------

## 2. 기본 `init.lua`

`~/.config/nvim/init.lua`

``` lua
require("config.options")
require("config.keymaps")

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

require("lazy").setup("plugins")
```

`lazy.nvim`은 **플러그인 관리자**입니다. `LazyVim`과는 다릅니다.

``` text
lazy.nvim → Neovim Plugin Manager
LazyVim   → lazy.nvim을 기반으로 미리 만들어진 Neovim 배포판
```

이번에는 LazyVim을 사용하지 않고 직접 환경을 구성합니다.

------------------------------------------------------------------------

## 3. 기본 옵션

`~/.config/nvim/lua/config/options.lua`

``` lua
local opt = vim.opt

-- Line numbers
opt.number = true
opt.relativenumber = true

-- Indentation
opt.tabstop = 2
opt.shiftwidth = 2
opt.softtabstop = 2
opt.expandtab = true
opt.smartindent = true

-- Search
opt.ignorecase = true
opt.smartcase = true

-- UI
opt.cursorline = true
opt.termguicolors = true
opt.signcolumn = "yes"
opt.scrolloff = 8

-- Editing
opt.wrap = false
opt.swapfile = false
opt.backup = false
opt.undofile = true

-- macOS clipboard
opt.clipboard = "unnamedplus"

-- Split
opt.splitright = true
opt.splitbelow = true
```

------------------------------------------------------------------------

## 4. 기본 키맵

`~/.config/nvim/lua/config/keymaps.lua`

``` lua
vim.g.mapleader = " "
vim.g.maplocalleader = " "

local keymap = vim.keymap.set

keymap("n", "<Esc>", "<cmd>nohlsearch<CR>")

keymap("n", "<C-h>", "<C-w>h")
keymap("n", "<C-j>", "<C-w>j")
keymap("n", "<C-k>", "<C-w>k")
keymap("n", "<C-l>", "<C-w>l")

keymap("n", "<leader>w", "<cmd>w<CR>", { desc = "Save" })
keymap("n", "<leader>q", "<cmd>q<CR>", { desc = "Quit" })

keymap("v", "<", "<gv")
keymap("v", ">", ">gv")

keymap("t", "<C-h>", [[<C-\><C-n><C-w>h]])
keymap("t", "<C-j>", [[<C-\><C-n><C-w>j]])
keymap("t", "<C-k>", [[<C-\><C-n><C-w>k]])
keymap("t", "<C-l>", [[<C-\><C-n><C-w>l]])
```

Leader key는 Space입니다.

------------------------------------------------------------------------

## 5. Telescope --- 파일과 코드 검색

먼저 검색 도구를 설치합니다.

``` bash
brew install ripgrep
brew install fd
```

`~/.config/nvim/lua/plugins/telescope.lua`

``` lua
return {
  {
    "nvim-telescope/telescope.nvim",
    dependencies = {
      "nvim-lua/plenary.nvim",
    },
    config = function()
      local builtin = require("telescope.builtin")

      vim.keymap.set("n", "<leader>ff", builtin.find_files)
      vim.keymap.set("n", "<leader>fg", builtin.live_grep)
      vim.keymap.set("n", "<leader>fb", builtin.buffers)
      vim.keymap.set("n", "<leader>fh", builtin.help_tags)
    end,
  },
}
```

주요 키:

``` text
Space ff → 파일 검색
Space fg → 프로젝트 전체 텍스트 검색
Space fb → 열린 buffer 검색
Space fh → Neovim help 검색
```

### 주의: 오래된 Telescope 버전 고정

`tag = "0.1.8"`처럼 구버전을 고정했을 때 Neovim 0.12 환경에서
`ft_to_lang` 관련 오류가 발생했습니다. 특정 구버전 tag를 고정하지 않고
업데이트된 Telescope를 사용합니다.

------------------------------------------------------------------------

## 6. Tree-sitter --- 코드를 구조적으로 이해하기

Tree-sitter는 소스 코드를 syntax tree로 분석합니다.

``` text
Tree-sitter → 코드의 문법적 구조 이해
LSP         → 코드의 의미와 프로젝트 관계 이해
```

CLI를 설치합니다.

``` bash
brew install tree-sitter-cli
tree-sitter --version
```

`~/.config/nvim/lua/plugins/treesitter.lua`

``` lua
return {
  {
    "nvim-treesitter/nvim-treesitter",
    lazy = false,
    build = ":TSUpdate",

    config = function()
      local treesitter = require("nvim-treesitter")

      treesitter.setup()

      treesitter.install({
        "lua",
        "vim",
        "vimdoc",
        "bash",
        "python",
        "go",
        "javascript",
        "typescript",
        "json",
        "yaml",
        "markdown",
      })
    end,
  },
}
```

오래된 자료의 `require("nvim-treesitter.configs").setup(...)` 설정은
현재 main branch와 맞지 않을 수 있습니다.

`ENOENT ... cmd: 'tree-sitter'` 오류가 발생한다면 `tree-sitter-cli` 설치
여부를 확인합니다.

------------------------------------------------------------------------

## 7. LSP --- 코드의 의미 이해하기

`~/.config/nvim/lua/plugins/lsp.lua`

``` lua
return {
  {
    "williamboman/mason.nvim",
    config = function()
      require("mason").setup()
    end,
  },

  {
    "williamboman/mason-lspconfig.nvim",
    dependencies = {
      "williamboman/mason.nvim",
      "neovim/nvim-lspconfig",
    },
  },

  {
    "neovim/nvim-lspconfig",
    dependencies = {
      "saghen/blink.cmp",
    },

    config = function()
      local capabilities = require("blink.cmp").get_lsp_capabilities()

      vim.diagnostic.config({
        virtual_text = true,
        signs = true,
        underline = true,
        update_in_insert = false,
        severity_sort = true,
      })

      local servers = {
        pyright = {},
        gopls = {},
        ts_ls = {},
      }

      for server, server_config in pairs(servers) do
        server_config.capabilities = capabilities
        vim.lsp.config(server, server_config)
        vim.lsp.enable(server)
      end

      vim.keymap.set("n", "gd", vim.lsp.buf.definition)
      vim.keymap.set("n", "K", vim.lsp.buf.hover)
      vim.keymap.set("n", "<leader>rn", vim.lsp.buf.rename)
      vim.keymap.set("n", "<leader>ca", vim.lsp.buf.code_action)

      vim.keymap.set("n", "]d", function()
        vim.diagnostic.jump({ count = 1 })
      end, { desc = "Next diagnostic" })

      vim.keymap.set("n", "[d", function()
        vim.diagnostic.jump({ count = -1 })
      end, { desc = "Previous diagnostic" })

      vim.keymap.set("n", "<leader>e", vim.diagnostic.open_float, {
        desc = "Show diagnostic",
      })

      vim.keymap.set("n", "<leader>dq", vim.diagnostic.setloclist, {
        desc = "Diagnostics list",
      })
    end,
  },
}
```

Neovim 0.12에서는 `vim.lsp.config(...)`, `vim.lsp.enable(...)` 방식을
사용합니다.

LSP 서버를 설치합니다.

``` vim
:MasonInstall pyright
:MasonInstall gopls
:MasonInstall typescript-language-server
```

상태 확인:

``` vim
:checkhealth vim.lsp
```

------------------------------------------------------------------------

## 8. blink.cmp --- 자동완성

`~/.config/nvim/lua/plugins/completion.lua`

``` lua
return {
  {
    "saghen/blink.cmp",
    version = "1.*",

    dependencies = {
      "rafamadriz/friendly-snippets",
    },

    opts = {
      keymap = {
        preset = "default",
      },

      completion = {
        documentation = {
          auto_show = true,
          auto_show_delay_ms = 500,
        },
      },

      sources = {
        default = {
          "lsp",
          "path",
          "snippets",
          "buffer",
        },
      },

      fuzzy = {
        implementation = "prefer_rust_with_warning",
      },
    },
  },
}
```

주요 키:

``` text
Ctrl-Space → completion 열기
Ctrl-n     → 다음 항목
Ctrl-p     → 이전 항목
Ctrl-y     → 선택
Ctrl-e     → 닫기
```

------------------------------------------------------------------------

## 9. Oil.nvim --- 파일 탐색

`~/.config/nvim/lua/plugins/oil.lua`

``` lua
return {
  {
    "stevearc/oil.nvim",

    dependencies = {
      "nvim-tree/nvim-web-devicons",
    },

    config = function()
      require("oil").setup({
        default_file_explorer = true,
        view_options = {
          show_hidden = true,
        },
      })

      vim.keymap.set("n", "-", "<cmd>Oil<CR>", {
        desc = "Open parent directory",
      })
    end,
  },
}
```

``` text
-      → 상위 디렉터리
Enter  → 파일/디렉터리 열기

Telescope → 원하는 것을 바로 검색
Oil       → 주변 디렉터리 구조 탐색
```

------------------------------------------------------------------------

## 10. Gitsigns --- Git 변경사항

`~/.config/nvim/lua/plugins/git.lua`

``` lua
return {
  {
    "lewis6991/gitsigns.nvim",

    config = function()
      require("gitsigns").setup({
        on_attach = function(bufnr)
          local gs = require("gitsigns")

          vim.keymap.set("n", "]c", function()
            gs.nav_hunk("next")
          end, { buffer = bufnr, desc = "Next Git hunk" })

          vim.keymap.set("n", "[c", function()
            gs.nav_hunk("prev")
          end, { buffer = bufnr, desc = "Previous Git hunk" })

          vim.keymap.set("n", "<leader>hp", gs.preview_hunk, {
            buffer = bufnr,
            desc = "Preview Git hunk",
          })

          vim.keymap.set("n", "<leader>hb", gs.blame_line, {
            buffer = bufnr,
            desc = "Git blame line",
          })

          vim.keymap.set("n", "<leader>hd", gs.diffthis, {
            buffer = bufnr,
            desc = "Git diff",
          })
        end,
      })
    end,
  },
}
```

------------------------------------------------------------------------

## 11. Neovim Terminal

``` vim
:terminal
```

또는:

``` vim
:split | terminal
```

Terminal input mode에서 Normal mode로 돌아오기:

``` text
Ctrl-\ Ctrl-n
```

------------------------------------------------------------------------

## 12. Claude Code 연결

Claude Code CLI를 Neovim terminal에서 실행하거나 Neovim integration을
사용할 수 있습니다.

실제 사용 중 Visual mode에서 선택 영역을 전달할 때 다음 오류가 발생했습니다.

``` text
[ClaudeCode] [server] [ERROR]
WebSocket server error:
Client read error: ECONNRESET
```

VS Code, Conductor 등 Claude Code integration을 사용하는 다른 IDE를
종료하자 정상 동작했습니다. WebSocket 관련 문제가 발생하면 다른 Claude Code
IDE integration이 동시에 실행되고 있는지 확인합니다.

------------------------------------------------------------------------

## 13. Diagnostics

예:

``` python
def add(a: int, b: int) -> int:
    return a + b

result = add("1", 2)
```

Pyright가 타입 오류를 표시합니다.

``` text
]d       → 다음 diagnostic
[d       → 이전 diagnostic
Space e  → 상세 메시지
Space dq → diagnostics 목록
```

------------------------------------------------------------------------

## 14. Formatting --- Conform.nvim

사용 formatter:

``` text
Python                  → Ruff
Go                      → gofmt
JavaScript/TypeScript   → Prettier
JSON/YAML/Markdown      → Prettier
```

설치:

``` bash
brew install ruff
npm install -g prettier
```

확인:

``` bash
go version
which gofmt
ruff --version
prettier --version
```

`gofmt`에는 `--version` 옵션이 없으므로 `gofmt --version`으로 확인하지
않습니다.

`~/.config/nvim/lua/plugins/format.lua`

``` lua
return {
  {
    "stevearc/conform.nvim",
    lazy = false,

    opts = {
      formatters_by_ft = {
        python = { "ruff_format" },
        go = { "gofmt" },
        javascript = { "prettier" },
        typescript = { "prettier" },
        javascriptreact = { "prettier" },
        typescriptreact = { "prettier" },
        json = { "prettier" },
        yaml = { "prettier" },
        markdown = { "prettier" },
      },

      format_on_save = {
        timeout_ms = 1000,
        lsp_format = "fallback",
      },
    },

    keys = {
      {
        "<leader>f",
        function()
          require("conform").format({
            async = true,
            lsp_format = "fallback",
          })
        end,
        desc = "Format buffer",
      },
    },
  },
}
```

`lazy = false`가 중요합니다. 없을 때는 `Space f` 수동 포맷은 됐지만 첫
저장에서 자동 포맷이 되지 않았습니다.

``` text
:w       → 저장 + 자동 format
Space f  → 수동 format
```

------------------------------------------------------------------------

## 15. which-key --- 단축키 안내

`~/.config/nvim/lua/plugins/which-key.lua`

``` lua
return {
  {
    "folke/which-key.nvim",
    event = "VeryLazy",

    opts = {
      delay = 300,
    },

    config = function(_, opts)
      local wk = require("which-key")

      wk.setup(opts)

      wk.add({
        { "<leader>a", group = "AI / Claude" },
        { "<leader>h", group = "Git Hunk" },
        { "<leader>d", group = "Diagnostics" },
        { "<leader>f", group = "Find / Format" },
      })
    end,
  },
}
```

Normal mode에서 `Space`를 누르고 잠시 기다리면 등록된 명령을 확인할 수
있습니다.

------------------------------------------------------------------------

## 16. Nerd Font 설치

``` bash
brew install --cask font-jetbrains-mono-nerd-font
```

iTerm2:

``` text
Settings
→ Profiles
→ Text
→ Font
→ JetBrainsMono Nerd Font Mono
```

`JetBrainsMonoNL Nerd Font Mono`의 `NL`은 No Ligatures 버전입니다.

Nerd Font 설정 전에는 which-key 아이콘이 `?`로 표시되었지만 설정 후 정상
표시됐습니다.

------------------------------------------------------------------------

## 17. lualine --- Status line

`~/.config/nvim/lua/plugins/statusline.lua`

``` lua
return {
  {
    "nvim-lualine/lualine.nvim",

    dependencies = {
      "nvim-tree/nvim-web-devicons",
    },

    opts = {
      options = {
        theme = "auto",
        globalstatus = true,
      },

      sections = {
        lualine_a = { "mode" },
        lualine_b = { "branch", "diff", "diagnostics" },
        lualine_c = { "filename" },
        lualine_x = { "filetype" },
        lualine_y = { "progress" },
        lualine_z = { "location" },
      },
    },
  },
}
```

화면 아래에서 mode, branch, diff, diagnostics, filename, filetype,
progress, location을 확인할 수 있습니다.

------------------------------------------------------------------------

## 18. 최종 키맵

| Key | 기능 |
| --- | --- |
| `Space` | which-key |
| `Space w` | 저장 |
| `Space q` | 종료 |
| `Space ff` | 파일 검색 |
| `Space fg` | 문자열 검색 |
| `Space fb` | buffer 검색 |
| `-` | Oil 파일 탐색 |
| `gd` | Definition |
| `K` | Hover |
| `Space rn` | Rename |
| `Space ca` | Code Action |
| `]d` | 다음 diagnostic |
| `[d` | 이전 diagnostic |
| `Space e` | Diagnostic 상세 |
| `Space dq` | Diagnostics 목록 |
| `]c` | 다음 Git hunk |
| `[c` | 이전 Git hunk |
| `Space hp` | Git hunk preview |
| `Space hb` | Git blame |
| `Space hd` | Git diff |
| `Space f` | Format |
| `Ctrl-h/j/k/l` | Window 이동 |

------------------------------------------------------------------------

## 19. 새 Mac에서 다시 설치할 때

순서는 다음과 같습니다.

``` text
1. Homebrew
2. Neovim
3. ~/.config/nvim 구성
4. lazy.nvim
5. Telescope
6. ripgrep / fd
7. tree-sitter-cli
8. Tree-sitter
9. Mason / LSP
10. Pyright / gopls / typescript-language-server
11. blink.cmp
12. Oil
13. Gitsigns
14. Claude Code
15. Ruff / Prettier
16. Conform
17. which-key
18. Nerd Font
19. lualine
```

시스템 패키지:

``` bash
brew install \
  neovim \
  ripgrep \
  fd \
  tree-sitter-cli \
  ruff
```

``` bash
npm install -g prettier
brew install --cask font-jetbrains-mono-nerd-font
```

Neovim에서:

``` vim
:MasonInstall pyright
:MasonInstall gopls
:MasonInstall typescript-language-server
```

------------------------------------------------------------------------

## 20. 직접 겪은 문제와 해결 방법

### `No specs found for module "plugins"`

`plugins/` 디렉터리가 비어 있는 상태에서
`require("lazy").setup("plugins")`를 실행해서 발생했습니다. 플러그인 spec을
하나 이상 추가하면 해결됩니다.

### Telescope `ft_to_lang` 오류

오래된 Telescope `0.1.8`을 고정해서 발생했습니다. `tag = "0.1.8"`을 제거하고
Telescope를 업데이트해서 해결했습니다.

### `nvim-treesitter.configs`를 찾지 못함

오래된 Tree-sitter 설정 API를 사용한 것이 원인이었습니다. 현재 main branch
API에 맞춰 설정했습니다.

### Tree-sitter `ENOENT: tree-sitter`

``` bash
brew install tree-sitter-cli
```

로 해결했습니다.

### `<C-l>`이 netrw에서 동작하지 않음

netrw가 buffer-local `<C-l>` mapping을 가지고 있었습니다.

``` vim
:verbose nmap <C-l>
```

로 확인할 수 있습니다.

### `:LspInfo`가 없음

Neovim 0.12에서는:

``` vim
:checkhealth vim.lsp
```

로 확인했습니다.

### `gofmt --version` 오류

설치 오류가 아닙니다. `gofmt`에 `--version` 옵션이 없습니다.

``` bash
go version
which gofmt
```

로 확인합니다.

### `:ConformInfo`가 없음

Conform이 lazy-loaded 상태여서 아직 로드되지 않은 것이 원인이었습니다.

### 저장할 때 자동 format이 안 됨

`Space f`는 동작했지만 `:w`에서는 포맷되지 않았습니다. `lazy = false`를
추가하여 해결했습니다.

### which-key 아이콘이 `?`로 표시됨

Nerd Font가 설정되지 않은 문제였습니다. JetBrainsMono Nerd Font Mono를
설치하고 iTerm2 font로 지정해서 해결했습니다.

### Claude Code `ECONNRESET`

VS Code/Conductor 등 Claude Code integration을 사용하는 다른 IDE가
동시에 실행 중이었습니다. 다른 IDE를 종료하자 정상 동작했습니다.

------------------------------------------------------------------------

## 21. 최종 개발 Workflow

프로젝트에서:

``` bash
cd ~/ws/src/my-project
nvim .
```

일상적인 흐름:

``` text
Telescope / Oil
       ↓
코드 탐색
       ↓
Tree-sitter + LSP
       ↓
blink.cmp
       ↓
코드 작성
       ↓
Claude Code
       ↓
Diagnostics
       ↓
Conform
       ↓
Gitsigns
       ↓
Git commit
```

작업 방식은 다음처럼 나눌 수 있습니다.

``` text
하나의 이슈를 집중해서 순차적으로 개발
→ Neovim + Claude Code

여러 이슈/에이전트를 병렬로 개발
→ Conductor + Git worktree
```

현재 정도면 실제 개발에 필요한 핵심 기능은 충분합니다. 이후에는 플러그인을
계속 추가하기보다 실제 프로젝트에서 사용하면서 불편한 점이 생길 때
필요한 기능만 추가하는 것이 좋습니다.

------------------------------------------------------------------------

## 마무리

각 구성 요소의 역할을 정리하면 다음과 같습니다.

``` text
Tree-sitter  → 문법 구조
LSP          → 코드 의미
blink.cmp    → 자동완성
Telescope    → 검색
Oil          → 파일 탐색
Gitsigns     → Git 변경사항
Conform      → Formatting
which-key    → 키맵 발견
lualine      → 상태 정보
Claude Code  → AI-assisted development
```

`~/.config/nvim` 자체를 Git 저장소로 관리해두는 것도 권장합니다. 그러면
다음 Mac에서는 dotfiles를 clone하고 Homebrew/Mason의 외부 의존성만
설치하면 거의 그대로 환경을 복원할 수 있습니다.
