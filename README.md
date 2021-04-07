# VFIO_DOC
Personal Single GPU Passthrough Setup

## Specs  
Ryzen 5 3600  
GTX 1070ti  
32gb DDR4  

## Useful Scripts
Convenient script to get latest version on Ubuntu/PopOS, comes with easy option to add ACS Override patch  
https://gist.github.com/mdPlusPlus/031ec2dac2295c9aaf1fc0b0e808e21a

Decent script for viewing iommu groups, will link to original source if I find it.
'''bash
#!/bin/bash
shopt -s nullglob
for g in `find /sys/kernel/iommu_groups/* -maxdepth 0 -type d | sort -V`; do
    echo "IOMMU Group ${g##*/}:"
    for d in $g/devices/*; do
        echo -e "\t$(lspci -nns ${d##*/})"
    done;
done;
'''
