# JB Docs

## The Case File

These papers concern our investigation into the curious machinery of `MH2p_SD_ModKit`, a contrivance which at first glance seemed merely a small repository of shell scripts and manifests, but has since revealed itself to be a rather elegant signed-update framework for smuggling mods into MH2p units by way of SD card or flash media.

Like any respectable mystery, it contains:

- a public facade that appears simple
- hidden actors working behind the wallpaper
- a few accomplices in neighboring repositories
- and at least two principal suspects whose origins remain obscure: `fecswap` and `mnfc`

This folder is our detective's notebook. It gathers what we have confirmed, what we strongly suspect, and what still skulks in the fog.

## What We Have Learned

The broad outline of the affair is now reasonably clear.

`MH2p_SD_ModKit` is the installer and execution framework. It presents itself to the target unit as a valid signed update, runs `modkit.sh`, hands off to `modkit_install.sh`, and then executes mod payloads from `/Mods/`. The mod system is divided into `Update`, `Post`, and `Persist` stages, which gives it a surprisingly orderly structure for something discovered through reverse engineering.

The Android Auto story is no longer conjecture. From the public `MH2p_AndroidAuto` scripts we now know that the mod:

- edits `/mnt/persist_new/fec/granted.fecs` with `fecswap`
- adds FECs for AMI/USB, Bluetooth, and Google Android Auto
- applies 5F-related changes afterward with the built-in `pc` tool

That means Android Auto enablement in this ecosystem is not a mere menu toggle. It is a layered operation involving both feature entitlement and control-module configuration.

We have also learned that the Porsche multi-phone bug fix is not merely a side note. The public evidence indicates that the Android Auto workflow for certain Porsche firmware ranges now includes:

- deployment of `aafix.jar`
- repair of persistence data via `fix_partition_1008`
- special handling for affected `PO` units in the `26xx` to `28xx` range

Finally, the wider MH2p ecosystem appears capable of much more than FEC manipulation. The CarPlay fullscreen work shows that jars placed in `/mnt/app/eso/hmi/lsd/jars` are loaded before `lsd.jar`, which means the platform supports UI and behavior overrides as well as coding and persistence edits.

## The Chief Mysteries Still Unsolved

Some facts remain maddeningly out of reach.

- `fecswap` is clearly important and clearly used, yet its source is not present here.
- `mnfc` plainly generated the package metadata, yet the tool itself and its signing process remain absent from the evidence locker.
- The Porsche bug-fix behavior is well described in public, but we do not yet have the full patch mechanics laid out from source in our local notes.

So the shape of the crime is known, but not every fingerprint has been lifted.

## The Papers In This Folder

- [repo-orientation.md](repo-orientation.md)
  A floor plan of the manor: what is in this repo, what each directory does, and how the runtime flow proceeds from signed update to mod execution.

- [binary-sources-and-build-notes.md](binary-sources-and-build-notes.md)
  Notes on the suspicious artifacts: bundled binaries, generated manifests, external dependencies, and the question of what appears authored here versus imported from elsewhere.

- [research-questions-and-findings.md](research-questions-and-findings.md)
  The active case board: top research questions, answers with solid evidence, and the lines of inquiry that still lack proper documentation.

- [MH2p_FEC_SWAP_Analysis_Report.md](MH2p_FEC_SWAP_Analysis_Report.md)
  A wider comparative inquiry into MH2p, FEC/SWaP, Android Auto enablement, M.I.B., and the surrounding ecosystem of related projects.

## Present Condition Of The Inquiry

- We have good evidence for how Android Auto is enabled in the MH2p ecosystem.
- We have good evidence that 5F coding is part of that path, not an optional flourish.
- We have decent evidence for how the Porsche persistence bug is conceptualized and integrated.
- We do not yet have strong source-level evidence for how `fecswap` or `mnfc` were built.

In short: the trail is warm, the lantern is lit, and the quarry is very likely still ahead of us.
