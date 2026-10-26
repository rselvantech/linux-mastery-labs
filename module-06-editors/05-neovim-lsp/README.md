# Demo M06-05 — Neovim: Built-in LSP, Lua Config & checkhealth

## Overview

Demos M06-01/M06-02/M06-03/M06-04 taught Vim — and everything in them works
identically in Neovim, because Neovim's default modes are Vim-compatible
by design. This demo covers what's actually *different*: the one feature
that genuinely separates the two for a working engineer is Neovim's
**built-in LSP (Language Server Protocol) client** — inline diagnostics,
hover documentation, and go-to-definition, natively, with no plugin
required. Plain Vim has none of this built in; getting equivalent
behavior in Vim means installing and configuring a plugin (`ALE`,
`coc.nvim`, etc.) yourself. This demo also covers Neovim's native Lua
configuration (just enough to enable LSP) and `:checkhealth`, the
diagnostic tool for verifying your setup actually works.

**One thing this demo explicitly does NOT claim is Neovim-only:**
`:terminal` (an integrated terminal inside the editor) exists in both —
Vim added it in version 8. It's commonly, incorrectly cited as a
Neovim-exclusive feature; it isn't, and won't be covered here as a
differentiator.

## Demo Objectives

1. Explain what Neovim provides natively that plain Vim does not — the
   built-in LSP client and native Lua configuration — and correctly rule
   out `:terminal` as a genuine differentiator
2. Write a minimal `init.lua` that enables a language server for a
   project
3. Read inline LSP diagnostics and navigate between them
4. Use go-to-definition and hover documentation via the built-in LSP
   client, with no plugin installed
5. Use `:checkhealth` to diagnose a broken or incomplete setup
6. Explain when the switch to Neovim's built-in tooling is actually worth
   it, versus staying with plain Vim and the patterns from Demos
   M06-01/M06-02/M06-03/M06-04

## Directory Structure

```
05-neovim-lsp/
├── README.md                   # this file
├── m06-05-neovim-lsp-anki.csv  # Anki flashcard deck
└── m06-05-neovim-lsp-quiz.md   # standalone quiz
```

## Environments

| Task | Ubuntu 24.04 | Rocky Linux 9.8 | EC2 Amazon Linux |
|---|---|---|---|
| All lab tasks | ✅ (`sudo apt install neovim` + a language server, e.g. `pip install python-lsp-server`) | ✅ (`sudo dnf install neovim` + a language server) | Not required |

> **Verification note — read before treating anything below as settled
> fact.** This demo could not be verified live in this sandbox to a
> *greater* degree than Demos M06-01/M06-02/M06-03/M06-04: not only is there no
> network to install Neovim itself, there's also no way to install or
> run an actual language server binary, which this topic genuinely
> depends on. Core LSP concepts (diagnostics, hover, go-to-definition,
> the client/server model) are stable and well-documented. **Default
> keybindings for LSP actions specifically have changed across Neovim
> versions** — this demo shows commonly-documented mappings and marks
> anything version-dependent explicitly. Confirm actual current defaults
> with `:help lsp-defaults` (or your installed version's equivalent)
> before treating any specific keymap below as guaranteed present without
> configuring it yourself.

## Recall Check — 04-buffers-windows-tabs

Answer these from memory before continuing — no looking back at Demo M06-04.

1. What's the difference in scope between `:argdo` and `:bufdo`?
2. Why use `| update` after an `:argdo`/`:bufdo` substitution instead of
   a blanket `:w`?
3. How do you save a file you opened without realizing it required root
   permission to write, without losing your in-progress edit?

<details>
<summary>Answers</summary>

1. `:argdo` operates only on the explicit `:args` list; `:bufdo` operates
   on every currently loaded buffer, which can include files opened ad
   hoc mid-session that were never part of the argument list.

2. `update` only writes a buffer if it was actually modified, leaving
   untouched files' content and mtimes completely unchanged; a blanket
   `:w` would rewrite every file's mtime regardless of whether anything
   changed.

3. `:w !sudo tee % > /dev/null` — pipes the buffer's current content to
   a `sudo`-elevated `tee`, saving the exact in-progress edit with no
   copy-paste or re-typing needed.

</details>

## Part 1 — Concepts

### 1.1 What's actually different — and what isn't

```
Genuinely Neovim-specific (this demo covers these):
  Built-in LSP client       vim.lsp.* — no plugin needed for the core
                             diagnostics/hover/go-to-definition loop
  Native Lua configuration   init.lua as a first-class config language,
                             not a workaround bolted onto Vimscript
  :checkhealth               structured, extensible setup diagnostics

Commonly assumed Neovim-only, but actually NOT (this demo does not
cover these as differentiators):
  :terminal (integrated terminal)   Vim has had this since version 8 —
                                     a genuinely common misconception,
                                     not a real distinguishing feature
```

The reason this matters practically: if you're deciding whether learning
Neovim specifically is worth the switch beyond what Demos M06-01/M06-02/M06-03/M06-04
already taught, the honest answer hinges entirely on whether the built-in
LSP client's value is worth it *for you* — not on features both editors
already share.

### 1.2 What LSP actually is, and why it matters without a plugin

The Language Server Protocol is a standardized way for an editor to talk
to a separate, language-specific background process (the "language
server" — e.g. `pyright` for Python, `gopls` for Go) that understands the
actual code: it can report errors, jump to a symbol's definition, and
show documentation on hover, without the editor itself needing any
language-specific knowledge baked in.

```
Editor (Neovim)  <---LSP protocol--->  Language server (e.g. pyright)
     |                                          |
  vim.lsp.*                              understands Python's actual
  built-in client                        syntax, types, imports, etc.
```

Plain Vim has no built-in half of this conversation — getting the same
capability in Vim means installing a plugin that implements an LSP
client itself (`coc.nvim`, or a plugin wrapping `vim-lsp`). Neovim ships
the client side natively as `vim.lsp` — you still need to install the
language server binary itself (that part is never built in, in either
editor, since it's language-specific), but the editor-side plumbing to
talk to it requires no plugin.

### 1.3 A minimal init.lua

Neovim's native configuration file is `~/.config/nvim/init.lua` (Lua),
replacing the Vimscript `.vimrc` — though Neovim still reads a `.vimrc`
if that's what you already have, for compatibility. A minimal LSP setup:

```lua
-- ~/.config/nvim/init.lua

vim.lsp.enable('pylsp')
-- 'pylsp' here refers to a language server Neovim knows how to launch
-- and connect to by name, via its built-in default client
-- configurations — you still need the actual pylsp binary installed
-- separately (e.g. `pip install python-lsp-server`); this line tells
-- Neovim to use the built-in LSP client to talk to it once it's running
```

**The exact configuration API shape (`vim.lsp.enable`, versus older
patterns built around the separate `nvim-lspconfig` plugin) has changed
across Neovim versions as built-in LSP support has matured.**
`[Unverified — test before use]`
Confirm the current recommended setup pattern for your specific installed
version — via `:help lsp` — before assuming the snippet above is exactly
correct for your setup.

### 1.4 Diagnostics — reading and navigating inline errors

Once a language server is attached to a buffer, it reports diagnostics
(errors, warnings, hints) inline:

```
:lua vim.diagnostic.open_float()   -- show the diagnostic under the
                                    -- cursor in a floating window
]d / [d                            -- jump to next / previous diagnostic
                                    -- in the current buffer
:lua vim.diagnostic.setloclist()   -- populate the location list with
                                    -- every diagnostic in the buffer
```

Diagnostics typically also show as underlines or gutter signs directly in
the buffer, without needing to run any command at all — the commands
above are for actively inspecting or navigating them, not for making them
appear in the first place.

### 1.5 Go-to-definition and hover — the core payoff

```
gd     -- go to definition of the symbol under the cursor (jumps
          directly to where it's actually defined, potentially in a
          different file entirely — the language server resolves this,
          not a text search)
K      -- show hover documentation for the symbol under the cursor
          (type signature, docstring, etc., in a floating window)
```

**This is the direct, tangible payoff over plain Vim:** `gd` here jumps to
an *actual, semantically-resolved* definition — understanding imports,
scope, and type information — not a text pattern match the way Demo M06-01's
`/pattern` search or `ctags`-based navigation would. `K` similarly shows
real documentation the language server understands, not just a `man`
page lookup.

**Whether these specific letters (`gd`, `K`) are bound by default without
any configuration, or whether you need to set them yourself in your
`init.lua`, has genuinely changed across Neovim versions.**
`[Unverified — test before use]`
Check `:help lsp-defaults` for your installed version's actual current
answer before assuming either way.

### 1.6 :checkhealth — diagnosing your own setup

```
:checkhealth
```

This runs a structured set of self-checks covering Neovim's core systems
and any installed plugins that provide their own health checks — Lua
runtime status, clipboard integration, Python/Node provider availability
(if configured), and LSP-related checks among them. The output is
organized into sections with clear ✓/✗/⚠ status per check, specifically
designed so that "why isn't my LSP setup working" has a structured place
to look first, rather than guessing from silence or an unclear error.

### 1.7 When the switch to Neovim's built-in tooling is actually worth it

If your work is mostly editing config files, scripts, and infrastructure
code on remote servers — the kind of editing Demos M06-01/M06-02/M06-03/M06-04 already
cover thoroughly — plain Vim, universally available with zero setup, is
often genuinely sufficient. The built-in LSP client earns its complexity
specifically when you're doing enough active code-writing (not just
editing) in a language with a good language server available, that
real-time diagnostics, semantic go-to-definition, and hover documentation
meaningfully speed up the work — and you're willing to maintain a
per-language-server setup, which plain Vim's universal, zero-dependency
availability doesn't ask of you at all.

## Part 2 — Hands-On Lab

### Commands used in this demo

| Command | Purpose |
|---|---|
| `vim.lsp.enable('server_name')` | Enable a configured language server (in `init.lua`) |
| `]d` / `[d` | Jump to next / previous diagnostic |
| `:lua vim.diagnostic.open_float()` | Show the diagnostic under the cursor |
| `gd` | Go to definition (semantic, via the language server) |
| `K` | Show hover documentation |
| `:checkhealth` | Run Neovim's structured setup diagnostics |

### Task 1 — Install Neovim and a language server

```bash
# Ubuntu
sudo apt install neovim
pip install python-lsp-server

# Rocky9
sudo dnf install neovim
pip install python-lsp-server
```

Confirm both are present:

```bash
nvim --version | head -1
pylsp --help > /dev/null && echo "pylsp installed"
```

### Task 2 — Minimal init.lua and :checkhealth

```bash
mkdir -p ~/.config/nvim
cat > ~/.config/nvim/init.lua << 'EOF'
vim.lsp.enable('pylsp')
EOF
nvim
```

Inside Neovim: `:checkhealth`. Review the output sections — confirm there
is no explicit failure related to Lua or your LSP configuration line
itself (a missing `pylsp` *binary* would surface as a runtime issue when
you actually open a Python file, not necessarily as a `:checkhealth`
failure — the two checks aren't identical).

### Task 3 — Diagnostics on a real file

```bash
printf 'import os\n\ndef greet(name)\n    print("hello " + name)\n' > /tmp/broken.py
nvim /tmp/broken.py
```

This file has a genuine syntax error (a missing colon after
`def greet(name)`) and an unused import (`os`). Confirm diagnostics
appear — underlines and/or gutter signs. Use `]d` to jump to the first
one, `:lua vim.diagnostic.open_float()` to read its full message, `[d` to
go back.

### Task 4 — Go-to-definition and hover

```bash
printf 'def helper():\n    return 42\n\ndef main():\n    result = helper()\n    print(result)\n' > /tmp/gd-demo.py
nvim /tmp/gd-demo.py
```

Place the cursor on `helper()` inside `main()`, press `gd` — confirm the
cursor jumps to `helper`'s actual definition a few lines above. Press
`K` on `helper` to see whatever hover documentation the language server
provides for it (a plain function with no docstring will show minimal
info — this is expected, not a failure).

### Task 5 — Recognize the Neovim-vs-Vim boundary

You're editing a single `nginx.conf` on a remote server over SSH, making
a quick one-line change. Separately, you're actively writing a 300-line
Python module locally, with imports across several files. Per §1.7, which
scenario benefits meaningfully from Neovim's built-in LSP client, and
which is served just as well by plain Vim and the patterns from Demos
M06-01/M06-02/M06-03/M06-04?

## What You Learned

- ✅ Explained what Neovim provides natively that plain Vim does not, and
  correctly ruled out `:terminal` as a genuine differentiator
- ✅ Wrote a minimal `init.lua` enabling a language server for a project
- ✅ Read inline LSP diagnostics and navigated between them
- ✅ Used go-to-definition and hover documentation via the built-in LSP
  client, with no plugin installed
- ✅ Used `:checkhealth` to diagnose setup problems
- ✅ Explained when switching to Neovim's built-in tooling is actually
  worth it versus staying with plain Vim

## Part 3 — Break-Fix Scenario

### Scenario — NovaTech Ops: two Neovim LSP incidents

> **Incident report — NovaTech Ops, ticket #4921**
>
> Alert 1: "A new hire says `gd` and `K` 'just don't do anything' on their
> fresh Neovim install." Alert 2: "Diagnostics show up fine, but `:lua
> vim.diagnostic.open_float()` says there's nothing to show, even though
> there's clearly a red underline right under the cursor."

**Incident 1 — gd/K appear to do nothing**

```
(fresh Neovim install, init.lua has vim.lsp.enable('pylsp'))
(opens a .py file, presses gd on a function call)
(nothing happens — cursor doesn't move)
```

Tasks:
1. Reference §1.2 and Task 1 — what are the two genuinely separate
   things that both need to be true for `gd`/`K` to work at all, and
   which one is easy to forget is a completely separate installation
   step from Neovim itself?
2. What command from §1.6 would be the first, structured place to check
   what's actually missing, rather than guessing?
3. Reference §1.5 — even once the language server is confirmed running,
   what else might be worth checking specifically about this Neovim
   version, given the caveat in that section?

**Incident 2 — diagnostic visible but open_float finds nothing**

```
(a red underline is clearly visible under the cursor)
:lua vim.diagnostic.open_float()
(No informational text found, or similar)
```

Tasks:
1. What's the most likely explanation for a visual diagnostic marker
   existing while a command specifically meant to show that diagnostic's
   text reports nothing — is this necessarily a bug, or could cursor
   position itself be the actual issue?
2. What would confirm which explanation is correct, without guessing?

<details>
<summary>Answers</summary>

**Incident 1:** Two genuinely separate things both need to be true: (1)
Neovim's built-in LSP *client* needs to be configured to talk to a server
(the `vim.lsp.enable('pylsp')` line in `init.lua`), and (2) the actual
`pylsp` *binary* needs to be installed and runnable on the system — this
second part is never built into Neovim itself, in the same way the actual
language server is always a separate, language-specific install (§1.2).
It's easy for a new hire to configure the client-side line correctly and
still have nothing work at all if the server binary itself was never
installed. `:checkhealth` (§1.6) is the first, structured place to check
this — its LSP-related section (when available for the configured
server) is designed to surface exactly this kind of "client configured,
server missing" mismatch rather than a silent no-op. Beyond that, given
this section's own caveat, it's also worth directly confirming whether
this specific Neovim version binds `gd`/`K` by default at all, or expects
them to be set explicitly in `init.lua` — a config gap there would produce
exactly this same "nothing happens" symptom, for a completely different
reason than a missing server binary.

**Incident 2:** This isn't necessarily a bug — a diagnostic underline can
span a range of characters on a line, and `vim.diagnostic.open_float()`
shows diagnostics for the exact line (and, more precisely, position) the
cursor is currently on. If the cursor isn't actually positioned on the
specific character range the diagnostic applies to — even while visually
appearing to be "under" the underline depending on how the terminal
renders it — the command can legitimately report nothing for that exact
spot. Moving the cursor directly onto the underlined characters and
retrying, or using `]d`/`[d` (§1.4) to jump precisely to the diagnostic's
actual recorded position rather than trusting eyeballed cursor placement,
is the direct way to confirm which explanation is correct.

</details>

## Interview Prep

**Q1. What's the actual, defensible difference between Neovim and Vim that would justify recommending one over the other for a specific engineer, beyond personal preference?**

The defensible, non-preference-based difference is Neovim's built-in LSP
client — real-time diagnostics, semantic go-to-definition, and hover
documentation without installing a plugin, which plain Vim genuinely
lacks natively. Features like `:terminal` are NOT a valid part of that
argument, since Vim has had them since version 8 — citing them as
Neovim-exclusive is a common but incorrect justification. The honest
recommendation hinges on whether that LSP value is worth the added setup
and per-language-server maintenance for that specific engineer's actual
workload.

---

**Q2. A colleague says "Neovim is just better because of `:terminal`." How would you respond?**

That's a common but factually incorrect justification — Vim added
`:terminal` in version 8, so it isn't a Neovim-exclusive feature at all.
The actual, defensible differentiator is the built-in LSP client (native
diagnostics/hover/go-to-definition with no plugin), not the terminal
feature.

---

**Q3. `gd` is pressed on a function call and nothing happens. What are the two genuinely separate things worth checking, and why does confusing them waste debugging time?**

Whether the LSP *client* is actually configured to talk to a server at
all (an `init.lua`/config issue), versus whether the actual language
server *binary* is installed and running (a completely separate,
external, language-specific installation step that Neovim never bundles).
Confusing the two wastes time because the fix for each is entirely
different — reconfiguring Lua won't help if the server binary was simply
never installed, and reinstalling the server won't help if the client-side
config never enabled it in the first place. `:checkhealth` is the
structured way to distinguish which one is actually the problem.

---

**Q4. Why does this demo explicitly avoid asserting specific default keybindings (like whether `gd`/`K` work with zero configuration) as settled fact?**

Because Neovim's default LSP keybindings have genuinely changed across
versions as built-in LSP support has matured — asserting a specific
version's defaults as universally true would be presenting an unverified,
version-dependent claim as settled fact, which risks being simply wrong
for whatever version someone's actually running. Checking `:help
lsp-defaults` against the actual installed version is the reliable way to
know, rather than trusting an assumed constant across all versions.

---

## Key Takeaways

1. **Neovim's genuine differentiator is the built-in LSP client — not
   `:terminal`, which Vim has had since version 8.** Citing `:terminal`
   as a reason to prefer Neovim is a common but factually incorrect
   argument.
2. **LSP requires two genuinely separate things to both be true: the
   editor-side client configured to talk to a server, and the actual
   language server binary installed and running.** Neovim provides the
   first natively; the second is never built into either editor and is
   always a separate, language-specific install.
3. **`gd` and `K` give semantically-resolved results — an actual
   definition location and real documentation — not a text-pattern match**
   the way Demo M06-01's `/search` or basic navigation would.
4. **`:checkhealth` is the structured first stop for diagnosing a broken
   setup**, rather than guessing from a silent failure or an unclear
   symptom — it's specifically designed to distinguish "client
   misconfigured" from "server missing" and similar setup mismatches.
5. **Default LSP keybindings have genuinely changed across Neovim
   versions** — never assume a specific keymap is bound by default
   without checking `:help lsp-defaults` against your actual installed
   version.
6. **The switch to Neovim's built-in tooling is worth it specifically for
   active code-writing in a language with a good language server
   available** — not for the kind of config/script editing Demos
   M06-01/M06-02/M06-03/M06-04 already cover thoroughly, where plain Vim's universal,
   zero-setup availability is often genuinely sufficient on its own.

> **Demo scope:** Primary concept: Neovim's built-in LSP client for
> diagnostics, hover, and go-to-definition with no plugin. Supporting
> concepts: minimal Lua configuration, `:checkhealth`, correcting the
> `:terminal` misconception.
> Estimated completion time: 38 minutes (reading + hands-on + verification).
> Checkpoints: 5 natural stopping points (after Task 1 through Task 5).

## Quick Commands Reference

### Configuration and diagnostics

| Command | Purpose |
|---|---|
| `vim.lsp.enable('server_name')` | Enable a language server (in `init.lua`) |
| `:checkhealth` | Run structured setup diagnostics |
| `]d` / `[d` | Jump to next / previous diagnostic |
| `:lua vim.diagnostic.open_float()` | Show the diagnostic under the cursor |
| `:lua vim.diagnostic.setloclist()` | Populate the location list with all diagnostics |

### Navigation and documentation

| Command | Purpose |
|---|---|
| `gd` | Go to definition (semantic, via the language server) |
| `K` | Show hover documentation |

## Appendix A — Anki Cards

**`m06-05-neovim-lsp-anki.csv`:**

```
#deck:Linux Mastery::Module 6 - Editors::Demo M06-05 - Neovim Built-in LSP
#separator:Comma
#columns:Front,Back,Tags
"What is the genuine, defensible feature difference between Neovim and Vim, as opposed to a commonly-cited but incorrect one?","Genuine: Neovim's built-in LSP client (native diagnostics, hover, go-to-definition, no plugin required). Commonly incorrect: :terminal — Vim has had an integrated terminal since version 8, so it isn't Neovim-exclusive at all.","demo-m06-05,neovim,differentiator,common-mistake"
"What two genuinely separate things both need to be true for gd/K to actually work?","1) The editor-side LSP client needs to be configured to talk to a server (e.g. vim.lsp.enable('pylsp') in init.lua). 2) The actual language server binary itself needs to be installed and running — this is never built into either editor, and is always a separate, language-specific install step.","demo-m06-05,neovim,lsp,common-mistake"
"What's the practical difference between gd (go to definition) and a text search like /pattern for finding where something is defined?","gd uses the language server's actual semantic understanding of the code (imports, scope, types) to jump to the real definition location. A text search just matches literal text patterns and has no understanding of whether a match is actually the definition versus just another mention of the same name.","demo-m06-05,neovim,lsp,gd,cross-demo"
"What is :checkhealth, and why is it the recommended first stop when an LSP setup seems broken?","A structured set of self-checks covering Neovim's core systems and installed providers/plugins, with clear per-check status. It's the first stop specifically because it's designed to distinguish common setup mismatches (like client configured but server binary missing) rather than leaving you to guess from a silent failure.","demo-m06-05,neovim,checkhealth,troubleshooting"
"Why does this demo avoid stating a specific Neovim version's default LSP keybindings as settled fact?","Default LSP keybindings have genuinely changed across Neovim versions as built-in LSP support has matured — asserting one version's defaults universally would risk being simply wrong for whatever version someone is actually running. :help lsp-defaults against the actual installed version is the reliable source.","demo-m06-05,neovim,lsp,version-dependent"
"What is the relationship between Neovim's built-in LSP client and an actual language server like pylsp or gopls?","Neovim provides the client side of the LSP conversation natively (vim.lsp). The language server itself (pylsp, gopls, etc.) is a completely separate, language-specific binary that's never bundled with either editor and must be installed on its own.","demo-m06-05,neovim,lsp,fundamentals"
"When is switching to Neovim's built-in LSP tooling actually worth it, versus staying with plain Vim and Demos M06-01/M06-02/M06-03/M06-04's patterns?","When doing enough active code-writing (not just config/script editing) in a language with a good language server available, that real-time diagnostics and semantic navigation meaningfully speed up the work — and you're willing to maintain a per-language-server setup, which plain Vim's zero-dependency availability doesn't require.","demo-m06-05,neovim,vim-comparison,cross-demo"
```

## Appendix B — Quiz

**`m06-05-neovim-lsp-quiz.md`:**

````markdown
# Quiz — Demo M06-05: Neovim: Built-in LSP, Lua Config & checkhealth

> One correct answer per question unless stated otherwise.
> Target: 80% or above before moving to Demo 24.

---

**Q1.** A colleague argues Neovim is better than Vim primarily because
of its integrated `:terminal`. Is this a valid, defensible argument?

- A) Yes — `:terminal` is exclusive to Neovim
- B) No — Vim has had an integrated terminal since version 8; `:terminal` is not a genuine Neovim-vs-Vim differentiator
- C) Yes, but only on Linux; `:terminal` is Neovim-only on other operating systems
- D) The question is meaningless — neither editor has ever supported `:terminal`

<details>
<summary>Answer</summary>

**B** — This is exactly the misconception this demo corrects up front in
§1.1. The genuine differentiator is the built-in LSP client, not
`:terminal`.

</details>

---

**Q2.** `gd` is pressed on a function call and nothing happens. What are
the two genuinely separate possibilities worth checking?

- A) Whether the file has been saved recently, and whether Neovim is the correct editor version
- B) Whether the LSP client is actually configured to talk to a server (init.lua), and whether the actual language server binary is installed and running — two separate requirements
- C) Whether the terminal window is wide enough to display the jump
- D) `gd` requires a mouse click first in modern Neovim versions

<details>
<summary>Answer</summary>

**B** — This is the exact structure of Break-Fix Incident 1 — client
configuration and server installation are two separate things that both
need to be true, and confusing which one is broken wastes debugging
effort on the wrong fix.

</details>

---

**Q3.** What does `:checkhealth` actually do?

- A) It automatically fixes any detected configuration problems
- B) It runs a structured set of self-checks across Neovim's core systems and installed providers/plugins, reporting clear per-check status, designed to surface exactly the kind of setup mismatch that causes silent LSP failures
- C) It only checks network connectivity
- D) It is a Vim-only command with no Neovim equivalent

<details>
<summary>Answer</summary>

**B** — `:checkhealth` is diagnostic, not corrective — it tells you
what's wrong (or confirms nothing is), rather than fixing anything
automatically.

</details>

---

**Q4.** Why does `gd` (go to definition) via the LSP client behave
differently from a plain text search like `/functionname` for finding
where something is defined?

- A) They behave identically — `gd` is just a shortcut for the equivalent search
- B) `gd` uses the language server's actual semantic understanding (scope, imports, types) to jump to the real definition; a text search only matches literal characters and has no way to distinguish the actual definition from any other mention of the same text
- C) `gd` only works within the same file; text search can cross files
- D) Text search is always faster and more reliable than `gd`

<details>
<summary>Answer</summary>

**B** — This is the concrete payoff described in §1.5 — semantic
resolution versus pattern matching, the same underlying distinction as
why `ctags`-free, LSP-based navigation is more reliable on real codebases
with many similarly-named symbols.

</details>

---

**Q5.** What is the relationship between Neovim and an actual language
server like `pylsp`?

- A) `pylsp` ships bundled inside Neovim and requires no separate install
- B) Neovim provides the client side of the LSP conversation natively; the language server itself is a completely separate, language-specific binary that must be installed on its own, in either editor
- C) Language servers only work with Vim, not Neovim
- D) Neovim automatically detects and installs the correct language server for any file type opened

<details>
<summary>Answer</summary>

**B** — Neither editor bundles language servers themselves — they're
always a separate install, since they're language-specific. Neovim's
native contribution is the client-side plumbing to talk to whichever
server you've installed.

</details>

---

**Q6.** Per this demo's stance on version-dependent claims, why does it
avoid stating that `gd`/`K` are bound by default in every Neovim
installation?

- A) Because they never work by default in any version, full stop
- B) Because default LSP keybindings have genuinely changed across Neovim versions, and asserting one version's behavior as universal risks being factually wrong for a different installed version
- C) Because keybindings are randomized per installation for security reasons
- D) Because this is a purely hypothetical concern with no real basis

<details>
<summary>Answer</summary>

**B** — This is a real, documented pattern of change across Neovim's LSP
support maturing over time — not a hypothetical or invented caveat.

</details>

---

**Q7.** You're editing a single-line change in `nginx.conf` on a remote
server over SSH. Per §1.7, is Neovim's built-in LSP client likely to add
meaningful value here?

- A) Yes — LSP diagnostics improve every editing task regardless of context
- B) Not meaningfully — this is exactly the kind of config-file editing Demos M06-01/M06-02/M06-03/M06-04 already cover thoroughly, where plain Vim's zero-setup universal availability is typically sufficient on its own
- C) Yes, but only if the remote server has a GUI installed
- D) No editor can be used over SSH, making this question moot

<details>
<summary>Answer</summary>

**B** — LSP's value shows up specifically in active code-writing with a
good language server available — a quick config-file edit over SSH is
squarely in plain Vim's wheelhouse, per §1.7's actual criterion for when
the switch is worth it.

</details>

---

Score guide:

| Score | Action |
|---|---|
| 7/7 | Import Anki cards, move to Demo 24 |
| 6/7 | Review the wrong answer, then proceed |
| 5/7 | Re-read the relevant section, retry those questions |
| Below 5/7 | Re-read the full demo and redo the walkthrough before proceeding |
````