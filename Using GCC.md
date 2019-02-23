# Using GCC for compiling Add-Ins for fx-9860G (II)

This guide assumes you're using Linux or similar environment.

## Compiling the Toolchain

Tested with:
  - binutils 2.32
  - gcc 8.2.0

"weak" dependencies (acquiring and extracting sources):
  - xz-utils
  - wget
  - git

Dependencies for building GCC (list might be incomplete):
  - mpfr
  - mpc
  - gmp
  - gcc
  - make

This guide aims to build and install:
  - *sh3eb-elf* toolchain
  - g1a-wrapper by Lephenixnoir

### Steps

#### 1. Set the variables and directories
We will install our toolchain to `~/fx-sh3-toolchain`.  
Begin by setting the `PREFIX` variable which we will use as the destination for
our toolchain:
```
export PREFIX="~/fx-sh3-toolchain"
mkdir -p $PREFIX
```
Afterwards we set `TARGET` to `sh3eb-elf` which we will pass to GCC during it's
build process.
```
export TARGET="sh3eb-elf"
```

To make this guide non-version specific, at the time of the writing this
toolchain was comprised of **binutils-2.32** + **gcc-8.2.0**. You may try to
change these variables to a newer version of them should they be available.
```
export BINUTILS_VER="2.32"
export GCC_VER="8.2.0"
```

Finally, to avoid polluting the current directory, we'll do the work in a
separate directory, `~/fx-building` for example. Let's create it:
```
mkdir -p ~/fx-building
```

##### Optional
If you want to speedup the building process and you have a multi-core system,
set `MAKE_THREADS` to the **number of workers/cores you want to use** during
the process.
```
export MAKE_THREADS=4
```


#### 2. Build binutils
Execute:
```
cd ~/fx-building
wget -N ftp://ftp.gnu.org/gnu/binutils/binutils-${BINUTILS_VER}.tar.xz
tar -xf binutils-${BINUTILS_VER}.tar.xz
cd binutils-${BINUTILS_VER}
mkdir obj-sh3
cd obj-sh3
../configure --prefix=$PREFIX --target=$TARGET --disable-nls --enable-lto
make -j $MAKE_THREADS
make install
```

#### 3. Build GCC (1st time)
```
export PATH=$PATH:$PREFIX/bin

cd ~/fx-building
wget -N ftp://gcc.gnu.org/pub/gcc/releases/gcc-${GCC_VER}/gcc-${GCC_VER}.tar.xz
tar -xf gcc-${GCC_VER}.tar.xz
cd gcc-${GCC_VER}
mkdir obj-sh3
cd obj-sh3
../configure --prefix=$PREFIX --target=$TARGET --enable-languages=c,c++ \
  --disable-nls --disable-libssp --without--headers --with-newlib
make -j $MAKE_THREADS
make install
```

#### 4. Build newlib
```
cd ~/fx-building
git clone https://git.planet-casio.com/Memallox/libc newlib-fx
cd newlib-fx
mkdir obj-sh3
cd obj-sh3
../configure --prefix=$PREFIX --target=$TARGET --enable-target-optspace
make -j $MAKE_THREADS
make install
```

#### 5. Rebuild GCC
```
cd ~/fx-building
cd gcc-${GCC_VER}
rm -rf obj-sh3
mkdir obj-sh3
cd obj-sh3
../configure --prefix=$PREFIX --target=$TARGET --enable-languages=c,c++ \
  --disable-nls --disable-libssp --without--headers --with-newlib --enable-lto
make -j $MAKE_THREADS
make install
```

#### 6. Build g1a-wrapper
```
cd ~/fx-building
git clone https://bitbucket.org/Lephenixnoir/add-in-wrapper.git
cd add-in-wrapper
make
cp build/g1a-wrapper $PREFIX/bin
```



## Compiling a project

According to
https://web.archive.org/web/20110112033531/https://sourceforge.net/apps/trac/fxsdk/wiki/UsingGCC, the following scripts and code are supposed to mimic the
official SDK behavior.  
An example project containing all the following files can be obtained from:
http://www.planet-casio.com/files/forums/projet-142401.zip or if you're just
interested in the linker and crt0.s files they can be found below:

#### addin.ld
```
OUTPUT_ARCH(sh3)
ENTRY(initialize)
MEMORY
{
        rom  : o = 0x00300200, l = 512k
        ram  : o = 0x08100000, l = 64k  /* pretty safe guess */
}
SECTIONS
{
        .text : {
                *(.pretext)     /* init stuff */
                *(.text)
        } > rom
        .rodata : {
                *(.rodata)
                *(.rodata.str1.4)
                _romdata = . ;  /* symbol for initialization data */
        } > rom
        .bss : {
                _bbss = . ;
                _bssdatasize = . ;
                LONG(0);        /* bssdatasize */
                *(.bss) *(COMMON);
                _ebss = . ;
        } > ram
        .data BLOCK(4) : AT(_romdata) {
                _bdata = . ;
                *(.data);
                _edata = . ;
        } > ram
}
```
#### crt0.s
```
.section .pretext
.global initialize
initialize:
sts.l   pr, @-r15

! set up TLB
mov.l   Hmem_SetMMU, r3
mov.l   address_one, r4 ! 0x8102000
mov.l   address_two, r5 ! 0x8801E000
jsr     @r3    ! _Hmem_SetMMU
mov     #108, r6

! clear the BSS
mov.l   bbss, r4   ! start
mov.l   ebss, r5   ! end
bra     L_check_bss
mov     #0, r6
L_zero_bss:
mov.l   r6, @r4        ! zero and advance
add     #4, r4
L_check_bss:
cmp/hs  r5, r4
bf      L_zero_bss

! Copy the .data
mov.l   bdata, r4  ! dest
mov.l   edata, r5  ! dest limit
mov.l   romdata, r6        ! source
bra     L_check_data
nop
L_copy_data:
mov.l   @r6+, r3
mov.l   r3, @r4
add     #4, r4
L_check_data:
cmp/hs  r5, r4
bf      L_copy_data

mov.l   bbss, r4
mov.l   edata, r5
sub     r4, r5              ! size of .bss and .data sections
add     #4, r5
mov.l   bssdatasize, r4
mov.l   r5, @r4

mov.l   GLibAddinAplExecutionCheck, r2
mov     #0, r4
mov     #1, r5
jsr     @r2    ! _GLibAddinAplExecutionCheck(0,1,1);
mov     r5, r6

mov.l   CallbackAtQuitMainFunction, r3
mov.l   exit_handler, r4
jsr     @r3    ! _CallbackAtQuitMainFunction(&exit_handler)
nop
mov.l   main, r3
jmp     @r3    ! _main()
lds.l   @r15+, pr

_exit_handler:
mov.l   r14, @-r15
mov.l   r13, @-r15
mov.l   r12, @-r15
sts.l   pr, @-r15

mov.l   Bdel_cychdr, r14
jsr     @r14 ! _Bdel_cychdr
mov     #6, r4
jsr     @r14 ! _Bdel_cychdr
mov     #7, r4
jsr     @r14 ! _Bdel_cychdr
mov     #8, r4
jsr     @r14 ! _Bdel_cychdr
mov     #9, r4
jsr     @r14 ! _Bdel_cychdr
mov     #10, r4

mov.l   BfileFLS_CloseFile, r12
mov     #4, r14
mov     #0, r13
L_close_files:
jsr     @r12 ! _BfileFLS_CloseFile
mov     r13, r4
add     #1, r13
cmp/ge  r14, r13
bf      L_close_files

mov.l   flsFindClose, r12
mov     #0, r13
L_close_finds:
jsr     @r12 ! _flsFindClose
mov     r13, r4
add     #1, r13
cmp/ge  r14, r13
bf      L_close_finds

lds.l   @r15+, pr
mov.l   @r15+, r12
mov.l   @r15+, r13
mov.l   Bkey_Set_RepeatTime_Default, r2
jmp     @r2    ! _Bkey_Set_RepeatTime_Default
mov.l   @r15+, r14

.align 4
address_two:    .long 0x8801E000
address_one:    .long 0x8102000
Hmem_SetMMU:    .long _Hmem_SetMMU
GLibAddinAplExecutionCheck:     .long _GLibAddinAplExecutionCheck
CallbackAtQuitMainFunction:     .long _CallbackAtQuitMainFunction
Bdel_cychdr:    .long _Bdel_cychdr
BfileFLS_CloseFile:     .long _BfileFLS_CloseFile
flsFindClose:   .long _flsFindClose
Bkey_Set_RepeatTime_Default:    .long _Bkey_Set_RepeatTime_Default
bbss:      .long _bbss
ebss:      .long _ebss
edata:    .long _edata
bdata:    .long _bdata
romdata:        .long _romdata
bssdatasize:    .long _bssdatasize

exit_handler:   .long _exit_handler
main:      .long _main

_Hmem_SetMMU:
mov.l   sc_addr, r2
mov.l   1f, r0
jmp     @r2
nop
1:      .long 0x3FA

_Bdel_cychdr:
mov.l   sc_addr, r2
mov.l   1f, r0
jmp     @r2
nop
1:      .long 0x119

_BfileFLS_CloseFile:
mov.l   sc_addr, r2
mov.l   1f, r0
jmp     @r2
nop
1:      .long 0x1E7

_Bkey_Set_RepeatTime_Default:
mov.l   sc_addr, r2
mov.l   1f, r0
jmp     @r2
nop
1:      .long 0x244

_CallbackAtQuitMainFunction:
mov.l   sc_addr, r2
mov.l   1f, r0
jmp     @r2
nop
1:      .long 0x494

_flsFindClose:
mov.l   sc_addr, r2
mov.l   1f, r0
jmp     @r2
nop
1:      .long 0x218

_GLibAddinAplExecutionCheck:
mov.l   sc_addr, r2
mov     #0x13, r0
jmp     @r2
nop
sc_addr:        .long 0x80010070
.end
```

Place both these files in the project directory.  
You will also need *libfx.a* and *dispbios.h, endian.h, filebios.h, fxlib.h,
keybios.h* and *timer.h*.  
  - *libfx.a* can be obtained from the archive above.  
  This file can be created either by using *sh-elf-convrenesaslib* which is
  included with prebuilt GCC toolchains from
  https://gcc-renesas.com/sh/download-latest-toolchains/ or
  by manual extraction of modules from *fx9860G_library.lib*.
  Extracting them manually only involves using *optlnk.exe* from SHC already
  included with the official Casio SDK.
  (Found at `C:\Program Files\CASIO\fx-9860G SDK\OS\SH\BIN\`).  
  *fx9860G_library.lib* can found in
  `C:\Program Files\CASIO\fx-9860G SDK\OS\FX\lib\` directory.  
  Listing of the modules can be done with
  `optlnk -FO=library -LIB=fx9860G_library -LIS=fx9860_library.lst`
  and extracting them with
  `optlnk -FO=library -LIB=fx9860G_library -LIS=fx9860_library.lst
  -OU=%module%.obj -EXT=%module%`, where `%module%` is the function name listed
  in the *fx9860_library.lst* (extract them individually).
  - *dispbios.h, endian.h, filebios.h, fxlib.h,
  keybios.h* and *timer.h* can be found in the archive or at
  `C:\Program Files\CASIO\fx-9860G SDK\OS\FX\include\`.

Suppose your project consists of a single C source file *addin.c* and you have
*libfx.a* and *include/* directory with the official SDK header files.  
To compile your project:
```
$ sh3eb-elf-gcc -m3 -mb -mrenesas -ffreestanding -nostdlib -T addin.ld crt0.s addin.c -o addin.elf -I include -lgcc -L . -lfx -O2
```

Options of Interest:  

| GCC Toggle     | Effect                             |
| -------------- | ---------------------------------- |
| -O9            | Enables full optimizations         |
| -m3            | Generate code for SH3              |
| -mb            | Generate big-endian code           |
| -mrenesas      | Produce code compatible with fxlib |
| -ffreestanding | Do not assume existence of stdlib  |
| -nostdlib      | stdlib is contained within fxlib   |
| -lgcc          | Link against libgcc (required!)    |
| -lfx           | Link against fxlib                 |


After compilation, an ELF file is obtained. We need it in binary. Therefore:  
```
sh3eb-elf-objcopy -R .comment -R .bss -O binary addin.elf addin.bin
```

Then, we need to generate a header so that our binary file is turned into
something that can be understood by the calculator. Do:  
```
g1a-wrapper addin.bin -o addin.g1a -i icon.bmp
```

This generates *addin.g1a* that can be installed by the calculator.


# Information taken from:
  - https://www.planet-casio.com/Fr/programmation/tutoriels.php?id=61 (in French)
  - https://web.archive.org/web/20110112033531/https://sourceforge.net/apps/trac/fxsdk/wiki/UsingGCC
  - http://www.ifp.illinois.edu/%7Enakazato/tips/xgcc.html
  - https://www.nongnu.org/avr-libc/user-manual/install_tools.html
  - https://wiki.gentoo.org/wiki/Embedded_Handbook/General/Creating_a_cross-compiler
  - https://gcc.gnu.org/onlinedocs/gcc/SH-Options.html#SH-Options
  - Extracting modules from .lib
    - http://www.casiopeia.net/forum/viewtopic.php?t=1472
