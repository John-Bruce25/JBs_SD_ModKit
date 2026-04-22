# Porsche Android Auto Fix Mechanics

This note documents the Porsche-specific Android Auto repair path using actual public source files from `fifthBro/pcm5-androidauto-connect-fix` and the current `LawPaul/MH2p_AndroidAuto` integration.

## What the fix is for

Affected scenario:

1. Phone 1 connects and Android Auto works
2. Phone 2 later connects and Android Auto works
3. Phone 1 reconnects and only charging works

The public explanation attributes this to a bad state transition in the Android Auto / CarPlay device activation logic on Porsche PCM5 (MH2P), especially on `26xx` to `28xx` firmware.

## The two repair layers

The fix is not one change. It has two distinct layers:

1. Java/bytecode behavior repair
2. Persistence blob/database repair

## 1. Java/bytecode behavior repair

README claim:

- `DeviceManager$DeviceActivationRequestHandler.moveSelectionMarker()` incorrectly demotes inactive devices to `NATIVE_SELECTED`
- intended fix removes:

```java
device.setUserAcceptState(NATIVE_SELECTED);
```

The public mod installer confirms the deployment mechanism:

- copy `aafix.jar` into `/mnt/app/eso/hmi/lsd/jars/aafix.jar`

That aligns with the wider MH2p ecosystem pattern where jars in the LSD jars directory are loaded before `lsd.jar` and can override behavior.

## 2. Persistence repair

The public source file:

- `https://github.com/fifthBro/pcm5-androidauto-connect-fix/blob/main/fix_partition_1008/fix_partition_1008.c`

shows the persistence repair is real code, not just README prose.

### What the tool targets

- SQLite database: `/mnt/persist_new/persistence/persistence.sqlite`
- partition name: `1008`
- device list key: `1`

### What it looks for

The tool scans the serialized blob for:

- `NATIVE_SELECTED`

and replaces it with:

- `DISCLAIMER_ACCEPTED`

It does this using explicit binary patterns:

- native pattern length: `17`
- disclaimer pattern length: `21`

### Important implementation details confirmed from source

- opens the SQLite database directly with `sqlite3`
- resolves the actual partition row from `"persistence-partitions"`
- reads the blob from `"persistence-data"`
- validates CRC32 over the payload
- parses device names from the serialized structure for reporting
- rewrites the blob with longer replacement strings
- recalculates CRC32 and writes it back as big-endian 64-bit value
- updates the database row in place
- supports:
  - `--list`
  - `--dry-run`
  - `--fix`
  - `--db-path`
  - `--no-backup`

### Confirmed serialized structure

From source comments and parser logic:

```text
[8 bytes CRC32]
[4 bytes version]
[4 bytes device count]
[for each device]
  deviceUniqueId
  smartphoneType
  has name
  name
  userAcceptState
  wasDisclaimerPreviouslyAccepted
  storeUserAcceptState
  lastmode
  lastConnectionType
```

This is stronger than the README alone because the parser logic, lookup queries, and replacement algorithm all appear in source.

## Mod wrapper mechanics

The mod wrapper script:

- `https://github.com/fifthBro/pcm5-androidauto-connect-fix/blob/main/MH2p_ModKit_Mod/AndroidAuto_Fix/Update/install.sh`

confirms the install path:

- only runs for `OEM == "PO"`
- only runs for firmware `26xx` or `28xx`
- mounts `/mnt/app` read-write
- copies `aafix.jar` into `/mnt/app/eso/hmi/lsd/jars`
- backs up `persistence.sqlite`
- runs `fix_partition_1008 --fix` with `LD_LIBRARY_PATH` including:
  - `/armle/usr/lib`
  - `/usr/lib`
- stores pre- and post-patch SQLite copies in `Backup/`

The matching uninstall script removes `aafix.jar`, but does not attempt to reverse the persistence blob changes.

## Relationship to `MH2p_AndroidAuto`

Current upstream `MH2p_AndroidAuto` `Update/install.sh` now contains nearly the same Porsche-specific logic:

- same OEM gate
- same firmware gate
- same `aafix.jar` deployment
- same `fix_partition_1008 --fix` execution

So the once-separate Porsche fix has effectively been folded into the Android Auto install path for affected units.

## What is still not fully closed

- We still do not have the actual source or class diff for `aafix.jar`
- We do not yet have a sourced loader script proving the exact classpath or bootclasspath mechanism beyond README claims
- We therefore have high confidence in the persistence repair path, and medium confidence in the exact Java injection mechanics

## Best current conclusion

The Porsche Android Auto fix is now well-supported at the source level for the persistence side and operational wrapper side.

What it definitely does:

- deploys a jar-based behavior patch
- repairs partition `1008` device state entries in `persistence.sqlite`
- recalculates CRC32 correctly
- applies only to Porsche `26xx` to `28xx` firmware

What remains partially opaque:

- the internals of `aafix.jar`
- the exact loader mechanism that gives the jar precedence over stock behavior
