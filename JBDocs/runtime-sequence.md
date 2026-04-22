# Runtime Sequence

This note documents the actual execution chain of `MH2p_SD_ModKit` from signed update package to persistent mod execution.

## End-to-end sequence

1. `Meta/auto.mnf`
   Defines the top-level signed update package.
   Includes:
   - `ExceptionList`
   - `MMX2P_POSTSCRIPT`

2. `Meta/MMX2P_POSTSCRIPT/1.0.0.mnf`
   Declares a script package that runs `ScriptInstaller`.

3. `Data/MMX2P_POSTSCRIPT.script_2021623-0210/0/modkit.sh`
   This is the first script entrypoint.
   It:
   - searches removable media mount points:
     - `/fs/sda0`
     - `/fs/sdb0`
     - `/fs/usb0_0`
   - verifies the media contains:
     - `Data`
     - `Meta`
     - `Mods`
   - exports `MEDIA_PATH`
   - tests media write access
   - ensures `Logs/` exists on the media
   - mounts `/mnt/app`
   - displays `/mnt/persist_new/splashscreen.jpg`
   - copies `/mnt/persist_new/fecswap` into `/bin/fecswap`
   - executes `/mnt/persist_new/modkit_install.sh`
   - cleans up helper files after completion

4. `Data/ExceptionList.fec_2017327-0730/0/modkit_install.sh`
   This is the main update-time installer.
   It:
   - exports environment variables such as:
     - `RELEASE_VERSION`
     - `REGION`
     - `OEM`
     - `TYPE`
     - `RELEASE_TYPE`
     - `SOFTWARE_VERSION`
   - mounts `/mnt/ota`
   - iterates over `"$MEDIA_PATH/Mods/*"`
   - for each mod:
     - runs `Update/install.sh` or `Update/uninstall.sh` immediately
     - copies `Post/` into `/mnt/ota/modkit/Mods/<mod>`
     - copies `Persist/` into `/mnt/ota/modkit/Mods/<mod>` unless uninstalling
   - copies any previously accumulated logs from `/mnt/ota/modkit/Logs` back to media
   - patches startup by replacing:
     - original binary: `/mnt/app/eso/bin/servicemgrmibhigh`
     - backup copy: `/mnt/app/eso/bin/servicemgrmibhigh0`
     - wrapper script: `/mnt/app/eso/bin/servicemgrmibhigh`
   - persists `modkit_persist.sh` into `/mnt/ota/modkit/modkit_persist.sh`

5. `Data/ExceptionList.fec_2017327-0730/0/servicemgrmibhigh.sh`
   This is the startup wrapper installed over the original service manager.
   It:
   - starts `/eso/bin/servicemgrmibhigh0` in the background
   - if present, launches `/mnt/ota/modkit/modkit_persist.sh`
   - logs its output to `/mnt/ota/modkit/Logs/_ModKit_Persist.log`

6. `Data/ExceptionList.fec_2017327-0730/0/modkit_persist.sh`
   This is the every-boot persistence script.
   It:
   - waits for removable media
   - checks for `failsafe.sh` on removable media and runs it if present
   - rebuilds the same release metadata environment variables used during update
   - iterates over `/mnt/ota/modkit/Mods/*`
   - for each mod:
     - runs `Post/install.sh` or `Post/uninstall.sh` once, then deletes `Post/`
     - runs `Persist/install.sh` every boot unless uninstalling
   - deletes mod directories that no longer need to persist
   - triggers persistent reboot with `gem-reset` if a `Post` stage ran

## Important filesystem locations

- `/mnt/persist_new`
  Temporary/persistent update staging used during update.
  In this project it carries helper payloads such as:
  - `fecswap`
  - `modkit_install.sh`
  - `modkit_persist.sh`
  - `servicemgrmibhigh.sh`
  - `splashscreen.jpg`

- `/mnt/ota/modkit`
  Long-lived ModKit persistence root created by `modkit_install.sh`.
  Holds:
  - persisted mod content
  - `modkit_persist.sh`
  - mod logs

- `/mnt/app/eso/bin`
  Startup patch point.
  `servicemgrmibhigh` is replaced with a wrapper, while the original binary is preserved as `servicemgrmibhigh0`.

- `/mnt/app/eso/hmi/lsd/jars`
  UI/Java patch point used by some mods, including Android Auto and fullscreen CarPlay-related work.

## Why `Post` exists

The split between `Update` and `Post` is intentional.

The ModKit README says `Post` is useful for work such as 5F coding that does not succeed during the update itself. That matches the public Android Auto mod, which:

- modifies FEC files during `Update`
- applies 5F-related `pc` writes during `Post`

So `Post` exists to run code after the unit has come back up in a runtime state where certain persistence or module operations can succeed.
