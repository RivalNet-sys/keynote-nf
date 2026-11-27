# KeyNote NF — translation workflow (UI, tips, help)

This document describes how the Czech and Slovak translations of KeyNote NF 2.1.7.1
were produced, so that other languages can follow the same (reviewable) process.
Everything is done with plain files; the translation itself is text work, not code.

## 1. Sources

- `Lang/LNG_2.1.7/Keynote.english.lng` — UI strings (KDL format, sections like
  `[TMainForm]`, lines `Key=Value`; Humanize escapes `\^` and `\+`).
- `Lang/LNG_2.1.7/keynote.english.tip` — 115 tips, one per section.
- `src/misc_files/KeyNoteNF_Help.knt` (compiled help) / `doc/help/KeyNoteNF_Help.knt`
  (plain text) — `.knt` notebook: `GFKNZ32` + zlib body + image trailer, or plain
  `#!GFKNT` text with external images.

## 2. Extract the strings

- Parse the `.lng` into `(section, key, value)` records; keep section/key/order.
- Parse the `.tip` into indexed entries.
- Unpack the help `.knt`: decompress the zlib body, tokenize the RTF and keep every
  raw formatting token, hyperlink field (`fldinst`/`fldrslt`) and image reference
  intact. Only spans that contain translatable letters become units.
  Paragraphs are identified as `folder:paragraph`; topic names are separate units.

## 3. Chunk for the model

Split strings into chunks of ~8–10 k paragraphs / ~250 strings so that each model
call stays small and the output is easy to validate.

## 4. Prompt templates

**UI / tips** (one chunk per call):

```
Translate these KeyNote NF strings into <language>.
Read prompt.json and write translation.json with the same ids.
Rules:
- keep placeholders: %s %d %% {0}
- keep menu accelerators "&" and Humanize escapes \^ \+
- keep product names, file names, paths, URLs, shortcuts (Ctrl+..., F1)
- no newlines inside a string
- terminology: note = ..., folder = ..., tree = ..., node = ..., tab = ...
Return only OK.
```

**Help paragraphs** (one chunk per call):

```
Translate these help paragraphs into <language>.
Each text contains formatting markers ⟪0⟫ ⟪1⟫ ...; keep all of them,
in the same order and count, and move them to the corresponding place
in the translated sentence.
Keep: KeyNote NF, RTF, HTML, INI, DDE, ZIP, UTF-8, .knt, URLs, e-mails,
file names, keyboard shortcuts, text in backticks.
No newlines; keep trailing punctuation and whitespace.
Return only OK.
```

**Hyperlink labels** (the visible text of help links) are translated in a separate
pass from a list of unique labels, then reinserted by the rebuild step.

## 5. Automatic checks (before a human sees it)

- every id appears exactly once, same order as the source;
- marker counts/order are identical (`⟪n⟫`);
- placeholders/accelerators preserved;
- no newlines;
- source strings that are purely technical stay unchanged;
- rebuild the `.lng`, `.tip` and `.knt` and open them in KeyNote to verify that
  formatting, links and images still work.

## 6. Native review (the important part)

A native speaker of the target language gets:

- a CSV with `type,id,english,ai_draft,correction,note` (generated from the draft),
- a portable KeyNote build with the language preview,
- a short guide (what to check: terminology, grammar, length, shortcuts,
  placeholders, diacritics).

The reviewer fills only the `correction` column; corrections are applied
mechanically by id. The review is mandatory before publishing.

## 7. Build and publish

- apply corrections, rebuild all files, test the UI, tips and help in the released
  KeyNote binary;
- update `keynote.lan` (`Name`, `LangFile`, `TipFile`, `Translator`);
- add the help notebooks to `src/misc_files` (compiled) and `doc/help` (plain text);
- open a PR with the language files, and add a translator credit line to the
  "How to use this help" topic of the translated help.

## 8. Licence and attribution

The translations are contributed under MPL 2.0 (same as KeyNote NF) and the
translator/reviewer is credited in `keynote.lan` and in the help file.

## Notes

- The application stores `LanguageUI` in the INI in the system ANSI code page; on
  Czech/Slovak Windows the diacritics round-trip correctly. For languages whose
  names cannot be represented in the ANSI code page, use an ASCII `Name` in
  `keynote.lan` (e.g. `Name=Greek`).
- The help is by far the largest part (~7 400 paragraphs); the UI and tips are small.
- Keep the original English master untouched; translations are added as new files.
