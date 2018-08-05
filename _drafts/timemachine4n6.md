---
layout: post
title:  "Forensicating Apple Time Machine without a Mac"
date:   2018-08-05 12:41:00 -0500
categories: dmg osx timemachine dfir
---

We won't be covering installation of the required tools.

Requirements:

* Ubuntu 16.04 box (newer versions may work)
* [afpfs-ng](https://github.com/simonvetter/afpfs-ng)
* [sparsebundlefs](https://github.com/torarnv/sparsebundlefs)
* p7zip-full (install from ubuntu)


Goals:

* Determine activities relating to [Time Machine](https://www.apple.com/in/support/timemachine/)
* 


Two typical situaitons

Local process

* Forensically capture as normal and create working copy
* Browse to sparse bundle
* Capture sparsebundle  as normal

 

Remote
* Remote mount with [afpfs-ng](https://github.com/simonvetter/afpfs-ng) 
*
* Browse to sparsebundle
* Capture sparsebundle  as normal (this may take a WHILE)

Mount sparsebunle [sparsebundlefs](https://github.com/torarnv/sparsebundlefs)

`sparsebundlefs ~/MyDiskImage.sparsebundle /tmp/my-disk-image`

`mount -o loop -t hfsplus /tmp/my-disk-image/sparsebundle.dmg /mnt/my-disk`

`mount_afp afp://simon:mypassword@delorean.local/time_travel /mnt/timetravel`




Remote:
mount_afp afp://jbeley:xxxx@xxxxx.local /mnt
The afpfs daemon does not appear to be running for uid xx, let me start it for you
Mounting  from xxxxx on /mnt
Volume  does not exist on server airport.
Choose from: xxxxx, xxxxx

mount_afp afp://jbeley:xxxxx@xxxxxxxx/REALMOUNT /mnt


`rsync -avzH --progress --stats --partial --inplace  ./voxpopuli.sparsebundle /data/tm4n6/`

Warning: the afpfs-ng daemon runs in userspace, your performance may vary. Additioanlly, it does not appear to be multi-threaded, so other acceses to the mount point will likely impact the performance of the capture. 
Make sure the trailing `/` is omitted from the sparsebundle directory name.

Plaso
`docker run --rm -it jbeley/plaso`

Carving

Listing contents

 
`7z l /my/dmg `

dmg2img
https://github.com/Lekensteyn/dmg2img


