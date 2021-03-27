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
    args = {'%filepath'},
    rootPatterns = {'.stylelintrc'},
    debounce = 100,
    formatPattern = {
      [[(\d+):(\d+)\s\s(.)\s\s(.+?)\s+([a-zA-Z\/\-]+)$]],
      {
        line = 1,
        column = 2,
        security = 3,
        message = {4, ' [', 5, ']'},
      },
    },
    securities = {
      ['⚠'] = 'warning',
      ['✖'] = 'error',
    },
  },
}

nvim_lsp.diagnosticls.setup {
  filetypes = vim.tbl_keys(filetypes),
  init_options = {
    filetypes = filetypes,
    linters = linters,
  }
}
```

Voila!

## Configuring the message

Let's take a look at the stylelint output when something is ill-formatted.

```nocode
client/src/components/clients/pricing/page/FeatureSection.tsx
  244:2  ⚠  Expected height to come before margin                       order/properties-alphabetical-order
  245:2  ⚠  Expected border-bottom-right-radius to come before height   order/properties-alphabetical-order
  247:3  ✖  Expected indentation of 1 tab                               indentation
```

As you can see, we have the line number, column number, severity level marked by an icon, the description, and the rule id in the output. This is where `formatPattern` field comes into play — to translate the linter output to the LSP diagnostic engine.

This regex pattern `(\d+):(\d+)\s\s(.)\s\s(.+?)\s+([a-zA-Z\/\-]+)$` enables us to capture all those information chunks into groups; the first group being the line number, the second being the column number, the third being the icon (severity), the fourth and the fifth being the message and the rule id respectively. Now that we know which group number corresponds to which chunk, you can start customizing to your liking the message format you'd like to see in the `message` field. Personally I like to see the message with the rule id in bracket.

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

## Using stylelint-lsp

If you'd like to have code actions capabilities, you can set up [stylelint-lsp](https://github.com/bmatcuk/stylelint-lsp) in your config. I'm not gonna talk about how to set it up in this blog post, but you can take a look at this [PR](https://github.com/neovim/nvim-lspconfig/pull/800/files) and just follow the instruction written in the `docs.description` field.

{{< figure src="/uploads/stylelint-lsp-codeactions.png" alt="Stylelint code actions" caption="Stylelint code actions" class="fig-center" >}}

One note tho, I haven't been able to set up "auto fix" action just yet. Guess I'll just rely on running `stylelint --fix` manually for now. It's not that _annoying_ 😉.

---

Hope it helps :)
