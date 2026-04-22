# VW To MH2p Architecture Overview

## Purpose

This document is the readable version of the case.

The other notes in `JBDocs` collect evidence, scripts, paths, and open questions. This one is meant to tell the whole story in one pass:

- what Volkswagen Group platforms appear to have out there already
- how feature licensing and activation are structured
- how the modding community traditionally approaches those systems
- what the MH2p ecosystem changes
- and how all of it comes together to get Android Auto working on a Porsche PCM5 / MH2p-style unit

It is not a step-by-step modification guide. It is an architecture and ecosystem map.

## The vendor side: what VW Group appears to ship

At a high level, the infotainment systems in this ecosystem are not built around a single hidden "enable Android Auto" switch.

They appear to use several layers at once:

1. a base firmware/runtime platform
2. a feature licensing layer
3. module coding / adaptation data
4. persistence storage for device state and settings
5. application and UI logic running on top

Across VW/VAG materials and community tools, the feature licensing layer is typically described as **FEC/SWaP**.

In plain language, FEC/SWaP is the mechanism by which the unit decides whether a feature should be recognized as licensed and available. The publicly visible feature lists include things like:

- Navigation
- Bluetooth
- MirrorLink
- Apple CarPlay
- Google Android Auto

That matters because it tells us Android Auto is not merely a user-interface capability. In this ecosystem it is treated as a licensable head-unit function.

## The old community model: how modders approached VAG infotainment before MH2p ModKit

Before the MH2p-specific approach enters the scene, the broader VAG modding world already had a recognizable pattern.

The older toolbox-style flow looks roughly like this:

1. identify the exact head-unit family and firmware
2. get the unit into a state where it will accept modified feature data
3. add or generate the needed FEC/SWaP material
4. apply companion coding or adaptations
5. repair or clear related state if necessary

Projects like **M.I.B. More Incredible Bash** are useful because they show that this was never just about toggling one flag. The community was already dealing with:

- patching system images
- editing or generating FEC containers
- applying brand/platform-specific Android Auto or CarPlay fixes
- and handling unit-specific edge cases

So the MH2p work did not invent the general concept. It inherits the same class of problem: the unit must be convinced, across several subsystems, that Android Auto is both licensed and configured.

## The MH2p change in strategy: from patch toolbox to signed installer framework

`MH2p_SD_ModKit` is the key architectural shift.

Instead of being primarily a general-purpose patch toolbox for many unit families, it behaves like a **head-unit-specific signed installer framework**.

The important idea is this:

- the project presents itself as a valid checksummed and signed update
- that update gets accepted by the target unit
- the accepted update runs `modkit.sh`
- `modkit.sh` launches `modkit_install.sh`
- from there the framework can execute unsigned mod payloads from `/Mods/`

That is the pivot.

The hard part is no longer "how do I manually perform every modification from scratch?" The hard part becomes "how do I package my desired change as a mod once the signed update wrapper has granted execution?"

That is why `MH2p_SD_ModKit` feels more like an operating framework than a single hack.

## The hooks the MH2p ecosystem uses

Once the ModKit has execution, it has several hooks into the system.

### 1. Update-time execution

The `Update` stage runs during the software update process itself.

This is where a mod can:

- copy files
- edit data on mounted partitions
- prepare persistent payloads
- modify feature stores such as FEC-related files

For Android Auto, this is where the entitlement layer is handled.

### 2. Post-update execution

The `Post` stage runs once after the unit boots back into a more normal runtime state.

This exists because some operations are more reliable or only possible after reboot, especially module coding or persistence writes that do not succeed cleanly during the update transaction.

For Android Auto, this is where the 5F-related writes happen.

### 3. Persistent startup execution

The ModKit patches startup by replacing `servicemgrmibhigh` with a wrapper script and preserving the original binary as `servicemgrmibhigh0`.

That wrapper launches the original service manager and also launches `modkit_persist.sh`.

This gives the framework a durable every-boot hook.

With that hook, mods can:

- run every startup
- run one-time `Post` actions then clean themselves up
- perform failsafe recovery when removable media is inserted

### 4. File and persistence hooks

The public mod examples show several concrete places the community uses:

- `/mnt/persist_new/fec`
  for feature entitlement material
- `/mnt/persist_new/persistence/persistence.sqlite`
  for persisted device/application state
- `/mnt/app/eso/hmi/lsd/jars`
  for jar/classpath-based behavior overrides
- `/mnt/app/eso/bin`
  for startup wrapper insertion

### 5. Built-in device tooling hooks

The framework also leans on utilities already present in the unit environment.

Two especially important examples are:

- `pc`
  the persistence client used for reading/writing 5F-related values
- the original platform runtime and classpath behavior under LSD jars
  which allows jar-based overrides to influence application behavior

In other words, the modding framework does not replace the factory runtime. It piggybacks on it, stages data around it, and overrides selected seams in it.

## How Android Auto comes together in the MH2p ecosystem

The public `MH2p_AndroidAuto` mod is where the architecture becomes concrete.

The mod does not rely on a single change. It composes several layers, each solving a different piece of the puzzle.

### Layer 1: FEC entitlement

In `Update/install.sh`, the mod backs up the FEC area and then runs:

```sh
fecswap -a 00030000 00050000 00060900 -f /mnt/persist_new/fec/granted.fecs
```

This adds:

- AMI/USB
- Bluetooth
- Google Android Auto

This is the clearest proof that Android Auto enablement includes direct FEC-level entitlement work.

### Layer 2: 5F coding

In `Post/install.sh`, the mod uses the built-in `pc` tool to set specific 5F-related bits:

```sh
/mnt/app/armle/usr/bin/pc b:0x5F22:0x600:19.6 1
/mnt/app/armle/usr/bin/pc b:0x5F22:0x600:19.7 1
/mnt/app/armle/usr/bin/pc b:0x5F22:0x22AD:7.7 1
```

Those comments identify:

- USB1 data
- USB2 data
- Google GAL

So Android Auto is not considered complete after FEC insertion alone. The system also needs module-level configuration changes.

### Layer 3: Porsche-specific repair logic

For Porsche (`OEM == PO`) on affected firmware ranges (`26xx` and `28xx`), the install path also deploys a repair flow:

- copy `aafix.jar` into `/mnt/app/eso/hmi/lsd/jars/aafix.jar`
- run `fix_partition_1008 --fix` against `persistence.sqlite`

That means the Android Auto mod, in its current public form, already folds in the once-separate Porsche repair logic when the target matches.

## Why Porsche needs extra work

The Porsche-specific fix repo explains a second layer of trouble that appears after activation.

The issue is not that Android Auto is missing as a licensed feature.
The issue is that a device activation state machine in the Java-side Android Auto / CarPlay integration can demote previously accepted devices into a `NATIVE_SELECTED` state.

When that happens, reconnect behavior breaks down and the unit may fall back to charging-only behavior instead of clean Android Auto reconnects.

So on affected Porsche units there are really two separate problems:

1. **feature enablement**
   Android Auto must be licensed and configured
2. **runtime state correctness**
   the unit must remember device acceptance correctly across reconnects

The persistence repair tool confirms this in source.

It opens the SQLite persistence database, finds the serialized device list blob in partition `1008`, replaces `NATIVE_SELECTED` with `DISCLAIMER_ACCEPTED`, recalculates CRC32, and writes the fixed blob back.

That makes the Porsche path more layered than the generic Android Auto path.

## The role of jar-based patching

One of the most revealing facts in the wider MH2p ecosystem is that jars copied into `/mnt/app/eso/hmi/lsd/jars` are loaded before `lsd.jar`.

That gives the community a very useful application-layer hook:

- ship a jar with replacement or overriding classes
- let the platform classpath precedence favor those classes
- influence behavior without rebuilding the full stock UI jar

This is visible in the CarPlay fullscreen work, and it also explains why the Porsche Android Auto repair can plausibly use `aafix.jar` as part of its fix path.

This is important because it shows the community is working at more than one level:

- storage/data layer
- coding/configuration layer
- startup/execution hook layer
- application bytecode/classpath layer

## The full mental model

If you zoom all the way out, the architecture looks like this:

```text
VW / VAG infotainment platform
  -> firmware/runtime
  -> FEC/SWaP licensing
  -> module coding / persistence
  -> Java/UI behavior

Community toolbox era
  -> patch images / patch acceptance logic
  -> generate or inject FEC material
  -> apply coding/adaptation changes
  -> add platform-specific fixes

MH2p SD ModKit era
  -> use signed update wrapper to gain execution
  -> run modular payloads from /Mods/
  -> split work into Update / Post / Persist
  -> patch startup and optionally override app behavior with jars

Porsche Android Auto path
  -> add FEC entitlement
  -> apply 5F coding
  -> for affected Porsche firmware:
     -> deploy jar-based repair
     -> repair persistence state in sqlite/blob storage
```

## What this means for "how Android Auto works on my friend's Porsche"

The clean answer is:

Android Auto works on that Porsche not because one hidden checkbox was turned on, but because several layers were made to agree.

Those layers are:

- the unit now has the relevant feature entitlement in its FEC store
- the 5F-related configuration has been changed so the feature path is exposed
- on affected Porsche firmware, the runtime and persistence bug that breaks reconnect behavior has been repaired

And the reason the community can do that repeatably is that `MH2p_SD_ModKit` provides the delivery vehicle:

- a signed update wrapper to get code execution
- a structured mod layout
- persistent hooks after reboot
- the ability to combine file edits, coding changes, persistence fixes, and jar-based overrides in one install flow

## Bottom line

The VW/VAG vendor world provides the layered machinery:

- feature licensing
- coding/adaptations
- persistence
- application logic

The modding community learned how to work with, around, and through those layers.

The older ecosystem did that with broader patch toolboxes such as M.I.B.
The MH2p ecosystem does it with a signed-update wrapper and modular install stages.

For Porsche PCM5 / MH2p Android Auto, the final result is a composed solution:

- FECs make the feature licensable
- 5F changes make it configurable
- persistence and jar-side fixes make it behave correctly on affected units

That is how the pieces come together.
