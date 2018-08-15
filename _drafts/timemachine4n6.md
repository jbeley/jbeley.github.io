---
layout: post
title:  "Forensicating Apple Time Machine without a Mac"
date:   2018-08-05 12:41:00 -0500
categories: dmg osx timemachine dfir
---


# Goals:

* Determine user activities relating to [Time Machine](https://www.apple.com/in/support/timemachine/) on OSX. Note we are not dealing with encrypted time machine backups for now
* Acquire TimeMachine backups from both local and remote (AFP) based backups


# Requirements:

* Ubuntu 16.04 box (newer versions may work), I'm using the [SIFT workstation](https://digital-forensics.sans.org/community/downloads
)
* [afpfs-ng](https://github.com/simonvetter/afpfs-ng)
* [sparsebundlefs](https://github.com/torarnv/sparsebundlefs)
* [afpfs-ng](https://github.com/simonvetter/afpfs-ng)
* [docker](https://www.docker.com)
* p7zip-full (install from ubuntu)
* I used a vagrant environment as not to break my existing backups

We won't be covering installation of the required tools.


# Goals:

* Utilize [Time Machine](https://www.apple.com/in/support/timemachine/) to determine user activities on OSX

# Test environment
* [Vagrant environment I whipped together to write this post ](https://github.com/jbeley/tm4n6/)


# Local Time Machine Capture

* Forensically capture as normal and create working copy (we're skipping this as we have a virtual environment)

*  Because we're using a virtual environment, we're exposing the VirtualBox VDI to our SIFT workstation using the [Network Block Device](https://nbd.sourceforge.io/
) facility in Linux. And to save space, we're exposing the VDI to our SIFT VM using the shared filesystem of our hypervisor.

`modprobe nbd`

`qemu-nbd -c /dev/nbd0  /mnt/tm4n6_osx-1012_1534267651942_53821/
NewVirtualDisk1.vdi`

`mount /dev/nbd0p2 /mnt2 -o ro -t hfsplus`

* Browse to directory
` cd /mnt2/Backups.backupdb/HOSTNAME`

_SCREENSHOT_

* While browsing the exposed TimeMachine backup, I noted that there was approximately  337 MB of data. This appears to be the delta from the base operating system installation, which in certain conditions, is a good thing. There were no locally stored syslogs or ASL logs within the backups, so caveat forensicator.

_SCREENSHOT_

* I'm a big fan of creating forensic timelines. I created a [docker](https://hub.docker.com/r/jbeley/plaso/) container to run [plaso](https://github.com/log2timeline/plaso)


`docker pull jbeley/plaso`

_SCREENSHOT_

* We have to run log2timeine.py as root within the docker container in order to read files on the exposed HFS+ volume. Because we're not examining the full filesystem, this process is generally quick. On my Mid-2014 MacBook Pro, the process took 67 seconds.

`docker run -u root --rm -it -v $(pwd)/2018-08-14-202132:/data  -v /tmp/:/output jbeley/plaso log2timeline.py /output/osx-1012_2018-08-14-202132.pb /data/`

* We're going to run psort.py to examine the resultant protobuf file to ensure there were no errors in processing that would impact our investigation.


`docker run -u root --rm -it -v /tmp/:/output jbeley/plaso pinfo.py /output/osx-1012_2018-08-14-202132.pb`

_SCREENSHOT_

* Now that we know that the resultant protobuf file is usable. We can convert it to JSON for analysis

`docker run -u root --rm -it -v /tmp/:/output jbeley/plaso psort.py -w /output/osx-1012_2018-08-14-202132.json -o json_line   /output/osx-1012_2018-08-14-202132.pb`

#Remote
Setup netatalk
Setup avahi
Setup Users










Remote
* Remote mount with [afpfs-ng](https://github.com/simonvetter/afpfs-ng)
* Browse to sparsebundle
* Capture sparsebundle  as normal (this may take a WHILE)

Mount sparsebunle [sparsebundlefs](https://github.com/torarnv/sparsebundlefs)

`sparsebundlefs ~/MyDiskImage.sparsebundle /tmp/my-disk-image`

`mount -o loop -t hfsplus /tmp/my-disk-image/sparsebundle.dmg /mnt/my-disk`

`mount_afp afp://simon:mypassword@delorean.local/time_travel /mnt/timetravel`




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
