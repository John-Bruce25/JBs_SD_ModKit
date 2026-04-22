# Research Questions And Findings

This note captures the highest-value research questions for the `MH2p_SD_ModKit` ecosystem, what we can answer now from available sources, and which areas still do not have strong source data.

## Key research questions

1. How does `MH2p_AndroidAuto` actually enable Android Auto?
2. What does "codes 5F" mean in concrete implementation terms?
3. Is the Porsche multi-phone fix separate from Android Auto activation, or already folded into it?
4. What exactly is patched by the Porsche fix repo?
5. Does the MH2p ecosystem support more than FEC/coding changes?
6. Is `MH2p_SD_ModKit` reusing older M.I.B.-style FEC workflows, or doing something different?
7. Where do `fecswap` and the `mnfc` packaging/signing tool come from, and how are they built?

## Behavior now established

### 1) How `MH2p_AndroidAuto` enables Android Auto

Status: `answered well`

Direct evidence from `LawPaul/MH2p_AndroidAuto`:

- `Update/install.sh` backs up `/mnt/persist_new/fec`
- it removes `/mnt/persist_new/fec/illegal.fecs`
- it preserves `granted.fecs` to `granted.fecs.bak`
- then it runs:

```sh
fecswap -a 00030000 00050000 00060900 -f /mnt/persist_new/fec/granted.fecs
```

Interpretation:

- Android Auto is enabled partly by editing the granted FEC file directly with `fecswap`
- the mod adds:
  - `00030000` - AMI/USB
  - `00050000` - Bluetooth
  - `00060900` - Google Android Auto

This is stronger than the earlier README-only wording because it shows the exact command and target file.

Sources:

- `https://github.com/LawPaul/MH2p_AndroidAuto/blob/main/Update/install.sh`
- `https://github.com/LawPaul/MH2p_AndroidAuto/blob/main/README.md`

### 2) What "codes 5F" means in practice

Status: `answered well`

Direct evidence from `LawPaul/MH2p_AndroidAuto`:

`Post/install.sh` runs the built-in `pc` tool against 5F persistence/coding keys:

```sh
/mnt/app/armle/usr/bin/pc b:0x5F22:0x600:19.6 1    # USB1 data
/mnt/app/armle/usr/bin/pc b:0x5F22:0x600:19.7 1    # USB2 data
/mnt/app/armle/usr/bin/pc b:0x5F22:0x22AD:7.7 1    # Google GAL
```

Interpretation:

- "codes 5F" is not just a vague statement
- in this mod, it specifically means flipping bits through the `pc` persistence client against module 5F data
- the Android Auto mod separates this from the FEC insertion step:
  - `Update` phase: modify FECs
  - `Post` phase: apply 5F changes after reboot

This also matches the ModKit README, which says `Post` is useful for 5F coding that does not work during update.

Sources:

- `https://github.com/LawPaul/MH2p_AndroidAuto/blob/main/Post/install.sh`
- `https://github.com/LawPaul/MH2p_SD_ModKit/blob/main/README.md`

### 3) Whether the Porsche fix is separate from activation

Status: `answered well`

Direct evidence now shows the Porsche fix is conceptually separate, but operationally folded into the current `MH2p_AndroidAuto` mod.

`Update/install.sh` in `MH2p_AndroidAuto` contains:

- a Porsche OEM check: `if [ "$OEM" = "PO" ]`
- a firmware-range check for `26xx` or `28xx`
- copy of `aafix.jar` into `/mnt/app/eso/hmi/lsd/jars/aafix.jar`
- execution of `fix_partition_1008 --fix` against `persistence.sqlite`

Interpretation:

- The Android Auto mod does more than FEC + 5F coding.
- It now also includes the Porsche-specific multi-device fix logic when conditions match.
- So the separate `pcm5-androidauto-connect-fix` repo still explains the fix, but its core behavior has effectively been integrated into `MH2p_AndroidAuto`.

Sources:

- `https://github.com/LawPaul/MH2p_AndroidAuto/blob/main/Update/install.sh`
- `https://github.com/fifthBro/pcm5-androidauto-connect-fix/blob/main/README.md`

### 4) What the Porsche fix changes

Status: `answered well`

Direct evidence from the `pcm5-androidauto-connect-fix` README and source:

- it identifies a bug in `DeviceManager$DeviceActivationRequestHandler.moveSelectionMarker()`
- the buggy logic demotes inactive devices to `NATIVE_SELECTED`
- the bytecode fix removes the line setting `device.setUserAcceptState(NATIVE_SELECTED);`
- it also fixes persistence partition `1008` in `persistence.sqlite`
- the repair replaces `NATIVE_SELECTED` with `DISCLAIMER_ACCEPTED`
- it recalculates CRC32 and updates the serialized blob cleanly
- it says the patch is loaded via bootclasspath injection in `lsd.sh`
- the public `fix_partition_1008.c` source confirms the SQLite queries, blob parsing, replacement logic, and CRC32 rewrite path
- the public ModKit wrapper scripts confirm the install/uninstall deployment flow for `aafix.jar` and `fix_partition_1008`

Interpretation:

- There are two distinct repair layers:
  - Java/bytecode behavior fix
  - persisted state repair in SQLite/blob storage

What is still missing:

- We still do not have the actual `aafix.jar` internals or the exact loader patch content that gives it precedence.

Sources:

- `https://github.com/fifthBro/pcm5-androidauto-connect-fix/blob/main/README.md`
- `https://github.com/fifthBro/pcm5-androidauto-connect-fix/blob/main/fix_partition_1008/fix_partition_1008.c`
- `https://github.com/fifthBro/pcm5-androidauto-connect-fix/blob/main/MH2p_ModKit_Mod/AndroidAuto_Fix/Update/install.sh`
- `https://github.com/fifthBro/pcm5-androidauto-connect-fix/blob/main/MH2p_ModKit_Mod/AndroidAuto_Fix/Update/uninstall.sh`

### 5) Whether MH2p mods can do more than FEC/coding

Status: `answered well`

Direct evidence from `MH2p_CarPlay_FullScreen`:

- `fc.jar` is copied to `/mnt/app/eso/hmi/lsd/jars`
- jars in that directory are loaded before `lsd.jar`
- classes in `fc.jar` can therefore override or modify UI behavior

Interpretation:

- The ecosystem supports at least:
  - FEC file modification
  - 5F coding/adaptation changes
  - Java/JAR UI behavior overrides
  - persistence repair and platform-specific runtime fixes

Sources:

- `https://github.com/LawPaul/MH2p_CarPlay_FullScreen/blob/main/README.md`

### 6) Comparative context with older M.I.B./FEC workflows

Status: `answered moderately well`

Evidence from M.I.B. and related references:

- M.I.B. documents generation of `FecContainer.fec` based on an existing container plus `addfec.txt`
- M.I.B. also documents Android Auto and CarPlay related patches in the broader MIB2/Harman ecosystem
- the M.I.B. FEC overview lists `00060900` as Google Android Auto and `00060800` as Apple CarPlay

Interpretation:

- The older M.I.B. flow and the MH2p flow are clearly working in the same conceptual domain
- but the implementation differs:
  - M.I.B. focuses on container-based patching for MHI2/MHI2Q units
  - MH2p uses a signed update wrapper plus local script execution and direct `.fecs` editing via `fecswap`

So "same problem space, different delivery mechanism" is a defensible summary.

Sources:

- `https://github.com/Mr-MIBonk/M.I.B._More-Incredible-Bash`
- `https://github.com/Mr-MIBonk/M.I.B._More-Incredible-Bash/wiki/MHI2-MHI2Q-FEC-overview`
- `https://github.com/LawPaul/MH2p_AndroidAuto/blob/main/Update/install.sh`

## Implementation still unresolved

### 7) Where `fecswap` comes from and how it is built

Status: `weak data`

What we know:

- `fecswap` is a bundled prebuilt 32-bit little-endian ARM ELF in `MH2p_SD_ModKit`
- `MH2p_AndroidAuto` depends on it to modify `/mnt/persist_new/fec/granted.fecs`
- the LawPaul wiki documents the CLI behavior

What we do not have:

- source code
- build scripts
- a public repository containing the source
- a documented file format spec for `.fecs`

Current confidence:

- high confidence on what it does
- low confidence on how it was implemented internally

### 8) Where `mnfc` comes from and how package signing is performed

Status: `weak data`

What we know:

- manifest and installer metadata files are marked `Created-By: mnfc`
- the ModKit README says checksum/signing methods were discovered by reverse engineering MH2p binaries

What we do not have:

- `mnfc` source code
- public documentation for `mnfc`
- command lines or config used to generate the package
- signing implementation details

Current confidence:

- high confidence that `mnfc` generated the package metadata
- low confidence on how the tool works internally or how to reproduce the workflow exactly

### 9) Exact implementation details of the Porsche bytecode patch

Status: `partial data`

What we know:

- the README explains the exact bug and the intended fix
- `MH2p_AndroidAuto` deploys `aafix.jar`
- the fix is said to load through bootclasspath injection
- the persistence repair path is source-backed by `fix_partition_1008.c`
- the ModKit deployment wrapper is source-backed by the public install/uninstall scripts

What we do not yet have:

- the actual decompiled patched class
- a diff versus the stock class
- the exact `lsd.sh` or loader patch content

Current confidence:

- high confidence on the persistence repair mechanics
- medium confidence on the wrapper/deployment mechanics
- lower confidence on the exact Java loader precedence and `aafix.jar` internals

## Best next steps

1. Reverse-engineer `fecswap` enough to document the `.fecs` file structure and command behavior.
2. Inspect or fetch the actual contents of the Porsche fix repo's `fix_partition_1008.c` and mod wrapper files.
3. Keep `mnfc` and package-signing reconstruction as a separate track, since that likely requires reverse engineering or unpublished tooling.
