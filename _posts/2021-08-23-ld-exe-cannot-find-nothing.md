---
layout: post
comments: true
title: "Solution for /bin/ld.exe: cannot find : Invalid argument collect2.exe: error: ld returned 1 exit status"
excerpt: "Linker can not find nothing. How I repaired that."
date: 2021-08-23 23:00:00
tags: c++, linker, error, collect2, cannot find nothing
---

I decided to try build my [NetBurner's CMake template](https://github.com/lopshopedun/cmake-netburner-demo) in NNDK 2.9.x that ships with GCC 8.1.0.

I repeated the steps in the README and suddenly got a strange linker error:

```
C:/nburn/gcc-m68k/bin/m68k-elf-g++.exe 
--sysroot=C:/nburn/gcc-m68k/bin/../m68k-unknown-elf/sysroot -mcpu=54415 
-gdwarf-2 -falign-functions=4 -ffunction-sections -fmessage-length=0 
-fdata-sections -fno-rtti -DMOD5441X -DMCF5441X -DNBMINGW 
-IC:/nburn/gcc-m68k/m68k-unknown-elf/include      -IC:/nburn/include      
-IC:/nburn/MOD5441X/include     
-IC:/nburn/gcc-m68k/m68k-unknown-elf/include/c++/8.1.0      -O3     -DNDEBUG   
-mcpu=54415 -Wl,-n -TC:/nburn/MOD5441X/lib/MOD5441X.ld         
-Wl,-RC:/nburn/MOD5441X/lib/sys.ld  
"CMakeFiles/cmake-netburner-hello-world_0.0.5.dir/main.cpp.obj" 
"CMakeFiles/cmake-netburner-hello-world_0.0.5.dir/htmldata.cpp.obj"  
-o Release/cmake-netburner-hello-world_0.0.5.elf  
-Wl,-Map=G:/cmake-netburner-hello-world/build_m68k_gcc8/Release/cmake-netburner-hello-world_0.0.5.map  
-Wl,--start-group, C:/nburn/lib/MOD5441X.a C:/nburn/lib/NetBurner.a 
C:/nburn/lib/FatFile.a C:/nburn/lib/StdFFile.a -lstdc++  -Wl,--end-group 
-Wl,--gc-sections
c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/bin/ld.exe: cannot find : Invalid argument
collect2.exe: error: ld returned 1 exit status
make.exe[3]: *** [CMakeFiles/cmake-netburner-hello-world_0.0.5.dir/build.make:117: Release/cmake-netburner-hello-world_0.0.5.elf] Error 1
make.exe[3]: Leaving directory 'G:/cmake-netburner-hello-world/build_m68k_gcc8'
make.exe[2]: *** [CMakeFiles/Makefile2:126: CMakeFiles/cmake-netburner-hello-world_0.0.5.dir/all] Error 2
make.exe[2]: Leaving directory 'G:/cmake-netburner-hello-world/build_m68k_gcc8'
make.exe[1]: *** [CMakeFiles/Makefile2:106: CMakeFiles/app.dir/rule] Error 2
make.exe[1]: Leaving directory 'G:/cmake-netburner-hello-world/build_m68k_gcc8'
make.exe: *** [Makefile:137: app] Error 2
```

It says that it can not find nothing:

```
c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/bin/ld.exe: cannot find : Invalid argument
collect2.exe: error: ld returned 1 exit status
```

I really don't understand from that message what is the cause of the error. Is there something expanded to empty string?

Googling didn't help me because in the majority of cases on StackOverflow were asked simple cases.

`-Bstatic` and similar flags don't used for my case.

In the project a custom [\*.cmake](https://github.com/lopshopedun/cmake-netburner-demo/blob/develop/cmake/m68k-unknown-elf.cmake) file for NetBurner toolchain is used to configure a build.

It calls the helper function where a linking is defined in the same manner as it done in the original makefiles from NetBurner.

For GCC 5.2.0 it worked, but for GCC 8.1.0 it doesn't. Why? How to repair my cmake scripts?

### Solution

Let's try to add linker options `-t --verbose` in [that line](https://github.com/lopshopedun/cmake-netburner-demo/blob/develop/cmake/m68k-unknown-elf.cmake#L106) after the first `-Wl,` symbols and look at the huge output:

<details>
  <summary>Click to expand
</summary>

```
$ cmake --build . --target app -- VERBOSE=1
"C:/Program Files/CMake/bin/cmake.exe" -SG:/cmake-netburner-hello-world -BG:/cmake-netburner-hello-world/build_m68k_gcc8 --check-build-system CMakeFiles/Makefile.cmake 0
C:/nburn/gcc-m68k/bin/make.exe  -f CMakeFiles/Makefile2 app
make.exe[1]: Entering directory 'G:/cmake-netburner-hello-world/build_m68k_gcc8'
"C:/Program Files/CMake/bin/cmake.exe" -SG:/cmake-netburner-hello-world -BG:/cmake-netburner-hello-world/build_m68k_gcc8 --check-build-system CMakeFiles/Makefile.cmake 0
"C:/Program Files/CMake/bin/cmake.exe" -E cmake_progress_start G:/cmake-netburner-hello-world/build_m68k_gcc8/CMakeFiles 3
C:/nburn/gcc-m68k/bin/make.exe  -f CMakeFiles/Makefile2 CMakeFiles/app.dir/all
make.exe[2]: Entering directory 'G:/cmake-netburner-hello-world/build_m68k_gcc8'
C:/nburn/gcc-m68k/bin/make.exe  -f CMakeFiles/cmake-netburner-hello-world_0.0.5.dir/build.make CMakeFiles/cmake-netburner-hello-world_0.0.5.dir/depend
make.exe[3]: Entering directory 'G:/cmake-netburner-hello-world/build_m68k_gcc8'
"C:/Program Files/CMake/bin/cmake.exe" -E cmake_depends "Unix Makefiles" G:/cmake-netburner-hello-world G:/cmake-netburner-hello-world G:/cmake-netburner-hello-world/build_m68k_gcc8 G:/cmake-netburner-hello-world/build_m68k_gcc8 G:/cmake-netburner-hello-world/build_m68k_gcc8/CMakeFiles/cmake-netburner-hello-world_0.0.5.dir/DependInfo.cmake --color=
make.exe[3]: Leaving directory 'G:/cmake-netburner-hello-world/build_m68k_gcc8'
C:/nburn/gcc-m68k/bin/make.exe  -f CMakeFiles/cmake-netburner-hello-world_0.0.5.dir/build.make CMakeFiles/cmake-netburner-hello-world_0.0.5.dir/build
make.exe[3]: Entering directory 'G:/cmake-netburner-hello-world/build_m68k_gcc8'
[ 33%] Linking CXX executable Release/cmake-netburner-hello-world_0.0.5.elf
C:/nburn/gcc-m68k/bin/m68k-elf-g++.exe --sysroot=C:/nburn/gcc-m68k/bin/../m68k-unknown-elf/sysroot 
-mcpu=54415 -gdwarf-2 -falign-functions=4 -ffunction-sections -fmessage-length=0 
-fdata-sections -fno-rtti -DMOD5441X -DMCF5441X -DNBMINGW 
-IC:/nburn/gcc-m68k/m68k-unknown-elf/include      -I"C:\nburn/include"    
-I"C:\nburn/MOD5441X/include"   
-I"C:\nburn/gcc-m68k/m68k-unknown-elf/include/c++/8.1.0"    -O3     -DNDEBUG 
-mcpu=54415 -Wl,-t --verbose -Wl,-n -Wl,-TC:/nburn/MOD5441X/lib/MOD5441X.ld 
-Wl,-RC:/nburn/MOD5441X/lib/sys.ld  
-Wl,-Map=G:/cmake-netburner-hello-world/build_m68k_gcc8/Release/cmake-netburner-hello-world_0.0.5.map 
./CMakeFiles/cmake-netburner-hello-world_0.0.5.dir/main.cpp.obj 
./CMakeFiles/cmake-netburner-hello-world_0.0.5.dir/htmldata.cpp.obj  
-o Release/cmake-netburner-hello-world_0.0.5.elf -Wl,--gc-sections 
-Wl,--start-group, C:/nburn/lib/MOD5441X.a C:/nburn/lib/NetBurner.a 
C:/nburn/lib/FatFile.a C:/nburn/lib/StdFFile.a -lstdc++ -Wl,--end-group
Using built-in specs.
COLLECT_GCC=C:\nburn\gcc-m68k\bin\m68k-elf-g++.exe
COLLECT_LTO_WRAPPER=c:/nburn/gcc-m68k/bin/../libexec/gcc/m68k-unknown-elf/8.1.0/lto-wrapper.exe
Target: m68k-unknown-elf
Configured with: /home/ubuntu/build/.build/HOST-mingw32/m68k-unknown-elf/src/gcc/configure 
--build=x86_64-build_pc-linux-gnu --host=i686-host_pc-mingw32 
--target=m68k-unknown-elf 
--prefix=/home/ubuntu/x-tools/HOST-mingw32/m68k-unknown-elf 
--with-local-prefix=/home/ubuntu/x-tools/HOST-mingw32/m68k-unknown-elf/m68k-unknown-elf 
--with-headers=/home/ubuntu/x-tools/HOST-mingw32/m68k-unknown-elf/m68k-unknown-elf/include 
--with-newlib --enable-threads=no --disable-shared 
--with-pkgversion='crosstool-NG crosstool-ng-1.23.0-328-g590eddf' 
--enable-__cxa_atexit --disable-libgomp --disable-libmudflap --disable-libmpx 
--disable-libssp --disable-libquadmath --disable-libquadmath-support 
--with-gmp=/home/ubuntu/build/.build/HOST-mingw32/m68k-unknown-elf/buildtools/complibs-host 
--with-mpfr=/home/ubuntu/build/.build/HOST-mingw32/m68k-unknown-elf/buildtools/complibs-host 
--with-mpc=/home/ubuntu/build/.build/HOST-mingw32/m68k-unknown-elf/buildtools/complibs-host 
--with-isl=/home/ubuntu/build/.build/HOST-mingw32/m68k-unknown-elf/buildtools/complibs-host 
--disable-lto --with-host-libstdcxx='-static-libgcc -Wl,-Bstatic,-lstdc++ -lm' 
--enable-target-optspace --disable-nls --enable-multiarch 
--with-multilib-list=rmprofile --enable-languages=c,c++
Thread model: single
gcc version 8.1.0 (crosstool-NG crosstool-ng-1.23.0-328-g590eddf)
COMPILER_PATH=c:/nburn/gcc-m68k/bin/../libexec/gcc/m68k-unknown-elf/8.1.0/;c:/nburn/gcc-m68k/bin/../libexec/gcc/;c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/bin/
LIBRARY_PATH=c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/m54455/;c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455/;c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/;c:/nburn/gcc-m68k/bin/../lib/gcc/;c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/
COLLECT_GCC_OPTIONS='-mcpu=54415' '-gdwarf-2' '-falign-functions=4' 
'-ffunction-sections' '-fmessage-length=0' '-fdata-sections' '-fno-rtti' '-D' 
'MOD5441X' '-D' 'MCF5441X' '-D' 'NBMINGW' '-I' 
'C:/nburn/gcc-m68k/m68k-unknown-elf/include' '-I' 'C:\nburn/include' '-I' 
'C:\nburn/MOD5441X/include' '-I' 
'C:\nburn/gcc-m68k/m68k-unknown-elf/include/c++/8.1.0' '-O3' '-D' 'NDEBUG' 
'-mcpu=54415' '-v' '-o' 'Release/cmake-netburner-hello-world_0.0.5.elf'
 c:/nburn/gcc-m68k/bin/../libexec/gcc/m68k-unknown-elf/8.1.0/collect2.exe 
 --sysroot=C:/nburn/gcc-m68k/bin/../m68k-unknown-elf/sysroot 
 -o Release/cmake-netburner-hello-world_0.0.5.elf 
 c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/m54455/crtbegin.o 
 -Lc:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/m54455 
 -Lc:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455 
 -Lc:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0 
 -Lc:/nburn/gcc-m68k/bin/../lib/gcc 
 -Lc:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib 
 -t -n -TC:/nburn/MOD5441X/lib/MOD5441X.ld -RC:/nburn/MOD5441X/lib/sys.ld 
 -Map=G:/cmake-netburner-hello-world/build_m68k_gcc8/Release/cmake-netburner-hello-world_0.0.5.map 
 ./CMakeFiles/cmake-netburner-hello-world_0.0.5.dir/main.cpp.obj 
 ./CMakeFiles/cmake-netburner-hello-world_0.0.5.dir/htmldata.cpp.obj 
 --gc-sections --start-group "" C:/nburn/lib/MOD5441X.a C:/nburn/lib/NetBurner.a 
 C:/nburn/lib/FatFile.a C:/nburn/lib/StdFFile.a -lstdc++ --end-group -lstdc++ 
 -lm -lgcc -lc -lgcc 
 c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/m54455/crtend.o
c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/bin/ld.exe: mode m68kelf
nb-crt0.o (c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/m54455\nb-crt0.o)
c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/m54455/crtbegin.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libc.a)lib_a-atexit.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libc.a)lib_a-__atexit.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libc.a)lib_a-__call_atexit.o
./CMakeFiles/cmake-netburner-hello-world_0.0.5.dir/main.cpp.obj
./CMakeFiles/cmake-netburner-hello-world_0.0.5.dir/htmldata.cpp.obj
(C:/nburn/lib/NetBurner.a)ucosmcfa.o
(C:/nburn/lib/NetBurner.a)main.o
(C:/nburn/lib/NetBurner.a)ucos.o
(C:/nburn/lib/NetBurner.a)cppmain.o
(C:/nburn/lib/NetBurner.a)ucosmain.o
(C:/nburn/lib/NetBurner.a)fileio.o
(C:/nburn/lib/NetBurner.a)ip.o
(C:/nburn/lib/NetBurner.a)buffers.o
(C:/nburn/lib/NetBurner.a)arp.o
(C:/nburn/lib/NetBurner.a)udp.o
(C:/nburn/lib/NetBurner.a)tcp.o
(C:/nburn/lib/NetBurner.a)http.o
(C:/nburn/lib/NetBurner.a)httpinternal.o
(C:/nburn/lib/NetBurner.a)httpstricmp.o
(C:/nburn/lib/NetBurner.a)htmldecomp.o
(C:/nburn/lib/NetBurner.a)iosys.o
(C:/nburn/lib/NetBurner.a)new_ops.o
(C:/nburn/lib/NetBurner.a)autoip.o
(C:/nburn/lib/NetBurner.a)dhcpc.o
(C:/nburn/lib/NetBurner.a)netinterface.o
(C:/nburn/lib/NetBurner.a)taskmon.o
(C:/nburn/lib/NetBurner.a)smarttrap.o
(C:/nburn/lib/NetBurner.a)autoupdate.o
(C:/nburn/lib/NetBurner.a)device.o
(C:/nburn/lib/NetBurner.a)base64.o
(C:/nburn/lib/NetBurner.a)nbiprintf.o
(C:/nburn/lib/NetBurner.a)nbprintf.o
(C:/nburn/lib/NetBurner.a)nbsiprintf.o
(C:/nburn/lib/NetBurner.a)nbsprintf.o
(C:/nburn/lib/NetBurner.a)weakumain.o
(C:/nburn/lib/NetBurner.a)cusermain.o
(C:/nburn/lib/NetBurner.a)websocket_handler.o
(C:/nburn/lib/NetBurner.a)ipv6_addr.o
(C:/nburn/lib/NetBurner.a)ipv6_tcp.o
(C:/nburn/lib/NetBurner.a)ipv6_ip.o
(C:/nburn/lib/NetBurner.a)ipv6_udp.o
(C:/nburn/lib/NetBurner.a)ipv6_printhelp.o
(C:/nburn/lib/NetBurner.a)ipv6_extensions.o
(C:/nburn/lib/NetBurner.a)ipv6_multicast.o
(C:/nburn/lib/NetBurner.a)ipv6_show.o
(C:/nburn/lib/NetBurner.a)ipv6_dhcp.o
(C:/nburn/lib/NetBurner.a)ucosmcfc.o
(C:/nburn/lib/NetBurner.a)system.o
(C:/nburn/lib/NetBurner.a)utils.o
(C:/nburn/lib/NetBurner.a)extraio.o
(C:/nburn/lib/NetBurner.a)multicast.o
(C:/nburn/lib/NetBurner.a)random.o
(C:/nburn/lib/NetBurner.a)netrx.o
(C:/nburn/lib/NetBurner.a)nbfloatprint.o
(C:/nburn/lib/NetBurner.a)md5c.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)si_class_type_info.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)del_ops.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)eh_exception.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)eh_catch.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)eh_terminate.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)eh_throw.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)eh_personality.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)tinfo.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)del_op.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)class_type_info.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)eh_alloc.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)ios_init.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)globals_io.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)istream-inst.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)ios-inst.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)cxx11-ios_failure.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)string-inst.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)ext11-inst.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)fstream-inst.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)cxx11-stdexcept.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)functexcept.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)ostream-inst.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)ios.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)wlocale-inst.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)locale-inst.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)streambuf-inst.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)snprintf_lite.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)system_error.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)ctype.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)eh_aux_runtime.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)eh_unex_handler.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)eh_term_handler.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)new_opv.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)vmi_class_type_info.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)eh_globals.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)bad_typeid.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)eh_call.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)del_opv.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)guard.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)pure.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)new_handler.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)dyncast.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)bad_cast.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)new_opvnt.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)bad_array_new.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)vterminate.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)bad_alloc.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)stdexcept.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)time_members.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)c++locale.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)messages_members_cow.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)monetary_members_cow.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)codecvt.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)locale_init.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)streambuf.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)numeric_members_cow.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)collate_members_cow.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)basic_file.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)ios_locale.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)locale_facets.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)codecvt_members.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)ios_failure.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)locale.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)istream.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)cxx11-locale-inst.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)lt1-codecvt.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)cow-string-inst.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)iostream-inst.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)cow-stdexcept.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)ctype_members.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)random.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)cow-shim_facets.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)cxx11-shim_facets.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)cow-locale_init.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)cow-wstring-inst.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)ctype_configure_char.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)cxx11-wlocale-inst.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)compatibility.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)guard_error.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)cp-demangle.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)new_opnt.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)eh_type.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)misc-inst.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)monetary_members.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)collate_members.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)messages_members.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)numeric_members.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)sso_string.o
(c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455\libstdc++.a)wstring-inst.o
c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/bin/ld.exe: cannot find : Invalid argument
collect2.exe: error: ld returned 1 exit status

make.exe[3]: *** [CMakeFiles/cmake-netburner-hello-world_0.0.5.dir/build.make:117: Release/cmake-netburner-hello-world_0.0.5.elf] Error 1
make.exe[3]: Leaving directory 'G:/cmake-netburner-hello-world/build_m68k_gcc8'
make.exe[2]: *** [CMakeFiles/Makefile2:126: CMakeFiles/cmake-netburner-hello-world_0.0.5.dir/all] Error 2
make.exe[2]: Leaving directory 'G:/cmake-netburner-hello-world/build_m68k_gcc8'
make.exe[1]: *** [CMakeFiles/Makefile2:106: CMakeFiles/app.dir/rule] Error 2
make.exe[1]: Leaving directory 'G:/cmake-netburner-hello-world/build_m68k_gcc8'
make.exe: *** [Makefile:137: app] Error 2
```

</details>

Now we are interested in where `collect2` is called and which arguments are passing in.

```
c:/nburn/gcc-m68k/bin/../libexec/gcc/m68k-unknown-elf/8.1.0/collect2.exe 
--sysroot=C:/nburn/gcc-m68k/bin/../m68k-unknown-elf/sysroot 
-o Release/cmake-netburner-hello-world_0.0.5.elf 
c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/m54455/crtbegin.o 
-Lc:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/m54455 
-Lc:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib/m54455 
-Lc:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0 
-Lc:/nburn/gcc-m68k/bin/../lib/gcc 
-Lc:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/../../../../m68k-unknown-elf/lib 
-t -n -TC:/nburn/MOD5441X/lib/MOD5441X.ld -RC:/nburn/MOD5441X/lib/sys.ld 
-Map=G:/cmake-netburner-hello-world/build_m68k_gcc8/Release/cmake-netburner-hello-world_0.0.5.map 
./CMakeFiles/cmake-netburner-hello-world_0.0.5.dir/main.cpp.obj 
./CMakeFiles/cmake-netburner-hello-world_0.0.5.dir/htmldata.cpp.obj 
--gc-sections --start-group "" C:/nburn/lib/MOD5441X.a C:/nburn/lib/NetBurner.a 
C:/nburn/lib/FatFile.a C:/nburn/lib/StdFFile.a -lstdc++ --end-group 
-lstdc++ -lm -lgcc -lc -lgcc 
c:/nburn/gcc-m68k/bin/../lib/gcc/m68k-unknown-elf/8.1.0/m54455/crtend.o
```

Look. Wtf! Where did these quotation marks come from?

```
... --start-group "" ...
```

From extra comma after `--start-group`!

```
... -Wl,--start-group,${NBLIBS} ...
```

We just found the cause of the error! It was completely not obvious. Fix is simple:

```
... -Wl,--start-group ${NBLIBS} ...
```

Now it is successfully linking with both mentioned versions of GCC.

### Conclusions

If you have such error then most probably it is due to stupid typo(s) in linker arguments.

Try the following:

1. Enable tracing and verbose output when executing a linker
2. Thoroughly study the received output

Maybe it will help to someone.