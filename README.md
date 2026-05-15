# IPO Community Patches

A shared repository of community-contributed patch files for BMW
diagnostic / coding `.IPO` scripts — translations, label corrections,
threshold overrides, and other non-invasive constant-level tweaks.

Patches are applied with [**inpax-ipo-editor**](https://github.com/emdzej/inpax/tree/main/apps/ipo-editor),
part of the [inpax](https://github.com/emdzej/inpax) toolchain. They
never modify bytecode — only the values of constants the original IPO
already declares — so they're safe to layer on top of any compatible
script, and trivially reversible.

## What this is for

If you've ever wished:

- the German labels in a particular INPA screen were in your language;
- a default sample period was different;
- a unit-conversion factor was tweaked for your specific car;
- a confusing acronym was spelled out;

…and the change is local to *constants the IPO already declares*,
then a patch file is the right tool. Submit it here, others benefit.

This repo is not for new IPO scripts, not for forks of BMW source
code, and not for anything that requires recompiling from `.IPS`. For
those, see [inpax](https://github.com/emdzej/inpax) directly.

## A short tour of the BMW IPO world

**IPO** stands for the compiled bytecode format that BMW's diagnostic
scripts ship in (`.IPO` files). It's interpreted by an embedded VM
that lives inside three Windows applications BMW shipped to its
dealer network:

- **INPA** (Interpreter for Test Procedures) — the diagnostic /
  troubleshooting tool. Used to read fault codes, watch live sensor
  data, run actuator tests. Scripts live under `EC-APPS/INPA/`.
- **NFS** (*Nutzfahrzeug-Service*, commercial-vehicle service) — the
  same interpreter, with a script library aimed at BMW's commercial
  / motorcycle lineage. Scripts under `EC-APPS/NFS/`.
- **NCS Expert** — the coding tool. Reads/writes ECU "coding" data
  (option codes, country variants). Same VM, different script set
  under `NCSEXPER/`.

All three programs share **the same IPO interpreter** and **the same
script format** — but they ship **different script libraries** for
different jobs. INPA's library is diagnostic (read fault codes, watch
live sensor data), NCS Expert's is coding-oriented (read/write option
bytes), NFS targets commercial-vehicle ECUs. So a RADIO diagnostic
script is part of INPA/NFS, not NCS Expert; an NCS Expert coding
script wouldn't be useful inside INPA. The interpreter is shared, the
*purpose* of each app's script set is not.

What this means for patches: the same patch *format* and the same
**inpax-ipo-editor** tooling work across all three product trees.
You'll just always be patching a script in whichever tree it
naturally belongs to.

**IPS** is the source language IPO files are compiled from. C-like
syntax, BMW-internal. Only a handful of IPS sources have surfaced
publicly over the years — the vast majority of scripts in any BMW
install ship only as compiled IPOs, with no corresponding IPS — so
recompiling from source is rarely an option. Patches are what fills
the gap.

> **Why patches instead of just shipping modified IPOs?**
> Three reasons. **Auditability** — a patch is a small human-readable
> YAML file; a modified IPO is opaque bytes. **Provenance** — patches
> declare which exact IPO they target via SHA-256, so you can't apply
> a translation patch made for one revision to a different one by
> mistake. **Composability** — multiple patches stack cleanly (a base
> translation + a personal terminology override, say) where multiple
> modified IPOs can't.

## Directory layout

```
patches/
├── inpa/
│   ├── cfgdat/        ← config-driven IPOs (menus, settings)
│   └── sgdat/         ← SGBD-driven IPOs (per-ECU diagnostic scripts)
├── nfs/
│   ├── cfgdat/
│   └── sgdat/
└── ncsexpert/
    ├── cfgdat/
    └── sgdat/
```

Patches live in the same relative path as the IPO they target. A
patch for `EC-APPS/INPA/SGDAT/RADIO.IPO` lives in
`patches/inpa/sgdat/`.

File naming convention:

```
<IPO-basename>.<purpose>.<author-tag>.patch.yaml
```

Examples:

- `RADIO.en.emdzej.patch.yaml` — English translation by emdzej
- `MS43.pl.someone.patch.yaml` — Polish translation by someone
- `RADIO.shorter-labels.emdzej.patch.yaml` — non-translation tweak

Multiple patches targeting the same IPO with different purposes can
coexist in the same directory.

## Quick start — applying a patch from this repo

```bash
# 1. Install the tooling globally (one-time setup; requires Node 20+).
npm i -g @emdzej/inpax-ipo-editor

# 2. Grab the patch from this repo.
git clone https://github.com/emdzej/ipo-community-patches
cd ipo-community-patches

# 3. Apply.
ipo-editor patch apply \
  ~/Downloads/inpa/EC-APPS/INPA/SGDAT/RADIO.IPO \
  patches/inpa/sgdat/RADIO.en.example.patch.yaml \
  -o ~/Downloads/inpa/EC-APPS/INPA/SGDAT/RADIO.IPO.patched
```

If your local RADIO.IPO's SHA-256 doesn't match the patch's expected
checksum, apply will refuse with a clear error. That's the whole
point — you can't accidentally translate the wrong revision. If you
*know* the deviation is benign, override with `--ignore-checksum`,
but verify the result first.

## ⚠ Before you change anything — read this

**Not every string in an IPO is a user-facing label.** The constant pool
is a flat list of every literal the script uses, mixed together: UI
labels, job names passed to EDIABAS, ECU variant identifiers, format
specifiers, error keys, internal markers. They all look the same in
`patch init` output.

Modifying the wrong one will silently break the script in ways that are
hard to debug. Common traps:

- **EDIABAS job names** — strings like `"INITIALISIERUNG"`, `"IDENT"`,
  `"STATUS_LESEN"`, `"FS_LESEN"`. These get passed to `INPAapiJob` and
  must match a job name in the target SGBD (`.PRG` file) verbatim.
  Translating `"INITIALISIERUNG"` → `"INITIALIZATION"` makes the script
  call a non-existent job. The radio never initialises. Nothing displays.
- **SGBD / ECU variant names** — strings like `"RADIO"`, `"RAD_83"`,
  `"MS43"`. The script compares these against `VARIANTE` from EDIABAS;
  a mismatch trips the "Steuergerät nicht gefunden" error path and the
  script exits.
- **Result-field keys** — strings like `"ID_BMW_NR"`, `"JOB_STATUS"`,
  `"STAT_LL_EIN"`. Used to look up values from EDIABAS job results.
  Rewriting them returns "field not found".
- **Format strings** — `"4.2"`, `"%5.2f"`, etc. These control how
  numeric values render. Editing them changes precision / width.
- **Constant filenames** — `"inpa.h"` and other include references.
  Internal; leave alone.

**Always check where a constant is used before editing it.** The
[inpax CLI's disassembler](https://github.com/emdzej/inpax/tree/main/apps/cli)
makes this fast:

```bash
# inpax disasm comes from the inpax CLI; install once via:
#   npm i -g @emdzej/inpax-cli
inpax disasm RADIO.IPO | grep -B 2 -A 2 'const\[13\]'
```

…will show you the bytecode around every reference to `const[13]`,
so you can tell whether it's loaded as an argument to `text` (UI),
to `INPAapiJob` (job name — don't touch), or somewhere else.

**Default to the conservative path:**

1. Generate a patch in `--with-notes` mode.
2. Disassemble the IPO once and keep it open as a reference.
3. For every constant you intend to change, locate its usage. If you
   can't explain in one sentence what it does, **don't change it**.
4. Test the result before submitting. A patched IPO that crashes the
   script at load time is more useful as a learning experience than as
   a community contribution.

When in doubt: **smaller patches are better**. A patch that changes
three labels you're certain about is more valuable than one that
changes thirty and breaks two of them.

## Creating a patch — overview

A patch is generated from an IPO + your edits. The standard flow:

1. **Generate a starter patch** listing every string in the IPO.
2. **Disassemble the IPO** and verify which constants are UI labels
   (see warning above — this step is not optional for non-trivial
   patches).
3. **Edit the YAML** with your changes (translation, term swaps, …).
4. **Test by applying** to a fresh copy of the IPO.
5. **Open a PR** to this repo.

### 1. Generate a starter

```bash
ipo-editor patch init \
  ~/Downloads/inpa/EC-APPS/INPA/SGDAT/RADIO.IPO \
  --location inpa \
  --description 'English translation of RADIO labels' \
  --with-notes \
  -o RADIO.en.<your-tag>.patch.yaml
```

That produces a YAML file looking like:

```yaml
inpax_patch_version: 1
original:
  name: RADIO.IPO
  location: inpa
  checksum:
    algorithm: sha256
    value: 8b3edfee580000ac8b71b79660daf2cdb922e4ab511ab82bd77def28b89e3341
target_encoding: cp1252
description: English translation of RADIO labels
created_at: 2026-05-15T07:21:26.402Z
patches:
  - index: 13
    type: string
    value: Hauptmenü
    notes: |
      const[13] at offset 0x1A2 (10 bytes)
  - index: 15
    type: string
    value: Info
  …
```

### 2. Edit the YAML

Open the file in your editor. Change `value:` fields to your
translations. Leave `index`, `type`, and `original.checksum` alone.
Trim entries you're not translating — only entries you actually
changed need to ship.

### 3. Apply and test

```bash
ipo-editor patch apply \
  ~/Downloads/inpa/EC-APPS/INPA/SGDAT/RADIO.IPO \
  RADIO.en.<your-tag>.patch.yaml \
  -o /tmp/RADIO.en.IPO

# Sanity-check by viewing the patched file in the TUI:
ipo-editor /tmp/RADIO.en.IPO
```

If your patch is for a non-cp1252 language (Polish, Czech, Cyrillic,
Greek, …), **read the encoding caveat below before submitting**.

## Submitting a patch — fork → PR workflow

1. **Fork** this repository on GitHub.
2. **Clone your fork** locally.
3. **Add your patch file** under the appropriate
   `patches/<product>/<kind>/` directory using the naming convention
   `<IPO>.<purpose>.<author>.patch.yaml`.
4. **Commit** with a short, descriptive message:
   ```
   patch(inpa/sgdat): English translation for RADIO.IPO
   ```
5. **Push to your fork** and **open a Pull Request** against `main` of
   the upstream repo.

The PR template will ask for:

- Which IPO the patch targets, and where in the install it lives
- What language / purpose
- Whether you tested the result in a real application (`inpax`, INPA
  on Windows, etc.) and what the outcome was
- Source attribution if the translation is derivative of another
  community translation

Maintainers will run the patch through `inpax-ipo-editor patch apply`
on a reference IPO to verify the YAML parses, the checksum is real,
and the file ends up byte-valid. **Patches with wrong checksums or
schema errors will get a comment, not a rejection** — small fixes
are common and we'd rather help land the patch than turn people away.

## Encoding caveat (important — please read)

The IPO format has **no embedded encoding marker**. Stock INPA, NFS,
and NCS Expert always interpret string constants as **cp1252**
(Windows Western European). This limits what you can faithfully
translate without breaking compatibility:

- **English, German, French, Spanish, Italian, Portuguese, Dutch, etc.** —
  fully expressible in cp1252. Translate freely. Set `target_encoding: cp1252`.
- **Polish, Czech, Slovak, Croatian, Hungarian, Romanian** — need cp1250.
  Patches in cp1250 work in inpax (with `--input-encoding cp1250`) but
  **render as garbage in real INPA**. Set `target_encoding: cp1250`
  and accept that limitation.
- **Russian, Bulgarian, Serbian (Cyrillic), Greek, Turkish** — same
  story, different codepage (`cp1251`, `cp1253`, `cp1254` respectively).

Full explanation:
[docs/research/ipo-encoding.md](https://github.com/emdzej/inpax/blob/main/docs/research/ipo-encoding.md)
in the inpax repo.

Practically: if your translation contains characters outside cp1252,
flag this in the PR description (`encoding: cp1250 — inpax-only,
not compatible with stock INPA`) so the patch is filed under the
right caveat.

## License

Patch files in this repository are released under [CC0 1.0
Universal](https://creativecommons.org/publicdomain/zero/1.0/) — public
domain — *for the translations and metadata the contributors create*.
The original German string contents that patches *reference by index*
remain BMW's; this repo distributes neither the IPO files themselves
nor the original string bytes.

If you submit a patch, you agree to release your translation under CC0.

## Related projects

- **[inpax](https://github.com/emdzej/inpax)** — browser- and CLI-based
  INPA runtime; this is the tool that applies the patches.
- **[ediabasx](https://github.com/emdzej/ediabasx)** — sibling project
  reimplementing EDIABAS (the layer below INPA — handles cable / DS2 /
  CAN protocol).
- **[bimmerz](https://github.com/emdzej/bimmerz)** — I-bus monitoring SDK
  for listen-only access to BMW peer-to-peer messages.

## Questions, issues

Open an issue in this repo for repo-specific questions
(patch-submission process, conventions, naming). For tooling issues
(`inpax-ipo-editor` bugs, patch-format extensions), file against
[inpax](https://github.com/emdzej/inpax/issues) instead.
