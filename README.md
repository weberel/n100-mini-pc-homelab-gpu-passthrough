# n100-mini-pc-homelab-gpu-passthrough
This is a guide for setting up a commonly avaliable n100 based mini pc as a home server with gpu passthorugh to a virtual machine for hdmi and sound output.

## General Setup
The general idea of the project is to us a budget mini pc as a media device that is connected to a TV to watch media. Additionally the efficient mini pc is used as a homeserver running services like nextcloud, open media vault, piehole or similar. This is achived using the virtualization platform Proxmox. Proxmox is installed on the mini pc, where a virtual machine running linux (or whatever you want) is used to watch media. To get video and sound over hdmi the gpu must be passed through to the vm, which is the primary challenge and why i decided to write this guide. Naturally because we run proxmox any other virtual machine or container can be run with what every servies you need.

### Hardware 
As the envisiond system could be achived with thousands of different computing devices I won't go into detail about my hardware choice. I chose the following mini pc avaliable from aliexpress with the following specs, mostly due to its low price, efficiency and adequate computing power. It also features a m.2 2242 sata and a m.2 2280 pcie slot, which i plan to use for future storage expansion using a jmb585 M.2 to sata adapter.

GMKtec Nuc G3:
- CPU: N100, 4 core, 3.40 GHz, Integrated Graphics
- RAM: 16GB, ddr4, 3200mt/s
- M.2 2242 Sata: 512GB SSD , used for operating system
- M.2 2280 pcie: 512GB SSD, used for storage (slot for future jmb585 adapter)

My TV is very old and does not have any smart features, but does have multiple HDMI inputs.

## Step-by-step
This step-by-step guid assumes you have Proxmox instaleld and set up on you device. Minimal terminal knowlegde is required. Most operations are done thorugh the proxmox webinterface. For the specific device this guide uses, all nececarry bios setings were already enabled. As this is system specific I cannot list all possible seting to enable. You should also know how to create a vm running you desired operating system in proxmox.

### 1. Get necesary ROM
First of all we need a rom file from https://github.com/d3hl/ms01-igpu-passthrough. 

For the n100 compatible file we can use the following commands to download and copy the file to where it is necessary:
```bash
# Download gen12_igd.rom
wget https://github.com/d3hl/ms01-igpu-passthrough/raw/main/gen12_igd.rom

# Move the ROM files to where Proxmox expects them
sudo mv gen12_igd.rom /usr/share/kvm/
```

### 2. Blacklist i915 Driver
We need to block the host operating system Proxmox from claiming and using the integrated graphics, we do this by blocking the i915 driver.

```bash
# Add i915 to the blacklist
echo "blacklist i915" >> /etc/modprobe.d/pve-blacklist.conf

# Verify it was added
cat /etc/modprobe.d/pve-blacklist.conf
```

### 3. Bind vfio-pci to Your GPU
Next the gpu must be bound to a driver for virtualization. First we get the gpu device ID, 8086:46d1 in my case. Then we bind it to the driver:

```bash
# Get the specific PCI ID (vendor:device format)
lspci -nn | grep -E "VGA|Display"

# Create/edit the vfio configuration file
nano /etc/modprobe.d/vfio.conf

# Add this line (replace 8086:46d1 with your actual PCI ID):
options vfio-pci ids=8086:46d1
```

Now we need to load the required VFIO modules:
```bash
# Edit the modules file
nano /etc/modules

# Add these lines:
vfio
vfio_iommu_type1
vfio_pci
vfio_virqfd
```
Now we update and reboot:

```bash
# Update the initial RAM filesystem
update-initramfs -u

# Reboot the system
reboot
```

### 4. Grub configuration
We also need to configure the GRUB bootloader configuration to enable IOMMU.
```bash
# Edit GRUB configuration
nano /etc/default/grub

# Replace the existing file with this
GRUB_DEFAULT=0
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null |>
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on i>
GRUB_CMDLINE_LINUX=""
```
After changing the file update grub and reboot:

```bash
# Update GRUB and reboot
update-grub
reboot
```

### 5. Create VM and install OS
Create the VM using the iso of the desired operating system, in my case this is Pop OS! Most settings in creation are not important, as we will configure them later, however we will use the following:
- bios: ovmf
- CPU: host
- machine: i440
- 2 cores and 8GB of ram, disable balloning.

Now we install the operating system. We do not setup up proper gpu passthorugh for the VM just now, as this hinders us from using the proxmox webinterface, which in turn might leave us with no screen at all before we install OS.

### 6. Configure VM
With the OS installed through the web interface, we can go on to configure the VM using the following:

After creating the VM use the pve shell to configure the VM by using its ID, in my case 1001. Open the document and replace all lines with the following:

```bash
#open config file
nano /etc/pve/qemu-server/1001.conf

#replace relevant lines until config looks like this:
args: -set device.hostpci0.addr=02.0 -set device.hostpci0.x-igd-gms=0x2 -set device.hostpci0.x-igd-opregion=on
balloon: 0
bios: ovmf
boot: order=scsi0;ide2;net0
cores: 3
cpu: host
efidisk0: local-lvm:vm-1001-disk-0,efitype=4m,pre-enrolled-keys=0,size=4M
hostpci0: 0000:00:02.0,legacy-igd=1,romfile=gen12_igd.rom
hostpci1: 0000:00:1f.3
ide2: local:iso/pop-os_22.04_amd64_intel_55.iso,media=cdrom,size=2728112K
machine: pc-i440fx-8.0
memory: 8192
meta: creation-qemu=9.2.0,ctime=1754164308
name: tv-vm
net0: virtio=BC:24:11:00:DE:04,bridge=vmbr0,firewall=1
numa: 0
onboot: 1
ostype: l26
scsi0: local-lvm:vm-1001-disk-1,iothread=1,size=128G
scsihw: virtio-scsi-single
smbios1: uuid=958796f2-e531-4fc9-b766-505d35bb681c
sockets: 1
usb0: host=3-8
usb1: host=3-3
usb2: host=3-2
usb3: host=3-4,usb3=1
vga: none
vmgenid: 2131e419-443d-4d84-9a59-3d3ca4f04384
```

name, cores, memory, usb devices, vmgenid can be different. Obviously if your hadware or operating system is different you might need a slightly differen file.

### 7. Done
After this reboot the system and start the VM. The OS should show up on your connected monitor or TV and HDMI sound should be a visible as audio source.

```bash
# Reboot
reboot
```

## Disclamers
This is just what wored for me. I haven't tested this extensively. etc.