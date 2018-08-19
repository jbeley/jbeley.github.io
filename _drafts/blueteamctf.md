---
layout: post
title:  "Creating a Blue Team entrance exam"
date:   2018-08-18 12:41:00 -0500
categories: DFIR CTF
---


# Goals:

Create a real life scenario to test domain knowledge of common incidents to:

* keep blue team sharp during down timelines

* create an objective entrance exam

* use created environment to develop and automate detection capabilities


# Requirements:

* vagrant

* disk space

* time

* Linux (preferably) host with at least 16 GB of RAM. My test system had 32 GB of RAM.


# Setting up the Virtual environment

## DetectionLab

 Before setting up this environment, I had never used vagrant in any capacity.  I found [DetectionLab](https://github.com/clong/DetectionLab) had many of the features I needed and would start as a good base to enhance.

### Additions to DetectionLab:
* bro and suricata applications to the build environment

* custom Microsoft Windows using boxcutter vagrant boxes with the Python and the Salt Stack installed as well as being fully patched

* [RTA](LINK) to automate some of the anti-forensic measures used

* a script to automate the capture of evidence to create an "evidence package" containing a memory image, an E01 comprising the virtual disk, and output from volatility and plaso

* a script to automate the creation of an OVA from a running vagrant VM (running under VirtualBox)

* a "analyst" workstation with typical tools used in incident response

* a Makefile to automate the build of the virtual environment

* modification to the "victim" setup to allow infection of Windows hosts via a network shared

* Linux victims


## Building the environment

### Building the custom Windows Vagrant boxes

I already had the ISO images for Windows 10 and Windows Server 2016 downloaded. If you don't have them downloaded, the boxcutter build process will download it automatically.  I added the following to the `Makefile.local` to ensure the Salt Stack was installed.

 ```
EVAL_WIN7_X64 := file:///data/msdn/win7eval.iso
EVAL_WIN10_X64 := file:///data/msdn/eval-win10x64-enterprise.iso
EVAL_WIN2012R2_X64 := file:///data/msdn/eval-win2012r2-standard.iso
EVAL_WIN2016_X64 := file:///data/msdn/eval-win2016-standard.iso
CM := salt
CM_VERSION := 2018.3.2-Py2
HEADLESS := true
GENERALIZE := true
```

I then ran:

```
make virtualbox/eval-win10x64-enterprise virtualbox/eval-win2016-standard
```

This can take a while depending on network bandwidth and system speed.
To add the newly created box files to your vagrant environment:

```
vagrant box add box/virtualbox/eval-win10x64-enterprise-salt2018.3.2-Py2-1.0.4.box --name eval-win10x64-enterprise-salt2018.3.2-Py2-1.0.4
vagrant box add box/virtualbox/eval-win2016-standard-salt2018.3.2-Py3-1.0.4.box --name eval-win2016-standard-salt2018.3.2-Py3-1.0.4
```




# Setting up the CTF


# Lessons Learned

* Virtualizing lots of machines on a single laptop is a juggling act

*
