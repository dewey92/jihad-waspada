---
title: "How to Setup Stylelint with Neovim LSP"
date: 2021-03-27T17:54:11+01:00
description: "Stylelint + NVIM diagnostic"
tags: ["neovim", "lsp", "stylelint"]
categories: ["code", "editor", "neovim"]
draft: false
---

We're going to use [diagnostic language server](https://github.com/iamcco/diagnostic-languageserver) to make stylelint work. Make sure you have it installed by running `npm i -g diagnostic-languageserver`. Then, with the help of [nvim-lspconfig](https://github.com/neovim/nvim-lspconfig), we can easily glue stylelint with Neovim LSP.

```lua
local nvim_lsp = require('lspconfig')

local filetypes = {
  typescript = {'stylelint'},
  typescriptreact = {'stylelint'},
}
local linters = {
  stylelint = {
    sourceName = 'stylelint',
    command = 'stylelint',
    args = {'--formatter', 'compact', '%filepath'},
    rootPatterns = {'.stylelintrc'},
    debounce = 100,
    formatPattern = {
      [[: line (\d+), col (\d+), (warning|error) - (.+?) \((.+)\)]],
      {
        line = 1,
        column = 2,
        security = 3,
        message = {4, ' [', 5, ']'},
      },
    },
    securities = {
      warning = 'warning',
      error = 'error',
    },
  },
}

local formatters = {
  stylelint = {
    command = 'stylelint',
    args = {'--fix', '--stdin', '--stdin-filename', '%filepath'},
  }
}
local formatFiletypes = {
  typescript = {'stylelint'},
  typescriptreact = {'stylelint'},
}

nvim_lsp.diagnosticls.setup {
  filetypes = vim.tbl_keys(filetypes),
  init_options = {
    filetypes = filetypes,
    linters = linters,
    formatters = formatters,
    formatFiletypes = formatFiletypes,
  }
}
```

We specified the `--formatter` option to `compact` as I find it more convenient to create the pattern in `formatPattern` compared to the default one, which is `--formatter string`. You can of couse choose the other options as well, for instance the default one if you want to show the cute warning and error icons in the editor. But so far this setup is enough to get your linter running and do the formatting job for you.

## Configuring the message

Let's take a look at the stylelint output when something is ill-formatted.

```nocode
/path/Filename.tsx: line 469, col 2, warning - Expected left to come before position (order/properties-alphabetical-order)
/path/Filename.tsx: line 472, col 2, warning - Expected pointer-events to come before z-index (order/properties-alphabetical-order)
/path/Filename.tsx: line 470, col 3, error - Expected indentation of 1 tab (indentation)
```

As you can see, we have the line number, column number, severity level, the description, and the rule id in the output. This is where `formatPattern` field comes into play — to translate the linter output to the LSP diagnostic engine.

This regex pattern `: line (\d+), col (\d+), (warning|error) - (.+?) \((.+)\)` enables us to capture all those information chunks into groups; the first group being the line number, the second being the column number, the third being the severity level, the fourth and the fifth being the message and the rule id respectively. Now that we know which group number corresponds to which chunk, you can start customizing to your liking the message format you'd like to see in the `message` field. Personally I like to see the message with the rule id in bracket.

```lua
formatPattern = {
  ...,
  {
    ...,
    message = {4, ' [', 5, ']'},
  }
}
````

{{< figure src="/uploads/stylelint-lsp-diagnostic.png" alt="Stylelint working" caption="Stylelint error messages" class="fig-center" >}}

## Using efm-langserver

Another alternative would be using efm-langserver which is a general purpose language server. The setup is also fairly simple.

```lua
local stylelint = {
  lintCommand = 'stylelint --stdin --stdin-filename ${INPUT} --formatter compact',
  lintIgnoreExitCode = true,
  lintStdin = true,
  lintFormats = {
    '%f: line %l, col %c, %tarning - %m',
    '%f: line %l, col %c, %trror - %m',
  },
  formatCommand = 'stylelint --fix --stdin --stdin-filename ${INPUT}',
  formatStdin = true,
}
nvim_lsp.efm.setup {
  cmd = { 'efm-langserver' },
  init_options = {
    documentFormatting = true,
    rename = false,
    hover = false,
    completion = false,
  },
  filetypes = { 'typescript', 'typescriptreact' },
  settings = {
    rootMarkers = { '.git', 'package.json' },
    languages = {
      typescript = { stylelint },
      typescriptreact = { stylelint },
    },
  },
}
```

## Using stylelint-lsp

If you'd like to have code actions capability, you can set up [stylelint-lsp](https://github.com/bmatcuk/stylelint-lsp) in your config. I'm not gonna talk about how to set it up in this blog post, but you can take a look at this [PR](https://github.com/neovim/nvim-lspconfig/pull/800/files) and just follow the instruction written in the `docs.description` field.

{{< figure src="/uploads/stylelint-lsp-codeactions.png" alt="Stylelint code actions" caption="Stylelint code actions" class="fig-center" >}}

So far I find `stylint-lsp` to be the fastest compared to the other two: you open (or update) a file and you'll see the linter message instantly if errors are present. One note tho, I haven't been able to set up "auto fix" action just yet. I think I'll just rely on efm formatting for the time being.

UPDATE: PR was merged, and this is all you need to do:

```lua
nvim_lsp.stylelint_lsp.setup {
  settings = {
    stylelintplus = {
      autoFixOnSave = true,
      autoFixOnFormat = true,
      -- other settings...
    }
  },
}
```

---

Hope it helps :)
