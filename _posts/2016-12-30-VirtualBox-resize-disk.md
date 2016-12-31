---
layout: post
title: Resizing an Ubuntu image on Virtual Box
published: true
categories: 
            - Linux
            - Virtual Box
            - Partioning

---

## Motivation
I work on a Macbook running OS X. However I still like to log into a virtual box ubuntu environment to use some of the applications I like. I got a prompt saying I was out of disk space on my ubuntu virtual environment. It took me a while to workout how to increase the size of the disk by another 10 Gig. I shall attempt to recall what I did.


## Run some virtual box commands to increase the virtual capacity

This command shows you the disk information
```
VBoxManage showhdinfo /path/to/vm/Ubuntu.vdi 
```
Resize the capacity to 20GB
```
VBoxManage modifyhd --resize 20000 ~/VirtualBox\ VMs/Ubuntu/Ubuntu.vdi
```

Rerun the first command to confirm the values have changed
```
VBoxManage showhdinfo /path/to/vm/Ubuntu.vdi 
```

## Use GParted to make use of the newly availble space
What we've done so far is simply created additional capacity. We now need to make use of it and this was what I found painful.

Step 1: Download the GParted iso image from http://gparted.org/download.php. In my case I chose the 64 bit one.

Step 2: Go to Virtual Box and select your Ubuntu image and then go to  Settings > Storage.

Step 3: Add an optical disk attachment under the IDE controller and select Live CD.

![Step 3](/images/Virtual-box-gparted-iso.png "Step 3")

Step 4: Check the boot order of your Ubuntu vm to ensure CD boots first before the hard drive. 
Settings > System > Boot order tab

Step 5: Start the ubuntu image and you will be prompted by GParted. Answer prompts with relevant answers (i.e UK English) and then on the last promp answer (0) to launch GParted automatically.

Step 6: You will see the extra 10 Gig sitting in unallocated. The tricky part to extending the partions.
I followed some of the steps here: http://gparted.org/display-doc.php%3Fname%3Dmoving-space-between-partitions

Basically you have to ensure your new unallocated partion moves up the block tree until it is next to the partion you want to grow. 

