The Ultimate Single GPU Passthrough Guide

# Prerequisites
## AMD USERS
If you have an AMD CPU you must enable the following in your BIOS.

  - IOMMU
  - NX Mode
  - SVM Mode

NOTE: If you cannot find any of these configurations in your BIOS, your CPU / Motherboard might not be capable of virtualization.

## INTEL USERS
If you have an Intel CPU you must enable the following in your BIOS.
  
  - VT-d
  - VT-x

NOTE: If you cannot find any of these configurations in your BIOS, your CPU / Motherboard might not be capable of virtualization.

## ALL USERS
Make sure your system is in UEFI mode, if your system is fairly old you might want to take a look at this setting, else, you can proceed.

# CONFIGURING GRUB

## Arch Linux / Manjaro / Gentoo Linux (GRUB) / Ubuntu / Linux Mint / OpenSUSE
In the file /etc/default/grub, in the line that contains GRUB_CMDLINE_LINUX_DEFAULT="...", add the following:
  - amd_iommu=on / intel_iommu=on
  - iommu=pt
  - video=efifb:off (Helps AMD users, adding it is recommended, but not requiered)

#### Manjaro / Ubuntu / Linux Mint -> Update GRUB with: "sudo update-grub"
#### Arch Linux / Gentoo Linux (GRUB) -> Update GRUB with: "sudo grub-mkconfig -o /boot/grub/grub.cfg"
#### OpenSUSE -> Update GRUB with: "sudo grub2-mkconfig -o /boot/grub2/grub.cfg"

## POP_OS Users
In the file /boot/efi/loader/entries/Pop_OS-current.conf, in the line that contains options root=UUID=, add the following:
  - amd_iommu=on / intel_iommu=on
  - iommu=pt
  - video=efifb:off (Helps AMD users, adding it is recommended, but not requiered)

#### Update GRUB with "sudo bootctl update"

## Fedora Users
In the file /etc/default/grub, in the line that contains GRUB_CMDLINE_LINUX="...", add the following:
  - amd_iommu=on / intel_iommu=on
  - iommu=pt
  - video=efifb:off (Helps AMD users, adding it is recommended, but not requiered)

#### Update GRUB with "sudo grub2-mkconfig -o /boot/efi/EFI/fedora/grub.cfg"

#### ALL USERS -> REBOOT

# IOMMU GROUPS
Execute the following script in your terminal, copying and pasting it is fine, but if that doesn't work, place the text in a file, make it executable, and open it through your terminal.
```
#!/bin/bash
shopt -s nullglob
for g in /sys/kernel/iommu_groups/*; do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
done;
```
You're going to have to look for your graphics card in the output of the script.

The following is the output of the script in my computer:
```
Group 15:
	2d:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 14 [Radeon RX 5500/5500M / Pro 5500M] [1002:7340] (rev c5)
Group 16:
	2d:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 10 HDMI Audio[1002:ab38]
```
You have to (USUALLY) search for a VGA compatible controller and for a HDMI Audio.
#### NOTE: Several situations can happen when executing the script, I will name the ones I know about:
  - The GPU and it's sound device are in diferent IOMMU groups, this is of no importance and you should proceed normally
  -  The GPU and it's sound device are in the same IOMMU group, this is completely normal and you should give zero atention to this fact.
  -  Another device is in the IOMMU group alongside the GPU and it's audio device, if this is the case, you'll HAVE to include the other devices in the following steps.

## Take note of these values
```
Group 15:
	2d:00.0 VGA compatible controller [0300]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 14 [Radeon RX 5500/5500M / Pro 5500M] [1002:7340] (rev c5)
Group 16:
	2d:00.1 Audio device [0403]: Advanced Micro Devices, Inc. [AMD/ATI] Navi 10 HDMI Audio[1002:ab38]
```
2d:00.0 & 2d:00.1 -> These values will be important, save them.
[1002:7340] & [1002:ab38] -> These values will also be important, save them

#### NOTE: DO NOT ADD BRIDGE DEVICES

# INSTALLING REQUIRED PACKAGES
