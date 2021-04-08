# VFIO_DOC
Personal Single GPU Passthrough Setup

## Specs  
Ryzen 5 3600  
GTX 1070ti  
32gb DDR4  

## Distro
PopOS

## Useful Scripts
Convenient script to get latest version on Ubuntu/PopOS, comes with easy option to add ACS Override patch  
https://gist.github.com/mdPlusPlus/031ec2dac2295c9aaf1fc0b0e808e21a

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
###### VFIO
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
###### KVM - CPU Pinning
CPU Pinning for Ryzen 5 3600  
```XML
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
## Commands
###### Kernel Options
Enables IOMMU for AMD CPUs, as well as enabling ACS for as many devices as possible.  
Probably not a good thing, but it works fine for now.
```bash
sudo kernelstub -a "amd_iommu=on amd_iommu=pt pcie_acs_override=downstream,multifunction"
```
