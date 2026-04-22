# JBs SD ModKit — Outstanding Questions Plan

## Vetting notes

This plan is directionally strong and worth executing, with three practical adjustments:

1. `mnfc` should remain a background lane, not the critical path.
2. The Porsche fix track can advance immediately from public source files, not just README text.
3. `fecswap` reverse engineering should be split into:
   - `phase 1`: local fingerprinting and behavior notes
   - `phase 2`: full disassembler-assisted analysis

With those changes, the order of work becomes:

1. source-backed runtime and Android Auto documentation
2. Porsche fix mechanics from actual source
3. `fecswap` partial reverse-engineering notes
4. `mnfc` reconstruction as longer-haul research

## What is already well-supported

From the current `JBDocs` set plus upstream repos, the following are already on solid ground:

- `MH2p_SD_ModKit` is the signed update wrapper and execution framework.
- `MH2p_AndroidAuto` enables Android Auto by editing `granted.fecs` with `fecswap` and then applying 5F-related changes with the built-in `pc` tool.
- The Porsche multi-phone bug fix is conceptually separate from activation, but current upstream Android Auto install logic folds it into the install path for affected Porsche firmware.
- MH2p mods can do more than FEC edits; the ecosystem also supports JAR/classpath-based behavior overrides and persistence repairs.

## The real remaining unknowns

The unresolved questions now cluster into three buckets:

1. **`fecswap` internals**
   - What is the `.fecs` file structure?
   - Are entries signed blobs, indexed records, or simple tagged containers?
   - Does `fecswap` validate signatures, preserve them, or just rearrange entries?

2. **`mnfc` packaging/signing workflow**
   - How are `.mnf`, `.cks`, and `_S.sig` actually generated?
   - Is `mnfc` a private helper, an extracted vendor tool, or a reverse-engineered reimplementation?
   - Which parts are reproducible from repo contents alone versus hidden tooling?

3. **Porsche fix mechanics at source level**
   - Exact wrapper layout around `aafix.jar`
   - exact `fix_partition_1008.c` behavior in code, not just README description
   - precise loader / bootclasspath integration details

## Recommended work plan

### Track 1 — Finish source-level documentation of known behavior

**Goal:** turn the repo from “good narrative notes” into “auditably sourced reference”.

1. Add a new doc: `runtime-sequence.md`
   - Show the end-to-end execution chain:
     - `Meta/auto.mnf`
     - `modkit.sh`
     - `modkit_install.sh`
     - `Update`
     - persisted `Post`
     - persisted `Persist`
     - `servicemgrmibhigh` wrapper
   - Include the exact persistence locations involved (`/mnt/persist_new`, `/mnt/ota/modkit`, `/mnt/app/eso/bin`).

2. Add a new doc: `android-auto-activation-path.md`
   - Split the Android Auto flow into:
     - FEC entitlement layer
     - 5F coding layer
     - Porsche-only repair layer
   - Include the exact command lines already recovered.

3. Tighten `research-questions-and-findings.md`
   - Reclassify some items from “open question” to “implementation unknown but behavior known”.
   - In practice, questions 1–6 are mostly answered; 7–9 are the real open items.

### Track 2 — Reverse-engineer `fecswap`

**Goal:** answer the highest-value remaining question with source-like evidence.

1. Pull the bundled ARM ELF from the repo and record:
   - SHA-1
   - `file` output
   - interesting strings
   - symbol table presence/absence

2. Open in Ghidra or IDA and answer these first:
   - argument parsing for `-a`, `-r`, `-rf`, `-f`
   - whether `.fecs` is parsed as a fixed record format or variable-length container
   - whether any crypto or signature verification library is called
   - whether FEC IDs are stored as strings, integers, or wrapped objects

3. Produce a new doc: `fecswap-reverse-engineering-notes.md`
   - CLI behavior
   - inferred file structure
   - edit algorithm
   - unresolved parts

**Why this track matters most:** it is the shortest path from “mystery” to hard evidence, and it directly supports both Android Auto and general MH2p feature-mod work.

### Track 3 — Separate package-generation mystery from mod-development work

**Goal:** avoid blocking useful repo progress on `mnfc`.

1. Treat `mnfc` reconstruction as a parallel research lane, not a prerequisite.
2. Document exactly what appears generated:
   - `Meta/*.mnf`
   - `*.cks`
   - `*_S.sig`
   - `installer.txt`
3. Compare multiple released packages to identify stable patterns:
   - field ordering
   - checksum placement
   - module naming
   - version encoding
4. Only then attempt reproduction.

**Working assumption:** package reproduction is harder and less urgent than understanding how mods actually work after execution begins.

### Track 4 — Close the Porsche-fix gap with actual source artifacts

**Goal:** replace README-level confidence with code-level confidence.

1. Pull the actual contents of the Porsche fix repo files, not just its README.
2. Capture:
   - `fix_partition_1008.c`
   - any wrapper scripts
   - any loader / `lsd.sh` patch material
3. Diff the documented intent against source reality.
4. Produce a new doc: `porsche-aa-fix-mechanics.md`.

## What I can already help with right now

### 1. Improve the question framing

Your current “top questions” list mixes solved behavior questions with unresolved implementation questions.
A tighter breakdown would be:

#### Behavior questions (mostly answered)
- How Android Auto is enabled
- What “codes 5F” means
- whether the Porsche fix is separate or integrated
- whether MH2p supports more than FEC edits
- whether MH2p is conceptually related to older M.I.B./FEC workflows

#### Implementation questions (still genuinely open)
- how `fecswap` works internally
- how `.fecs` is structured
- where `mnfc` comes from and how signing works
- exact source mechanics of the Porsche repair path

That reframing will make the repo feel more advanced and less stuck.

### 2. Suggest the next best concrete deliverables

If you want the repo to become a serious research notebook, the next three markdown files should be:

1. `android-auto-activation-path.md`
2. `fecswap-reverse-engineering-notes.md`
3. `porsche-aa-fix-mechanics.md`

Those three would close most of the practical gaps.

### 3. Offer a practical execution order

Best order:

1. `android-auto-activation-path.md` — fast win, mostly synthesis
2. `porsche-aa-fix-mechanics.md` — medium effort, likely high confidence gain
3. `fecswap-reverse-engineering-notes.md` — hardest, but highest long-term value
4. `mnfc` reconstruction notes — long-haul track

## Suggested edits to your existing docs

### `research-questions-and-findings.md`

- Rename **“Top questions”** to **“Key research questions”**.
- Split the section into:
  - **Behavior now established**
  - **Implementation still unresolved**
- Move question 6 (M.I.B. relationship) into a “comparative context” section rather than keeping it as a live unknown.

### `binary-sources-and-build-notes.md`

- Add a short subsection called **“What can probably be ignored for now”**
  - example: device-native binaries like `pc` and `showimage` are important operationally but not the main research bottleneck.
- Add a subsection called **“Priority unknowns”**
  - `fecswap`
  - `mnfc`

### `repo-orientation.md`

- Expand the execution path into a numbered sequence diagram style.
- Add one paragraph on why `Post` exists at all: because some 5F-related changes only succeed after reboot into a different runtime state.

## My best recommendation

If the goal is to make the repo genuinely useful to future modders and to yourself six months from now, focus next on **documenting the Android Auto path cleanly** and **reverse-engineering `fecswap`**, while treating `mnfc` as a secondary packaging problem.

That is the highest payoff path because it improves both understanding and future mod authoring without requiring you to completely crack the packaging toolchain first.
