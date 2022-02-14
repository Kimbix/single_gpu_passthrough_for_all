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
In the file "/etc/default/grub", in the line that contains GRUB_CMDLINE_LINUX_DEFAULT="...", add the following:
  - amd_iommu=on / intel_iommu=on
  - iommu=pt
  - video=efifb:off (Helps AMD users, adding it is recommended, but not requiered)

#### Manjaro / Ubuntu / Linux Mint -> Update GRUB with: "sudo update-grub"
#### Arch Linux / Gentoo Linux (GRUB) -> Update GRUB with: "sudo grub-mkconfig -o /boot/grub/grub.cfg"
#### OpenSUSE -> Update GRUB with: "sudo grub2-mkconfig -o /boot/grub2/grub.cfg"

## POP_OS Users
In the file "/boot/efi/loader/entries/Pop_OS-current.conf", in the line that contains options root=UUID=, add the following:
  - amd_iommu=on / intel_iommu=on
  - iommu=pt
  - video=efifb:off (Helps AMD users, adding it is recommended, but not requiered)

#### Update GRUB with "sudo bootctl update"

## Fedora Users
In the file "/etc/default/grub", in the line that contains GRUB_CMDLINE_LINUX="...", add the following:
  - amd_iommu=on / intel_iommu=on
  - iommu=pt
  - video=efifb:off (Helps AMD users, adding it is recommended, but not requiered)

#### Update GRUB with "sudo grub2-mkconfig -o /boot/efi/EFI/fedora/grub.cfg"

## ALL USERS
Reboot

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

## Arch Linux / Manjaro
Install the following packages:
```
sudo pacman -S virt-manager qemu vde2 ebtables iptables-nft nftables dnsmasq bridge-utils ovmf
```
#### NOTE: If the following message appears, select Y.
```
:: iptables and iptables-nft are in conflict. Remove iptables? [y/N]
```

## Fedora
Install the following package group:
```
sudo dnf install @virtualization
```

## POP_OS / Ubuntu / Linux Mint
Install the following packages:
```
sudo apt install qemu-kvm libvirt-clients libvirt-daemon-system bridge-utils virt-manager ovmf
```

## OpenSUSE
Install the following packages:
```
sudo zypper in libvirt libvirt-client libvirt-daemon virt-manager virt-install virt-viewer qemu qemu-kvm qemu-ovmf-x86_64 qemu-tools
```

## ALL USERS
### Editing the libvirt configuration files
In the following file "/etc/libvirt/libvirt.conf", look for the following lines and remove the "#" from them:
```
#unix_sock_group = "libvirt"
#unix_sock_rw_perms = "0770"
```
If you want detailed logs, in the same file, add the following lines at the end of the file:
```
log_filters="1:qemu"
log_outputs="1:file:/var/log/libvirt/libvirtd.log"
```
Now, we are going to add our user to the libvirt group, to do so, execute the following command:
```
sudo usermod -a -G libvirt $(whoami)
```
We can verify that we we're added to the group if the output of the following command contains libvirt:
```
sudo groups $(whoami)
```
Once we have edited the configuration files and added our user to the libvirt group, we have to start / restart libvirtd and make it autostart on boot:
```
systemd:
sudo systemctl start libvirtd (If service wasn't started already)
sudo systemctl enable libvirtd

OpenRC:
rc-service libvirtd start (If service wasn't started already)
rc-service add libvirtd default
```

### Editing QEMU configuration files
In the file "/etc/libvirt/qemu.conf", look for the following lines, remove the "#" from them, and add the requested information:
```
#user = "root" 	-> user = "[your username]"
#group = "root" -> group = "[your username]"
```
Once the file has been edited, restart the libvirtd service with the following command:
```
systemd:
sudo systemctl restart libvirtd

OpenRC:
sudo rc-service libvirtd restart
```

#### NOTE: For specific use in distros like Ubuntu / Linux Mint / POP_OS, add yourself to the kvm group with the following command:
```
sudo usermod -a -G kvm $(whoami)
```
We can verify that we we're added to the group if the output of the following command contains kvm:
```
sudo groups $(whoami)
```

### Enabling the virtual machine default network
To make the virtual machine network start on boot, execute the following commands:
```
sudo virsh net-autostart default
```
And to manually start it execute the following command:
```
sudo virsh net-start default
```
