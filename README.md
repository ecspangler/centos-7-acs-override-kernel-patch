# ACS Override Patch for Kernel Version 3.10.0-957.27.2.el7.acs_kernel.x86_64

A common use case in virtualization is the need to passthrough a PCIe device to a guest VM.

https://lkml.org/lkml/2013/5/30/513

Due to file differences between the kernel version used in Alex Williamson's original patch and the kernel version that ships with CentOS 7, a new patch is needed to apply the ACS override to CentOS 7 (without changing the CentOS kernel version).

This repository provides patch files and application instructions for the kernel version 3.10.0-957.27.2.el7.acs_kernel.x86_64.

Note: The ACS override patch is typically considered a 'last resort' for PCIe passthrough if other methods don't work. It's not suitable for production systems, but may be useful for dev environments or homelabs.


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

### Add ACS Override Patch Files and Build Custom Kernel



### Boot System Using the New Custom Kernel







