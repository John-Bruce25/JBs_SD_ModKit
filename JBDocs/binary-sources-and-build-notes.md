# Binary Sources And Build Notes

## Goal

This note tracks what we can currently prove about bundled binaries and generated package artifacts in this repository, and where the source trail goes cold.

## Confirmed findings

### `fecswap`

Path:

- `Data/ExceptionList.fec_2017327-0730/0/fecswap`

What is confirmed locally:

- It is an ELF executable.
- The ELF header identifies it as a 32-bit little-endian ARM binary.
- SHA-1 matches the checksum recorded in `Data/ExceptionList.fec_2017327-0730/0/installer.txt`:
  - `2E7ED478BAF70A4DCB48A6AFE60E82DF1C001C52`
- Git history shows it was introduced in commit `82f9452` (`ModKit v2`).

What public sources confirm:

- The project wiki has a page titled `Utilities: fecswap (ModKit addition)` describing its behavior.
- The wiki says it "Adds or removes signed FECs" and documents flags such as `-a`, `-r`, `-rf`, and `-f`.

What is not yet found:

- Source code for `fecswap`
- A public build script for `fecswap`
- A separate public repository containing `fecswap`

Current conclusion:

- `fecswap` is a bundled project utility whose behavior is publicly documented, but whose source is not present in this repository and was not found in the public sources reviewed so far.

### `mnfc`-generated package metadata

Affected files include:

- `Meta/auto.mnf`
- `Meta/ExceptionList/9999.0.0.mnf`
- `Meta/ExceptionList/fec/9999.0.0.mnf`
- `Meta/MMX2P_POSTSCRIPT/1.0.0.mnf`
- `Meta/MMX2P_POSTSCRIPT/script/1.0.0.mnf`
- `Data/*/installer.txt`

What is confirmed:

- These files contain `Created-By` markers such as `mnfc 0.32` and `mnfc 0.39`.
- They describe the package tree, included modules, install operations, script payloads, and checksums used by the MH2p update process.

What is not yet found:

- Source code for `mnfc`
- Public documentation for `mnfc`
- A reproducible build recipe showing how the current `Meta/` and `installer.txt` files were generated

Current conclusion:

- `mnfc` appears to be an external packaging tool used to emit MH2p manifest and installer metadata. The generated artifacts are in this repo, but the tool itself is not.

## Other bundled or external components

### `splashscreen.jpg`

Path:

- `Data/ExceptionList.fec_2017327-0730/0/splashscreen.jpg`

Current conclusion:

- Bundled asset used by `modkit.sh` via `showimage`.
- No source asset history or generation process is documented in this repo.

### Device-native tools referenced by scripts

The following are referenced by repo scripts but are not included in the repository:

- `/mnt/app/eso/bin/apps/showimage`
- `/mnt/app/armle/usr/bin/pc`
- `/mnt/app/eso/bin/servicemgrmibhigh`

Current conclusion:

- These are target-device binaries or scripts from the MH2p environment, not project-owned source in this repo.
- The wiki documents `pc` usage, but not its implementation source.

## Related external tools

### `vwcoding.ru` FEC generator

Reference:

- `https://vwcoding.ru/en/utils/fec/`

What it is:

- A public FEC/SWaP code generator for VAG platforms.
- The page describes generating FEC codes from `VIN` and `VCRN`.
- It lists many of the same FEC identifiers seen in the broader MIB ecosystem, including:
  - `00060800` - Apple CarPlay
  - `00060900` - Google AndroidAuto
  - `FFFFFFF9` through `FFFFFFFE` - clear/remove-style FEC operations

Why it is relevant:

- It confirms that the FEC IDs used by `fecswap` are part of a broader VAG FEC/SWaP ecosystem, not unique to this repo.
- It gives outside evidence that the repo's FEC operations are working in a real, shared format/domain.

Why it is not the same thing as `fecswap`:

- The `vwcoding.ru` tool is presented as a code generator for upload through tools such as OBDeleven or ODIS.
- `fecswap` is documented in the ModKit wiki as a local utility that adds or removes signed FECs in a `.fecs` file on the target system.
- No public evidence was found that `vwcoding.ru` publishes the source for `fecswap`, or that `fecswap` was built from that project.

Current conclusion:

- `vwcoding.ru` is related in subject matter and useful for context, but it is not currently evidence of the origin or build process for the `fecswap` binary in this repository.

## How these artifacts appear to be made

Based on the repo contents plus the public wiki:

1. Repo-authored shell scripts implement the mod framework behavior.
2. A packaging tool called `mnfc` is used to generate `.mnf` manifest files and `installer.txt` metadata.
3. The top-level package is then represented by:
   - `Meta/*.mnf`
   - checksum files such as `.cks`
   - signature files such as `_S.sig`
4. Bundled binaries like `fecswap` are dropped into the payload tree under `Data/...` and then referenced from the generated installer metadata with hashes.

Important caveat:

- The exact command line, config format, and signing workflow used to produce the current package have not yet been recovered from this repo.
- The README states that checksum and signing methods were discovered by reverse engineering MH2p binaries, but it does not include the implementation details.

## Best current answer to "where do the binaries come from?"

Short answer:

- Some binaries are shipped from the target system itself and only referenced at runtime.
- `fecswap` is shipped inside this repo as a prebuilt ARM executable.
- The source for `fecswap` and the source/tooling for `mnfc` are not present here and were not found in the public sources checked so far.

## Recommended next investigation steps

1. Reverse-engineer `fecswap` locally with Ghidra or another ELF-capable disassembler to recover CLI parsing and file format behavior.
2. Search the project wiki and linked community resources for mentions of `mnfc`, manifest generation, or signing workflows.
3. Compare this repo against downloadable mod bundles from the project site to see whether package metadata is reproduced consistently.
4. If source recovery matters, inspect additional LawPaul repos and community posts for unpublished tooling references.

## Sources checked

- Local repo files and git history
- `README.md` in this repo
- LawPaul GitHub repo page for `MH2p_SD_ModKit`
- LawPaul project site: `https://lawpaul.github.io/MH2p_SD_ModKit_Site/`
- `vwcoding.ru` FEC generator: `https://vwcoding.ru/en/utils/fec/`
- LawPaul wiki pages mirrored by `github-wiki-see.page`, including:
  - `Utilities: fecswap (ModKit addition)`
  - `Utilities: pc`
  - `Development: software update`

## Working confidence

- High confidence:
  - `fecswap` is bundled here as a prebuilt ARM ELF.
  - `mnfc` generated the manifest and installer metadata.
  - source for both is absent from this repo.
- Medium confidence:
  - `mnfc` is likely a private or unpublished helper tool rather than a common public utility.
- Low confidence:
  - any specific implementation details for how `fecswap` was written or how `mnfc` performs signing internally.
