# penguin-swap — Enable swap in the ChromeOS Linux container (Crostini / Flex)

[日本語版 (Japanese)](README.ja.md)

## Background: penguin has no swap

The ChromeOS Linux container ("penguin") ships with **no swap at all**: `/proc/swaps` is empty and `free` reports `Swap: 0`. ChromeOS exposes no setting to add any. When memory runs out, processes simply die.

## Problem

Creating a swap file the usual way does not work. The container's root filesystem is btrfs, and that is the wall:

```console
$ sudo fallocate -l 2G /swapfile
$ sudo chmod 600 /swapfile
$ sudo mkswap /swapfile
$ sudo swapon /swapfile
swapon: /swapfile: swapon failed: Invalid argument
```

The error does not tell you why. The platform is not forbidding swap — the **file attributes** are wrong.

## Why it fails

On btrfs, the kernel requires a swap file to be:

- NOCOW (copy-on-write disabled)
- fully allocated, with no holes
- owned by root, mode 600

A file created the ordinary way is copy-on-write, so `swapon` rejects it with `Invalid argument`.

## Recipe

Set NOCOW on the empty file first, then allocate it in full:

```bash
# Create a 2 GiB swap file
sudo touch /swapfile
sudo chattr +C /swapfile          # NOCOW — must be set while the file is empty
sudo dd if=/dev/zero of=/swapfile bs=1M count=2048 status=progress
sudo chmod 600 /swapfile
sudo mkswap /swapfile
sudo swapon /swapfile

# Verify
cat /proc/swaps
free -h
```

To grow further, add a second file (capacities add up):

```bash
sudo touch /swapfile2
sudo chattr +C /swapfile2
sudo dd if=/dev/zero of=/swapfile2 bs=1M count=2048 status=progress
sudo chmod 600 /swapfile2
sudo mkswap /swapfile2
sudo swapon /swapfile2
```

## Operational notes

- **Swap does not survive a container restart.** The files persist, so after each boot just run `sudo swapon /swapfile` (for each file you created). `mkswap` is only needed when creating a file.
- Swap is shared by the whole container; it is not per-application.
- Swap adds *capacity*, not *speed*: data that is actively used still needs to live in RAM. On an SSD, however, page-in latency is small, so swap absorbs transient peaks very well — nothing like the thrashing death spiral of the HDD era.
- To stop: `sudo swapoff /swapfile`, then delete the file to reclaim the disk space.

## Measured results (2026-11-19)

- Hardware: MacBook Air 13-inch (Early 2014) running ChromeOS Flex; Linux container (Debian 13, btrfs). The container's memory total is fixed at 2,792 MB with no supported way to raise it.
- Workload: a Windows game under Wine 11.0 whose home screen needs ~2.3 GB plus ~0.6 GB of resident processes — more than the container total, so it always died at the announcement / memorial-lobby peak.
- No swap: 8 launch attempts — reached home 4 times, sustained 0 times.
- 2 GiB swap: reached home and stayed usable for several minutes (settings screen, BGM playing), but with almost no headroom left after the peak, it died under continued use.
- 4 GiB swap: sustained at home; the 3D café and the student list worked. Suppressing the announcement WebView ("Don't show again today") removed the single largest spike (~950 MB) and made it stable in practice.

## License

MIT License.
