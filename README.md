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

The `results` directory contains all the Off-target code generated for all functions evaluated for the `lk` target system which is the primary artifact to be verified as the main theme of the paper is extraction and generation of a correct C code from a larger S/W system. Additionally there are also results from the testing campaigns (fuzzer logs and artifacts) executed to properly evaluate the `AoT` usefullness (collected data from all test sessions are also available in the form of `.csv` files). All the Off-targets code was generated using two tools recently released to open source, i.e.:
* [Auto Off-Target](https://github.com/Samsung/auto_off_target)
* [CAS](https://github.com/Samsung/CAS)

These are the core engines that were release to open source. The test management scripts are not present and it was not included in the open sourced packages.


## Off-target generation

The Auto Off-Target project is able to generate off-target (OT) code for a given function automatically. More information can be found on the [AoT](https://github.com/Samsung/auto_off_target) project page. In order for the `AoT` project to work properly it needs Code Database (CodeDB) for a given project we're working on. Code Database can be generated using the Code Aware Services ([CAS](https://github.com/Samsung/CAS)) project. Please refer to the [README.md](https://github.com/Samsung/CAS/blob/master/README.md) file for more information how to setup and use the `CAS` project. Having the `CAS` setup properly the Code DB for the `lk` project can be created as follows.

First clone and build the repository under `CAS` tracer:
```bash
git clone https://github.com/littlekernel/lk.git && cd lk
export LK_DIR=$(pwd)
git checkout 77fa084cd05459a1f360a2b825e14ea6e60518e5
etrace scripts/make-parallel qemu-virt-arm32-test
````

Next create the Build Database:
```bash
export CAS_DIR=<path_to_CAS_repository>
${CAS_DIR}/etrace_parser .nfsdb
python3 ${CAS_DIR}/bas/postprocess.py .nfsdb.json
${CAS_DIR}/bas/postprocess.py --create-nfsdb-cache .nfsdb.json
```

In the next step we have to prepare some files needed for the Code Database creation:
```bash
export PYTHONPATH=${CAS_DIR}
git clone https://github.com/bzator/ASE2022_AOT_artifacts.git
ASE2022_AOT_artifacts/extract-lk-info-for-ftdb .nfsdb.json.img
```

Now create the Code Database:
```bash
export CLANG_PROC=${CAS_DIR}/clang-proc/clang-proc
${CAS_DIR}/clang-proc/create_json_db -P ${CLANG_PROC} -p fops -o fops.json
${CAS_DIR}/clang-proc/create_json_db -p db -o db.json -P $CLANG_PROC -F fops.json -m lk -V "77fa084cd05459a1f360a2b825e14ea6e60518e5" -A -cdm cdm.json -j4
${CAS_DIR}/tests/ftdb_cache_test --only-ftdb-create db.json
```

Finally setup the `AoT` project repository as described in the [Readme.md](https://github.com/Samsung/auto_off_target/blob/master/README.md) file. Make sure that the `AOT_DIR` environmental variable points to the proper `AOT` repository. First create some required configuration files (inside the `lk` project source code):
```bash
echo "{ \"BASserver\": \"https://localhost\" }" > cfg.json
cp ${AOT_DIR}/src/known_functions .
cp ${AOT_DIR}/src/lib_functions .
cp ${AOT_DIR}/src/always_include .
echo "[]" > init.json
````

Next import the Code Database for the `lk` product to be used by `AoT`:
```bash
${AOT_DIR}/src/aot.py --config=cfg.json --product=lk --version=77fa084cd05459a1f360a2b825e14ea6e60518e5 --build-type=eng --import-json=db.json --rdm-file=rdm.json --known-funcs-file=known_functions --lib-funcs-file=lib_functions --always-inc-funcs-file=always_include --init-file=init.json --source-root=${LK_DIR}
```

Now you can generate off-target (OT) for a given function (let's take the `minip_parse_ipaddr` function as an example):
```bash
${AOT_DIR}/src/aot.py --config=cfg.json --product=lk --version=77fa084cd05459a1f360a2b825e14ea6e60518e5 --build-type=eng --db=db.img --output-dir out_minip_parse_ipaddr --functions minip_parse_ipaddr --external-inclusion-margin 1 --init --verify-struct-layout
```
Now the OT for the `minip_parse_ipaddr` function should have been created in the `out_minip_parse_ipaddr` directory.

## Testing the off-target

In order to test the generated off-target the first thing to do is to compile the OT sources:
```bash
(cd out_minip_parse_ipaddr && make)
```

This will produce the `native` binary with the compiled (and linked) off-target code. Adding `AFL` instrumentation requires different target to build:
```bash
(cd out_minip_parse_ipaddr && make afl)
```

There are other targets like `asan`, `msan`, `ubasn`, `dfsan`, `klee` etc. Please consult the generated `Makefile` for details (building some of the targets might require latest clang version 15 installed). After the build of `afl` target completes the OT is ready to be fuzzed.

```bash
cd out_minip_parse_ipaddr
mkdir -p testcase && dd if=/dev/zero of=testcase/zero bs=4 count=1
afl-fuzz -m none -i testcase -o findings -- ./afl @@
```

Other testing techniques (like symbolic execution using the `KLEE`) can be used on other targets as appropriate.