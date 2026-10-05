---
title: "Linux Disk Unmounted After Reboot - Recovering the Boot"
date: 2026-10-05T00:00:00Z
draft: false
description: "Root partition stops being mounted after a crash or unclean shutdown; recover from a live USB by fixing fstab, the initramfs filesystem driver and the bootloader on any Linux distro"
categories: ["System Administration"]
tags: ["linux", "grub", "fstab", "initramfs", "mkinitcpio", "dracut", "efi", "recovery"]
---

## Problem

After a crash, power loss or unclean shutdown, the machine drops to an emergency shell or fails to find the root filesystem:

```
ALERT! /dev/mapper/... does not exist. Dropping shell.
# or
error: /dev/nvme0n1p2 does not exist
```

From a live USB, `lsblk -f` shows the partitions are intact (an ext4 or btrfs root, a vfat EFI partition), but the system no longer mounts them at boot. Usually at least one of these is true:

- `/etc/fstab` has a stale or wrong `UUID=` (the disk was cloned, the partition table was regenerated, or `genfstab`/`blkid` was run before the disk settled).
- The initramfs does not include a driver for the EFI partition's filesystem, so the kernel cannot read the bootloader and the boot chain never reaches the root filesystem.
- The EFI bootloader entries were removed or never written to the new EFI partition.

This is distro-independent: only the chroot helper, the initramfs generator, and the EFI mount point differ. The table below maps them, everything else is identical.

| Distro family | Chroot | Initramfs | EFI partition mounted at |
| --- | --- | --- | --- |
| Arch, Manjaro, EndeavourOS | `arch-chroot /mnt` | `mkinitcpio -P` | `/boot` |
| Fedora, RHEL, Rocky, AlmaLinux | `chroot /mnt` | `dracut --regenerate-all --force` | `/boot` |
| openSUSE (GRUB) | `chroot /mnt` | `dracut --regenerate-all --force` | `/boot` |
| Debian, Ubuntu (GRUB) | `chroot /mnt` | `update-initramfs -u -k all` | `/boot/efi` |
| Any distro with a unified kernel image | `chroot /mnt` | `kernel-install add $(ls /usr/lib/modules)` | `/boot/efi` |

## Attempted Solutions

- **Booting the installed system directly** — failed; the bootloader stage itself depends on the initramfs being able to read the EFI partition.
- **Reinstalling GRUB from the live USB only** — GRUB installed fine but the machine still failed, because the initramfs could not mount the root filesystem. The missing `vfat` driver in the initramfs was the real blocker.

## Final Solution

Boot any Linux live image (Arch, Ubuntu, Fedora, SystemRescue, ...). On Arch ISO pick **Arch Linux (no password)**; on Ubuntu, **Try Ubuntu**.

### 1. Identify the partitions

```bash
lsblk -f
```

```text
NAME          FSTYPE LABEL  UUID                                 MOUNTPOINTS
sda
├─sda1        vfat   EFI    1A2B-3C4D                           <- EFI
├─sda2        ext4   arch   5e6f7a8b-9c0d-1e2f-3a4b-5c6d7e8f9a0b  <- root
└─sda3        swap   swap   0a1b2c3d-4e5f-6a7b-8c9d-0e1f2a3b4c5d
```

Naming is not fixed: `sda1`/`sda2` on SATA, `nvme0n1p1`/`nvme0n1p2` on NVMe, `vda1` on virtio. Use the `FSTYPE` and `LABEL` columns, not the numbers, to identify them. Note the UUIDs, you will compare them in step 4.

Confirm the layouts below. Any of these is valid, just match what you see:

```bash
blkid                          # all partitions with UUIDs, from the live env
ls -d /sys/firmware/efi/efivars >/dev/null && echo "booted via UEFI"   # if this fails, the disk is BIOS/MBR, use BIOS boot paths below
```

### 2. (Optional) Check the filesystem

An unclean shutdown leaves the ext4 journal dirty, which is usually replayed automatically on mount. If the mount fails or reports errors, check it explicitly:

```bash
fsck.ext4 -f /dev/sda2            # ext4 root
# for btrfs, before mounting:
mount -o ro /dev/sda2 /mnt && btrfs scrub start -Bd /mnt && umount /mnt
```

Do not run `fsck` on a mounted filesystem.

### 3. Mount root, then EFI

```bash
mount /dev/sda2 /mnt
mount /dev/sda1 /mnt/boot     # /boot/efi on Debian, Ubuntu, Gentoo
```

Mount the EFI partition where the installed system expects it: `/boot` for Arch/Fedora/openSUSE, `/boot/efi` for Debian/Ubuntu/Gentoo. You can tell by what already exists in `/mnt/boot` (a `grub/` directory means `/boot`; an `efi/` directory means `/boot/efi`).

If your root is a Btrfs subvolume or LVM, mount the top-level device or volume first and use its subvolume, e.g. `mount -o subvol=@ /dev/sda2 /mnt`.

### 4. Chroot into the installed system

```bash
arch-chroot /mnt          # Arch and derivatives
# or, on everything else:
chroot /mnt
```

With a plain `chroot`, bind the pseudo-filesystems so package tools and the initramfs generator work:

```bash
mount --rbind /proc  /mnt/proc
mount --rbind /sys   /mnt/sys
mount --rbind /dev   /mnt/dev
mount --rbind /run   /mnt/run
```

`arch-chroot` does these for you.

### 5. Check fstab against the real UUIDs

```bash
cat /etc/fstab
blkid
```

Every `UUID=` in `fstab` must match a `UUID` reported by `blkid`. Fix any mismatch:

```bash
nano /etc/fstab
```

Or regenerate the whole file from the currently mounted tree:

```bash
genfstab -U /mnt > /mnt/etc/fstab.new && mv /mnt/etc/fstab.new /mnt/etc/fstab
cat /mnt/etc/fstab   # verify before rebooting
```

Comment out any stale `resume=` swap line rather than pointing it at a UUID that does not exist; a bad resume entry hard-hangs the boot before you ever see a prompt.

### 6. Make sure the initramfs can read the EFI filesystem

The initramfs must contain a `vfat` driver, otherwise the kernel cannot read the partition holding the bootloader.

Arch and Manjaro, `/etc/mkinitcpio.conf`:

```conf
MODULES=(vfat)
```

Fedora, RHEL, openSUSE, `/etc/dracut.conf.d/fat.conf`:

```conf
add_drivers+=" vfat "
```

Alternatively, force it from the kernel command line with `rd.driver.pre=vfat` (dracut) or `MODULES=(vfat)`.

Debian and Ubuntu already include the FAT driver in the default initramfs, so this step usually needs nothing; verify with `lsinitramfs /boot/initrd.img-$(uname -r) | grep -i vfat` if the boot still fails.

### 7. Regenerate the initramfs

```bash
mkinitcpio -P                                  # Arch, Manjaro, EndeavourOS
dracut --regenerate-all --force                # Fedora, RHEL, openSUSE
update-initramfs -u -k all                     # Debian, Ubuntu
```

For a unified kernel image (no GRUB), install the kernel and its UKI:

```bash
kernel-install add $(ls /usr/lib/modules)
```

### 8. Reinstall the bootloader

Check which bootloader the EFI partition actually has before reinstalling, and reinstall only that one:

```bash
ls /mnt/boot/efi/EFI/ 2>/dev/null || ls /mnt/boot/EFI/
```

**GRUB** — the common case. `GRUB` or `ubuntu` means GRUB is in use:

```bash
grub-install --target=x86_64-efi --efi-directory=/mnt/boot --bootloader-id=GRUB
grub-mkconfig -o /mnt/boot/grub/grub.cfg
```

Adjust `--efi-directory` to `/mnt/boot/efi` on Debian/Ubuntu. On Debian and Ubuntu, installing the distribution's `grub-efi-amd64-bin` package first is safer than running `grub-install` from the live image, because the live image's GRUB version may differ from the installed one:

```bash
apt install grub-efi-amd64-bin   # in the chroot, on Debian/Ubuntu
```

**systemd-boot / unified kernel images** — a `Linux` directory with `*.efi` files means UKI, and `grub-install` does not apply:

```bash
# systemd-boot: restore the loader entry
bootctl install
bootctl list
```

**BIOS / MBR system** (if `ls /sys/firmware/efi` failed in step 1) — install to the disk's MBR instead, with the disk, not the partition, as target:

```bash
grub-install --target=i386-pc /dev/sda
grub-mkconfig -o /boot/grub/grub.cfg
```

### 9. Exit, unmount, reboot

```bash
exit
umount -R /mnt       # -R unmounts /mnt/boot/efi and /mnt/boot before /mnt
sync
reboot
```

## Why It Works

The boot chain is: firmware → `\EFI\GRUB\grubx64.efi` on the vfat EFI partition → GRUB → kernel + initramfs in `/boot` → root filesystem. Every link can break silently and produce the same "cannot mount root" symptom:

- **fstab** is read after the root filesystem is mounted, so a wrong UUID there does not block the kernel stage, but it breaks the normal path, `resume=`, and any service that mounts `/boot`.
- **The initramfs** is built from the distribution's config (`MODULES=` on Arch, `add_drivers=` on dracut distros). A missing `vfat` driver means the early userspace cannot mount `/boot` to fetch the kernel and bootloader, so the boot stalls regardless of a correct GRUB install. This is the most commonly missed step.
- **grub-install** rewrites the EFI entry and the loader image, repairing the firmware → GRUB link after a reinstall, a firmware update, or a disk clone.

Running the fstab, initramfs, and bootloader steps together from the live image fixes every link at once, which is why the sequence works even when all you knew was the symptom "the disk is not mounted".