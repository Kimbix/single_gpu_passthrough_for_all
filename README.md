The Ultimate Single GPU Passthrough Guide

# NOTES
  - This guide is made for people that want a gaming VM but don't really have THAT much experience with the topic, so I'm making it as DETAILED as possible to make sure every nook and cranny is perfect and every option I have come across is presented.
  - This guide is written from the perspective of an Arch User, if any of the steps differ in your distribution, or I have missed something in your distribution, you can make an Issue and I'll take a look at it, you can also write to my Discord: Kimbix#0234.
  - If you also feel like something is poorly redacted, I am sorry, English is not my native language and you can send corrections to my Discord (Kimbix#0234), or make an Issue.
  - When a command contains brackets "[]" the content inside them are not to be taken literally and you HAVE to replace them with what is written inside them. Example: "sudo groups [your username]" I would write "sudo groups kimbix"

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

# CREATING THE VIRTUAL MACHINE IN VIRT-MANAGER
## Prerequisites
	- A Windows 10 ISO: https://www.microsoft.com/en-us/software-download/windows10ISO.
	- VirtIO Drivers: https://fedorapeople.org/groups/virt/virtio-win/direct-downloads/stable-virtio/virtio-win.iso.

## Creating the VM
  - Select the QEMU/KVM Connection.
  - Select to install the OS using local media.
  - Select Browse -> Browse Local -> Your Windows 10 ISO file.
  - Leave the RAM and CPU settings the default (we will change them later).
  - Create a disk image for the virtual machine (If using for gaming my recommended size is 128GB), you can use .qcow2 or .raw, it doesn't really affect performance that much.
  - **IMPORTANT**: Check the box "Customize configuration before install", and select Finish.

## Configuring the VM
  - Overview: Make sure the chipset is set to Q35 and the Firmware option is set to UEFIx86_64:/usr/share/edk2-ovmf/x64/OVMF_CODE.fd (You might have multiple OVMF_CODE.fd available to you, the preferred option is the one that has in it's path x64).
  - CPUs: Uncheck the box Copy host CPU configuration, and set the Model to host-passthrough; Under the topology section, check the box Manually set CPU topology, and set it to whatever your CPU is like, usually you only have 1 socket, and then multiply the Cores * Threads to get your vCPU allocation, Virt-Manager will indicate you if your configuration is invalid.
  - Memory: Allocate memory to the virtual machine (NOTE: If your memory allocation is NOT valid, the VM might not boot, the safest values are multiples of 2048).
  - Boot Options: Check the SATA CDROM 1 box and move it to the top of the list.
  - SATA Disk 1: Change the Disk bus to VirtIO; Under the Advanced options section change the Cache mode to writeback.
  - Network Section (2 computers): Change the Device model to virtio.
  - Add Hardware: Under the storage section, select device type CDROM device, and select the VirtIO drivers (Manage -> Browse Local -> VirtIO drivers).
	
#### You are now ready to boot into your VM and install windows.

## Installing Windows
Once you are inside of the Windows installation, once you are prompted to select a type of install, select Custom Install; Windows will not be able to read the virtual disk without the VirtIO drivers, so you'll have to load them manually.
Select Load Drivers and Windows will automatomatically detect the VirtIO drivers and will ask you to select one, select the w10 drivers and wait for them to install. Otherwise if Windows does NOT automatically detect the VirtIO drivers, look for them in your filesystem or check that you added the hardware properly.
Once the Windows installation media can detect the Virtual Disk, select the disk to perform the instalation and wait for the installation to complete.

## Finalizing the Windows setup
Once you're inside of Windows, install the VirtIO drivers and turn off the VM.

## Finishing touches for the VM
Once you have finished the Windows installation and the VM is turned off, you can remove any Spice Hardware, as it won't be needed anymore.

# GETTING AND PATCHING YOUR VBIOS
Now, to get the GPU working, we will have to look for it's vBIOS.

There are multiple ways to go about doing this, and I will cover every way I know of doing it:

## Downloading it from the internet
The only reliable website I have knowledge of that provides GPUs vBIOS, is TechPowerup.com, if you're planning on aquiring your vBIOS this way, you HAVE to know the EXACT model of your GPU, otherwise it will not work.

Techpowerup website: https://www.techpowerup.com/vgabios/

## Dumping it from Windows 10
If you have a way to boot into Windows 10 using your GPU (The VM doesn't work), you can use a program like CPU-Z to aquire the vBIOS.

CPU-Z website: https://www.cpuid.com/softwares/cpu-z.html

## Dumping it from Linux
If you don't want to boot into Windows 10 and don't want to download it from the internet, you can dump it directly from Linux with these programs:

NVIDIA: https://www.techpowerup.com/download/nvidia-nvflash/
AMD: https://www.techpowerup.com/download/ati-atiflash/

To dump it using these programs you will have to follow these steps:
  - Ctrl + Alt + F2 into a Terminal
  - Stop your display manager with "sudo systemctl stop [Your display manager].service" or "sudo rc-service stop [Your display manager]".
  - Unload your GPU modules with "sudo rmmod nvidia, nvidia_uvm, nvidia_modeset" or "sudo rmmod amdgpu" (I don't know if this step is necesary for AMD users, as I did not do it when dumping my vBIOS).
  - cd into the directory where the file is located and make the file executable with the following command "chmod u+x [nvflash / amdvbflash]".
  - Execute the following command to save the vBIOS in your current directory "sudo ./nvflash_linux --save test.rom" or "sudo ./ati --save test.rom".
  - Load your modules once more "sudo modprobe nvidia, nvidia_uvm, nvidia_modeset" or "sudo modprobe amdgpu" and start your display manager again "sudo systemctl start [Your display manager].service" or "sudo rc-service start [Your display manager]".
	
## Patching your vBIOS
This is a complicated (and honestly annoying) step.

First, download a HEX editor and open the rom file with it.
Using the search function of your preferred editor, search for the text "VIDEO", once you find it, remove everything before the first letter U to the left of "VIDEO".

These images might help visualize what I'm trying to say:
![image](https://user-images.githubusercontent.com/83105263/153798510-93397ab1-fb46-4953-8567-0188c23692d6.png)
![image](https://user-images.githubusercontent.com/83105263/153798518-7b265542-8fd8-4a07-80c7-fcb3d733248f.png)

#### NOTE: IF YOU CANNOT FIND VIDEO IN YOUR vBIOS, THIS MIGHT MEAN IT'S ALREADY "PATCHED", IF THE FIRST LETTER OF YOUR VBIOS IS A U, THIS MIGHT IN FACT BE THE CASE.

Save the file and proceed to the next step.

## If you use selinux (Fedora)

  - Make the following directory "sudo mkdir /var/lib/libvirt/vbios".
  - Place the rom in above directory with "sudo mv [path to your .rom file] /var/lib/libvirt/vbios/GPU.rom".
  - cd into the directory with "cd /var/lib/libvirt/vbios".
  - Add the propper permissions to the file with "sudo chmod -R 660 GPU.rom".
  - Tell the system you own the file with "sudo chown [your username:your username] GPU.rom".
  - Change your SELinux configuration with "sudo semanage fcontext -a -t virt_image_t /var/lib/libvirt/vbios/GPU.rom".
  - Change the context of the file with "sudo restorecon -v /var/lib/libvirt/vbios/GPU.rom".

## If you use anything else

  - Make the following directory "sudo mkdir /usr/share/vgabios"
  - Place the rom in above directory with "sudo mv [path to your .rom file] /usr/share/vgabios/GPU.rom".
  - cd into the directory with "cd /usr/share/vgabios".
  - Change the permissions of the file with "sudo chmod -R 660 <ROMFILE>.rom".
  - Tell the system you own the file with "sudo chown username:username <ROMFILE>.rom ".
