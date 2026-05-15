# Patch Index

Hand-maintained list of every patch in this repository. When you
submit a new patch, please add a row to the appropriate section
**in the same PR** — keeps this file honest without a separate
"who updates the index" step.

Last reviewed: 2026-05-15

> If you spot an entry that's drifted (file renamed, removed,
> description outdated), open a quick PR — index hygiene is welcome.

---

## INPA — diagnostics

### `patches/inpa/sgdat/` — per-ECU diagnostic scripts

| Patch | Target IPO | Language | Encoding | Description |
|---|---|---|---|---|
| [RADIO.en.example](patches/inpa/sgdat/RADIO.en.example.patch.yaml) | `RADIO.IPO` | English | cp1252 | Example template; 4 menu labels (Hauptmenü, Ident, Fehler, Steuern → English) |

### `patches/inpa/cfgdat/` — INPA config-driven IPOs

*(no patches yet — be the first)*

---

## NFS — flashing

### `patches/nfs/sgdat/`

*(no patches yet)*

### `patches/nfs/cfgdat/`

*(no patches yet)*

---

## NCS Expert — coding

### `patches/ncsexpert/sgdat/`

*(no patches yet)*

### `patches/ncsexpert/cfgdat/`

*(no patches yet)*

---

## How rows are filled

| Column | What goes here |
|---|---|
| **Patch** | Markdown link to the patch file, with the basename (drop `.patch.yaml`) as the link text |
| **Target IPO** | The IPO basename **with** extension, in `inline code`. Matches `original.name` in the patch YAML. |
| **Language** | Full English name for translations (English, Polish, Cyrillic / Russian, …) or `—` for non-translation patches. |
| **Encoding** | Whatever the patch's `target_encoding` says — usually `cp1252`. Non-cp1252 entries get a 🚧 prefix in the description. |
| **Description** | One sentence. Keep it under ~100 characters. Reference the original-language labels if useful. |

Multiple patches for the same IPO get separate rows, listed in
filename-alphabetical order so diffs are predictable.
