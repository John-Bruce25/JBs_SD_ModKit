# MH2p / FEC-SWaP / Android Auto Analysis

## Purpose

This report summarizes how the `MH2p_SD_ModKit` ecosystem appears to relate to older VW/VAG **FEC/SWaP** workflows documented on **vwcoding.ru**, with a focus on **adding Android Auto to systems that did not originally expose or support it**.

It is written as a practical repo drop-in for engineering notes, not as a how-to for modifying a vehicle.

---

## Executive summary

My current read is:

1. **FEC/SWaP** is the VW/VAG feature licensing layer for infotainment features such as Navigation, CarPlay, Android Auto, Bluetooth, MirrorLink, etc.
2. On older or more general VW/VAG workflows, those features are often enabled by a combination of:
   - a **patched head unit** that will accept modified/added FEC entries,
   - a **generated or injected FEC/SWaP container/code**,
   - and sometimes additional **5F coding / adaptations / patching**.
3. The **vwcoding.ru** material is mostly a **documentation + generator + workflow reference** for that older model, especially around MIB/MQB units.
4. `MH2p_SD_ModKit` looks like a **different delivery mechanism** for the same broad class of problem on **MH2p / PCM5 / MIB2+ High style units**: instead of using dealer tools or the older M.I.B. patch flow directly, it packages a **valid signed update** that runs custom install scripts from `/Mods/`.
5. The repo `LawPaul/MH2p_AndroidAuto` strongly suggests that, in the MH2p world, Android Auto enablement is done by **adding Android Auto FEC plus 5F coding**, delivered through the ModKit.
6. For Porsche PCM5/MH2p specifically, there is also a separate fix repo for a known **multi-phone / persistence bug**, which means “FEC present” is **necessary but not always sufficient** for a clean Android Auto experience.

So in plain English:

> `MH2p_SD_ModKit` appears to be the **installation framework**, `MH2p_AndroidAuto` appears to be the **feature-enablement mod**, and the older **FEC/SWaP** pages on vwcoding describe the underlying licensing/activation concept that this newer MH2p flow is effectively automating or replacing for a different head-unit family.

---

## Key sources

### MH2p ecosystem

- `MH2p_SD_ModKit` repo: <https://github.com/LawPaul/MH2p_SD_ModKit>
- MH2p Mod Kit site: <https://lawpaul.github.io/MH2p_SD_ModKit_Site/>
- `MH2p_AndroidAuto` repo: <https://github.com/LawPaul/MH2p_AndroidAuto>
- Porsche PCM5 Android Auto bug fix repo: <https://github.com/fifthBro/pcm5-androidauto-connect-fix>
- `MH2p_CarPlay_FullScreen` repo: <https://github.com/LawPaul/MH2p_CarPlay_FullScreen>

### Older / broader VW-VAG FEC-SWaP references

- vwcoding.ru main site: <https://vwcoding.ru/en/>
- MIB FEC/SWaP generator: <https://vwcoding.ru/en/utils/fec/>
- SWaP for DM and CM units: <https://vwcoding.ru/en/MQB/headDeviceFunctions/>
- Unlocking and SWaP of Pro: <https://vwcoding.ru/en/MQB/headDeviceFunctionsPro/>
- Site history/changelog: <https://vwcoding.ru/en/history/>
- M.I.B. More Incredible Bash repo: <https://github.com/Mr-MIBonk/M.I.B._More-Incredible-Bash>

---

## What FEC / SWaP means in this context

Across VW/VAG infotainment ecosystems, **FEC** (sometimes adjacent to **FSC** in related ecosystems) is used as a feature licensing / activation mechanism. The vwcoding generator page is explicit that it is generating **FEC codes** for a MIB unit and that those codes are meant to be uploaded to the infotainment system. The same page lists feature examples such as Navigation, Bluetooth, MirrorLink, Apple CarPlay, and Google Android Auto.

That makes FEC/SWaP relevant to your Android Auto question because Android Auto is treated as one of the **licensed / activatable head-unit functions**, not just as a UI toggle.

### Relevant vwcoding page

- MIB FEC/SWaP generator: <https://vwcoding.ru/en/utils/fec/>

### Feature examples shown there

The page exposes feature codes including:

- `00040100` — Navigation
- `00050000` — Bluetooth
- `00060300` — MirrorLink
- `00060800` — Apple CarPlay
- `00060900` — Google Android Auto

That is the clearest direct signal that **Android Auto is part of the FEC/SWaP activation model**.

---

## What vwcoding.ru adds to the picture

The vwcoding site appears to serve three roles:

1. **Documentation** for platform-specific coding and activation workflows.
2. **Utilities**, including a browser-based **FEC generator**.
3. A historical record of when FEC/SWaP-related tooling and guides were added.

### Specific findings

#### 1) FEC generator exists and is browser-based

The page **“MIB FEC/SWaP Code Generator”** says it generates FEC codes for **MIB STD2 Technisat**, requires a **VIN** and **VCRN**, and says the codes can be uploaded with **OBDeleven or ODIS**. It also says the unit must already be **SWaP patched** for the generated codes to be recognized as valid.

That implies a classic three-part model:

- derive/generate the right FEC material,
- patch the unit so it will accept it,
- then inject/upload it using external tooling.

#### 2) SWaP for Pro references M.I.B. bash explicitly

The page **“Unlocking and SWaP of Pro”** states that actions are performed using **M.I.B. bash**, and that only an SD card is needed for activation in that workflow. That is very similar in spirit to the later MH2p SD-card-only delivery model, although it applies to a different ecosystem and patch stack.

#### 3) Site changelog lines up with FEC/SWaP support evolution

The site changelog says:

- **22 Apr 2021**: SWaP codes and parametry were added for extra main-unit functionality (CM and DM).
- **03 May 2021**: SWaP code for ACC/pACC activation and unlock/SWaP codes for MIB2 Pro were added.
- **11 Dec 2023**: a **FEC/SWAP generator for MIB STD2 Technisat** was added.

This looks like an evolution from static guides toward a more automated online generator.

---

## What `MH2p_SD_ModKit` appears to be

`MH2p_SD_ModKit` is best understood as a **general-purpose mod delivery framework for MH2p units**.

### Core design

From the repo README and site:

- It is a **valid checksummed and signed update** for MH2p.
- The author says the checksum/signing methods were discovered by **reverse engineering MH2p binaries**.
- During installation, it runs `modkit.sh`, then `modkit_install.sh`, then executes mod scripts found under `/Mods/`.
- Mods are split into:
  - **Update** — runs during update,
  - **Post** — runs once after update/reboot,
  - **Persist** — runs on every startup.
- Logs are written under `/Logs/[modname].log` on the media.
- A `failsafe.sh` mechanism can run very early in startup to recover from mistakes/boot loops.

### Why that matters

This is a **delivery abstraction**. It means the project is not one single Android Auto hack; it is a platform for shipping multiple low-level changes through a signed update wrapper.

That is important because it explains why Android Auto support lives in a **separate mod repo** rather than inside the core ModKit.

### Relevant links

- Repo: <https://github.com/LawPaul/MH2p_SD_ModKit>
- Site: <https://lawpaul.github.io/MH2p_SD_ModKit_Site/>

---

## What `MH2p_AndroidAuto` appears to do

The dedicated Android Auto repo is unusually direct about its behavior.

### Repo summary

`LawPaul/MH2p_AndroidAuto` says:

- it **activates Android Auto**,
- it is **for use with MH2p SD ModKit**,
- it **adds Android Auto FEC & codes 5F**,
- and it references an external fix for a bug causing issues when multiple phones are involved.

### Interpretation

This strongly suggests the mod is doing **two distinct classes of work**:

1. **Feature entitlement / activation work**
   - “adds Android Auto FEC”
   - this is the licensing/activation side
2. **Control-module configuration work**
   - “codes 5F”
   - this is the infotainment/controller coding side

That is exactly the kind of combined flow you would expect if FEC alone is not enough.

### Important nuance

The repo description says **wired Android Auto only** and explicitly says **wireless not supported**.

### Relevant link

- <https://github.com/LawPaul/MH2p_AndroidAuto>

---

## What the Porsche PCM5 bug-fix repo adds

The repo `fifthBro/pcm5-androidauto-connect-fix` matters because it explains a failure mode that sits **after** activation.

### What it claims

The README says it:

- is based on **MH2p SD ModKit**,
- automatically checks whether the unit is **Porsche** and on **26xx–28xx firmware**,
- installs a **bytecode fix**,
- repairs the **persistence partition** in SQLite including blob length / CRC32 issues,
- is meant to be installed **alongside** the `MH2p_AndroidAuto` mod.

### Interpretation

This implies a layered architecture:

1. **Enable Android Auto** (FEC + 5F coding)
2. **Fix runtime/device-state bugs** specific to PCM5/MH2p behavior

That is a useful clue for understanding how the whole stack fits together: **activation** and **stability/behavior fixes** are separate concerns.

### Relevant link

- <https://github.com/fifthBro/pcm5-androidauto-connect-fix>

---

## Evidence that MH2p mods patch application/UI behavior too

The ModKit ecosystem is not limited to FEC/coding changes.

For example, the `MH2p_CarPlay_FullScreen` repo says it copies `fc.jar` into a location from which jars are loaded **before** `lsd.jar`, allowing classes in `fc.jar` to override or modify UI behavior.

That matters because it shows the ModKit can be used for at least three kinds of modification:

1. **FEC / activation changes**
2. **5F coding / configuration changes**
3. **Bytecode / jar / UI behavior overrides**

This makes it a broader mod platform than a simple FEC injector.

### Relevant link

- <https://github.com/LawPaul/MH2p_CarPlay_FullScreen>

---

## How this compares to M.I.B. / older FEC-SWaP workflows

This is the key conceptual comparison.

### Older / broader workflow (vwcoding + M.I.B.)

The older workflow typically looks like this:

1. Identify the head-unit family and firmware.
2. Patch the unit (or patch an ifs-root-stage2 image) so feature checks can be bypassed or new FECs can be accepted.
3. Generate or add FEC entries.
4. Push the FEC/SWaP material to the unit.
5. Clear SVM / perform related maintenance.
6. Apply extra coding/adaptations as needed.

The M.I.B. project explicitly says it can:

- patch `ifs-root`,
- do **SVM fix**,
- generate **custom `FecContainer.fec`** files based on existing containers plus `addfec.txt`,
- and includes model-specific Android Auto / widescreen fixes.

That is a very direct “toolbox” approach.

### MH2p ModKit workflow

The MH2p approach appears to be:

1. Use the ModKit’s signed-update wrapper to gain a safe/automated execution path.
2. Deliver one or more mods through `/Mods/`.
3. Let individual mods perform the required FEC, 5F, bytecode, persistence, or UI modifications.
4. Reboot and validate.

### Bottom line

**M.I.B.** feels like the older **general-purpose patching toolbox** for Harman MIB 2.x style units.

**MH2p_SD_ModKit** feels like a **head-unit-specific signed installer framework** that packages the relevant changes into installable mods for MH2p/PCM5/MIB2+ High type systems.

So they are not the same project, but they solve adjacent problems in a similar problem space.

---

## Likely architecture for Android Auto enablement in your MH2p repo context

Based on the public descriptions, the likely stack looks like this:

```text
Android Auto support on target vehicle
        │
        ├─ Requires head-unit support path to be unlocked/exposed
        │
        ├─ FEC / SWaP layer
        │    └─ add Android Auto entitlement / feature activation
        │
        ├─ 5F coding layer
        │    └─ enable/configure infotainment behavior for the feature
        │
        ├─ Platform-specific runtime fixes (optional but sometimes required)
        │    └─ Porsche PCM5 persistence / multi-device Android bug fix
        │
        └─ UI / behavior mods (optional, separate concern)
             └─ fullscreen / windowed fullscreen / jar overrides
```

And the delivery mechanism looks like this:

```text
MH2p_SD_ModKit
   └─ signed update wrapper
      └─ executes mods from /Mods/
         ├─ MH2p_AndroidAuto
         ├─ AndroidAuto_Fix (Porsche-specific runtime fix)
         ├─ CarPlay / fullscreen mods
         └─ other future mods
```

---

## Where “FECswap” probably fits

I did **not** find a literal artifact named `FECswap` on vwcoding or in the GitHub material I reviewed.

What I **did** find is repeated use of:

- **FEC/SWaP**
- **SWAP/FEC**
- FEC container generation and insertion
- Android Auto as a FEC-governed feature

So I suspect your “FECswap” label is either:

1. a shorthand for **FEC/SWaP** generally, or
2. a memory of a local script / folder / variable name inside your repo or build process that wraps this concept.

If that string exists in your private/local repo, I would expect it to be one of these:

- a wrapper around adding Android Auto FEC entries,
- a helper for building / copying a FEC payload,
- a step that pairs FEC injection with 5F coding,
- or an internal naming convention borrowed from the broader FEC/SWaP concept.

---

## Practical takeaways for your repo analysis

If you are documenting how your repo fits together, this is the framing I would use:

### Suggested interpretation

- `MH2p_SD_ModKit` = **the installer/runtime framework**
- `MH2p_AndroidAuto` = **the Android Auto activation mod**
- FEC/SWaP = **the underlying licensing/feature-enablement model**
- 5F coding = **the required module configuration complement**
- `pcm5-androidauto-connect-fix` = **the Porsche-specific post-activation stability fix**
- fullscreen / UI mods = **separate optional UX changes**

### Suggested wording for repo notes

> The Android Auto path on MH2p is not just “turning on a hidden menu.” It appears to require a combination of feature entitlement (FEC/SWaP), 5F coding, and in some Porsche PCM5 cases an additional persistence/bytecode fix. `MH2p_SD_ModKit` provides the signed SD-card update framework that installs these changes as modular packages.

---

## What is still uncertain

A few things are still not proven from the public descriptions alone:

1. **Exactly how `MH2p_AndroidAuto` writes or injects the FEC**
   - The README says it adds Android Auto FEC, but not the implementation details.
2. **Whether the mod is creating a FEC container, editing an existing container, or calling a built-in platform pathway**.
3. **Whether “codes 5F” means direct coding values, adaptations, scriptable diagnostics, or a file-based patch flow**.
4. **How much of the older M.I.B. approach is conceptually reused vs independently reimplemented for MH2p**.

So the public evidence is enough to map the architecture at a high level, but not enough to fully reverse-engineer the internals without reading the actual mod scripts/files in your working tree.

---

## Recommended next step inside your repo

If you want the next layer of understanding, inspect these areas in your local repo/build tree:

1. `Mods/MH2p_AndroidAuto/Update/`
2. `Mods/MH2p_AndroidAuto/Post/`
3. any files referencing:
   - `FEC`
   - `SWAP`
   - `5F`
   - `AndroidAuto`
   - `addfec`
   - `FecContainer`
   - `sqlite`
   - `crc`
   - `persist`
4. any references to copying jars, patching bytecode, or editing persistence blobs

What I would specifically look for:

- shell scripts that write to a persistence area,
- coding/adaptation commands for module **5F**,
- data blobs or container files that correspond to Android Auto entitlement,
- firmware / OEM / region checks,
- and separation between **Update**, **Post**, and **Persist** behavior.

---

## Short conclusion

The pieces fit together coherently:

- **vwcoding.ru** explains the **FEC/SWaP** activation world and provides examples showing **Android Auto is one of those licensable features**.
- **M.I.B.** represents an older, broader patch/toolbox approach for Harman MIB 2.x units.
- **MH2p_SD_ModKit** is a newer SD-card mod framework tailored to **MH2p / PCM5 / related units**.
- **MH2p_AndroidAuto** appears to implement the actual Android Auto enablement by adding the **Android Auto FEC** and doing **5F coding**.
- A separate Porsche-specific repo fixes a **runtime/persistence bug**, showing that feature activation and stable operation are distinct layers.

That is likely the right mental model for documenting how your `MH2p_SD_ModKit`-based Android Auto build works.

---

## Source notes / page snippets worth remembering

### vwcoding

- Generator requires **VIN + VCRN** and says the unit must already be **SWaP patched**.
- Android Auto appears explicitly in the generator’s feature list.
- Pro SWaP page references **M.I.B. bash** and SD-card activation.
- Changelog shows a timeline from SWaP guides (2021) to online FEC generator (2023).

### MH2p

- ModKit is a **signed update wrapper** discovered via reverse engineering.
- Mods run as **Update / Post / Persist** units.
- Android Auto mod says it **adds Android Auto FEC & codes 5F**.
- Porsche fix repo says it repairs persistence / CRC32 issues and is installed **alongside** the Android Auto mod.

---

## Appendix: source list for quick clicking

- vwcoding main: <https://vwcoding.ru/en/>
- vwcoding FEC generator: <https://vwcoding.ru/en/utils/fec/>
- vwcoding SWaP for DM/CM: <https://vwcoding.ru/en/MQB/headDeviceFunctions/>
- vwcoding Unlocking and SWaP of Pro: <https://vwcoding.ru/en/MQB/headDeviceFunctionsPro/>
- vwcoding history: <https://vwcoding.ru/en/history/>
- M.I.B. More Incredible Bash: <https://github.com/Mr-MIBonk/M.I.B._More-Incredible-Bash>
- MH2p Mod Kit site: <https://lawpaul.github.io/MH2p_SD_ModKit_Site/>
- MH2p SD ModKit repo: <https://github.com/LawPaul/MH2p_SD_ModKit>
- MH2p Android Auto repo: <https://github.com/LawPaul/MH2p_AndroidAuto>
- Porsche Android Auto fix: <https://github.com/fifthBro/pcm5-androidauto-connect-fix>
- MH2p CarPlay FullScreen: <https://github.com/LawPaul/MH2p_CarPlay_FullScreen>
