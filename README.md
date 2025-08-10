# N100 Mini PC Homelab with GPU Passthrough Guide
This is a guide for setting up a commonly available N100-based mini PC as a home server with GPU passthrough to a virtual machine for HDMI and audio output.

## General Setup
The general idea of the project is to use a budget mini PC as a media device that is connected to a TV to watch media. Additionally, the efficient mini PC is used as a homeserver running services like Nextcloud, OpenMediaVault, Pi-hole, or similar. This is achieved using the virtualization platform Proxmox. Proxmox is installed on the mini PC, where a virtual machine running Linux (or any desired operating system) is used to watch media. To get video and audio over HDMI, the GPU must be passed through to the VM, which is the primary challenge and why I decided to write this guide. Naturally, because we run Proxmox, any other virtual machine or container can be run with whatever services you need.

### Hardware 
As the envisioned system could be achieved with thousands of different computing devices, I won't go into detail about my hardware choice. I chose the following mini PC available from AliExpress with the following specs, mostly due to its low price, efficiency, and adequate computing power. It also features an M.2 2242 SATA and an M.2 2280 PCIe slot, which I plan to use for future storage expansion using a JMB585 M.2 to SATA adapter.

GMKtec Nuc G3:
- CPU: N100, 4 core, 3.40 GHz, integrated graphics
- RAM: 16GB, DDR4, 3200 MT/s
- M.2 2242 SATA: 512GB SSD, used for operating system
- M.2 2280 PCIe: 512GB SSD, used for storage (slot for future JMB585 adapter)

My TV is very old and does not have any smart features, but does have multiple HDMI inputs.

## Step-by-step
This step-by-step guide assumes you have Proxmox installed and set up on your device. Minimal terminal knowledge is required. Most operations are done through the Proxmox web interface. For the specific device this guide uses, all necessary BIOS settings were already enabled. As this is system-specific, I cannot list all possible settings to enable. You should also know how to create a VM running your desired operating system in Proxmox.

### 1. Get Necessary ROM
First, we need a ROM file from https://github.com/d3hl/ms01-igpu-passthrough. 

For the N100-compatible file, we can use the following commands to download and copy the file to where it is necessary:
```bash
# Download gen12_igd.rom
wget https://github.com/d3hl/ms01-igpu-passthrough/raw/main/gen12_igd.rom

# Move the ROM files to where Proxmox expects them
sudo mv gen12_igd.rom /usr/share/kvm/
```

### 2. Blacklist i915 Driver
We need to block the host operating system Proxmox from claiming and using the integrated graphics. We do this by blacklisting the i915 driver.

```bash
# Add i915 to the blacklist
echo "blacklist i915" >> /etc/modprobe.d/pve-blacklist.conf

# Verify it was added
cat /etc/modprobe.d/pve-blacklist.conf
```

### 3. Bind vfio-pci to Your GPU
Next, the GPU must be bound to a driver for virtualization. First, we get the GPU device ID (8086:46d1 in my case). Then we bind it to the driver:

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

### 4. GRUB Configuration
We also need to configure the GRUB bootloader to enable IOMMU (Input-Output Memory Management Unit) for GPU passthrough.
```bash
# Edit GRUB configuration
nano /etc/default/grub

# Replace the existing file with this
GRUB_DEFAULT=0
GRUB_TIMEOUT=5
GRUB_DISTRIBUTOR=`lsb_release -i -s 2> /dev/null || echo Debian`
GRUB_CMDLINE_LINUX_DEFAULT="quiet intel_iommu=on iommu=pt pcie_acs_override=downstream,multifunction initcall_blacklist=sysfb_init video=simplefb:off video=vesafb:off video=efifb:off video=vesa:off disable_vga=1 vfio_iommu_type1.allow_unsafe_interrupts=1 kvm.ignore_msrs=1 modprobe.blacklist=radeon,nouveau,nvidia,nvidiafb,nvidia-gpu,snd_hda_intel,snd_hda_codec_hdmi,i915"
GRUB_CMDLINE_LINUX=""
```
After changing the file, update GRUB and reboot:

```bash
# Update GRUB and reboot
update-grub
reboot
```

### 5. Create VM and install OS
Create the VM using the ISO of the desired operating system, in my case, this is Pop!_OS. Most settings during creation are not important, as we will configure them later. However, we will use the following:
- BIOS: OVMF (UEFI)
- CPU: host
- Machine: i440fx
- 2 cores and 8GB of RAM, disable ballooning

Now we install the operating system. We do not set up proper GPU passthrough for the VM just yet, as this prevents us from using the Proxmox web interface, which in turn might leave us with no screen output at all before we install the OS.

### 6. Configure VM
With the OS installed through the web interface, we can go on to configure the VM using the following:

After creating the VM, use the PVE shell to configure the VM by using its ID (in my case, 1001). Open the configuration file and replace all lines with the following:

```bash
#open config file
nano /etc/pve/qemu-server/1001.conf

#replace relevant lines until config looks like this:
args: -set device.hostpci0.addr=02.0 -set device.hostpci0.x-igd-gms=0x2 -set device.hostpci0.x-igd-opregion=on
balloon: 0
bios: ovmf
boot: order=scsi0;ide2;net0
cores: 2
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


Note: name, cores, memory, USB devices, and vmgenid can be different for your setup. Obviously, if your hardware or operating system is different, you might need a slightly different configuration. The USB passthrough lines (usb0-usb3) are optional and specific to my setup. To include specific usb devices, the easiest option is to add them in the hardware section of the VM.

### 7. Done
After this, reboot the system and start the VM. The OS should show up on your connected monitor or TV, and HDMI audio should be visible as an audio source.

```bash
# Reboot
reboot
```

## Disclaimer

### Important Notice

This guide documents what worked for me on my specific hardware configuration. I put together different parts of other guides. I am not an expert in virtualization or GPU passthrough. I'm just sharing my experience. Your results may vary.

**Never run terminal commands you don't understand.** Always research what a command does before executing it. These modifications can potentially:
- Make your system unbootable
- Cause data loss
- Require a complete reinstall if something goes wrong

### Before You Start

- **Make backups** of any important data
- This guide worked on my specific N100 mini PC, but different hardware may need different configurations
- GPU passthrough can be tricky and doesn't work on all hardware
- Have a Proxmox installation USB ready in case you need to start over

### No Warranty

I provide this guide as-is with no guarantees. I am not responsible for any damage to your hardware or data loss that may occur. By following this guide, you accept all risks.

If you run into issues, I cannot provide support. You'll need to:
- Search online forums for help
- Check the official Proxmox documentation
- Be prepared to troubleshoot on your own

### Final Note

This guide was created in 2025 and tested only on a GMKtec Nuc G3 with N100 CPU. Technology changes quickly, so some steps may become outdated.

Good luck!
