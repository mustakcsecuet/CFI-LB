# CFI-LB: Adaptive Call-site Sensitive Control Flow Integrity

CFI-LB: Adaptive Call-site Sensitive Control Flow Integrity will publish in EuroS&P' 2019. The source code is available here. The protoype is build with Clang/LLVM, Intel pin, Radare2, and Triton (Symbolic Execution Engine)(each of them have multiple dependencies). To build Clang/LLVM, it requires 20GB memory along, so please make sure your machine can support that load. The **run.sh** may ask for sudo permission to install dependent library and enable/disable ASLR for process memory dump to use in concolic process.

[Join us in the slack](https://join.slack.com/t/opencfi/shared_invite/enQtNzQ2MTM5MTA5NzM0LTdmMTQwZDU1YzEwNmE2ZDY4OTZiY2ExMDI1ZGVkOTdjYmYyNTNjNzVkOTYwNzdkNmY2OWNmMzhjMTUyNTJhZjc)

## IMPORTANT: Licensing

This project is licensed in GPLv3 with the following additional conditions: 

1. If you plan to benchmark, compare, evaluate this project with intention to publish the results (including in a paper), you must contact us with your real identity, affiliation, and advisors, and a short description of how you will use our source code before using and/or downloading this project. In addition, you will provide an opportunity for us to comment on and help with technical and other issues related to this project you have during the development. Examples include but are not limited to failure to compile or incomplete protection.

2. If you use any part of this project (excluding third-party software) and published a paper about it, you agree to open-source your project within one month of the official paper publication.

If you do not agree to these conditions, please do not use our source code.

**Justfication:** this is a research prototype. Its sole purpose is to demonstrate that the original idea works. It is expected to have implementation flaws. We welcome efforts to re-produce/evaluate our results but request an opportunity to fix implementation flaws. Generally speaking, we value design flaws more but will try to fix implementation issues.
If you plan to use this project in production, we would love to hear about it and provide help if needed. 

## Project Structure
**CFILB reference monitor implementation:** cfilbLibs/

**Reference monitor instrumentation:** llvm-project/clang/lib/CodeGen/CGCall.cpp

**Clang libtool to prepare the source:** llvm-project/clang/tools/clangCodePrep/

**Intel pin dynamic CFG generator:** dCFG/

**Intel pin process memory dump:** cHelper/

**LLVM static CFG generator:** llvm-project/llvm/lib/Transforms/sCFG/

**LLVM instrument CFG:** llvm-project/llvm/lib/Transforms/instCFG

**Symbol table extraction:** utils/extract.py

**Adaptive filter algorithm:** utils/filter.py

**Concolic process helper:** utils/symHelper.py

**Concolic CFG generator:** cCFG/src/examples/symCFG

**Run script to execute the CFILB process:** run.sh

**Makefile for CPU2006 Spec Benchmark:** spec2006-cfilb.cfg

## Overall Process
Step 1: A clang libtool will prepare the target code base.

Step 2: Copy the CFILB runtime library to the source directory.

Step 3: Build the source with clang (with reference monitor instrumentation) and generate the bitcode.

Step 4: Run a LLVM Pass analysis to calculate the static CFG and instrument the table back to bitcode.

Step 5: Build the binary from the step 4 bitcode. (This binary is protected with static CFG)

Step 6: Extract symbol table from the elf binary.

Step 7: Execute the binary with seed input using intel pin tool to generate dynamic CFG.

Step 8: Execute the binary with seed input using intel pin tool to dump memory for concolic process. (ASLR Disabled) [slow process]

Step 9: Run a radare2 python script to collect point of interest (POI) for concolic process.

Step 10: Run the concolic CFG generator (for each POI from step 9) with dump info from step 8. [can have crash issue, please report]

Step 11: Run a python script to apply the adaptive algorithm.

Step 12: Run another LLVM Pass to instrument the adaptive dynamic CFG table in the bitcode.

Step 13: Build the final binary from the step 12 bitcode. The binary will be named as: benchmarkname_cfg

Optional: Due to instrument CFG the code instruction address can be changed from concolic CFG, so there may be a repeat of step 11-13 with an additional check using a python script.

## Installation Guideline
1. Install required binary:
```text
sudo apt install wget clang cmake subversion g++ gcc bash git python-pip libcapstone-dev libboost-all-dev libz3-dev
pip install pyelftools
pip install r2pipe
```

2. Git clone the project:
```text
git clone git@github.com:mustakcsecuet/CFI-LB.git
cd CFILB
# copy the project path and save it
EDITOR ~/.profile
export CFILB_PATH="$HOME/../CFI-LB"
```
***Note: You can skip step 2, 3, 4, 6, and 7 if you have already configured Gold plugin for another compiler.***

3. Install required library for Gold plugin:
```text
sudo apt-get install linux-headers-$(uname -r) csh gawk automake libtool bison flex libncurses5-dev
# Check 'makeinfo -v'. If 'makeinfo' does not exist
sudo apt-get install apt-file texinfo texi2html
sudo apt-file update
sudo apt-file search makeinfo
```

4. Download binutils source code:
```text
cd ~
git clone --depth 1 git://sourceware.org/git/binutils-gdb.git binutils
```

5. Build binutils:
```text
mkdir build
cd build
../binutils/configure --enable-gold --enable-plugins --disable-werror
make
```

6. Build the compiler (use the binutils directory if you already have one):
```text
cd $CFILB_PATH/llvm-project
mkdir build
cmake -DLLVM_BINUTILS_INCDIR="path_to_binutils/include"  -DLLVM_ENABLE_PROJECTS=clang -G "Unix Makefiles" ../llvm
make -j8
```

7. Backup ar, nm, ld and ranlib:
```text
cd ~
mkdir backup
cd /usr/bin/
cp ar ~/backup/
cp nm ~/backup/
cp ld ~/backup/
cp ranlib ~/backup/
```

8. Replace ar, nm, ld and ranlib:
```text
cd /usr/bin/
sudo cp ~/build/binutils/ar ./
sudo rm nm
sudo cp ~/build/binutils/nm-new ./nm
sudo cp ~/build/binutils/ranlib ./
sudo cp ~/build/gold/ld-new ./ld
```

9. install LLVMgold.so to /usr/lib/bfd-plugins:
```text
cd /usr/lib
sudo mkdir bfd-plugins
cd bfd-plugins
sudo cp $CFILB_PATH/llvm_project/build/lib/LLVMgold.so ./
sudo cp $CFILB_PATH/llvm_project/build/lib/libLTO.* ./
```

10. Download intel-pin-3.5 source:
```text
cd $CFILB_PATH
wget http://software.intel.com/sites/landingpage/pintool/downloads/pin-3.5-97503-gac534ca30-gcc-linux.tar.gz
tar -xvzf pin-3.5-97503-gac534ca30-gcc-linux.tar.gz
rm pin-3.5-97503-gac534ca30-gcc-linux.tar.gz
mv pin-3.5-97503-gac534ca30-gcc-linux intel-pin
```

11. Build the dynamic CFG generation pin:
```text
cd $CFILB_PATH/dCFG
make PIN_ROOT=../intel-pin/
make PIN_ROOT=../intel-pin/ obj-intel64/dCFG.so
```

12. Build the dynamic process memory dump for concolic process:
```text
cd $CFILB_PATH/cHelper
make PIN_ROOT=../intel-pin/
make PIN_ROOT=../intel-pin/ obj-intel64/sym-dump.so
```

13. Build z3 solver from the source:
```text
cd $CFILB
git clone https://github.com/Z3Prover/z3.git
cd z3
python scripts/mk_make.py
cd build
make
sudo make install
```

14. Build the concolic system with Triton:
```text
cd $CFILB_PATH/cCFG
mkdir build
cd build
cmake ..
sudo make -j2 install
```

## Spec Benchmark Build Guideline
1. Put spec2006-cfilb.cfg file into folder $CPU2006_HOME/config and analyze CPU2006 to generate bc files
```text
cd $CPU2000_HOME
. ./shrc
rm -rf benchspec/CPU2006/*/exe/
runspec  --action=run --config=spec2006-cfilb.cfg --tune=base --size=test --iterations=1 --noreportable all
```
2. Change the Makefile.spec in the build directory of the benchmark (e.g. ~/spec/benchspec/CPU2006/456.hmmer/build/build_base_amd64-m64-softbound-nn.0000/Makefile.spec):
```text
# add cfilb.c in the source list, keep others same
SOURCES=cfilb.c ...
```
3. Use the run.sh to start the system. It is a long process and will ask for user input.
