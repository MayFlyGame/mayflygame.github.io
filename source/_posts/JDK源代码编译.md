---
title: JDK源代码编译
date: 2022-05-04 21:23:08
tags:
---


# JDK源码编译

# 1. 源码准备

## 1.1 源码版本及下载

确定要编译哪个版本的JDK，有两种方法：

- Github：[https://github.com/openjdk/jdk](https://github.com/openjdk/jdk) 可以从github找到对应的版本来下载
- Java.Net: [https://jdk.java.net/java-se-ri/18](https://jdk.java.net/java-se-ri/18) 也可以去次网站直接下载源码压缩包。

我这里是通过第二种方式下载的，下载的是：openjdk_18+36版本。

```jsx
godwit@Firebase  ~/Source  ll
total 339496
drwxr-xr-x@ 19 godwit  staff   608B  5  2 22:15 openjdk-18+36
-rw-r--r--@  1 godwit  staff   166M  5  2 22:11 openjdk-18+36_src.zip
 godwit@Firebase  ~/Source  cd openjdk-18+36
 godwit@Firebase  ~/Source/openjdk-18+36 
```

# 2. 环境准备

## 2.1 Bootstrap JDK

假设需要编译大版本号为N的JDK，那么我们需要另外准备一个大版本号至少为N-1的，已经编译好的JDK，这是因为OpenJDK由多个部分构成，其中一部分（HotSpot）代码是用C、C++编写，而更多的代码是用Java语言来实现的。因此编译这些Java代码就需要使用另外一个编译期可用的JDK，官方称这个为Bootstrap JDK。

我本机已经安装了JDK环境，直接使用即可。

```bash
godwit@Firebase  ~  java -version
java 17.0.2 2022-01-18 LTS
Java(TM) SE Runtime Environment (build 17.0.2+8-LTS-86)
Java HotSpot(TM) 64-Bit Server VM (build 17.0.2+8-LTS-86, mixed mode, sharing)

```

## 2.2 其他依赖库

缺少什么，直接brew install xxx即可。

- FreeType：
- Autoconf：
- ccache：
- clang/clang++: XCode自带，所以没有额外配置安装。

```bash
godwit@Firebase  ~  clang --version
Apple clang version 13.1.6 (clang-1316.0.21.2.3)
Target: arm64-apple-darwin21.4.0
Thread model: posix
InstalledDir: /Library/Developer/CommandLineTools/usr/bin

godwit@Firebase  ~  clang++ --version
Apple clang version 13.1.6 (clang-1316.0.21.2.3)
Target: arm64-apple-darwin21.4.0
Thread model: posix
InstalledDir: /Library/Developer/CommandLineTools/usr/bin

godwit@Firebase  ~  autoconf --version
autoconf (GNU Autoconf) 2.71
Copyright (C) 2021 Free Software Foundation, Inc.
License GPLv3+/Autoconf: GNU GPL version 3 or later
<https://gnu.org/licenses/gpl.html>, <https://gnu.org/licenses/exceptions.html>
This is free software: you are free to change and redistribute it.
There is NO WARRANTY, to the extent permitted by law.

Written by David J. MacKenzie and Akim Demaille.

godwit@Firebase  ~  make --version
GNU Make 3.81
Copyright (C) 2006  Free Software Foundation, Inc.
This is free software; see the source for copying conditions.
There is NO warranty; not even for MERCHANTABILITY or FITNESS FOR A
PARTICULAR PURPOSE.

This program built for i386-apple-darwin11.3.0

godwit@Firebase  ~  freetype-config --ftversion
2.12.0
```

# 3. 编译

## 3.1 配置

```bash
# --with-debug-level=slowdebug参数是为了显示更多调试信息.debug level有4种：release, fastdebug, slowdebug, optimized. 默认是 release
# --with-native-debug-symbols hotspot编译之后是一个名为libjvm.so的动态库,在java命令里通过dlopen加载,如果不加上--with-native-debug-symbols=internal,需要手动解压jdk/lib/amd64/server里的libjvm.debuginfo,或者指定动态库路径,调试起来会很复杂.
# --with-jvm-variants=server：仅包含Server模式
# --disable-warnings-as-errors：因为tag版本可能还不是很完美，存在warning，所以这里允许warning的存在
$ bash ./configure  --with-debug-level=slowdebug --with-native-debug-symbols=internal --with-jvm-variants=server --enable-dtrace --disable-warnings-as-errors

```

### 3.1.1 错误1 - No xcodebuild tool and no system framework headers found

```bash
...
checking which variants of the JVM to build... server
checking if absolute paths should be allowed in the build output... no, release build
checking for sdk name...
configure: error: No xcodebuild tool and no system framework headers found, use --with-sysroot or --with-sdk-name to provide a path to a valid SDK
/Users/godwit/Source/openjdk-18+36/build/.configure-support/generated-configure.sh: line 84: 5: Bad file descriptor
configure exiting with result code 1
 ✘ godwit@Firebase  ~/Source/openjdk-18+36 
#### 解决方案如下：
 ✘ godwit@Firebase  ~/Source/openjdk-18+36  sudo xcode-select --switch /Applications/Xcode.app/Contents/Developer
Password:
 godwit@Firebase  ~/Source/openjdk-18+36  
```

### 3.1.2 错误2 - error: parameter 'SizeOfTag' set but not used

```jsx
=== Output from failing command(s) repeated here ===
* For target support_native_java.desktop_liblcms_cmstypes.o:
/Users/godwit/Source/openjdk-18+36/src/java.desktop/share/native/liblcms/cmstypes.c:3441:132: error: parameter 'SizeOfTag' set but not used [-Werror,-Wunused-but-set-parameter]
void *Type_ProfileSequenceId_Read(struct _cms_typehandler_struct* self, cmsIOHANDLER* io, cmsUInt32Number* nItems, cmsUInt32Number SizeOfTag)
                                                                                                                                   ^
/Users/godwit/Source/openjdk-18+36/src/java.desktop/share/native/liblcms/cmstypes.c:5137:125: error: parameter 'SizeOfTag' set but not used [-Werror,-Wunused-but-set-parameter]
void *Type_Dictionary_Read(struct _cms_typehandler_struct* self, cmsIOHANDLER* io, cmsUInt32Number* nItems, cmsUInt32Number SizeOfTag)
                                                                                                                            ^
2 errors generated.

* All command lines available in /Users/godwit/Source/openjdk-18+36/build/macosx-aarch64-server-release/make-support/failure-logs.
=== End of repeated output ===

### 解决方法：--disable-warnings-as-errors
```

### 3.1.3 配置成功

出现以下输出，说明配置成功。

```jsx

......
====================================================
A new configuration has been successfully created in
/Users/godwit/Source/openjdk-18+36/build/macosx-aarch64-server-release
using default settings.

Configuration summary:
* Name:           macosx-aarch64-server-release
* Debug level:    release
* HS debug level: product
* JVM variants:   server
* JVM features:   server: 'cds compiler1 compiler2 dtrace epsilongc g1gc jfr jni-check jvmci jvmti management nmt parallelgc serialgc services shenandoahgc vm-structs zgc'
* OpenJDK target: OS: macosx, CPU architecture: aarch64, address length: 64
* Version string: 18-internal+0-adhoc.godwit.openjdk-1836 (18-internal)

Tools summary:
* Boot JDK:       java version "17.0.2" 2022-01-18 LTS Java(TM) SE Runtime Environment (build 17.0.2+8-LTS-86) Java HotSpot(TM) 64-Bit Server VM (build 17.0.2+8-LTS-86, mixed mode, sharing) (at /Library/Java/JavaVirtualMachines/jdk-17.0.2.jdk/Contents/Home)
* Toolchain:      clang (clang/LLVM from Xcode 13.3)
* C Compiler:     Version 13.1.6 (at /usr/bin/clang)
* C++ Compiler:   Version 13.1.6 (at /usr/bin/clang++)

Build performance summary:
* Build jobs:     10
* Memory limit:   32768 MB
```

# 4. 编译

## 4.1 make

```jsx
godwit@Firebase  ~/Source/openjdk-18+36  make all
```

## 4.2 错误 - guarantee(val < (1ULL << nbits)) failed: Field too big for insn

```jsx
=== Output from failing command(s) repeated here ===
* For target jdk__optimize_image_exec:
#
# A fatal error has been detected by the Java Runtime Environment:
#
#  Internal Error (assembler_aarch64.hpp:250), pid=72987, tid=9731
#  guarantee(val < (1ULL << nbits)) failed: Field too big for insn
#
# JRE version:  (18.0) (build )
# Java VM: OpenJDK 64-Bit Server VM (18-internal+0-adhoc.godwit.openjdk-1836, mixed mode, sharing, tiered, compressed oops, compressed class ptrs, g1 gc, bsd-aarch64)
# No core dump will be written. Core dumps have been disabled. To enable core dumping, try "ulimit -c unlimited" before starting Java again
#
# An error report file with more information is saved as:
# /Users/godwit/Source/openjdk-18+36/make/hs_err_pid72987.log
#
#

* All command lines available in /Users/godwit/Source/openjdk-18+36/build/macosx-aarch64-server-release/make-support/failure-logs.
=== End of repeated output ===

No indication of failed target found.
Hint: Try searching the build log for '] Error'.
Hint: See doc/building.html#troubleshooting for assistance.

make[1]: *** [main] Error 2
make: *** [all] Error 2
```

解决方案：

```bash
可以根据commit修改一下src/hotspot/cpu/aarch64/immediate_aarch64.cpp
// src/hotspot/cpu/aarch64/immediate_aarch64.cpp 
uint64_t replicate(uint64_t bits, int nbits, int count)
{
  // 在函数开头，添加下main代码 
  assert(count > 0, "must be");
  assert(nbits > 0, "must be");
  assert(count * nbits <= 64, "must be");

  // Special case nbits == 64 since the shift below with that nbits value
  // would result in undefined behavior.
  if (nbits == 64) {
    return bits;
  }
  //-----------------------

  uint64_t result = 0;
  uint64_t mask = ones(nbits);
```

## 4.3 编译成功

编译成功后，在build/macosx-aarch64-server-slowdebug/jdk/bin目录下执行：java -version

能够看到正常输出。

```jsx
✘ godwit@Firebase  ~/Source/openjdk-18+36/build/macosx-aarch64-server-slowdebug/jdk/bin  java -version
openjdk version "1.8.0_322"
OpenJDK Runtime Environment (Zulu 8.60.0.21-CA-macos-aarch64) (build 1.8.0_322-b06)
OpenJDK 64-Bit Server VM (Zulu 8.60.0.21-CA-macos-aarch64) (build 25.322-b06, mixed mode)

```