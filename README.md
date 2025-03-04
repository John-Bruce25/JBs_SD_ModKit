# MH2p SD ModKit
Free ModKit that allows for modifying the MH2p unit used in some Volkswagen AG vehicles using only an SD card or flash drive.
## License
 - This file is part of MH2p_SD_ModKit, licensed under CC BY-NC-SA 4.0.
 - https://creativecommons.org/licenses/by-nc-sa/4.0/
 - See the LICENSE file in the project root for full license text.
 - NOT FOR COMMERCIAL USE
## Mod development
 - each mod should be self-contained in its own folder `/Mods/[modname]`
 - mod can include Update, Post, and Persist parts
     - Update: runs during update
         - if included, needs `install.sh` and `uninstall.sh`
     - Post: runs once after update, followed by persistant reboot (useful for 5F coding that does not work during an update)
         - if included, needs `install.sh` and `uninstall.sh`
     - Persist: runs at every startup
         - if included, needs `install.sh`
 - mod output is logged to `/Logs/[modname].log`
 - if `uninstall.txt` is present, `uninstall.sh` scripts are run
 - otherwise, `install.sh` scripts are run
 - users can choose to install or uninstall by adding or removing `uninstall.txt`
 - MH2p ModKit exports a few useful variables:
     - `MODKIT_VERSION`: current release is 2
     - `RELEASE_VERSION`: release version string ex: MH2p_US_PO416_P2870
     - `REGION`: region ex: AS, CN, ER, US, ...
     - `OEM`: car brand ex: VW, AU, PO, LB, ...
     - `TYPE`: screen type ex: 416, 636, G33, G35, G36, ...
     - `RELEASE_TYPE`: E: engineering, K: customer update, P: production, S: security update
     - `SOFTWARE_VERSION`: software version number ex: 9830, 2870, ...
     - `MOD`: name of mod (same as mod's folder name) ex: `[modname]`
     - `MOD_PATH`: path to mod part's folder ex: `/fs/sdb0/Mods/[modname]/Update`
     - `MEDIA_PATH`: path to SD card (only available in Update) ex: `/fs/sdb0`
## How it works
 - the ModKit is a valid checksummed and signed update for MH2p
     - I discovered checksum and signing methods through reverse engineering MH2p binaries
 - the update runs `modkit.sh` which runs `modkit_install.sh` which runs scripts under `/Mods/` that are not checksummed or signed
 - this allows easier development and installation of mods
## Fail-Safe
 - after running the ModKit, if update media is inserted with `failsafe.sh` at the root, the script will be run early in the startup process
     - this can be used to clean up a mistake and escape a boot loop without needing to use a PL2303TA cable to enter emergency SWUP