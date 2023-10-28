# Overlay-Root
### Use an overlayfs on top of a read-only root filesystem
With overlay-root it is possible to provide the root file system with temporary write protection by mounting it as read-only. All write accesses will be temporarily written to the main memory (RAM) on a ramdisk.
For example, overlayroot opens up the following options:
* Safe and hardware-related testing of software without using a VM
* Shared computers that return to their initial state after a reboot 
