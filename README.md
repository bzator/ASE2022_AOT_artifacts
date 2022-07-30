## Introduction

This repository contains information regarding artifacts generated for the purpose of the  "Auto Off-Target: Enabling Thorough and Scalable Testing for Complex Software Systems" paper written by Tomasz Kuchta and Bartosz Zator. This paper describes experiments with off-target (OT) generation for four target systems: Android Linux kernel running on Pixel 6 device, the Little Kernel Embedded Operating System (`lk`), Das U-Boot (`uboot`) - a bootloader bootloader for embedded devices and the IUH module of the Osmocom project implementing he IUH interface for femtocell communication from a 3GPP standard. Although originally artifacts were generated for all target systems this repository contains information only about artifacts for the `lk` project. This is due to the enormous size of the artifact directories. The directory for the `lk` project itself (which is the smallest of the four) contains 12GB of data in its original form (unpacked). The principles of how the artifacts are used or what they represent don't change across the projects therefore it is assumed that the artifacts from the `lk` project can serve as a representative to the paper understanding and verification.

## Artifact repository

The public repository that contains the `lk` artifacts can be found at the following link: [aot_lk_artifact.tar.gz](https://figshare.com/s/c464484f996b9e49b835). It can be downloaded and extracted with the following Linux commands:
```bash
wget "https://figshare.com/ndownloader/files/35010967?private_link=c464484f996b9e49b835" -O aot_lk_artifact.tar.gz
tar -xf aot_lk_artifact.tar.gz
```

After extraction the repository contains the following files:

* cdm.json : compilation dependency map, which is a list of compiled files for all relevant modules (single linked lk binary in this case)
* compile_commands.json : a list of commands used to compile each source file
* ndb.json : Code DB for the `lk` linked module
* rdm.json : reverse dependency map, which maps all relevant files to the corresponding modules that used it
* results : a directory with the generated 1000 off-target programs for `lk`
