# linux-vm
My Arch-based linux virtual machine host setup with GPU passthrough using EndeavourOS Linux distribution. Documenting so I can remember how to set it up in the event I reformat my desktop or need to revert back manipulation.

## Reference Materials
- EndeavourOS: https://endeavouros.com/
- Arch Linux PCI Passthrough via OVMF docs: https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF
- YouTube video tutorial followed: https://www.youtube.com/watch?v=h7SG7ccjn-g

# Setup GPU Passthrough
Configuration steps to set up GPU passthrough.

### Enable IOMMU support by setting the correct kernel parameter
- Open terminal and run command: `sudo nano /etc/default/grub`
- Find `GRUB_CMDLINE_LINUX_DEFAULT`
- Input `intel_iommu=on iommu=pt` after `quiet`
- Ctrl + X → y to save → enter

### Rebuild bootloader/re-generate the grub.cfg file so IOMMU enablement comes into effect
- Run command: sudo grub-mkconfig -o /boot/grub/grub.cfg
- Restart computer with command: reboot

### Check and validate that IOMMU has been correctly enabled
- Open terminal and run command: `sudo dmesg | grep -i -e DMAR -e IOMMU`
- Should see the following message around the 5th line to confirm IOMMU is enabled: `DMAR: IOMMU enabled`

### Find out if you can hijack the GPU and retrieving its device ID (needed in order to pass it through a virtual machine)
- Run the below script:
```
shopt -s nullglob
for g in `find /sys/kernel/iommu_groups/* -maxdepth 0 -type d | sort -V`; do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
done;
```
- Copy the entire IOMMU Group where the GPU is listed (Usually in IOMMU Group 1) e.g. `IOMMU Group 1 00:01.0 PCI bridge [0604]: Intel Corporation Xeon E3-1200 v5/E3-1500 v5/6th Gen Core Processor PCIe Controller (x16) [8086:1901] (rev 07)`
- Open a text editor → save file as ‘IOMMU Group1GPU’ to desktop for reference

### Isolating the GPU - bind vfio-pci with the GPU’s hardware IDs (taken from the IOMMU Group copied from last step)
- Run command: sudo nano /etc/modprobe.d/vfio.conf
- Copy each of the device IDs in brackets near the end of each line in the IOMMU Group
- Input: `options vfio-pci ids=[DEVICEID],[DEVICEID]` where DEVICEID is specified by the GPU device IDs, e.g. `options vfio-pci ids=**10de:13c2**,**10de:0fbb**`
- Ctrl + X → ‘y’ to save → enter

### Force vfio-pci to load early before the graphics drivers have a chance to bind to the card
- Run command: `sudo nano /etc/mkinitcpio.conf`
- Find the `MODULES=””` line
- Replace the quotes `“”` with a parentheses `()`
- Inside the parentheses, input: `vfio_pci vfio vfio_iommu_type1 vfio_virqfd` e.g. `MODULES=(vfio_pci vfio vfio_iommu_type1 vfio_virqfd)`
- Scroll down to bottom of file → ensure that the `modconf` hook is included in the HOOKS list → replace quotes `“”` with parentheses `()`
- Ctrl + X → ‘y’ to save → enter

### Rebuild kernel image
- Run command: `sudo mkinitcpio -p linux`
_**NOTE:** Once restarting at the next step, no signal from the GPU will go to the display as it is now in passthrough state. This means it will no longer be usable on the host machine until the the manipulation is reversed. Connect an HDMI cord to the motherboard's HDMI port._
- Run command: `reboot`

### Set motherboard to display with onboard graphics
- Immediately after rebooting, access the BIOS by tapping F2 at boot → Go to **Advanced Mode → Advanced → Chipset Configuration** → configure the following settings:
    - Primary Graphics Adapter: `Onboard`
    - IGPU Multi-Monitor: `Enabled`
- Exit → Save Changes and Exit

### Check and validate that the GPU passthrough configuration worked
- Open terminal and run command: lspci -nnk
- Find GPU → ensure it says below it: `Kernel driver in use: vfio-pci` This means that the device is captured by VFIO (Virtual Function)

### Install virt-manager, qemu, ovmf, libvirt, and ebtables dependencies required for OVMF-based guest virtual machine
- Run command: `sudo pacman -S qemu libvirt ovmf virt-manager ebtables`

### Configuring libvirt and making virtual network start automatically
- Run command: `sudo systemctl enable libvirtd.service`
- Run command: `sudo systemctl start libvirtd.service`
- Run command: `sudo systemctl enable virtlogd.socket`
- Run command: `sudo systemctl start virtlogd.socket`
- Run command: `sudo virsh net-autostart default`
