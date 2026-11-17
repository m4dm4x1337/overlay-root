# Overlay-Root

## Description

### Read-only root filesystem with a temporary OverlayFS write layer

Overlay-Root is a Debian package for Debian and Ubuntu systems. It uses
[OverlayFS](https://en.wikipedia.org/wiki/OverlayFS) to keep the real root
filesystem read-only while placing a temporary, writable `tmpfs` layer on top.
Applications can use the root filesystem normally, but changes made through the
overlay are stored in memory and discarded at the next reboot.

Overlay-Root is a small, straightforward alternative to Ubuntu's more
feature-rich [overlayroot package](https://wiki.ubuntuusers.de/overlayroot/)
([English machine translation](https://wiki-ubuntuusers-de.translate.goog/overlayroot/?_x_tr_sl=de&_x_tr_tl=en&_x_tr_hl=de&_x_tr_pto=wapp)).
It focuses on a temporary RAM-backed write layer and broad compatibility across
Debian and Ubuntu releases that provide compatible `initramfs-tools` and
OverlayFS interfaces.

This is useful for purposes such as:

* Testing software on real hardware without permanently changing the system
* Kiosk and shared computers that return to a known state after every reboot
* Reducing persistent writes to USB flash drives and other storage devices

## Compatibility

The package has been tested with Debian Stretch and later releases and Ubuntu
Trusty and later releases. It is expected to remain compatible with future
releases that provide compatible `initramfs-tools` and OverlayFS interfaces.

The initramfs scripts support both kernel module names used by these releases:

* `overlay` on current kernels
* `overlayfs` on legacy kernels

OverlayFS may also be built directly into the kernel instead of being provided
as a module.

The complete list of OverlayFS mount options is available in the
[Linux kernel documentation](https://docs.kernel.org/filesystems/overlayfs.html).
Overlay-Root uses the kernel defaults unless explicitly configured otherwise.

## Installation

Run the administrative commands in this guide from a root shell.

Download the Debian package and install it with:

```bash
apt-get install ./overlay-root_4.0.0_all.deb
```

The package automatically updates all existing initramfs images. Reboot the
system to activate Overlay-Root.

## Usage

After rebooting, the merged root filesystem is mounted at `/`. The two layers
remain accessible separately:

* `/mnt/root-ro` is the real, read-only root filesystem.
* `/mnt/root-rw` is the temporary `tmpfs` used by OverlayFS.
* `/mnt/root-rw/upper` contains the changed and newly created files.
* `/mnt/root-rw/work` is an internal OverlayFS working directory.

The active mounts can be inspected with:

```bash
findmnt /
findmnt /mnt/root-ro
findmnt /mnt/root-rw
```

The filesystem type reported for `/` should be `overlay` or, on a legacy
kernel, `overlayfs`.

### Optional Volatile Mode

Linux 5.10 and newer support a volatile OverlayFS mode that omits sync calls to
the temporary upper layer. To enable it for one boot, add this kernel command
line parameter:

```text
root-ro=volatile
```

The default remains unchanged when this parameter is absent. On an older
kernel, Overlay-Root prints a warning and uses the standard mount options.

## Persistent and Temporary Data

Changes made to files stored on the root filesystem are temporary. This also
includes user data under `/home` when `/home` is part of the root filesystem.

Filesystems mounted separately through `/etc/fstab`, such as a dedicated
`/home` partition, remain persistent. Overlay-Root removes the original root
entry and all swap entries from the temporary `/etc/fstab` used during the
running session. Other entries are retained.

The writable layer uses `tmpfs`, so writes consume memory. Large updates,
downloads, caches, or log files can exhaust the available space and cause
write failures.

Known high-usage directories can be moved to a separately mounted persistent
filesystem and replaced with symbolic links. For example, when `/home` is a
separate persistent filesystem, APT data can be stored there:

```bash
mkdir --parents /home/overlay-root/var/cache /home/overlay-root/var/lib
mv /var/cache/apt /home/overlay-root/var/cache/
mv /var/lib/apt /home/overlay-root/var/lib/
ln --symbolic /home/overlay-root/var/cache/apt /var/cache/apt
ln --symbolic /home/overlay-root/var/lib/apt /var/lib/apt
```

Prepare these links while Overlay-Root is disabled. Do not use a directory on
the root filesystem as the target because it would still be part of the
temporary overlay.

The systemd journal should also use volatile storage. By default, journald may
use up to 10% of the filesystem that contains its journal, capped at 4 GiB.
Because `/run` is usually a relatively small `tmpfs`, the runtime journal
normally has a much smaller limit than a journal stored below `/var/log`.

Configure journald while Overlay-Root is disabled:

```bash
mkdir --parents /etc/systemd/journald.conf.d
cat > /etc/systemd/journald.conf.d/90-overlay-root.conf << 'EOF'
[Journal]
Storage=volatile
EOF
rm --recursive --force /var/log/journal
```

After rebooting, journal files are stored below `/run/log/journal` and are
discarded during shutdown.

## Disabling Overlay-Root

### Temporarily

1. Reboot and select the normal entry in the GRUB menu.
2. Press `e` to edit it.
3. Add `disable-root-ro=true` to the end of the line beginning with `linux`.
4. Press `F10` to boot the edited entry.

The change applies only to this boot. To make future edits easier,
`disable-root-ro=` can be added to the kernel command line in
`/etc/default/grub`. Run `update-grub` afterward. The empty value leaves
Overlay-Root enabled; add `true` in the GRUB editor when it needs to be
disabled.

### Permanently

First boot once with `disable-root-ro=true` as described above. The normal root
filesystem is then writable, so the persistent marker file can be created:

```bash
touch /disable-root-ro
```

Overlay-Root remains disabled after subsequent reboots. To enable it again:

```bash
rm /disable-root-ro
reboot
```

## Updating or Removing the Package

Package changes made while Overlay-Root is active are temporary. To update or
remove the package permanently:

1. Disable Overlay-Root with the kernel parameter or marker file described
   above.
2. Reboot and confirm that `/` is no longer an OverlayFS mount.
3. Update or remove the package normally.

To remove it:

```bash
apt-get purge overlay-root
```

The package updates the initramfs again when it is removed.

## Troubleshooting

Boot messages from Overlay-Root use the tag `root-ro`. On systems with systemd,
they can be searched with:

```bash
journalctl --boot | grep --fixed-strings root-ro
```

The package installs an initramfs hook that adds the OverlayFS module matching
each target kernel. Kernels with OverlayFS built in do not require a module to
be copied.

## Changelog

### v4.0.0

* Replaced the static `modules.d` entry with a hook that selects the correct
  overlay module for the target kernel version
* Added detection for kernels with OverlayFS built in
* Moved the init-bottom script to `/usr/share/initramfs-tools/scripts/init-bottom`
* Added support for both `overlay` and the legacy `overlayfs` driver
* Skipped `workdir` when using the legacy `overlayfs` driver
* Removed AUFS support and the `root-ro-driver` kernel parameter because AUFS
  keeps direct references to its branch mounts, preventing systemd from
  cleanly unmounting them during shutdown; AUFS is also out-of-tree and no
  longer available in current distributions
* Fixed creation order for the writable upperdir after mounting `tmpfs`
* Added the optional `root-ro=volatile` mode for Linux 5.10 and newer
* Renumbered error messages consistently

### v3.0.0

* Resolved root filesystem symlinks such as `/dev/mapper/...`
* Took tab characters in `/etc/fstab` into account

## Contributing

Contributions are welcome. Feel free to fork the repository and submit a pull
request.

## License

This project is licensed under the GNU General Public License, version 3 or
later (`GPL-3.0-or-later`). See the `COPYING` file for details.

## Authors

The Overlay-Root Debian package is maintained by m4dm4x1337. The initramfs
script is based on earlier work by Axel Heider, Pascal Rosin, Sebastian P., and
Nicholas A. Schembri. See `debian/copyright` and the script header for details.

## Donations 🥺

❤️ Please donate ❤️

![QR code for donations](https://raw.githubusercontent.com/m4dm4x1337/tor-router-gnome/master/tor-router-gnome/usr/share/pixmaps/tor-router-gnome-donation.png)

This project is open source and maintained in the author's spare time. If it is
useful to you, please consider supporting its continued development with a
donation. Thank you!

[bitcoin:bc1q9ha0l0tt7dghcpgext8jppejandefeshcukpxx](bitcoin:bc1q9ha0l0tt7dghcpgext8jppejandefeshcukpxx)
