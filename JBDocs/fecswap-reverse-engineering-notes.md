# Fecswap Reverse-Engineering Notes

This note is a staged reverse-engineering record for the bundled `fecswap` binary in `MH2p_SD_ModKit`.

## Current stage

This is a `phase 1` note.

It captures:

- local fingerprinting
- executable format details
- known behavioral use from upstream scripts
- what still requires a disassembler pass

It does not yet claim full internal understanding.

## Local fingerprint

Path:

- `Data/ExceptionList.fec_2017327-0730/0/fecswap`

SHA-1:

- `2E7ED478BAF70A4DCB48A6AFE60E82DF1C001C52`

Size:

- `347900` bytes

ELF header facts recovered locally:

- magic: `ELF`
- class: `ELF32`
- endianness: `LittleEndian`
- machine: `0x0028`
- entrypoint: `0x0804A6C4`

Interpretation:

- `0x0028` is ARM in ELF machine identifiers
- this aligns with the binary being a prebuilt ARM executable for the target MH2p environment

## Structural observations

Recovered locally:

- program header offset: `0x00000034`
- program header entry size: `32`
- program header count: `4`
- section header offset: `0x00054840`
- section header entry size: `40`
- section header count: `19`

Interpretation:

- the file is not a tiny stub
- it has a normal section/program header layout for a real compiled ELF
- section headers are present, though we have not yet resolved named sections or symbol content in this environment

## Known behavior from upstream usage

From the ModKit wiki and public Android Auto scripts, `fecswap` is used to modify `.fecs` files.

Known CLI usage:

- add:

```sh
fecswap -a 00030000 00050000 00060900 -f /mnt/persist_new/fec/granted.fecs
```

- remove:

```sh
fecswap -r 00030000 00050000 00060900 -f /mnt/persist_new/fec/granted.fecs
```

Documented flags from the public wiki:

- `-a` add FECs
- `-r` remove FECs
- `-rf` force remove FECs
- `-f` target `.fecs` file

So behaviorally, we have high confidence that `fecswap` is a file editor for the granted/illegal FEC store used by MH2p.

## What we do not yet know

- the actual `.fecs` file format
- whether entries are fixed-length records or variable containers
- whether signatures are preserved, regenerated, or simply copied through
- whether the tool validates signatures or only rearranges existing signed records
- whether FEC IDs are stored as strings, integers, or wrapped structures internally

## Best current hypothesis

Given the upstream usage pattern, the most likely behavior is:

1. read the target `.fecs` file
2. parse existing FEC entries
3. add or remove selected FEC IDs
4. write the modified file back in a format the MH2p system accepts

But this remains a hypothesis until a disassembler session confirms the parser and write path.

## Recommended next step

Open `fecswap` in Ghidra or IDA and answer these first:

1. where argument parsing handles `-a`, `-r`, `-rf`, and `-f`
2. whether the file parser uses fixed record lengths or variable-length structures
3. whether any obvious crypto or signature routines are called
4. where add/remove decisions are applied before writeback

## Confidence

- high confidence:
  - `fecswap` is a bundled ARM executable
  - it edits `.fecs` files
  - it is used directly in the Android Auto enablement path
- low confidence:
  - internal file format handling
  - signature/crypto behavior
  - implementation details of the write path
