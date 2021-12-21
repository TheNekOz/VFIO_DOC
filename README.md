# VFIO_DOC
Personal Single GPU Passthrough Setup
This is mostly notes for personal usage, but I have made this public hoping it's useful for other people.  

## Specs  
Ryzen 5 3600  
GTX 1070ti  
32gb DDR4  

## Distro
PopOS

## Useful Scripts
Convenient script to get latest kernel on Ubuntu/PopOS, comes with easy option to add ACS Override patch  
https://gist.github.com/mdPlusPlus/031ec2dac2295c9aaf1fc0b0e808e21a  
Kernel used by PopOS should has ACSO Patch on by default afaik.
Decent script for viewing iommu groups, will link to original source if I find it.
```bash
#!/bin/bash
shopt -s nullglob
for g in `find /sys/kernel/iommu_groups/* -maxdepth 0 -type d | sort -V`; do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
done;
```
## Guides
##### VFIO
Huge parts of my setup is based on this guide.  
https://github.com/joeknock90/Single-GPU-Passthrough  

Worth checking out, not used this.  
https://github.com/QaidVoid/Complete-Single-GPU-Passthrough  

For some unfortunate souls, such as me, the ACSO patch is needed. Arch Wiki comes to the rescue!  
https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#Bypassing_the_IOMMU_groups_(ACS_override_patch)

###### KVM Optimization
One of many sources used to understand CPU pinning, can take time to understand this.  
https://wiki.archlinux.org/index.php/PCI_passthrough_via_OVMF#CPU_pinning  

## Configuration
##### KVM - CPU Pinning
CPU Pinning for Ryzen 5 3600  
```XML
  <vcpu placement="static">10</vcpu>
  <cputune>
    <vcpupin vcpu="0" cpuset="1"/>
    <vcpupin vcpu="1" cpuset="7"/>
    <vcpupin vcpu="2" cpuset="2"/>
    <vcpupin vcpu="3" cpuset="8"/>
    <vcpupin vcpu="4" cpuset="3"/>
    <vcpupin vcpu="5" cpuset="9"/>
    <vcpupin vcpu="6" cpuset="4"/>
    <vcpupin vcpu="7" cpuset="10"/>
    <vcpupin vcpu="8" cpuset="5"/>
    <vcpupin vcpu="9" cpuset="11"/>
  </cputune>
```
CPU Pinning locks CPU cores and threads to the VM. A core is left behind as a "house-keeping" core for Linux, performance in VM is really good. CPU Pin configuration is heavily based on the topology of your CPU. If you have a Ryzen 5 3600 like me, you should be able to use my configuration.
##### Libvirt scripts
**/etc/libvirt/hooks/qemu.d/[vm_name]/prepare/begin/start.sh**  
```bash

#!/bin/bash
# Helpful to read output when debugging
set -x

cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
for file in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do echo "performance" > $file; done
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# Start Samba Service
systemctl start smbd
# Start OpenSSH Server
systemctl start sshd
# Stop display manager
systemctl stop display-manager.service
## Uncomment the following line if you use GDM
killall gdm-x-session

# Unbind VTconsoles
echo 0 > /sys/class/vtconsole/vtcon0/bind

# Unbind EFI-Framebuffer
echo efi-framebuffer.0 > /sys/bus/platform/drivers/efi-framebuffer/unbind

# Avoid a Race condition by waiting 2 seconds. This can be calibrated to be shorter or longer if required for your system
sleep 3

# Unload all Nvidia drivers
modprobe -r nvidia_drm
modprobe -r nvidia_modeset
modprobe -r nvidia_uvm
modprobe -r nvidia

# Unbind the GPU from display driver
virsh nodedev-detach pci_0000_08_00_0
virsh nodedev-detach pci_0000_08_00_1

# Load VFIO Kernel Module  
modprobe vfio-pci  
```
**/etc/libvirt/hooks/qemu.d/[vm_name]/release/end/revert.sh**  
```bash
#!/bin/bash
set -x

#Revert to On Demand governor
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor
for file in /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor; do echo "ondemand" > $file; done
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

#Stop samba service
systemctl stop smbd

# Unload VFIO-PCI Kernel Driver
modprobe -r vfio-pci
modprobe -r vfio_iommu_type1
modprobe -r vfio
  
# Re-Bind GPU to Nvidia Driver
virsh nodedev-reattach pci_0000_08_00_1
virsh nodedev-reattach pci_0000_08_00_0

# Rebind VT consoles
echo 1 > /sys/class/vtconsole/vtcon0/bind
# Some machines might have more than 1 virtual console. Add a line for each corresponding VTConsole
#echo 1 > /sys/class/vtconsole/vtcon1/bind

nvidia-xconfig --query-gpu-info > /dev/null 2>&1
echo "efi-framebuffer.0" > /sys/bus/platform/drivers/efi-framebuffer/bind

modprobe nvidia_drm
modprobe nvidia_modeset
modprobe nvidia_uvm
modprobe nvidia

# Restart Display Manager
systemctl start display-manager.service
```
## Commands
##### Kernel Options
Enables IOMMU for AMD CPUs, as well as enabling ACS for as many devices as possible.  
Probably not a good thing, but it works fine for now.
```bash
sudo kernelstub -a "amd_iommu=on amd_iommu=pt pcie_acs_override=downstream,multifunction"
```
