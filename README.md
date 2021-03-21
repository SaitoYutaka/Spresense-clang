# Spresense-clang
How to build Spresense SDK with clang.

:warning:
This page is not still complete.  
I can not run nuttx on spresense.  
So this page say that I tried to build with clang.  
I think that It may help someone to try build nuttx with clang.  
Please some advice if you find something wrong.

# Setting up environment

## Spresense SDK Getting Started Guide (CLI)

I used the following page as a reference.  
https://developer.sony.com/develop/spresense/docs/sdk_set_up_en.html

## Clang

Download clang(Ver11.0.0).

https://releases.llvm.org/download.html

```
$ clang --version
clang version 11.0.0 (https://github.com/llvm/llvm-project.git 0160ad802e899c2922bc9b29564080c22eb0908c)
Target: x86_64-unknown-linux-gnu
Thread model: posix
InstalledDir:
```

## compiler-rt

Build [compiler-rt](https://llvm.org/docs/HowToCrossCompileBuiltinsOnArm.html) for armv7-m.

```
cmake ../ -G Ninja \
-DCMAKE_TRY_COMPILE_TARGET_TYPE=STATIC_LIBRARY \
-DCOMPILER_RT_OS_DIR="baremetal" \
-DCOMPILER_RT_BUILD_BUILTINS=ON \
-DCOMPILER_RT_BUILD_SANITIZERS=OFF \
-DCOMPILER_RT_BUILD_XRAY=OFF \
-DCOMPILER_RT_BUILD_LIBFUZZER=OFF \
-DCOMPILER_RT_BUILD_MEMPROF=OFF \
-DCOMPILER_RT_BUILD_PROFILE=OFF \
-DCMAKE_C_COMPILER=/home/saitoyutaka/clang/clang+llvm-11.0.0-x86_64-linux-gnu-ubuntu-20.04/bin/clang-11 \
-DCMAKE_EXE_LINKER_FLAGS="-fuse-ld=lld" \
-DCMAKE_C_COMPILER_TARGET="arm-none-eabi" \
-DCMAKE_ASM_COMPILER_TARGET="arm-none-eabi" \
-DCMAKE_AR=/home/saitoyutaka/clang/clang+llvm-11.0.0-x86_64-linux-gnu-ubuntu-20.04/bin/llvm-ar \
-DCMAKE_NM=/home/saitoyutaka/clang/clang+llvm-11.0.0-x86_64-linux-gnu-ubuntu-20.04/bin/llvm-nm \
-DCMAKE_RANLIB=/home/saitoyutaka/clang/clang+llvm-11.0.0-x86_64-linux-gnu-ubuntu-20.04/bin/llvm-ranlib \
-DCOMPILER_RT_BAREMETAL_BUILD=ON \
-DCOMPILER_RT_DEFAULT_TARGET_ONLY=ON \
-DLLVM_CONFIG_PATH=/home/saitoyutaka/clang/clang+llvm-11.0.0-x86_64-linux-gnu-ubuntu-20.04/bin/llvm-config \
-DCMAKE_C_FLAGS="-DCOMPILER_RT_ARMHF_TARGET -march=armv7-m -mcpu=cortex-m4 -mfloat-abi=hard -mfpu=vfp -mthumb -fno-strict-aliasing -fomit-frame-pointer -I/home/saitoyutaka/arm-gcc/gcc-arm-none-eabi-10-2020-q4-major/arm-none-eabi/include/ --sysroot=/home/saitoyutaka/arm-gcc/gcc-arm-none-eabi-10-2020-q4-major/arm-none-eabi/include/" -DCMAKE_ASM_FLAGS="-DCOMPILER_RT_ARMHF_TARGET -march=armv7-m -mcpu=cortex-m4 -mfloat-abi=hard -mfpu=vfp -mthumb" 
```

libclang_rt.builtins-arm.a will be created.

```
$ file libclang_rt.builtins-arm.a
libclang_rt.builtins-arm.a: current ar archive
```

# Edit Spresense SDK files

## sdk/tools/scripts/Make.defs

I had modified [sdk/tools/scripts/Make.defs](https://github.com/sonydevworld/spresense/blob/master/sdk/tools/scripts/Make.defs) as follows.

```diff
git diff sdk/tools/scripts/Make.defs
diff --git a/sdk/tools/scripts/Make.defs b/sdk/tools/scripts/Make.defs
index 4eb5ddaa..6ddc8742 100644
--- a/sdk/tools/scripts/Make.defs
+++ b/sdk/tools/scripts/Make.defs
@@ -61,15 +61,15 @@ else
   HOSTEXEEXT =
 endif
 
-CC = $(CROSSDEV)gcc
-CXX = $(CROSSDEV)g++
-CPP = $(CROSSDEV)gcc -E
-LD = $(CROSSDEV)ld
-AR = $(ARCROSSDEV)ar rcs
-NM = $(ARCROSSDEV)nm
-OBJCOPY = $(CROSSDEV)objcopy
-OBJDUMP = $(CROSSDEV)objdump
-STRIP = $(CROSSDEV)strip
+CC = clang
+CXX = clang++
+CPP = clang -E
+LD = ld.lld
+AR = llvm-ar rcs
+NM = llvm-nm
+OBJCOPY = llvm-objcopy
+OBJDUMP = llvm-objdump
+STRIP = llvm-strip
 
 MKNXFLAT = mknxflat
 LDNXFLAT = ldnxflat
@@ -82,16 +82,22 @@ ifeq ($(CONFIG_DEBUG_SYMBOLS),y)
 endif
 
 ifneq ($(CONFIG_DEBUG_NOOPT),y)
-  ARCHOPTIMIZATION += $(MAXOPTIMIZATION) -fno-strict-aliasing -fno-strength-reduce -fomit-frame-pointer
+  #ARCHOPTIMIZATION += $(MAXOPTIMIZATION) -fno-strict-aliasing -fno-strength-reduce -fomit-frame-pointer
+  ARCHOPTIMIZATION += $(MAXOPTIMIZATION) -fno-strict-aliasing -fomit-frame-pointer
 endif
 
 ARCHCFLAGS = -fno-builtin -mabi=aapcs -ffunction-sections -fdata-sections
-ARCHCXXFLAGS = -fno-builtin -fno-exceptions -fno-rtti -std=c++98
+ARCHCXXFLAGS = -fno-builtin -fno-exceptions -fno-rtti -std=c++98 -fcheck-new
 ARCHWARNINGS = -Wall -Wstrict-prototypes -Wshadow -Wundef
 ARCHWARNINGSXX = -Wall -Wshadow -Wundef
 ARCHDEFINES =
 ARCHPICFLAGS = -fpic -msingle-pic-base -mpic-register=r10
 
+#ARCHCFLAGS += -ffreestanding -target arm-none-eabi -march=armv7-m -mcpu=cortex-m4
+#ARCHCXXFLAGS += -ffreestanding -target arm-none-eabi -march=armv7-m -mcpu=cortex-m4 -DCONFIG_WCHAR_BUILTIN
+ARCHCFLAGS += -target arm-none-eabi -march=armv7-m -mcpu=cortex-m4
+ARCHCXXFLAGS += -target arm-none-eabi -march=armv7-m -mcpu=cortex-m4 -DCONFIG_WCHAR_BUILTIN
+
 CFLAGS = $(ARCHCFLAGS) $(ARCHWARNINGS) $(ARCHOPTIMIZATION) $(ARCHCPUFLAGS) $(ARCHINCLUDES) $(ARCHDEFINES) $(EXTRADEFINES) -pipe
 CPICFLAGS = $(ARCHPICFLAGS) $(CFLAGS)
 CXXFLAGS = $(ARCHCXXFLAGS) $(ARCHWARNINGSXX) $(ARCHOPTIMIZATION) $(ARCHCPUFLAGS) $(ARCHXXINCLUDES) $(ARCHDEFINES) $(EXTRADEFINES) -pipe
@@ -129,12 +135,13 @@ endif
 ASMEXT = .S
 OBJEXT = .o
 LIBEXT = .a
-EXEEXT =
+EXEEXT = 
 
+# LDFLAGS += -target arm-none-eabi -march=armv7-m -mcpu=cortex-m4
 LDFLAGS += --gc-sections
 
 ifneq ($(CROSSDEV),arm-nuttx-elf-)
-  LDFLAGS += -nostartfiles -nodefaultlibs
+   #LDFLAGS += -nostartfiles -nodefaultlibs
 endif
 ifeq ($(CONFIG_DEBUG_SYMBOLS),y)
   CFLAGS += -gdwarf-3
@@ -147,14 +154,14 @@ endif
 ifeq ($(WINTOOL),y)
   LDFLAGS += -Map="${shell cygpath -w $(TOPDIR)/nuttx.map}" --cref
 else
-  LDFLAGS += -Map=$(TOPDIR)/nuttx.map --cref
+   LDFLAGS += -Map=$(TOPDIR)/nuttx.map --cref
 endif
 
 ifneq ($(CONFIG_ASMP_MEMSIZE),)
   LDFLAGS += --defsym=__reserved_ramsize=$(CONFIG_ASMP_MEMSIZE)
 endif
 
-HOSTCC = gcc
+#HOSTCC = gcc
 HOSTINCLUDES = -I.
 HOSTCFLAGS = -Wall -Wstrict-prototypes -Wshadow -Wundef -g -pipe
 HOSTLDFLAGS =
```


## sdk/configs/default/defconfig

I had modified [sdk/configs/default/defconfig](https://github.com/sonydevworld/spresense/blob/master/sdk/configs/default/defconfig) as follows.


```diff
diff --git a/sdk/configs/default/defconfig b/sdk/configs/default/defconfig
index 370158b7..ebb2316b 100644
--- a/sdk/configs/default/defconfig
+++ b/sdk/configs/default/defconfig
@@ -20,7 +20,7 @@ CONFIG_ARCH_BOARD_SPRESENSE=y
 CONFIG_ARCH_CHIP="cxd56xx"
 CONFIG_ARCH_CHIP_CXD56XX=y
 CONFIG_ARCH_INTERRUPTSTACK=2048
-CONFIG_ARCH_MATH_H=y
+#CONFIG_ARCH_MATH_H=y
 CONFIG_ARCH_STACKDUMP=y
 CONFIG_ARMV7M_USEBASEPRI=y
 CONFIG_BATTERY_CHARGER=y
@@ -69,7 +69,8 @@ CONFIG_HAVE_CXX=y
 CONFIG_HAVE_CXXINITIALIZE=y
 CONFIG_LCD=y
 CONFIG_LCD_NOGETRUN=y
-CONFIG_LIBC_FLOATINGPOINT=y
+#CONFIG_LIBC_FLOATINGPOINT=y
+CONFIG_LIBM=y
 CONFIG_LIBC_IPv4_ADDRCONV=y
 CONFIG_LIBC_IPv6_ADDRCONV=y
 CONFIG_LIB_KBDCODEC=y
@@ -151,3 +152,239 @@ CONFIG_USBMSC_VENDORID=0x054c
 CONFIG_USBMSC_VENDORSTR="Sony"
 CONFIG_USERMAIN_STACKSIZE=8192
 CONFIG_USER_ENTRYPOINT="spresense_main"
+CONFIG_DEBUG_HARDFAULT_ALERT=y
+CONFIG_DEBUG_HARDFAULT_INFO=y
+CONFIG_HAVE_FUNCTIONNAME=y
+CONFIG_CPP_HAVE_VARARGS=y
+CONFIG_DEBUG_ALERT=y
+CONFIG_DEBUG_ERROR=y
+CONFIG_DEBUG_WARN=y
+CONFIG_DEBUG_INFO=y
+CONFIG_DEBUG_MM_ERROR=y
+CONFIG_DEBUG_MM_WARN=y
+CONFIG_DEBUG_MM_INFO=y
+CONFIG_DEBUG_SCHED_ERROR=y
+CONFIG_DEBUG_SCHED_WARN=y
+CONFIG_DEBUG_SCHED_INFO=y
+CONFIG_DEBUG_SYSCALL_ERROR=y
+CONFIG_DEBUG_SYSCALL_WARN=y
+CONFIG_DEBUG_SYSCALL_INFO=y
+CONFIG_DEBUG_PAGING_ERROR=y
+CONFIG_DEBUG_PAGING_WARN=y
+CONFIG_DEBUG_PAGING_INFO=y
+CONFIG_DEBUG_NET_ERROR=y
+CONFIG_DEBUG_NET_WARN=y
+CONFIG_DEBUG_NET_INFO=y
+CONFIG_DEBUG_POWER_ERROR=y
+CONFIG_DEBUG_POWER_WARN=y
+CONFIG_DEBUG_POWER_INFO=y
+CONFIG_DEBUG_WIRELESS_ERROR=y
+CONFIG_DEBUG_WIRELESS_WARN=y
+CONFIG_DEBUG_WIRELESS_INFO=y
+CONFIG_DEBUG_FS_ERROR=y
+CONFIG_DEBUG_FS_WARN=y
+CONFIG_DEBUG_FS_INFO=y
+CONFIG_DEBUG_CONTACTLESS_ERROR=y
+CONFIG_DEBUG_CONTACTLESS_WARN=y
+CONFIG_DEBUG_CONTACTLESS_INFO=y
+CONFIG_DEBUG_CRYPTO_ERROR=y
+CONFIG_DEBUG_CRYPTO_WARN=y
+CONFIG_DEBUG_CRYPTO_INFO=y
+CONFIG_DEBUG_INPUT_ERROR=y
+CONFIG_DEBUG_INPUT_WARN=y
+CONFIG_DEBUG_INPUT_INFO=y
+CONFIG_DEBUG_ANALOG_ERROR=y
+CONFIG_DEBUG_ANALOG_WARN=y
+CONFIG_DEBUG_ANALOG_INFO=y
+CONFIG_DEBUG_CAN_ERROR=y
+CONFIG_DEBUG_CAN_WARN=y
+CONFIG_DEBUG_CAN_INFO=y
+CONFIG_DEBUG_GRAPHICS_ERROR=y
+CONFIG_DEBUG_GRAPHICS_WARN=y
+CONFIG_DEBUG_GRAPHICS_INFO=y
+CONFIG_DEBUG_BINFMT_ERROR=y
+CONFIG_DEBUG_BINFMT_WARN=y
+CONFIG_DEBUG_BINFMT_INFO=y
+CONFIG_DEBUG_LIB_ERROR=y
+CONFIG_DEBUG_LIB_WARN=y
+CONFIG_DEBUG_LIB_INFO=y
+CONFIG_DEBUG_AUDIO_ERROR=y
+CONFIG_DEBUG_AUDIO_WARN=y
+CONFIG_DEBUG_AUDIO_INFO=y
+CONFIG_DEBUG_DMA_ERROR=y
+CONFIG_DEBUG_DMA_WARN=y
+CONFIG_DEBUG_DMA_INFO=y
+CONFIG_DEBUG_IRQ_ERROR=y
+CONFIG_DEBUG_IRQ_WARN=y
+CONFIG_DEBUG_IRQ_INFO=y
+CONFIG_DEBUG_LCD_ERROR=y
+CONFIG_DEBUG_LCD_WARN=y
+CONFIG_DEBUG_LCD_INFO=y
+CONFIG_DEBUG_LEDS_ERROR=y
+CONFIG_DEBUG_LEDS_WARN=y
+CONFIG_DEBUG_LEDS_INFO=y
+CONFIG_DEBUG_GPIO_ERROR=y
+CONFIG_DEBUG_GPIO_WARN=y
+CONFIG_DEBUG_GPIO_INFO=y
+CONFIG_DEBUG_I2C_ERROR=y
+CONFIG_DEBUG_I2C_WARN=y
+CONFIG_DEBUG_I2C_INFO=y
+CONFIG_DEBUG_I2S_ERROR=y
+CONFIG_DEBUG_I2S_WARN=y
+CONFIG_DEBUG_I2S_INFO=y
+CONFIG_DEBUG_PWM_ERROR=y
+CONFIG_DEBUG_PWM_WARN=y
+CONFIG_DEBUG_PWM_INFO=y
+CONFIG_DEBUG_RTC_ERROR=y
+CONFIG_DEBUG_RTC_WARN=y
+CONFIG_DEBUG_RTC_INFO=y
+CONFIG_DEBUG_MEMCARD_ERROR=y
+CONFIG_DEBUG_MEMCARD_WARN=y
+CONFIG_DEBUG_MEMCARD_INFO=y
+CONFIG_DEBUG_SENSORS_ERROR=y
+CONFIG_DEBUG_SENSORS_WARN=y
+CONFIG_DEBUG_SENSORS_INFO=y
+CONFIG_DEBUG_SPI_ERROR=y
+CONFIG_DEBUG_SPI_WARN=y
+CONFIG_DEBUG_SPI_INFO=y
+CONFIG_DEBUG_TIMER_ERROR=y
+CONFIG_DEBUG_TIMER_WARN=y
+CONFIG_DEBUG_TIMER_INFO=y
+CONFIG_DEBUG_USB_ERROR=y
+CONFIG_DEBUG_USB_WARN=y
+CONFIG_DEBUG_USB_INFO=y
+CONFIG_DEBUG_WATCHDOG_ERROR=y
+CONFIG_DEBUG_WATCHDOG_WARN=y
+CONFIG_DEBUG_WATCHDOG_INFO=y
+CONFIG_DEBUG_MM_ERROR=y
+CONFIG_DEBUG_MM_WARN=y
+CONFIG_DEBUG_MM_INFO=y
+CONFIG_DEBUG_SCHED_ERROR=y
+CONFIG_DEBUG_SCHED_WARN=y
+CONFIG_DEBUG_SCHED_INFO=y
+CONFIG_DEBUG_SYSCALL_ERROR=y
+CONFIG_DEBUG_SYSCALL_WARN=y
+CONFIG_DEBUG_SYSCALL_INFO=y
+CONFIG_DEBUG_PAGING_ERROR=y
+CONFIG_DEBUG_PAGING_WARN=y
+CONFIG_DEBUG_PAGING_INFO=y
+CONFIG_DEBUG_NET_ERROR=y
+CONFIG_DEBUG_NET_WARN=y
+CONFIG_DEBUG_NET_INFO=y
+CONFIG_DEBUG_POWER_ERROR=y
+CONFIG_DEBUG_POWER_WARN=y
+CONFIG_DEBUG_POWER_INFO=y
+CONFIG_DEBUG_WIRELESS_ERROR=y
+CONFIG_DEBUG_WIRELESS_WARN=y
+CONFIG_DEBUG_WIRELESS_INFO=y
+CONFIG_DEBUG_FS_ERROR=y
+CONFIG_DEBUG_FS_WARN=y
+CONFIG_DEBUG_FS_INFO=y
+CONFIG_DEBUG_CONTACTLESS_ERROR=y
+CONFIG_DEBUG_CONTACTLESS_WARN=y
+CONFIG_DEBUG_CONTACTLESS_INFO=y
+CONFIG_DEBUG_CRYPTO_ERROR=y
+CONFIG_DEBUG_CRYPTO_WARN=y
+CONFIG_DEBUG_CRYPTO_INFO=y
+CONFIG_DEBUG_INPUT_ERROR=y
+CONFIG_DEBUG_INPUT_WARN=y
+CONFIG_DEBUG_INPUT_INFO=y
+CONFIG_DEBUG_ANALOG_ERROR=y
+CONFIG_DEBUG_ANALOG_WARN=y
+CONFIG_DEBUG_ANALOG_INFO=y
+CONFIG_DEBUG_CAN_ERROR=y
+CONFIG_DEBUG_CAN_WARN=y
+CONFIG_DEBUG_CAN_INFO=y
+CONFIG_DEBUG_GRAPHICS_ERROR=y
+CONFIG_DEBUG_GRAPHICS_WARN=y
+CONFIG_DEBUG_GRAPHICS_INFO=y
+CONFIG_DEBUG_BINFMT_ERROR=y
+CONFIG_DEBUG_BINFMT_WARN=y
+CONFIG_DEBUG_BINFMT_INFO=y
+CONFIG_DEBUG_LIB_ERROR=y
+CONFIG_DEBUG_LIB_WARN=y
+CONFIG_DEBUG_LIB_INFO=y
+CONFIG_DEBUG_AUDIO_ERROR=y
+CONFIG_DEBUG_AUDIO_WARN=y
+CONFIG_DEBUG_AUDIO_INFO=y
+CONFIG_DEBUG_DMA_ERROR=y
+CONFIG_DEBUG_DMA_WARN=y
+CONFIG_DEBUG_DMA_INFO=y
+CONFIG_DEBUG_IRQ_ERROR=y
+CONFIG_DEBUG_IRQ_WARN=y
+CONFIG_DEBUG_IRQ_INFO=y
+CONFIG_DEBUG_LCD_ERROR=y
+CONFIG_DEBUG_LCD_WARN=y
+CONFIG_DEBUG_LCD_INFO=y
+CONFIG_DEBUG_LEDS_ERROR=y
+CONFIG_DEBUG_LEDS_WARN=y
+CONFIG_DEBUG_LEDS_INFO=y
+CONFIG_DEBUG_GPIO_ERROR=y
+CONFIG_DEBUG_GPIO_WARN=y
+CONFIG_DEBUG_GPIO_INFO=y
+CONFIG_DEBUG_I2C_ERROR=y
+CONFIG_DEBUG_I2C_WARN=y
+CONFIG_DEBUG_I2C_INFO=y
+CONFIG_DEBUG_I2S_ERROR=y
+CONFIG_DEBUG_I2S_WARN=y
+CONFIG_DEBUG_I2S_INFO=y
+CONFIG_DEBUG_PWM_ERROR=y
+CONFIG_DEBUG_PWM_WARN=y
+CONFIG_DEBUG_PWM_INFO=y
+CONFIG_DEBUG_RTC_ERROR=y
+CONFIG_DEBUG_RTC_WARN=y
+CONFIG_DEBUG_RTC_INFO=y
+CONFIG_DEBUG_MEMCARD_ERROR=y
+CONFIG_DEBUG_MEMCARD_WARN=y
+CONFIG_DEBUG_MEMCARD_INFO=y
+CONFIG_DEBUG_SENSORS_ERROR=y
+CONFIG_DEBUG_SENSORS_WARN=y
+CONFIG_DEBUG_SENSORS_INFO=y
+CONFIG_DEBUG_SPI_ERROR=y
+CONFIG_DEBUG_SPI_WARN=y
+CONFIG_DEBUG_SPI_INFO=y
+CONFIG_DEBUG_TIMER_ERROR=y
+CONFIG_DEBUG_TIMER_WARN=y
+CONFIG_DEBUG_TIMER_INFO=y
+CONFIG_DEBUG_USB_ERROR=y
+CONFIG_DEBUG_USB_WARN=y
+CONFIG_DEBUG_USB_INFO=y
+CONFIG_DEBUG_WATCHDOG_ERROR=y
+CONFIG_DEBUG_WATCHDOG_WARN=y
+CONFIG_DEBUG_WATCHDOG_INFO=y
+CONFIG_DEBUG_ERROR=y
+CONFIG_DEBUG_MM=y
+CONFIG_DEBUG_SCHED=y
+CONFIG_DEBUG_SYSCALL=y
+CONFIG_DEBUG_PAGING=y
+CONFIG_DEBUG_NET=y
+CONFIG_DEBUG_POWER=y
+CONFIG_DEBUG_WIRELESS=y
+CONFIG_DEBUG_FS=y
+CONFIG_DEBUG_CONTACTLESS=y
+CONFIG_DEBUG_INPUT=y
+CONFIG_DEBUG_ANALOG=y
+CONFIG_DEBUG_CAN=y
+CONFIG_DEBUG_GRAPHICS=y
+CONFIG_DEBUG_BINFMT=y
+CONFIG_DEBUG_LIB=y
+CONFIG_DEBUG_AUDIO=y
+CONFIG_DEBUG_DMA=y
+CONFIG_DEBUG_IRQ=y
+CONFIG_DEBUG_LCD=y
+CONFIG_DEBUG_LEDS=y
+CONFIG_DEBUG_GPIO=y
+CONFIG_DEBUG_I2C=y
+CONFIG_DEBUG_I2S=y
+CONFIG_DEBUG_PWM=y
+CONFIG_DEBUG_RTC=y
+CONFIG_DEBUG_MEMCARD=y
+CONFIG_DEBUG_SENSORS=y
+CONFIG_DEBUG_SPI=y
+CONFIG_DEBUG_TIMER=y
+CONFIG_DEBUG_USB=y
+CONFIG_DEBUG_WATCHDOG=y
+CONFIG_DEBUG_ALERT=y
+CONFIG_DEBUG_ERROR=y
+CONFIG_DEBUG_WARN=y
+CONFIG_DEBUG_INFO=y
```
## arch/arm/src/Makefile

I had modified [arch/arm/src/Makefile](https://github.com/sonydevworld/spresense-nuttx/blob/7686b71a7cfe17bf8f3fb1be7749594278b0c808/arch/arm/src/Makefile) as follows.  
I added libclang_rt.builtins-arm.a as a linker argument.

```diff
diff --git a/arch/arm/src/Makefile b/arch/arm/src/Makefile
index f658f371c0..cdd5369b75 100644
--- a/arch/arm/src/Makefile
+++ b/arch/arm/src/Makefile
@@ -181,10 +181,10 @@ board$(DELIM)libboard$(LIBEXT):
        $(Q) $(MAKE) -C board TOPDIR="$(TOPDIR)" libboard$(LIBEXT) EXTRADEFINES=$(EXTRADEFINES)
 
 nuttx$(EXEEXT): $(HEAD_OBJ) board$(DELIM)libboard$(LIBEXT)
-       $(Q) echo "LD: nuttx"
-       $(Q) $(LD) --entry=__start $(LDFLAGS) $(LIBPATHS) $(EXTRA_LIBPATHS) \
+       $(Q) echo "LD: nuttx $(LDFLAGS)"
+       $(Q) $(LD) -m armelf --entry=__start $(LDFLAGS) $(LIBPATHS) $(EXTRA_LIBPATHS) \
                -o $(NUTTX) $(HEAD_OBJ) $(EXTRA_OBJS) \
-               $(LDSTARTGROUP) $(LDLIBS) $(EXTRA_LIBS) $(LIBGCC) $(LDENDGROUP)
+               $(LDSTARTGROUP) $(LDLIBS) $(EXTRA_LIBS) $(LIBGCC) /home/saitoyutaka/clang/compiler-rt-11.0.0.src/build/lib/baremetal/libclang_rt.builtins-arm.a $(LDENDGROUP)
 ifneq ($(CONFIG_WINDOWS_NATIVE),y)
        $(Q) $(NM) $(NUTTX) | \
        grep -v '\(compiled\)\|\(\$(OBJEXT)$$\)\|\( [aUw] \)\|\(\.\.ng$$\)\|\(LASH[RL]DI\)' | \
```

## boards/arm/cxd56xx/spresense/scripts/ramconfig.ld

I had modified [boards/arm/cxd56xx/spresense/scripts/ramconfig.ld](https://github.com/sonydevworld/spresense-nuttx/blob/7686b71a7cfe17bf8f3fb1be7749594278b0c808/boards/arm/cxd56xx/spresense/scripts/ramconfig.ld) as follows.  
I'm not sure if this fix is better or not.

```diff
diff --git a/boards/arm/cxd56xx/spresense/scripts/ramconfig.ld b/boards/arm/cxd56xx/spresense/scripts/ramconfig.ld
index e43b556e5b..f532e58a27 100644
--- a/boards/arm/cxd56xx/spresense/scripts/ramconfig.ld
+++ b/boards/arm/cxd56xx/spresense/scripts/ramconfig.ld
@@ -104,7 +104,8 @@ SECTIONS
     /* __stack symbol is referred from mkspk tool
      * and means the end address of heap region */
     PROVIDE(__stack = ORIGIN(ram) + LENGTH(ram));
-    __stack -= DEFINED(__reserved_ramsize) ? __reserved_ramsize : 0;
+/*    __stack -= DEFINED(__reserved_ramsize) ? __reserved_ramsize : 0; */
+    __stack = __stack - 0;
 
     ASSERT(_ebss < __stack, "Error: Out of memory")
 
```

# build

First I had tried to build hello world.

```
$ cd spresense/sdk
$ tools/config.py examples/hello
$ make
...
make[2]: ディレクトリ '/home/saitoyutaka/spresense/nuttx/arch/arm/src' から出ます
Generating: nuttx.spk
File nuttx.spk is successfully created.
make[1]: ディレクトリ '/home/saitoyutaka/spresense/nuttx' から出ます
$ file nuttx
nuttx: ELF 32-bit LSB executable, ARM, EABI5 version 1 (SYSV), statically linked, with debug_info, not stripped
$ file nuttx.spk 
nuttx.spk: data
```

Download to Spresense.

```
$ tools/flash.sh -c /dev/ttyUSB0 nuttx.spk 
>>> Install files ...
install -b 115200
Install nuttx.spk
|0%-----------------------------50%------------------------------100%|
######################################################################

158432 bytes loaded.
Package validation is OK.
Saving package to "nuttx"
updater# sync
updater# Restarting the board ...
reboot
```

# Hard Fault

In the current situation, a hardware fault occurs as shown below.

```
up_hardfault: Hard Fault:
up_hardfault:   IRQ: 3 regs: 0xd03686c
up_hardfault:   BASEPRI: 000000e0 PRIMASK: 00000000 IPSR: 00000003 CONTROL: 00000000
up_hardfault:   CFAULTS: 00020000 HFAULTS: 40000000 DFAULTS: 00000000 BFAULTADDR: e000ed38 AFAULTS: 00000000
up_hardfault: PANIC!!! Hard fault: 40000000
up_assert: Assertion failed at file:armv7-m/up_hardfault.c line: 148 task: init
up_registerdump: R0: 0d023c04 0d03695c 0d00f005 00000101 0d026870 0d000000 0d02685c 0d02685c
up_registerdump: R8: 00000000 00000000 00000000 00000000 0d0343e0 0d036940 0d01e775 0d023c04
up_registerdump: xPSR: 80000000 BASEPRI: 000000e0 CONTROL: 00000000
up_registerdump: EXC_RETURN: ffffffe9
up_dumpstate: sp:     0d0282b0
up_dumpstate: IRQ stack:
up_dumpstate:   base: 0d028300
up_dumpstate:   size: 00000800
up_dumpstate:   used: 000000e0
up_stackdump: 0d0282a0: 0d0282b0 0d02311e 0d0343e0 0d0017a1 000000e0 00000000 00000000 0d0343e0
up_stackdump: 0d0282c0: 0d036940 0d01e775 0d023c04 00000094 40000000 00000000 e000ed38 00000000
up_stackdump: 0d0282e0: 00000000 0d00450d deadbeef 0d03686c 0d000000 0d02685c 0d02685c 0d00449f
up_dumpstate: sp:     0d036940
up_dumpstate: User stack:
up_dumpstate:   base: 0d036968
up_dumpstate:   size: 00001fec
up_dumpstate:   used: 00000100
up_stackdump: 0d036940: 00000000 00000000 00000000 00000000 00000000 0d00f02b 00000000 00000064
up_stackdump: 0d036960: 00000000 0d0032a1 deadbeef 0d036974 00000000 74696e69 deadbe00 8c574e5e
up_taskdump: Idle Task: PID=0 Stack Used=0 of 0
up_taskdump: hpwork: PID=1 Stack Used=304 of 2028
up_taskdump: lpwork: PID=2 Stack Used=304 of 2028
up_taskdump: lpwork: PID=3 Stack Used=304 of 2028
up_taskdump: lpwork: PID=4 Stack Used=304 of 2028
up_taskdump: cxd56_pm_task: PID=6 Stack Used=328 of 996
up_taskdump: init: PID=7 Stack Used=256 of 8172
```

