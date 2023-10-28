# Overlay-Root

## Description

### Use an overlayfs on top of a read-only root filesystem

With overlay-root it is possible to provide the root file system with temporary write protection by mounting it as read-only. All write accesses will be temporarily written to the main memory (RAM) on a ramdisk.
For example, overlayroot opens up the following options:
* Safe and hardware-related testing of software without using a VM
* Shared computers that return to their initial state after a reboot 

## Installation

Download the Debian package and run this commands:

```bash
sudo dpkg -i overlay-root_3.0.0_all.deb || sudo apt-get -f install
```

## Usage

Reboot to activate the overlay root filesystem.

## Contributing

Contributions are welcome! Feel free to fork this repository and submit pull requests.

## License

This project is licensed under the GPLv3 License. See the COPYING file for details.

## Author

The Tor-Router Debian package was created by m4dm4x1337.

## Donations 🥺

 ❤️ Please donate ❤️

![QR code for donations](https://raw.githubusercontent.com/m4dm4x1337/tor-router-gnome/master/tor-router-gnome/usr/share/pixmaps/tor-router-gnome-donation.png)

This project is open source, and the only income comes from the donators. If you like the project, please donate, thank you!

[bitcoin:bc1q9ha0l0tt7dghcpgext8jppejandefeshcukpxx](bitcoin:bc1q9ha0l0tt7dghcpgext8jppejandefeshcukpxx)
