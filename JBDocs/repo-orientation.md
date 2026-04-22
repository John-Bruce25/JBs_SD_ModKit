# Repo Orientation

## What this repo is

`MH2p_SD_ModKit` is a packaged software update for MH2p infotainment systems. Its purpose is to bootstrap user-provided mods from removable media by wrapping them inside a valid update structure the device accepts.

This is not a typical app repo with a local build system, tests, or source modules. The deliverable is the folder structure itself:

- `Meta/` - manifests, checksums, and signature files for the update package
- `Data/` - payload files that are copied or executed on the target device
- `Mods/` - extension point for end-user mods
- `Logs/` - output location for install and runtime logs copied back to removable media

## Main execution path

1. `Meta/auto.mnf` defines the top-level update package.
2. That package includes `ExceptionList` and `MMX2P_POSTSCRIPT`.
3. `MMX2P_POSTSCRIPT` runs `Data/MMX2P_POSTSCRIPT.script_2021623-0210/0/modkit.sh`.
4. `modkit.sh` finds the SD/USB media, shows a splash screen, copies in helper files, and invokes `modkit_install.sh`.
5. `modkit_install.sh` runs `Mods/*/Update`, copies `Post` and `Persist` mod content into persistent storage, and patches startup by replacing `servicemgrmibhigh` with a wrapper script.
6. On later boots, `servicemgrmibhigh.sh` starts the original service manager and also launches `modkit_persist.sh`.
7. `modkit_persist.sh` handles:
   - optional `failsafe.sh` execution from removable media
   - one-time `Post` mod steps
   - every-boot `Persist` mod steps
   - log collection back to media

## Repo-authored parts

These files look repo-authored and are the clearest source of project logic:

- `README.md`
- `Data/MMX2P_POSTSCRIPT.script_2021623-0210/0/modkit.sh`
- `Data/ExceptionList.fec_2017327-0730/0/modkit_install.sh`
- `Data/ExceptionList.fec_2017327-0730/0/modkit_persist.sh`
- `Data/ExceptionList.fec_2017327-0730/0/servicemgrmibhigh.sh`

They carry LawPaul copyright headers and implement the project-specific behavior.

## Generated or bundled parts

These look generated or externally produced rather than handwritten in this repo:

- `Meta/*.mnf`
- `Meta/*.cks`
- `Meta/*_S.sig`
- `Data/*/installer.txt`
- `Data/ExceptionList.fec_2017327-0730/0/fecswap`
- `Data/ExceptionList.fec_2017327-0730/0/splashscreen.jpg`

The manifest and installer files include `Created-By: mnfc`, which strongly suggests they were emitted by an external packaging tool.

## External runtime dependencies

The scripts depend on binaries and system paths that live on the target MH2p device and are not in this repo:

- `/mnt/app/eso/bin/apps/showimage`
- `/mnt/app/armle/usr/bin/pc`
- `/mnt/app/eso/bin/servicemgrmibhigh`
- QNX mount points such as `/mnt/app`, `/mnt/ota`, `/mnt/persist_new`
- removable media mount points such as `/fs/sda0`, `/fs/sdb0`, `/fs/usb0_0`

That means the repo explains the mod framework behavior, but not the implementation of those vendor/device-native tools.
