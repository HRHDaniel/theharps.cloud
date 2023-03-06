---
layout: post
title:  "Formatting a USB drive to FAT32"
date:   2023-03-05 10:00:00 -0600
---
Just had a need arise requiring a reformat of a USB stick to FAT32.  Easy enough - I stuck it in the PC and right clicked on the drive in explorer and selected format. Now I am annoyed because FAT32 isn't an option:

![Format Options](/imgs/format-options.png)

At first, I thought my annoyance should be directed at myself and that I should have paused and thought more about what I was doing. However, it is now directed more at MicroSoft. My USB drive is 64GB. FAT32 can be used for drives larger than 32GB, but many assume the 32 in the name is related to limiting to 32GB. It appears windows engineers may have fallen for that myth and limited their gui tools to only allowing FAT32 when formatting 32GB or less.

After a quick Internet search, I was provided with a couple options. The most common recommendation is to download third party tools.

Your second option is formatting from Powershell. Run Powershell as administrator and run `format /fs:fat32 F:` where `F:` is your USB drive letter. However, all the sites that mention that option state it doesn't allow a quick format and takes a very long time.

Every site I found limited you to those two options. However, there's a third option available they left out. For my particular case, I don't need more than 32GB and will not be leaving my flash drive in this format for long. If that is all you need, just drop the partition size down to 32GB before formatting. Here is the easy quick solution:

Hit your windows "Start" menu and type `Disk management` and select "Create and format hard disk partitions".

Find your USB drive, mine was `Disk 2` mounted as `F:`

![Disk Management - Disk 2](/imgs/disk-management-1.png)

Right click on the volume on that disk and select "Delete Volume.."

You will receive a warning that all the data on the volume will be lost. Take heed to that warning and make sure you are deleting the correct one and that you do not have anything on the disk you do not want to lose. Once you are sure, agree and continue.

Now you have a disk with unallocated space. Right click on that unallocated space and select "New Simple Volume". Set the volume size to `32768 MB`.

![New Volume](/imgs/new-volume.png)

You can choose to assign this to a drive letter or not. Then you will be given the option to format the partition with the option for `FAT32` and `Perform a quick format`

![Format Options](/imgs/format-options-2.png)

I now have a 32GB FAT32 flash drive that is working perfectly for the files I need to move around. I will repeat the above steps to return it to an exFAT drive when no longer needed.