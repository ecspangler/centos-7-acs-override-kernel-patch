# ACS Override Patch for Kernel Version 3.10.0-957.27.2.el7.acs_kernel.x86_64

A common use case in virtualization is the need to passthrough a PCI or VGA device to a guest VM. However, because all devices within a single IOMMU group must be passed through at once, it can pose challenges to passthrough if there are multiple devices in the same IOMMU group.

To address this issue, Alex Williamson created a kernel patch which allows each device to be placed into its own IOMMU group, easing passthrough:
https://lkml.org/lkml/2013/5/30/513

Due to file differences between the kernel version used in Alex Williamson's original patch and the kernel version that ships with CentOS 7, a new patch is needed to apply the ACS override to CentOS 7 (without changing the CentOS kernel version).

This repository provides patch files and application instructions for the kernel version 3.10.0-957.27.2.el7.acs_kernel.x86_64.

Note: The ACS override patch is typically considered a 'last resort' for PCIe passthrough if other methods don't work. It's not suitable for production systems (and neither are custom kernels), but may be useful for dev environments or homelabs.


References and ACS Override Alternatives:

https://heiko-sieger.info/iommu-groups-what-you-need-to-consider/
https://forum.level1techs.com/t/the-pragmatic-neckbeard-3-vfio-iommu-and-pcie/111251
https://bugzilla.redhat.com/show_bug.cgi?id=1113399
https://bugzilla.redhat.com/attachment.cgi?id=913028&action=diff



## Installation

### Confirm Current IOMMU Groups

Run the iommu-groups.sh helper script to view the current IOMMU groups:
```
# ./iommu-groups.sh
...
IOMMU Group 1:
	00:01.0 PCI bridge [0604]: Intel Corporation Xeon E3-1200 v5/E3-1500 v5/6th Gen Core Processor PCIe Controller (x16) [8086:1901] (rev 05)
	00:01.1 PCI bridge [0604]: Intel Corporation Xeon E3-1200 v5/E3-1500 v5/6th Gen Core Processor PCIe Controller (x8) [8086:1905] (rev 05)
	01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP106 [GeForce GTX 1060 6GB] [10de:1c03] (rev a1)
	01:00.1 Audio device [0403]: NVIDIA Corporation GP106 High Definition Audio Controller [10de:10f1] (rev a1)
	02:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP106 [GeForce GTX 1060 6GB] [10de:1c03] (rev a1)
	02:00.1 Audio device [0403]: NVIDIA Corporation GP106 High Definition Audio Controller [10de:10f1] (rev a1)
...
```

In the above example, two GPUs and two PCI bridges are in the same group.


### Download CentOS Kernel Source

Steps adapted from the below:
https://wiki.centos.org/HowTos/I_need_the_Kernel_Source
https://wiki.centos.org/HowTos/Custom_Kernel

As root, install these supporting packages:
```
# yum install asciidoc audit-libs-devel bash binutils binutils-devel bison bzip2 diffutils elfutils-devel
# yum install elfutils-libelf-devel findutils flex gawk gcc gnupg gzip hmaccalc m4 make module-init-tools
# yum install net-tools newt-devel patch patchutils perl perl-ExtUtils-Embed python python-devel
# yum install redhat-rpm-config rpm-build sh-utils tar xmlto zlib-devel
# yum groupinstall "Development Tools"
# yum install ncurses-devel
# yum install hmaccalc zlib-devel binutils-devel elfutils-libelf-devel
```

Switch to an ordinary non-root user and create the build tree:
```
$ mkdir -p ~/rpmbuild/{BUILD,BUILDROOT,RPMS,SOURCES,SPECS,SRPMS}
$ echo '%_topdir %(echo $HOME)/rpmbuild' > ~/.rpmmacros
```

Install the source package:
```
$ rpm -i http://vault.centos.org/7.6.1810/updates/Source/SPackages/kernel-3.10.0-957.27.2.el7.src.rpm 2>&1 | grep -v exist
```

Prepare the source files:
```
$ cd ~/rpmbuild/SPECS
$ rpmbuild -bp --target=$(uname -m) kernel.spec
```


### Add ACS Override Patch Files and Build Custom Kernel

Still as a non-root user, copy the currently running config to the BUILD directory:
```
$ cd ~/rpmbuild/BUILD/kernel-*/linux-*/
$ cp /boot/config-`uname -r` .config
```

Open the .config file and make sure the first line is:
```
# x86_64
```

Copy the .config file to the configs directory:
```
$ cp .config configs/kernel-3.10.0-`uname -m`.config
```

Copy configs directory to the SOURCES directory
```
$ cp configs/* ~/rpmbuild/SOURCES/
```

Change to the SPECS directory and copy the kernel.spec:
```
$ cd ~/rpmbuild/SPECS/
$ cp kernel.spec kernel.spec.distro
```

Modify the kernel.spec to include the following:
```
%define buildid .acs_kernel
```
```
Patch40000: kernel-params-acs-override.patch
Patch40001: quirk-acs-override.patch
```
```
ApplyOptionalPatch kernel-params-acs-override.patch
ApplyOptionalPatch quirk-acs-override.patch
```

In the example kernel.spec in this repo, these updates are at lines 8, 452-453 and 782-783.

Copy the kernel-params-acs-override.patch and quirk-acs-override.patch patch files from this repo into the ~/rpmbuild/SOURCES directory.

Build the kernel. Note: Do not build the kernel as root:
```
$ rpmbuild -bb --without kabichk --target=`uname -m` kernel.spec 2> build-err.log | tee build-out.log
```

Switch to root to install the kernel.

Change to the RPMS directory which contains the output of the build (replace <user> with the user used to build the kernel):
```
# cd /home/<user>/rpmbuild/RPMS/`uname -m`/
```

Install the new kernel (do not overwrite the existing kernel):
```
# yum localinstall kernel-*.rpm
```

Check that the new kernel is available for boot:
```
# awk -F\' '$1=="menuentry " {print i++ " : " $2}' /etc/grub2.cfg
0 : CentOS Linux (3.10.0-957.27.2.el7.acs_kernel.x86_64.debug) 7 (Core)
1 : CentOS Linux (3.10.0-957.27.2.el7.acs_kernel.x86_64) 7 (Core)
2 : CentOS Linux (3.10.0-957.27.2.el7.x86_64) 7 (Core)
3 : CentOS Linux (3.10.0-957.el7.x86_64) 7 (Core)
4 : CentOS Linux (0-rescue-ace3cce7f4a5477fabc3642f08598bec) 7 (Core)
```


### Boot System Using the New Custom Kernel

Based on the menu number of the custom kernel, set this to the default boot:
```
# grub2-set-default 1
```

Add the pcie_acs_override=downstream flag to GRUB_CMDLINE_LINUX (other options are multifunction and id:nnnn:nnnn, but downstream should be enough):
```
# vi /etc/sysconfig/grub
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR="$(sed 's, release .*$,,g' /etc/system-release)"
GRUB_DEFAULT=saved
GRUB_DISABLE_SUBMENU=true
GRUB_TERMINAL_OUTPUT="console"
GRUB_CMDLINE_LINUX="crashkernel=auto rd.lvm.lv=centos/root rd.lvm.lv=centos/swap rhgb quiet rd.driver.pre=vfio-pci intel_iommu=on iommu=pt nouveau.modeset=0 rd.driver.blacklist=nouveau modprobe.blacklist=nouveau video=vesafb:off,efifb:off pcie_acs_override=downstream"
GRUB_DISABLE_RECOVERY="true"
```

Regenerate the grub config:
```
# grub2-mkconfig -o /etc/grub2.cfg
```

Reboot:
```
# reboot
```


### Confirm Devices Separated

Run the iommu-groups.sh script again to confirm devices are now separated as expected:
```
# ./iommu-groups.sh
IOMMU Group 0:
	00:00.0 Host bridge [0600]: Intel Corporation Xeon E3-1200 v6/7th Gen Core Processor Host Bridge/DRAM Registers [8086:591f] (rev 05)
IOMMU Group 1:
	00:01.0 PCI bridge [0604]: Intel Corporation Xeon E3-1200 v5/E3-1500 v5/6th Gen Core Processor PCIe Controller (x16) [8086:1901] (rev 05)
IOMMU Group 10:
	00:1b.4 PCI bridge [0604]: Intel Corporation 200 Series PCH PCI Express Root Port #21 [8086:a2eb] (rev f0)
IOMMU Group 11:
	00:1c.0 PCI bridge [0604]: Intel Corporation 200 Series PCH PCI Express Root Port #1 [8086:a290] (rev f0)
IOMMU Group 12:
	00:1c.2 PCI bridge [0604]: Intel Corporation 200 Series PCH PCI Express Root Port #3 [8086:a292] (rev f0)
IOMMU Group 13:
	00:1c.4 PCI bridge [0604]: Intel Corporation 200 Series PCH PCI Express Root Port #5 [8086:a294] (rev f0)
IOMMU Group 14:
	00:1c.5 PCI bridge [0604]: Intel Corporation 200 Series PCH PCI Express Root Port #6 [8086:a295] (rev f0)
IOMMU Group 15:
	00:1c.6 PCI bridge [0604]: Intel Corporation 200 Series PCH PCI Express Root Port #7 [8086:a296] (rev f0)
IOMMU Group 16:
	00:1c.7 PCI bridge [0604]: Intel Corporation 200 Series PCH PCI Express Root Port #8 [8086:a297] (rev f0)
IOMMU Group 17:
	00:1d.0 PCI bridge [0604]: Intel Corporation 200 Series PCH PCI Express Root Port #9 [8086:a298] (rev f0)
IOMMU Group 18:
	00:1f.0 ISA bridge [0601]: Intel Corporation 200 Series PCH LPC Controller (Z270) [8086:a2c5]
	00:1f.2 Memory controller [0580]: Intel Corporation 200 Series/Z370 Chipset Family Power Management Controller [8086:a2a1]
	00:1f.3 Audio device [0403]: Intel Corporation 200 Series PCH HD Audio [8086:a2f0]
	00:1f.4 SMBus [0c05]: Intel Corporation 200 Series/Z370 Chipset Family SMBus Controller [8086:a2a3]
IOMMU Group 19:
	00:1f.6 Ethernet controller [0200]: Intel Corporation Ethernet Connection (2) I219-V [8086:15b8]
IOMMU Group 2:
	00:01.1 PCI bridge [0604]: Intel Corporation Xeon E3-1200 v5/E3-1500 v5/6th Gen Core Processor PCIe Controller (x8) [8086:1905] (rev 05)
IOMMU Group 20:
	01:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP106 [GeForce GTX 1060 6GB] [10de:1c03] (rev a1)
	01:00.1 Audio device [0403]: NVIDIA Corporation GP106 High Definition Audio Controller [10de:10f1] (rev a1)
IOMMU Group 21:
	02:00.0 VGA compatible controller [0300]: NVIDIA Corporation GP106 [GeForce GTX 1060 6GB] [10de:1c03] (rev a1)
	02:00.1 Audio device [0403]: NVIDIA Corporation GP106 High Definition Audio Controller [10de:10f1] (rev a1)
IOMMU Group 22:
	04:00.0 USB controller [0c03]: ASMedia Technology Inc. ASM2142 USB 3.1 Host Controller [1b21:2142]
IOMMU Group 23:
	07:00.0 Network controller [0280]: Qualcomm Atheros AR93xx Wireless Network Adapter [168c:0030] (rev 01)
IOMMU Group 3:
	00:02.0 VGA compatible controller [0300]: Intel Corporation HD Graphics 630 [8086:5912] (rev 04)
IOMMU Group 4:
	00:08.0 System peripheral [0880]: Intel Corporation Xeon E3-1200 v5/v6 / E3-1500 v5 / 6th/7th Gen Core Processor Gaussian Mixture Model [8086:1911]
IOMMU Group 5:
	00:14.0 USB controller [0c03]: Intel Corporation 200 Series/Z370 Chipset Family USB 3.0 xHCI Controller [8086:a2af]
IOMMU Group 6:
	00:16.0 Communication controller [0780]: Intel Corporation 200 Series PCH CSME HECI #1 [8086:a2ba]
IOMMU Group 7:
	00:17.0 SATA controller [0106]: Intel Corporation 200 Series PCH SATA controller [AHCI mode] [8086:a282]
IOMMU Group 8:
	00:1b.0 PCI bridge [0604]: Intel Corporation 200 Series PCH PCI Express Root Port #17 [8086:a2e7] (rev f0)
IOMMU Group 9:
	00:1b.2 PCI bridge [0604]: Intel Corporation 200 Series PCH PCI Express Root Port #19 [8086:a2e9] (rev f0)
```


