---
title: PCI passthrough via OVMF
source: https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#4c/2t_AMD_CPU_example_(Before_ComboPi_AGESA_bios_update)
kind: external
domain: compute
original_date: 2026-05-15
fetched_at: 2026-05-16
bookmark_title: PCI passthrough via OVMF - ArchWiki
tags: [external, compute]
---

> [!info] 外部文章 · 自动导入
> 来源：[wiki.archlinux.org](https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#4c/2t_AMD_CPU_example_(Before_ComboPi_AGESA_bios_update))
> 原始日期：2026-05-15
> 抓取日期：2026-05-16

# PCI passthrough via OVMF

# PCI passthrough via OVMF

The Open Virtual Machine Firmware (OVMF) is a project to enable UEFI support for virtual machines. Starting with Linux 3.9 and recent versions of QEMU, it is now possible to passthrough a graphics card, offering the virtual machine native graphics performance which is useful for graphic-intensive tasks.

Provided you have a desktop computer with a spare GPU you can dedicate to the host (be it an integrated GPU or an old OEM card, the brands do not even need to match) and that your hardware supports it (see #Prerequisites), it is possible to have a virtual machine of any OS with its own dedicated GPU and near-native performance. For more information on techniques see the background presentation (pdf).

## Prerequisites

A VGA Passthrough relies on a number of technologies that are not ubiquitous as of today and might not be available on your hardware. You will not be able to do this on your machine unless the following requirements are met:

- Your CPU must support hardware virtualization (for kvm) and IOMMU (for the passthrough itself).
- List of compatible Intel CPUs (Intel VT-x and Intel VT-d).
- All AMD CPUs from the Bulldozer generation and up (including Zen) should be compatible.

- Your motherboard must also support IOMMU.
- Both the chipset and the BIOS must support it. It is not always easy to tell at a glance whether or not this is the case, but there is a fairly comprehensive list on the matter on the Xen wiki as well as Wikipedia:List of IOMMU-supporting hardware.

- Your guest GPU ROM must support UEFI.
- If you can find any ROM in this list that applies to your specific GPU and is said to support UEFI, you are generally in the clear. All GPUs from 2012 and later should support this, as Microsoft made UEFI a requirement for devices to be marketed as compatible with Windows 8.


You will probably want to have a spare monitor or one with multiple input ports connected to different GPUs (the passthrough GPU will not display anything if there is no screen plugged in and using a VNC or Spice connection will not help your performance), as well as a mouse and a keyboard you can pass to your virtual machine. If anything goes wrong, you will at least have a way to control your host machine this way.

**Tip**You can do steps 1-3 with gpu-passthrough-manager

AUR. This is a simple graphical interface that lets you choose what graphics drivers you want to load on your system. This can also set up IOMMU and adds VFIO modules, but is limited to GRUB only. Select the graphics and audio devices you want to passthrough and then select LOAD VFIO. Reboot when prompted and then continue to step 4.

## Setting up IOMMU

**Note**

- IOMMU is a generic name for Intel VT-d and AMD-Vi.
- VT-d stands for
*Intel Virtualization Technology for Directed I/O*and should not be confused with VT-x*Intel Virtualization Technology*. VT-x allows one hardware platform to function as multiple “virtual” platforms while VT-d improves security and reliability of the systems and also improves performance of I/O devices in virtualized environments.

Using IOMMU opens to features like PCI passthrough and memory protection from faulty or malicious devices, see Wikipedia:Input-output memory management unit#Advantages and Memory Management (computer programming): Could you explain IOMMU in plain English?.

### Enabling IOMMU

Ensure that AMD-Vi or Intel VT-d is supported by your CPU and enabled in the BIOS settings. These options typically appear alongside other CPU features, which may be located in an overclocking-related menu. They may be listed under their actual names ("VT-d" or "AMD-Vi") or in more ambiguous terms such as "Virtualization Technology," which may or may not be explained in the motherboard manual.

Use dmesg to check whether IOMMU is enabled:

# dmesg | grep -i IOMMU

[ 0.000000] ACPI: DMAR 0x00000000BDCB1CB0 0000B8 (v01 INTEL BDW 00000001 INTL 00000001) [ 0.000000] Intel-IOMMU: enabled [ 0.028879] dmar: IOMMU 0: reg_base_addr fed90000 ver 1:0 cap c0000020660462 ecap f0101a [ 0.028883] dmar: IOMMU 1: reg_base_addr fed91000 ver 1:0 cap d2008c20660462 ecap f010da [ 0.028950] IOAPIC id 8 under DRHD base 0xfed91000 IOMMU 1 [ 0.536212] DMAR: No ATSR found [ 0.536229] IOMMU 0 0xfed90000: using Queued invalidation [ 0.536230] IOMMU 1 0xfed91000: using Queued invalidation [ 0.536231] IOMMU: Setting RMRR: [ 0.536241] IOMMU: Setting identity map for device 0000:00:02.0 [0xbf000000 - 0xcf1fffff] [ 0.537490] IOMMU: Setting identity map for device 0000:00:14.0 [0xbdea8000 - 0xbdeb6fff] [ 0.537512] IOMMU: Setting identity map for device 0000:00:1a.0 [0xbdea8000 - 0xbdeb6fff] [ 0.537530] IOMMU: Setting identity map for device 0000:00:1d.0 [0xbdea8000 - 0xbdeb6fff] [ 0.537543] IOMMU: Prepare 0-16MiB unity mapping for LPC [ 0.537549] IOMMU: Setting identity map for device 0000:00:1f.0 [0x0 - 0xffffff] [ 2.182790] [drm] DMAR active, disabling use of stolen memory

To manually enable IOMMU support, set the correct kernel parameter depending on the type of CPU in use:

- For Intel CPUs (VT-d) set
`intel_iommu=on`

, unless your kernel sets the`CONFIG_INTEL_IOMMU_DEFAULT_ON`

config option. - For AMD CPUs (AMD-Vi), IOMMU support is enabled automatically if the kernel detects IOMMU hardware support from the BIOS.

### Ensuring that the groups are valid

The following script should allow you to see how your various PCI devices are mapped to IOMMU groups. If it does not return anything, you either have not enabled IOMMU support properly or your hardware does not support it.

#!/bin/bash shopt -s nullglob for g in $(find /sys/kernel/iommu_groups/* -maxdepth 0 -type d | sort -V); do echo "IOMMU Group ${g##*/}:" for d in $g/devices/*; do echo -e "\t$(lspci -nns ${d##*/})" done; done;

Example output:

IOMMU Group 1: 00:01.0 PCI bridge: Intel Corporation Xeon E3-1200 v2/3rd Gen Core processor PCI Express Root Port [8086:0151] (rev 09) IOMMU Group 2: 00:14.0 USB controller: Intel Corporation 7 Series/C210 Series Chipset Family USB xHCI Host Controller [8086:0e31] (rev 04) IOMMU Group 4: 00:1a.0 USB controller: Intel Corporation 7 Series/C210 Series Chipset Family USB Enhanced Host Controller #2 [8086:0e2d] (rev 04) IOMMU Group 10: 00:1d.0 USB controller: Intel Corporation 7 Series/C210 Series Chipset Family USB Enhanced Host Controller #1 [8086:0e26] (rev 04) IOMMU Group 13: 06:00.0 VGA compatible controller: NVIDIA Corporation GM204 [GeForce GTX 970] [10de:13c2] (rev a1) 06:00.1 Audio device: NVIDIA Corporation GM204 High Definition Audio Controller [10de:0fbb] (rev a1)

An IOMMU group is the smallest set of physical devices that can be passed to a virtual machine. For instance, in the example above, both the GPU in 06:00.0 and its audio controller in 6:00.1 belong to IOMMU group 13 and can only be passed together. The frontal USB controller, however, has its own group (group 2) which is separate from both the USB expansion controller (group 10) and the rear USB controller (group 4), meaning that any of them could be passed to a virtual machine without affecting the others.

### Gotchas

#### Plugging your guest GPU in an unisolated CPU-based PCIe slot

Not all PCIe slots are the same. Most motherboards have PCIe slots provided by both the CPU and the PCH. Depending on your CPU, it is possible that your processor-based PCIe slot does not support isolation properly, in which case the PCI slot itself will appear to be grouped with the device that is connected to it.

IOMMU Group 1: 00:01.0 PCI bridge: Intel Corporation Xeon E3-1200 v2/3rd Gen Core processor PCI Express Root Port (rev 09) 01:00.0 VGA compatible controller: NVIDIA Corporation GM107 [GeForce GTX 750] (rev a2) 01:00.1 Audio device: NVIDIA Corporation Device 0fbc (rev a1)

This is fine so long as only your guest GPU is included in here, such as above. Depending on what is plugged in to your other PCIe slots and whether they are allocated to your CPU or your PCH, you may find yourself with additional devices within the same group, which would force you to pass those as well. If you are ok with passing everything that is in there to your virtual machine, you are free to continue. Otherwise, you will either need to try and plug your GPU in your other PCIe slots (if you have any) and see if those provide isolation from the rest or to install the ACS override patch, which comes with its own drawbacks. See #Bypassing the IOMMU groups (ACS override patch) for more information.

**Note**If they are grouped with other devices in this manner, pci root ports and bridges should neither be bound to vfio at boot, nor be added to the virtual machine.

## Isolating the GPU

In order to assign a device to a virtual machine, this device and all those sharing the same IOMMU group must have their driver replaced by a stub driver or a VFIO driver in order to prevent the host machine from interacting with them. In the case of most devices, this can be done on the fly right before the virtual machine starts.

However, due to their size and complexity, GPU drivers do not tend to support dynamic rebinding very well, so you cannot just have some GPU you use on the host be transparently passed to a virtual machine without having both drivers conflict with each other. Because of this, it is generally advised to bind those placeholder drivers manually before starting the virtual machine, in order to stop other drivers from attempting to claim it.

The following section details how to configure a GPU so those placeholder drivers are bound early during the boot process, which makes said device inactive until a virtual machine claims it or the driver is switched back. This is the preferred method, considering it has less caveats than switching drivers once the system is fully online.

**Warning**Once you reboot after this procedure, whatever GPU you have configured will no longer be usable on the host until you reverse the manipulation. Make sure the GPU you intend to use on the host is properly configured before doing this - your motherboard should be set to display using the host GPU.

Starting with Linux 4.1, the kernel includes vfio-pci. This is a VFIO driver, meaning it fulfills the same role as pci-stub did, but it can also control devices to an extent, such as by switching them into their D3 state when they are not in use.

### Binding vfio-pci via device ID

Vfio-pci normally targets PCI devices by ID, meaning you only need to specify the IDs of the devices you intend to passthrough. For the following IOMMU group, you would want to bind vfio-pci with `10de:13c2`

and `10de:0fbb`

, which will be used as example values for the rest of this section.

IOMMU Group 13: 06:00.0 VGA compatible controller: NVIDIA Corporation GM204 [GeForce GTX 970] [10de:13c2] (rev a1) 06:00.1 Audio device: NVIDIA Corporation GM204 High Definition Audio Controller [10de:0fbb] (rev a1)

**Note**

- You cannot specify which device to isolate using vendor-device ID pairs if the host GPU and the guest GPU share the same pair (i.e : if both are the same model). If this is your case, read #Using identical guest and host GPUs instead.
- If, as noted in #Plugging your guest GPU in an unisolated CPU-based PCIe slot, your pci root port is part of your IOMMU group, you
**must not**pass its ID to`vfio-pci`

, as it needs to remain attached to the host to function properly. Any other device within that group, however, should be left for`vfio-pci`

to bind with. - Binding the audio device (
`10de:0fbb`

in above's example) is optional. Libvirt is able to unbind it from the`snd_hda_intel`

driver on its own.

Providing the device IDs is done via the kernel module parameter `ids=10de:13c2,10de:0fbb`

for `vfio-pci`

.

In case of wanting to keep the HDAudio in the host it can be detached by using the kernel module parameters `gpu_bind=0`

for `snd-hda-core`

and `enable_acomp=n`

for `snd-hda-codec-hdmi`

.

You can use this bash script without kernel `vfio-pci`

ID. GPU can function in host after calling `unbind_vfio`

.

#!/bin/bash gpu="0000:06:00.0" aud="0000:06:00.1" gpu_vd="$(cat /sys/bus/pci/devices/$gpu/vendor) $(cat /sys/bus/pci/devices/$gpu/device)" aud_vd="$(cat /sys/bus/pci/devices/$aud/vendor) $(cat /sys/bus/pci/devices/$aud/device)" function bind_vfio { echo "$gpu" > "/sys/bus/pci/devices/$gpu/driver/unbind" echo "$aud" > "/sys/bus/pci/devices/$aud/driver/unbind" echo "$gpu_vd" > /sys/bus/pci/drivers/vfio-pci/new_id echo "$aud_vd" > /sys/bus/pci/drivers/vfio-pci/new_id } function unbind_vfio { echo "$gpu_vd" > "/sys/bus/pci/drivers/vfio-pci/remove_id" echo "$aud_vd" > "/sys/bus/pci/drivers/vfio-pci/remove_id" echo 1 > "/sys/bus/pci/devices/$gpu/remove" echo 1 > "/sys/bus/pci/devices/$aud/remove" echo 1 > "/sys/bus/pci/rescan" }

### Loading vfio-pci early

Since Arch's linux has vfio-pci built as a module, we need to force it to load before the graphics drivers have a chance to bind to the card. There are two methods: modprobe configuration, or adding the modules to initramfs.

**Note**When troubleshooting this step, to make sure all other conditions are fulfilled, verify the initramfs method works, since it is the stronger guarantee for isolating the card.

#### modprobe.d

This method loads `vfio`

when udev loads GPU drivers. This avoids bloating initramfs and slowing boot times unnecessarily.

/etc/modprobe.d/vfio.conf

softdep drm pre: vfio-pci

If you are passing through an NVIDIA GPU and have the proprietary NVIDIA driver installed, use the following instead:

/etc/modprobe.d/vfio.conf

softdep nvidia pre: vfio-pci

#### initramfs

**Warning**Since kernel 6.0, framebuffer output will freeze after loading VFIO modules and until GPU drivers are loaded. When using disk encryption, this will prevent the password prompt from showing. Make sure to add GPU drivers to initramfs when choosing this method. See this forum thread.

##### mkinitcpio

Add `vfio_pci`

, `vfio`

, and `vfio_iommu_type1`

to mkinitcpio:

/etc/mkinitcpio.conf

MODULES=(... vfio_pci vfio vfio_iommu_type1 ...)

**Note**

- As of kernel 6.2, the
`vfio_virqfd`

functionality has been folded into the base`vfio`

module. - If you also have another driver loaded this way for early modesetting (such as
`nouveau`

,`radeon`

,`amdgpu`

,`i915`

, etc.), all of the aforementioned VFIO modules**must**precede it. - If you are modesetting the
`nvidia`

driver, the`vfio-pci.ids`

must be embedded in the initramfs image. If given via kernel arguments, they will be read too late to take effect. Follow the instructions in #Binding vfio-pci via device ID for adding the ids to a modprobe conf file.

Also, ensure that the modconf hook is included in the HOOKS list of `mkinitcpio.conf`

:

/etc/mkinitcpio.conf

HOOKS=(... modconf ...)

Since new modules have been added to the initramfs configuration, you must regenerate the initramfs.

##### booster

Similar to mkinitcpio you need to specify modules to load early:

/etc/booster.yaml

modules_force_load: vfio_pci,vfio,vfio_iommu_type1

and then regenerate booster images.

##### dracut

Following the same idea, we need to ensure all vfio drivers are in the initramfs. Add the following file to `/etc/dracut.conf.d`

:

10-vfio.conf

force_drivers+=" vfio_pci vfio vfio_iommu_type1 "

Note that we used `force_drivers`

instead the usual `add_drivers`

option, which will ensure that the drivers are tried to be loaded early via modprobe (Dracut#Early kernel module loading).

As with mkinitcpio, you must regenerate the initramfs afterwards. See dracut for more details.

### Verifying that the configuration worked

Reboot and verify that vfio-pci has loaded properly and that it is now bound to the right devices.

# dmesg | grep -i vfio

[ 0.329224] VFIO - User Level meta-driver version: 0.3 [ 0.341372] vfio_pci: add [10de:13c2[ffff:ffff]] class 0x000000/00000000 [ 0.354704] vfio_pci: add [10de:0fbb[ffff:ffff]] class 0x000000/00000000 [ 2.061326] vfio-pci 0000:06:00.0: enabling device (0100 -> 0103)

It is not necessary for all devices (or even expected device) from `vfio.conf`

to be in *dmesg* output.
Even if a device does not appear, it might still be visible and usable in the guest virtual machine.

$ lspci -nnk -d 10de:13c2

06:00.0 VGA compatible controller: NVIDIA Corporation GM204 [GeForce GTX 970] [10de:13c2] (rev a1) Kernel driver in use: vfio-pci Kernel modules: nouveau nvidia

$ lspci -nnk -d 10de:0fbb

06:00.1 Audio device: NVIDIA Corporation GM204 High Definition Audio Controller [10de:0fbb] (rev a1) Kernel driver in use: vfio-pci Kernel modules: snd_hda_intel

## Setting up an OVMF-based guest virtual machine

OVMF is an open-source UEFI firmware for QEMU virtual machines. While it is possible to use SeaBIOS to get similar results to an actual PCI passthrough, the setup process is different and it is generally preferable to use the EFI method if your hardware supports it.

### Configuring libvirt

Libvirt is a wrapper for a number of virtualization utilities that greatly simplifies the configuration and deployment process of virtual machines. In the case of KVM and QEMU, the frontend it provides allows us to avoid dealing with the permissions for QEMU and make it easier to add and remove various devices on a live virtual machine. Its status as a wrapper, however, means that it might not always support all of the latest qemu features, which could end up requiring the use of a wrapper script to provide some extra arguments to QEMU.

Install qemu-desktop, libvirt, edk2-ovmf, and virt-manager. For the default network connection dnsmasq is required.

Follow Libvirt#Configuration to configure libvirt for use.

You may also need to activate the default libvirt network:

# virsh net-autostart default # virsh net-start default

**Note**The default libvirt network will only be listed if the virsh command is run as root.

### Setting up the guest OS

The process of setting up a virtual machine using `virt-manager`

is mostly self-explanatory, as most of the process comes with fairly comprehensive on-screen instructions.

However, you should pay special attention to the following steps:

- When the virtual machine creation wizard asks you to name your virtual machine (final step before clicking "Finish"), check the "Customize before install" checkbox.
- In the "Overview" section, set your firmware to "UEFI". If the option is grayed out, make sure that:
- Your hypervisor is running as a system session and not a user session. This can be verified by clicking, then hovering over the session in virt-manager. If you are accidentally running it as a user session, you must open a new connection by clicking "File" > "Add Connection..", then select the option from the drop-down menu station "QEMU/KVM" and not "QEMU/KVM user session".

- In the "CPUs" section, change your CPU model to "host-passthrough". If it is not in the list, you will have to either type it by hand or by using
`virt-xml`

. This will ensure that your CPU is detected properly, since it causes libvirt to expose your CPU capabilities exactly as they are instead of only those it recognizes (which is the preferred default behavior to make CPU behavior easier to reproduce). Without it, some applications may complain about your CPU being of an unknown model.*vmname*--edit --cpu host-passthrough - If you want to minimize IO overhead, it is easier to setup #Virtio disk before installing

The rest of the installation process will take place as normal using a standard QXL video adapter running in a window. At this point, there is no need to install additional drivers for the rest of the virtual devices, since most of them will be removed later on. Once the guest OS is done installing, simply turn off the virtual machine. It is possible you will be dropped into the UEFI menu instead of starting the installation upon powering your virtual machine for the first time. Sometimes the correct ISO file was not automatically detected and you will need to manually specify the drive to boot. By typing exit and navigating to "boot manager" you will enter a menu that allows you to choose between devices.

### Attaching the PCI devices

With the installation done, it is now possible to edit the hardware details in libvirt and remove virtual integration devices, such as the spice channel and virtual display, the QXL video adapter, the emulated mouse and keyboard and the USB tablet device.

**Note**When setting up Looking Glass to use the guest with the host keyboard and mouse, do not do this step just yet.

For example, remove the following sections from your XML file:

<channel type="spicevmc"> ... </channel> <input type="tablet" bus="usb"> ... </input> <input type="mouse" bus="ps2"/> <input type="keyboard" bus="ps2"/> <graphics type="spice" autoport="yes"> ... </graphics> <video> <model type="qxl" .../> ... </video>

Since that leaves you with no input devices, you may want to bind a few USB host devices to your virtual machine as well, but remember to **leave at least one mouse and/or keyboard assigned to your host** in case something goes wrong with the guest. This may be done by using `Add Hardware > USB Host Device`

.

At this point, it also becomes possible to attach the PCI device that was isolated earlier; simply click on "Add Hardware" and select the PCI Host Devices you want to passthrough. If everything went well, the screen plugged into your GPU should show the OVMF splash screen and your virtual machine should start up normally. From there, you can setup the drivers for the rest of your virtual machine.

### Video card driver virtualisation detection

Video card drivers by AMD incorporate very basic virtual machine detection targeting Hyper-V extensions. Should this detection mechanism trigger, the drivers will refuse to run, resulting in a black screen.

If this is the case, it is required to modify the reported Hyper-V vendor ID:

$ virsh editvmname

... <features> ... <hyperv> ... <vendor_id state='on' value='randomid'/> ... </hyperv> ... </features> ...

Nvidia guest drivers prior to version 465 exhibited a similar behaviour which resulted in a generic error 43 in the card's device manager status. Systems using these older drivers therefore also need the above modification. In addition, they also require hiding the KVM CPU leaf:

$ virsh editvmname

... <features> ... <kvm> <hidden state='on'/> </kvm> ... </features> ...

Note that the above steps do not equate 'hiding' the virtual machine from Windows or any drivers/programs running in the virtual machine. Also, various other issues not related to any detection mechanism referred to here can also trigger error 43.

### Passing keyboard/mouse via Evdev

If you do not have a spare mouse or keyboard to dedicate to your guest, and you do not want to suffer from the video overhead of Spice, you can setup evdev to share them between your Linux host and your virtual machine.

**Note**By default, press both left and right

`Ctrl`

keys at the same time to swap control between the host and the guest.
You can change this hotkeys. You need to set `grabToggle`

variable to one of available combination `ctrl-ctrl`

, `alt-alt`

, `shift-shift`

, `meta-meta`

, `scrolllock`

or `ctrl-scrolllock`

for your keyboard.
More information: https://github.com/libvirt/libvirt/blob/master/docs/formatdomain.rst#input-devices

First, find your keyboard and mouse devices in `/dev/input/by-id/`

. Only devices with `event`

in their name are valid. You may find multiple devices associated to your mouse or keyboard, so try `cat /dev/input/by-id/`

and either hit some keys on the keyboard or wiggle your mouse to see if input comes through, if so you have got the right device. Now add those devices to your configuration:
*device_id*

$ virsh editvmname

... <devices> ... <input type='evdev'> <source dev='/dev/input/by-id/MOUSE_NAME'/> </input> <input type='evdev'> <source dev='/dev/input/by-id/KEYBOARD_NAME' grab='all' repeat='on' grabToggle='ctrl-ctrl'/> </input> ... </devices>

Replace `MOUSE_NAME`

and `KEYBOARD_NAME`

with your device path. Now you can startup the guest OS and test swapping control of your mouse and keyboard between the host and guest by pressing both the left and right control keys at the same time.

You may also consider switching from PS/2 to Virtio inputs in your configurations. Add these two devices:

$ virsh editvmname

... <input type='mouse' bus='virtio'/> <input type='keyboard' bus='virtio'/> ...

The virtio input devices will not actually be used until the guest drivers are installed. QEMU will continue to send key events to the PS2 devices until it detects the virtio input driver initialization. Note that the PS2 devices cannot be removed as they are an internal function of the emulated Q35/440FX chipsets.

### Gotchas

#### Using a non-EFI image on an OVMF-based virtual machine

The OVMF firmware does not support booting off non-EFI mediums. If the installation process drops you in a UEFI shell right after booting, you m

[... 内容超长，已截断；完整原文见 source URL ...]
