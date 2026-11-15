# Nexus 9 (flounder) — 1 GiB LZ4 zRAM / 2 compression streams

This repository documents and tracks a verified zRAM configuration repair for the HTC Nexus 9 (device codename `flounder`), running Android 7.1.2 / CM14.1-based software with Linux kernel `3.10.103-g5cef0dc1890-dirty`.

## Project status

- **Device-side repair:** flashed to the boot partition and verified on the device.
- **Configuration:** 1 GiB zRAM, LZ4 compression, 2 compression streams.
- **Stress validation:** a 768 MiB anonymous-memory test produced real swap-out and swap-in activity; memory availability recovered after the test.
- **Repository contents:** this README and the detailed technical report only.
- **Final boot image binary:** not currently stored in this repository. Do not assume that a downloadable image is available here.

## Detailed report

See **[Nexus 9 zRAM 1 GiB / LZ4 / 2-streams repair and validation report](Nexus9_zram_1GiB_LZ4_2streams_report.md)** for the device details, root-cause analysis, exact fstab change, static image audit, flashing procedure, runtime checks, stress-test measurements, and recovery notes.

## What was fixed

The original init script wrote `2` to `/sys/block/zram0/max_comp_streams`, but CM14.1's `fs_mgr` initialized zRAM from the active `fstab.flounder` and applied its default stream count of `1`. The fstab initialization therefore overrode the earlier init-script setting.

The active fstab entry was changed to:

```text
/dev/block/zram0 none swap defaults zramsize=1073741824,zramstreams=2,notrim
```

Only the active `fstab.flounder` entry in the ramdisk was changed. The kernel binary, DTB data, and `fstab.flounder64` were not modified. The existing init configuration selects LZ4.

## Verified runtime configuration

After flashing the repaired image to the **boot** partition and rebooting, the device reported:

| Check | Verified value |
|---|---|
| Compression algorithm | `lzo [lz4]` (LZ4 is selected) |
| Maximum compression streams | `2` |
| zRAM disk size | `1073741824` bytes (1 GiB) |
| Initialization state | `1` |
| Swap device | `/dev/block/zram0`, enabled |

The 768 MiB stress test showed increasing `pswpout`, non-zero `pswpin`, and zRAM `mm_stat` usage. After releasing the test allocation, free memory recovered to approximately 937 MiB. A small amount of swap usage remained, which is normal and does not by itself indicate a fault. No OOM, reboot, or system crash was observed during the recorded test.

The approximately 5.12:1 compression ratio was measured for one test snapshot only; it is not a guaranteed ratio for other workloads.

## Final image identity

The locally verified final image is named:

`LNX_zram_1024MiB_zramstreams2_candidate.img`

- Size: **33,554,432 bytes** (32 MiB)
- SHA-256:

```text
b0d4887df788e87a0c14f3bddf7cd064701ac26a35a99082850710f89b05d1bc
```

**Important:** this repository currently does not contain that image binary. The SHA-256 above identifies the locally verified image; it does not mean the image has been uploaded here. Before flashing any copy, calculate its SHA-256 locally and require an exact match.

## Flashing notes

The recorded procedure used a temporary boot first, followed by a permanent flash to the boot partition:

```bash
fastboot boot LNX_zram_1024MiB_zramstreams2_candidate.img
```

After confirming the runtime values, the permanent flash command is:

```bash
fastboot flash boot LNX_zram_1024MiB_zramstreams2_candidate.img
fastboot reboot
```

Do not flash an image whose SHA-256 does not match the value above. The correct target is `boot`, not a partition named `LNX`. Keep a known-good original boot image available for recovery, and ensure the device has sufficient battery and a stable USB connection before flashing.

## Repository scope

Obsolete candidate images, diagnostic logs, and intermediate `.txt` artifacts have been removed from the repository. The detailed report preserves the relevant findings and measurements. The repository is intentionally kept to the README and report until the final verified image is uploaded.

## Disclaimer

This is a device-specific, community repair record, not an official Android, Google, HTC, or LineageOS release. Boot images are hardware- and build-specific. Do not use this image on another device or an unrelated Nexus 9 build without independently checking compatibility.
