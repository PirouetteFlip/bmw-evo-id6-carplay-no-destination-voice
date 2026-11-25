# BMW EVO ID6: Restoring CarPlay Voice Guidance Without a Destination

English translation of the Korean write-up · 2026-11-25

**License**: [CC BY 4.0](https://creativecommons.org/licenses/by/4.0/) © 2026 PirouetteFlip. You may copy, modify and redistribute this freely as long as you credit the author and link to the original. The risk notice and disclaimer below still apply.

## Risks and disclaimer

- **Risk of a non-booting head unit**: this modifies an executable on the head unit. If something goes wrong, it may no longer boot.
- **Back up first**: make backups in two places (the car's internal file system and a USB stick) before you start.
- **Responsibility**: everything you do is at your own risk, and it may affect your vehicle warranty.
- **Scope of the author**: the author only shares what was applied to and verified on a single car. The author cannot take responsibility or help if something goes wrong.
- **Other firmware / models**: the layout of the executable may differ. Before trying, use AI or other tools to check whether this applies to your car.
- **What is shared**: only a description of the method, not a patch program.

## Summary

In no-destination driving (CarPlay running with no route set; called "안심주행" in Korean navigation apps), CarPlay navigation voice prompts were silent. This was fixed by changing one 4-byte instruction in the `Connectivity` executable, and it was verified on one real car.

- **Symptom**: in no-destination driving, Tmap, Naver Map and Apple Maps all produce no voice guidance.
- **Cause**: when a destination is set, navigation ownership goes to the iPhone (`Navigation Owner: Apple Device` in the CarPlay log), and only then is the CarPlay guidance audio connected to the output channel (source 23 → sink 4). Without a destination the log says `Navigation Owner: None`, so the guidance audio is not delivered.
- **Fix**: inside `handleAltAudioUpdate`, replace one conditional branch (`beq`) that checks whether guidance is active with a NOP. The guidance audio is then delivered whether or not a destination is set, so sound works without a destination too.

## Test environment

**Note:** this was verified only on the combination below. Other firmware may not have the same bytes at the same position.

| Item | Details |
| --- | --- |
| Car | BMW F33 LCI (Korean market) |
| Head unit | EVO iDrive 6 (NBT EVO ID6), QNX-based, 6WB instrument cluster |
| Access | OBD-ENET cable + Ethernet adapter, SSH (shell prompt `hu-omap`) |
| Target file | `/opt/conn/bin/Connectivity` (ARM ELF, 7,529,956 bytes) |
| Original SHA-256 | `17a31d3cabfb4d1d3ae535d85d4e1999ef999d57de3face258401d4396641d5f` |

## How it works

`handleAltAudioUpdate` is called when the CarPlay guidance audio (Alt Audio) state changes. In no-destination driving, it skips the request for the guidance audio path. The following is a simplified version of the disassembly.

```
handleAltAudioUpdate(duckAudio, connectionID):
    log(bDuckAudio, mConnctionID, CountBuildHighSync, TurnByTurn)
    turnByTurn = CarPlay guidance state      # 0 in no-destination driving

    if turnByTurn == 0:                      # cmp r0,#0 ; beq  <- before the patch it jumps here
        return                               # guidance audio path is not requested -> no sound

    requestBuildHighSyncConnection(guidance audio path)   # nonzero: guidance audio is passed on
                                             # -> jacinto: source 23 -> sink 4 connection
```

1. **Compare** `cmp r0,#0`: checks whether the guidance state is 0 (off).
2. **Jump** `beq` (`19 00 00 0a`): if it is 0, the connection request is skipped. This is why there was no sound in no-destination driving.
3. **Connection request** `requestBuildHighSyncConnection`: if it is not 0, the guidance audio path (source 23 → sink 4) is requested.

✅ **What to change**: replacing `beq` with `nop` (`00 00 a0 e1`) removes the jump, so step 3 always runs regardless of the state.

**Limits**: the pseudocode simplifies only the area around this branch. The real function has other branches (ducking vs. non-ducking, handling of the connection count `CountBuildHighSync`, and so on), and I did not analyze all of them. That is why the "possible side effects" below remain open.

## Applying and restoring

The change is 4 bytes inside the `Connectivity` file. Writing directly into the memory of the running `Connectivity` process is blocked, so a patched copy of the original is swapped in and the unit is rebooted to apply it.

The partition holding the file, `/fs/sda0` (`/opt`), is **read-only** by default. So right before swapping the file, make it writable with `mount -uw /fs/sda0`, and after the swap, return it to read-only with `mount -ur /fs/sda0`. This is already included in the steps below.

| Item | Value |
| --- | --- |
| Position | block 1306899 in `dd bs=4` units |
| Original | `19 00 00 0a` (`beq`) |
| Patch | `00 00 a0 e1` (NOP) |
| Surrounding 20 bytes (original) | `2c 30 93 e5  33 ff 2f e1  00 00 50 e3  19 00 00 0a  24 30 94 e5` |

The car's shell has no `od`, `hexdump`, `xxd`, `touch`, `chown`, `awk` or `python`. Bytes were checked with `sed -n l`, and the binary string was produced with the ksh builtin `print -n`.

### Steps

**Prerequisite: SSH access**

This method requires SSH access to the head unit. SSH is not open by default, so you first have to open it using the methods already known in the community. This document does not cover how to open SSH.

- **Connection**: connect your terminal to the head unit with an OBD-ENET cable and an Ethernet adapter and log in over SSH. The author used a terminal app on an Android phone.
- **Check the connection**: if the shell prompt looks like `hu-omap:/dev/shmem>`, you are in the omap-side shell. Run all commands below in this shell.
- **Conditions**: do this while parked, and do not touch the file before the backup in step 1 is finished.

**What you need: a USB stick for the backup**

To keep one more copy of the original `Connectivity` outside the car, plug a writable USB stick into the car's USB port. The file is about 7.5 MB, so 8 MB or more of free space is enough. Before step 1, connect over SSH and check that `ls /fs/usb0` works.

**Pre-check: verify the original bytes**

Before patching, check that the target position matches the original described here. If the output differs, you probably have different firmware, so stop here.

```
dd if=/opt/conn/bin/Connectivity bs=4 skip=1306896 count=5 | sed -n l
# Expected output (original; computed from the bytes in the table above): shown on two lines
# ,0\223\3453\377/\341\000\000P\343\031\000\000$
# $0\224\345$
```

1. **Back up (two copies of the original)**. Create `t_Connectivity.orig` on the USB stick. The in-car backup (`Connectivity.orig`) is created during the swap in step 5.

```
cp /opt/conn/bin/Connectivity /fs/usb0/t_Connectivity.orig
ls -l /fs/usb0/t_Connectivity.orig   # check that it is 7529956 bytes
```

2. **Create the 4-byte NOP file** (`/dev/shmem` is RAM, so it is cleared when the car is switched off).

```
(dd if=/dev/zero bs=1 count=2 2>/dev/null; print -n '\0240\0341') > /dev/shmem/t_nop
```

3. **Patch a copy in RAM**. The live file is not touched. The goal is to overwrite exactly 4 bytes. The `dd` options below are simply the way the author did it. When it finishes, make sure the file size is still 7,529,956 bytes, and stop if it is not.

```
cp /opt/conn/bin/Connectivity /dev/shmem/t_Conn_patched
dd if=/dev/shmem/t_nop of=/dev/shmem/t_Conn_patched bs=4 seek=1306899 conv=notrunc
ls -l /dev/shmem/t_Conn_patched
```

4. **Verify the patch**. After patching, the output should be a single line like the one below (the original appears split over two lines because of its `0x0a` byte). The live file should still be the original at this point.

```
dd if=/dev/shmem/t_Conn_patched bs=4 skip=1306896 count=5 | sed -n l
# Expected output: ,0\223\3453\377/\341\000\000P\343\000\000\240\341$0\224\345$
```

5. **Swap**. If anything fails, stop this sequence and restore the original. Since `/fs/sda0` is read-only by default, make it writable briefly, and return it to read-only when done.

```
mount -uw /fs/sda0
cp /dev/shmem/t_Conn_patched /opt/conn/bin/Connectivity.new && chmod 775 /opt/conn/bin/Connectivity.new
mv /opt/conn/bin/Connectivity /opt/conn/bin/Connectivity.orig && mv /opt/conn/bin/Connectivity.new /opt/conn/bin/Connectivity
dd if=/opt/conn/bin/Connectivity bs=4 skip=1306896 count=5 | sed -n l
mount -ur /fs/sda0
```

6. **Reboot the head unit** (rebooting drops the SSH connection, so reconnect after it has booted). `Connectivity` is not restarted automatically if it dies (`MaxProcessRestarts=0`), so applying the change by rebooting is the safer way.
7. **Check**. Start CarPlay in no-destination driving (no route set) and see whether the guidance voice plays. Also check music, phone calls and the stock navigation. For the logs, run `on -f hu-jacinto sloginfo | grep "sourceID=23"` within 10 seconds after the sound (the log scrolls away quickly).

### Restore

```
mount -uw /fs/sda0
cp /opt/conn/bin/Connectivity.orig /opt/conn/bin/Connectivity.rst && mv /opt/conn/bin/Connectivity.rst /opt/conn/bin/Connectivity
mount -ur /fs/sda0
```

After restoring, reboot the head unit. `Connectivity` does not manage the SSH server (there is no SSH entry in its process-management config), so SSH is expected to stay alive even if this file is broken, but this was not actually tested. The `t_Connectivity.orig` on the USB stick is the last-resort backup.

### If something goes wrong

- **Error during step 5**: if `Connectivity` is missing or its size looks wrong, with the partition still writable run `mv /opt/conn/bin/Connectivity.orig /opt/conn/bin/Connectivity` to put it back, then run `mount -ur /fs/sda0`. A leftover `Connectivity.new` can be ignored.
- **CarPlay or sound is wrong after reboot, but SSH works**: run the restore procedure above and reboot again.
- **SSH does not work after reboot either**: this document cannot recover from that. The author has not been in this situation and has not verified any recovery method. So only reboot while parked and after the step 5 verification output matched.

## What was confirmed

These were confirmed in three ways: log comparison, binary reverse engineering, and a real-car test.

- **Log comparison**: in the log with a destination set, a `sourceID=23 → sinkID=4` connection appears on jacinto. In the log without a destination, it never appears. The stock navigation voice (`sourceID=17 → sinkID=4`) is always connected.
- **Where "Navigation Owner" comes from**: the CarPlay connection library (`libconn_carplay.so`) writes it to the omap-side log in the form `Modes changed: … Navigation Owner : Apple Device` or `… Navigation Owner : None`. Comparing this line with and without a destination shows whether you have the same symptom.
- **The audio stream does arrive**: even without a destination, the CarPlay Alt Audio (ducking audio) stream reaches the head unit, but with no output connection there is no sound.
- **Location of the branch**: in `handleAltAudioUpdate` of `Connectivity`, there is a `cmp r0,#0` + `beq` pair that skips the `requestBuildHighSyncConnection` call when the TurnByTurn value is 0. File offset `0x4FC44C` is that `beq`.
- **Real-car test**: after patching and rebooting, guidance voice plays in no-destination driving. Music, phone calls and the stock navigation are also normal. The patch survives reboots (because the file itself was replaced).
- **Attempts that had no effect**: `NativeNaviSupported=NO` in `carplay.ini` had no effect, and removing the AltAudio line from `FormatSupport` broke CarPlay sound entirely. Both were reverted.

## Cautions

- **Other firmware / models**: the offsets and bytes are for the file with the SHA-256 above. If your hash differs, do not use them as they are; first check whether the original bytes in the "Applying and restoring" table match.
- **Possible side effects**: because the guidance audio path is now always requested, I have not checked over a long period whether sound overlaps or cuts out in other situations such as phone calls, Siri, or music resuming. Behavior when a destination is set has also not been tested over the long term.
- **Unexplained log**: a `kDuplicateErr` AirPlay assert appeared only when there was no destination. I did not investigate its relation to the patch.
- **File owner**: the car has no `chown`, so the file's owner became root (permissions stay 775). No problem was observed.
- **Updates**: if a firmware update or coding replaces `Connectivity`, the patch is lost.

## Revision history

| Date | Change |
| --- | --- |
| 2026-11-25 | First version. Verified on one BMW F33 LCI |
