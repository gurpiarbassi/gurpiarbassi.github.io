---
layout: post
title: Partioning a SDXC card on OS X to use with Raspberry Pi
published: true
categories: 
            - Raspberry Pi
            - OS X
            - Partioning
            - FAT 32
---

# Motivation
I wanted to use an existing 64 GB SDXC card in a Raspberry Pi. However from reading various articles I gathered that installing NOOBS onto a 64 GB card may result in problems. So I needed to partition this card. However the partition button in OS X Disk Utility was always greyed out.


## Creating partions on OS X
1. Launch Disk Utility
2. Select the SD Card
3. Erase all and format it with the "OS X Extended (Journaled) option" using GUID based partitioning
4. The partition button now becomes clickable so click on it
5. From here create 2 partitions of approximately equal size using "OS X Extended (Journaled)" option for each partition 
6. Two partitions will now be created but we need to make them into FAT 32 partitions
7. Individually select each partition and erase it, selecting "MS-DOS (FAT)" this time

## Trying it out in a Pi
The first time it starts, NOOBS will actually automagically resize the two partitions back into one for you.