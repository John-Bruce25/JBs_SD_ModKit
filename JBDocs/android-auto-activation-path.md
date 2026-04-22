# Android Auto Activation Path

This note summarizes the Android Auto enablement flow in the MH2p ecosystem using source-backed behavior from the public `MH2p_AndroidAuto` repository.

## High-level model

Android Auto enablement is a layered path:

1. FEC entitlement layer
2. 5F coding layer
3. Porsche-only repair layer for affected firmware

The public scripts show these layers are separated across `Update` and `Post`.

## 1. FEC entitlement layer

Source:

- `https://github.com/LawPaul/MH2p_AndroidAuto/blob/main/Update/install.sh`

During `Update`, the mod:

- backs up `/mnt/persist_new/fec`
- removes `/mnt/persist_new/fec/illegal.fecs`
- preserves `granted.fecs` to `granted.fecs.bak`

Then it runs:

```sh
fecswap -a 00030000 00050000 00060900 -f /mnt/persist_new/fec/granted.fecs
```

This adds:

- `00030000` - AMI/USB
- `00050000` - Bluetooth
- `00060900` - Google Android Auto

This is the clearest source-backed proof that Android Auto is enabled by direct FEC file editing in the MH2p flow.

## 2. 5F coding layer

Source:

- `https://github.com/LawPaul/MH2p_AndroidAuto/blob/main/Post/install.sh`

During `Post`, the mod runs:

```sh
/mnt/app/armle/usr/bin/pc b:0x5F22:0x600:19.6 1
/mnt/app/armle/usr/bin/pc b:0x5F22:0x600:19.7 1
/mnt/app/armle/usr/bin/pc b:0x5F22:0x22AD:7.7 1
```

With comments in the upstream script:

- `USB1 data`
- `USB2 data`
- `Google GAL`

This means "codes 5F" is not vague marketing language.
In this mod, it specifically means writing 5F-related persistence/coding bits with the device-native `pc` tool.

## 3. Porsche-only repair layer

Source:

- `https://github.com/LawPaul/MH2p_AndroidAuto/blob/main/Update/install.sh`
- `https://github.com/fifthBro/pcm5-androidauto-connect-fix/blob/main/README.md`

The same `Update/install.sh` also includes a Porsche-specific branch:

- checks `OEM == "PO"`
- checks firmware version in `26xx` or `28xx`
- copies `aafix.jar` into `/mnt/app/eso/hmi/lsd/jars/aafix.jar`
- runs `fix_partition_1008 --fix` against `persistence.sqlite`

So for affected Porsche units, the activation path is:

1. add FECs
2. stage/enable Java-side repair
3. repair persistence state
4. later apply 5F bits in `Post`

## Why the flow is split across `Update` and `Post`

This layout matches the ModKit framework design:

- `Update` handles file manipulation and staging during the signed update process
- `Post` handles operations that need the runtime environment after reboot

That is consistent with the ModKit README, which explicitly calls out `Post` as useful for 5F coding that does not work during update.

## Source-backed conclusion

The MH2p Android Auto path is not a single toggle. It is a three-layer install path:

- FEC insertion via `fecswap`
- 5F persistence/coding writes via `pc`
- Porsche-specific runtime/persistence repair when needed

That is now established by public source files, not just README summaries.
