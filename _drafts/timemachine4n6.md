---
layout: post
title:  "Forensicating Apple Time Machine without a Mac"
date:   2018-08-18 12:41:00 -0500
categories: dmg osx timemachine dfir
---


# Goals:

* Determine user activities relating to [Time Machine](https://www.apple.com/in/support/timemachine/) on OSX. Note we are not dealing with encrypted time machine backups for now
* Acquire TimeMachine backups from both local (disk) and remote (AFP) based backups


# Requirements:

* Ubuntu 16.04 box (newer versions may work), I'm using the [SIFT workstation](https://digital-forensics.sans.org/community/downloads
) but a "plain" Ubuntu box should work equally as well
* [afpfs-ng](https://github.com/simonvetter/afpfs-ng)
* [sparsebundlefs](https://github.com/torarnv/sparsebundlefs)
* [afpfs-ng](https://github.com/simonvetter/afpfs-ng)
* [docker](https://www.docker.com)
* I used a [vagrant](https://www.vagrantup.com) environment as not to break my existing backups

We won't be covering installation of the required tools today.


# Goals:

* Utilize [Time Machine](https://www.apple.com/in/support/timemachine/) to determine user activities on OSX

# Test environment
* [Vagrant environment I whipped together to write this post ](https://github.com/jbeley/tm4n6/)


# Local Time Machine Capture
## Forensic Capture
* Forensically capture as normal and create working copy (we're skipping this as we have a virtual environment). The EWF tools can be used to expose the E01 to the operating system.

## Exposing the HFS+ filesystem

*  Because we're using a virtual environment, we're exposing the VirtualBox VDI to our SIFT workstation using the [Network Block Device](https://nbd.sourceforge.io/
) facility in Linux. And to save space and time, we're exposing the VDI to our SIFT VM using the shared filesystem of our hypervisor.

`modprobe nbd`

`qemu-nbd -c /dev/nbd0  /mnt/vms/tm4n6_osx-1012_1534267651942_53821/
NewVirtualDisk1.vdi`

* Create E01 from second partition of local Time Machine disk

`ewfacquire -v -u -l /tmp/4n6.ewf.log -t /vagrant/tm4n6  -S 2TB /dev/nbd0p2`

*  If you wish to mount the filesystem and browse

`mount /dev/nbd0p2 /target -o ro -t hfsplus`



# Remote Time Machine capture (Time Capsule)

I had an old Time Machine backup on a Time Capsule that I wanted to collect.


## Remote mounting of AFP (Time Capsule) volumes

* Remote mount the  with [afpfs-ng](https://github.com/simonvetter/afpfs-ng)

`mount_afp afp://jbeley:xxxx@xxxxx.local /mnt`

`The afpfs daemon does not appear to be running for uid xx, let me start it for you
Mounting  from xxxxx on /mnt
Volume  does not exist on server airport.
Choose from: xxxxx, xxxxx ``

`mount_afp afp://jbeley:xxxxx@xxxxxxxx/REALMOUNT /mnt`


* Capture sparsebundle using rsync (this may take a WHILE depending on network speeds and age/volume of backup data)

`rsync -avzH --progress --stats --partial --inplace  ./voxpopuli.sparsebundle /data/tm4n6/`

Warning: the afpfs-ng daemon runs in userspace, your performance may vary. Additioanlly, it does not appear to be multi-threaded, so other acceses to the mount point will likely impact the performance of the capture.
Make sure the trailing `/` is omitted from the sparsebundle directory name. Also, ensure `-H` is specified to deal with [hard links](https://en.wikipedia.org/wiki/Hard_link).

## Exposing the HFS+ filesystem
* Expose sparsebunle as HFS+ filesystem
Mount sparsebunle [sparsebundlefs](https://github.com/torarnv/sparsebundlefs)

`sparsebundlefs /data/tm4n6/HOSTNAME.sparsebunle /dmg`

* Automatically create loop devices from each partition

`kpartx -a -v /dmg/sparsebundle.dmg`

* Create EWF from DMG (HFS+ volume). This will also take a a while

`ewfacquire -q -u -l /tmp/4n6.ewf.log -t /tmp/tm4n6  -S 2TB /dev/mapper/loop1p2`

*  If you wish to mount the filesystem and browse

`mount -o loop,ro -t hfsplus /dev/mapper/loop0p2 /target`


# Common Steps once you have the sparsebunle exposed as an HFS+ filesystem  

* Browse to directory
` cd /target/Backups.backupdb/HOSTNAME`

[timemachine listing]({{ site.url }}/assets/tm4n6_ls.jpg)

* While browsing the exposed TimeMachine backup, I noted that there was approximately  337 MB of data. This appears to be the delta from the base operating system installation, which in certain conditions, is a good thing. There were no locally stored syslogs or ASL logs within the backups, so caveat forensicator.

[timemachine du]({{ site.url }}/assets/tm4n6_du.jpg)


* I'm a big fan of creating forensic timelines. I created a [docker](https://hub.docker.com/r/jbeley/plaso/) container to run [plaso](https://github.com/log2timeline/plaso)


`docker pull jbeley/plaso`

[timemachine docker pull]({{ site.url }}/assets/tm4n6_docker_pull.jpg)

* We have to run log2timeine.py as root within the docker container in order to read files on the exposed HFS+ volume. Because we're not examining the full filesystem, this process is generally quick. On my Mid-2014 MacBook Pro, the process took 67 seconds. jj

`docker run -u root --rm -it -v /target/Backups.backupdb/HOSTNAME/2018-08-14-202132:/data   -v  /case/:/output jbeley/plaso log2timeline.py  --hashers all --parsers all  --data /usr/share/plaso --artifact_definitions /usr/share/artifacts  --logfile /output/osx-1012_2018-08-14-202132.plaso.log /output/osx-1012_2018-08-14-202132.pb /data/`

* We're going to run psort.py to examine the resultant protobuf file to ensure there were no errors in processing that would impact our investigation.

`docker run -u root --rm -it -v /case/:/output jbeley/plaso pinfo.py -w /output/osx-1012_2018-08-14-202132.pinfo.log  /output/osx-1012_2018-08-14-202132.pb`

[timemachine psort]({{ site.url }}/assets/tm4n6_psort.jpg)

* Now that we know that the resultant protobuf file is usable. We can convert it to JSON (or CSV) for analysis.

* JSON :

`docker run -u root --rm -it -v /case/:/output jbeley/plaso psort.py -w /output/osx-1012_2018-08-14-202132.json -o json_line  --logfile /output/osx-1012_2018-08-14-202132.psort.log /output/osx-1012_2018-08-14-202132.pb`

* CSV :
`docker run -u root --rm -it -v /case/:/output jbeley/plaso psort.py -w /output/osx-1012_2018-08-14-202132.csv -o l2tcsv  --logfile /output/osx-1012_2018-08-14-202132.psort.log /output/osx-1012_2018-08-14-202132.pb`


* As noted above, we have multiple snapshots. By running the above subsequently, we can see when the deltas occur. It should be noted, that the default backup interval is 1 hour while the machine is online and has access to the backup medium.


* Bulk Extractor
`bulk_extractor -R /target/Backups.backupdb/HOSTNAME/2018-08-14-202132/ -o /case/osx-1012_2018-08-14-20213`

# Lessons Learned

* User space filesystems can take a long time to acquire

* Time Machine backups appear to be the delta of the base installation plus the user's files. No system logs are captured.

* [Sparsebunles](https://en.wikipedia.org/wiki/Sparse_image) are representations of HFS+ filesystems containing fixed-sized band files

* 
