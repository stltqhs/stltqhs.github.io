---
title: openjdk编译
date: 2018-03-31 15:34:38
tags:
---

# 一、`windows`下`openjdk`编译与`hotspot`调试过程
### 1.需要使用`cygwin`编译，同时需要安装`visual studio 2013`
注意不要和babun混用cygwin
以下需要修改的文件是：`common/autoconf/generated-configure.sh`
### 2.需要安装`cpio`，在`babun`的环境下直接使用pact安装
```
pact install cpio
```
使用`babun`安装的数据需要复制到`cygwin`环境
### 3.由于检查cygwin时可能存在路径链接深度太大，可以注释cygwin检查
```
@@ -7469,8 +7469,9 @@ $as_echo "$as_me: Rewriting SRC_ROOT to \"$new_path\"" >&6;}

   windows_path="$new_path"
   if test "x$OPENJDK_BUILD_OS_ENV" = "xwindows.cygwin"; then
-    unix_path=`$CYGPATH -u "$windows_path"`
-    new_path="$unix_path"
+    #unix_path=`$CYGPATH -u "$windows_path"`
+    #new_path="$unix_path"
+    echo 'Ignore cygwin'
```
### 4.如果无法检查系统的CPU类型，可以直接改
```
@@ -20376,7 +20377,8 @@ $as_echo "$as_me: The result from running with -V was: \"$COMPILER_VERSION_TEST\
     COMPILER_VERSION_TEST=`$COMPILER 2>&1 | $HEAD -n 1 | $TR -d '\r'`
     COMPILER_VERSION=`$ECHO $COMPILER_VERSION_TEST | $SED -n "s/^.*Version \([1-9][0-9.]*\) .*/\1/p"`
     COMPILER_VENDOR="Microsoft CL.EXE"
-    COMPILER_CPU_TEST=`$ECHO $COMPILER_VERSION_TEST | $SED -n "s/^.* for \(.*\)$/\1/p"`
+    #COMPILER_CPU_TEST=`$ECHO $COMPILER_VERSION_TEST | $SED -n "s/^.* for \(.*\)$/\1/p"`
+    COMPILER_CPU_TEST='x64'
```
### 5.如果Visual Studio的安装目录存在空格，需要使用软链接的方式链接到到Visual Studio的目录，同时在PATH中添加到VC/bin/amd64的变量
```
@@ -17678,7 +17679,7 @@ $as_echo "$as_me: Cannot locate a valid Visual Studio installation, checking cur
   # At this point, we should have corrent variables in the environment, or we can't continue.
   { $as_echo "$as_me:${as_lineno-$LINENO}: checking for Visual Studio variables" >&5
 $as_echo_n "checking for Visual Studio variables... " >&6; }
-
+PATH="/cygdrive/d/Application/VisualStudio12.0/VC/BIN/amd64":$PATH
```
同时需要创建软链接
```
ln -s /cygdrive/d/Program\ Files\ \(x86\)/Microsoft\ Visual\ Studio\ 12.0 /cygdrive/d/Application/VisualStudio12.0
```
### 6.fixpath报错时需要更改common/src/fixpath.c
```
@@ -266,6 +266,38 @@ char *fix_at_file(char *in)
   return atname;
 }

+char *replace_babun(char *in)
+{
+  char *str;
+  char *rep = "/d/Application/babun-1.2.0/.babun/cygwin";
+  int p = 0;
+
+  str = in;
+  while ((p = strstr(str, rep)))
+  {
+
+     str = replace_substring(str, rep, "");
+  }
+
+  return str;
+}
+
+char *replace_vs_link(char *in)
+{
+  char *str;
+  int p = 0;
+  char *substr = "Application/VisualStudio12.0";
+  char *replace = "Program Files (x86)/Microsoft Visual Studio 12.0";
+
+  str = in;
+  while ((p = strstr(str, substr)))
+  {
+     str = replace_substring(str, substr, replace);
+  }
+
+  return str;
+}
+
 int main(int argc, char **argv)
 {
     STARTUPINFO si;
@@ -302,12 +334,13 @@ int main(int argc, char **argv)
       fprintf(stderr, "Unknown mode: %s\n", argv[1]);
       exit(-1);
     }
-    line = replace_cygdrive(strstr(GetCommandLine(), argv[2]));
+    line = replace_babun(strstr(GetCommandLine(), argv[2]));
+    line = replace_cygdrive(line);

     for (i=1; i<argc; ++i) {
        if (argv[i][0] == '@') {
           // Found at-file! Fix it!
-          old_at_file = replace_cygdrive(argv[i]);
+          old_at_file = replace_cygdrive(replace_babun(argv[i]));
           new_at_file = fix_at_file(old_at_file);
           line = replace_substring(line, old_at_file, new_at_file);
        }
@@ -321,6 +354,8 @@ int main(int argc, char **argv)
     si.cb=sizeof(si);
     ZeroMemory(&pi,sizeof(pi));

+    line = replace_vs_link(line);
+
     rc = CreateProcess(NULL,
                        line,
                        0,

```
### 7.安装freetype
1)下载windows库
wget https://ftp.acc.umu.se/mirror/gnu.org/savannah/freetype/freetype-2.8.tar.gz
2)使用visual studio编译
使用vs打开freetype-2.8\builds\windows\vs2010\freetype.sln
3)配置编译项
点击菜单“PROJECT”->"Properties"
Configuration选择Release Multithreaded
在Configuration Properties选择General项
由于需要dll和lib文件，所以需要配置两次Configuration Type和Target Extension
4)负责编译输入文件到freetype-2.8\lib目录下
需要重命名文件为freetype.lib和freetype.dll
8.执行配置
进入cmd，执行VC\bin\amd64\vcvars64.bat
设置cygwin的bin目录为环境变量（如果找不到bash）：set PATH=D:\Application\cygwin64\bin:%PATH%
输入bash
```
./configure --enable-debug --with-boot-jdk=/cygdrive/d/Application/Java/jdk1.8.0_51 --with-freetype=/cygdrive/e/Project/freetype-2.8
```
配置成功的标志：
```
Configuration summary:
* Debug level:    fastdebug
* JDK variant:    normal
* JVM variants:   server
* OpenJDK target: OS: windows, CPU architecture: x86, address length: 64

Tools summary:
* Environment:    cygwin version 1.7.35(0.287/5/3) (root at /cygdrive/d/Applicat
ion/babun-1.2.0/.babun/cygwin)
* Boot JDK:       java version "1.8.0_51"  Java(TM) SE Runtime Environment (buil
d 1.8.0_51-b16)  Java HotSpot(TM) 64-Bit Server VM (build 25.51-b03, mixed mode)
   (at /cygdrive/d/Application/babun-1.2.0/.babun/cygwin/d/Application/Java/jdk1
.8.0_51)
* C Compiler:     Microsoft CL.EXE version 18.00.21005.1 (at /cygdrive/d/Applica
tion/babun-1.2.0/.babun/cygwin/d/Application/VisualStudio12.0/VC/BIN/amd64/cl)
* C++ Compiler:   Microsoft CL.EXE version 18.00.21005.1 (at /cygdrive/d/Applica
tion/babun-1.2.0/.babun/cygwin/d/Application/VisualStudio12.0/VC/BIN/amd64/cl)

Build performance summary:
* Cores to use:   4
* Memory limit:   9696 MB
* ccache status:  not available for your system
```
### 9.编译
```
make all
```
编译成功的标志
```
----- Build times -------
Start 2017-11-26 20:30:00
End   2017-11-26 20:39:53
00:00:03 corba
00:00:03 demos
00:07:10 docs
00:00:02 hotspot
00:02:04 images
00:00:01 jaxp
00:00:02 jaxws
00:00:14 jdk
00:00:02 langtools
00:00:01 nashorn
00:09:53 TOTAL
-------------------------
Finished building OpenJDK for target 'all'
```
编译时可能的错误
1)Could not create output file (/cygdrive/e/Project/openjdk/build/windows-x86_64-normal-server-fastdebug/langtools/dist/lib/src.zip)
cygwin的路径问题，不知道为什么不行，强制更改为windows路径，修改的文件是：langtools/make/BuildLangtools.gmk
```
@@ -195,10 +195,10 @@ ifeq ($(PROPS_ARE_CREATED), yes)

     $(eval $(call SetupZipArchive,ZIP_FULL_JAVAC_SOURCE, \
         SRC := $(LANGTOOLS_TOPDIR)/src/share/classes $(LANGTOOLS_OUTPUTDIR)/gensrc, \
-        ZIP := $(LANGTOOLS_OUTPUTDIR)/dist/lib/src.zip))
+        ZIP := e:/Project/openjdk/build/windows-x86_64-normal-server-fastdebug/langtools/dist/lib/src.zip))

     all: $(LANGTOOLS_OUTPUTDIR)/dist/lib/classes.jar \
-        $(LANGTOOLS_OUTPUTDIR)/dist/lib/src.zip \
+        e:/Project/openjdk/build/windows-x86_64-normal-server-fastdebug/langtools/dist/lib/src.zip \
         $(LANGTOOLS_OUTPUTDIR)/dist/bootstrap/lib/javac.jar

   endif
```
为了防止相同的错误，还需要修改以下文件：
A)corba/make/BuildCorba.gmk
```
@@ -198,7 +198,7 @@ ifeq ($(LOGWRAPPERS_ARE_CREATED), yes)
     # mimic behavior in old build system.
     $(eval $(call SetupZipArchive,ARCHIVE_BUILD_CORBA, \
         SRC := $(CORBA_TOPDIR)/src/share/classes $(CORBA_OUTPUTDIR)/gensrc $(CORBA_OUTPUTDIR)/logwrappers, \
-        ZIP := $(CORBA_OUTPUTDIR)/dist/lib/src.zip))
+        ZIP := e:/Project/openjdk/build/windows-x86_64-normal-server-fastdebug/corba/dist/lib/src.zip))

     $(BUILD_CORBA): $(BUILD_IDLS) $(LOGWRAPPER_DEPENDENCIES)

@@ -238,7 +238,7 @@ ifeq ($(LOGWRAPPERS_ARE_CREATED), yes)
     BIN_FILES := $(CORBA_TOPDIR)/src/share/classes/com/sun/tools/corba/se/idl/orb.idl \
         $(CORBA_TOPDIR)/src/share/classes/com/sun/tools/corba/se/idl/ir.idl

-    $(CORBA_OUTPUTDIR)/dist/lib/bin.zip: $(BIN_FILES) $(CORBA_OUTPUTDIR)/dist/lib/classes.jar
+    e:/Project/openjdk/build/windows-x86_64-normal-server-fastdebug/corba/dist/lib/bin.zip: $(BIN_FILES) $(CORBA_OUTPUTDIR)/dist/lib/classes.jar
        $(MKDIR) -p $(CORBA_OUTPUTDIR)/dist/lib
        $(MKDIR) -p $(CORBA_OUTPUTDIR)/lib
        $(RM) -f $@
@@ -254,8 +254,8 @@ ifeq ($(LOGWRAPPERS_ARE_CREATED), yes)
         $(CORBA_OUTPUTDIR)/btjars/logutil.jar \
         $(CORBA_OUTPUTDIR)/btjars/btcorba.jar \
         $(CORBA_OUTPUTDIR)/dist/lib/classes.jar \
-        $(CORBA_OUTPUTDIR)/dist/lib/src.zip \
-        $(CORBA_OUTPUTDIR)/dist/lib/bin.zip
+        e:/Project/openjdk/build/windows-x86_64-normal-server-fastdebug/corba/dist/lib/src.zip \
+        e:/Project/openjdk/build/windows-x86_64-normal-server-fastdebug/corba/dist/lib/bin.zip
   endif
 endif
```
B)jaxp/make/BuildJaxp.gmk
```
@@ -46,7 +46,7 @@ $(eval $(call SetupJavaCompilation,BUILD_JAXP, \
     SETUP := GENERATE_NEWBYTECODE_DEBUG, \
     SRC := $(JAXP_TOPDIR)/src, \
     BIN := $(JAXP_OUTPUTDIR)/classes, \
-    SRCZIP := $(JAXP_OUTPUTDIR)/dist/lib/src.zip))
+    SRCZIP := e:/Project/openjdk/build/windows-x86_64-normal-server-fastdebug/jaxp/dist/lib/src.zip))

 # Imitate the property cleaning mechanism in the old build. This will likely be replaced
 # by the unified functionality in JavaCompilation.gmk, but keep it the same as old build
@@ -66,6 +66,6 @@ $(eval $(call SetupArchive,ARCHIVE_JAXP, $(BUILD_JAXP) $(TARGET_PROP_FILES), \
     SUFFIXES := .class .properties, \
     JAR := $(JAXP_OUTPUTDIR)/dist/lib/classes.jar))

-all: $(JAXP_OUTPUTDIR)/dist/lib/classes.jar $(JAXP_OUTPUTDIR)/dist/lib/src.zip
+all: $(JAXP_OUTPUTDIR)/dist/lib/classes.jar e:/Project/openjdk/build/windows-x86_64-normal-server-fastdebug/jaxp/dist/lib/src.zip

 .PHONY: default all
```
C)jaxws/make/BuildJaxws.gmk
```
@@ -106,8 +106,8 @@ $(eval $(call SetupArchive,ARCHIVE_JAXWS, $(BUILD_JAXWS) $(BUILD_JAF) $(TARGET_P

 $(eval $(call SetupZipArchive,ZIP_JAXWS_SOURCES, \
     SRC := $(JAXWS_TOPDIR)/src/share/jaf_classes $(JAXWS_TOPDIR)/src/share/jaxws_classes, \
-    ZIP := $(JAXWS_OUTPUTDIR)/dist/lib/src.zip))
+    ZIP := e:/Project/openjdk/build/windows-x86_64-normal-server-fastdebug/jaxws/dist/lib/src.zip))

-all: $(JAXWS_OUTPUTDIR)/dist/lib/classes.jar $(JAXWS_OUTPUTDIR)/dist/lib/src.zip
+all: $(JAXWS_OUTPUTDIR)/dist/lib/classes.jar e:/Project/openjdk/build/windows-x86_64-normal-server-fastdebug/jaxws/dist/lib/src.zip

 .PHONY: default all
```
D)jdk/make/lib/CoreLibraries.gmk等
执行如下命令批量修改：
```
find jdk/make/lib -iname '*Libraries.gmk' | xargs sed -i 's/$(JDK_OUTPUTDIR)\/objs\/lib/e:\/Project\/openjdk\/build\/windows-x86_64-normal-server-fastdebug\/jdk\/objs\/lib/'
sed -i 's/$(JDK_OUTPUTDIR)/e:\/Project\/openjdk\/build\/windows-x86_64-normal-server-fastdebug\/jdk/g' jdk/make/CompileLaunchers.gmk
sed -i 's/$(JDK_OUTPUTDIR)/e:\/Project\/openjdk\/build\/windows-x86_64-normal-server-fastdebug\/jdk/g' jdk/make/CompileDemos.gmk
sed -i 's/$(IMAGES_OUTPUTDIR)/e:\/Project\/openjdk\/build\/windows-x86_64-normal-server-fastdebug\/images/g' jdk/make/CreateJars.gmk
sed -i 's/$(IMAGES_OUTPUTDIR)/e:\/Project\/openjdk\/build\/windows-x86_64-normal-server-fastdebug\/images/g' jdk/make/Profiles.gmk
```
2)adlc.exe.manifest : general error c1010070: Failed to load and parse the manifest. ???????????
修改文件hotspot/make/windows/makefiles/adlc.make
```
@@ -99,7 +99,7 @@ GENERATED_NAMES_IN_DIR=\

 adlc.exe: main.obj adlparse.obj archDesc.obj arena.obj dfa.obj dict2.obj filebuff.obj \
           forms.obj formsopt.obj formssel.obj opcodes.obj output_c.obj output_h.obj
-       $(LD) $(LD_FLAGS) /subsystem:console /out:$@ $**
+       $(LD) $(LD_FLAGS) /subsystem:console /MANIFEST /out:$@ $**
 !if "$(MT)" != ""
 # The previous link command created a .manifest file that we want to
 # insert into the linked artifact so we do not need to track it
```
3)ad_x86_64_pipeline.obj : error LNK2011: precompiled object not linked in; image may not run   \n   jvm.dll : fatal error LNK1120: 1 unresolved externals
由于拿到的源代码与VS2013不兼容
修改文件hotspot/make/windows/makefiles/vm.make
```
@@ -82,7 +82,7 @@ AGCT_EXPORT=/export:AsyncGetCallTrace

 # If you modify exports below please do the corresponding changes in
 # src/share/tools/ProjectCreator/WinGammaPlatformVC7.java
-LD_FLAGS=$(LD_FLAGS) $(STACK_SIZE) /subsystem:windows /dll /base:0x8000000 \
+LD_FLAGS=$(LD_FLAGS) $(STACK_SIZE) /subsystem:windows /manifest /dll /base:0x8000000 \
   /export:JNI_GetDefaultJavaVMInitArgs       \
   /export:JNI_CreateJavaVM                   \
   /export:JVM_FindClassFromBootLoader        \
@@ -132,6 +132,10 @@ CXX_USE_PCH=/Fp"vm.pch" /Yu"precompiled.hpp"
 # VS2012 requires this object file to be listed:
 LD_FLAGS=$(LD_FLAGS) _build_pch_file.obj
 !endif
+!if "$(COMPILER_NAME)" == "VS2013"
+# VS2012 requires this object file to be listed:
+LD_FLAGS=$(LD_FLAGS) _build_pch_file.obj
+!endif
 !else
 CXX_USE_PCH=$(CXX_DONT_USE_PCH)
 !endif
```
这个文件也可能需要改（出错就改这）hotspot/make/windows/makefiles/compile.make
@@ -112,6 +112,7 @@ CXX_FLAGS=$(CXX_FLAGS) /D TARGET_COMPILER_visCPP
 #      1500 is for VS2008
 #      1600 is for VS2010
 #      1700 is for VS2012
+#      1800 is for VS2013
 #    Do not confuse this MSC_VER with the predefined macro _MSC_VER that the
 #    compiler provides, when MSC_VER==1399, _MSC_VER will be 1400.
 #    Normally they are the same, but a pre-release of the VS2005 compilers
@@ -147,6 +148,9 @@ COMPILER_NAME=VS2010
 !if "$(MSC_VER)" == "1700"
 COMPILER_NAME=VS2012
 !endif
+!if "$(MSC_VER)" == "1800"
+COMPILER_NAME=VS2013
+!endif
 !endif

 # By default, we do not want to use the debug version of the msvcrt.dll file
```
4)mismatch detected for 'RuntimeLibrary' value 'MT_StaticRelease' doesn't match value 'MD_DynamicRelease' in awt_DnDDS.obj
修改文件jdk/make/lib/Awt2dLibraries.gmk
```
@@ -444,7 +444,7 @@ ifeq ($(OPENJDK_TARGET_OS), windows)
       WGLSurfaceData.c

   LIBAWT_LANG := C++
-  LIBAWT_CFLAGS += -EHsc -DUNICODE -D_UNICODE
+  LIBAWT_CFLAGS += -EHsc -DUNICODE -D_UNICODE -MT
   ifeq ($(OPENJDK_TARGET_CPU_BITS), 64)
     LIBAWT_CFLAGS += -DMLIB_OS64BIT
   endif
```
执行如下命令
```
make clean-jdk
```
再重新编译make all
5)make[2]: *** No rule to make target '/cygdrive/e/Project/openjdk/build/windows-x86_64-normal-server-fastdebug/jdk/demo/jpda/examples.jar', needed by 'all'.  Stop.
修改文件jdk/make/CompileDemos.gmk
```
$(JDK_OUTPUTDIR)/demo/jpda/examples.jar->e:/Project/openjdk/build/windows-x86_64-normal-server-fastdebug/jdk/demo/jpda/examples.jar
```
### 10.创建VC工程文件
1)修改hotspot/make/windows/create.bat
```
@@ -140,6 +140,7 @@ goto usage
 :test3
 if not "%HOTSPOTMKSHOME%" == "" goto makedir
 if exist c:\cygwin\bin set HOTSPOTMKSHOME=c:\cygwin\bin
+if exist D:\Application\cygwin64\bin set HOTSPOTMKSHOME=D:\Application\cygwin64\bin
 if not "%HOTSPOTMKSHOME%" == "" goto makedir
 echo Warning: please set variable HOTSPOTMKSHOME to place where
 echo          your MKS/Cygwin installation is
```
2)开始创建工程文件
进入cmd，执行VC\bin\amd64\vcvars64.bat
设置cygwin的bin目录为环境变量：set PATH=D:\Application\cygwin64\bin:%PATH%
执行create openjdk\build\windows-x86_64-normal-server-fastdebug\jdk
执行成功后会生成工程文件：openjdk\hotspot\build\vs-amd64\jvm.vcproj
11.构建jvm及调试
1)工程配置
在项目配置窗口（jvm Property Pages）打开Configuration Manager的窗口，修改Active solution configuration为compiler2_fastdebug,Active solution platform: x64
Configuration Properties>General选项，Output Directory和Intermediate Directory都改为：openjdk\build\windows-x86_64-normal-server-fastdebug\jdk\bin\server
Configuration Properties>Debugging选项，Command改为：openjdk\build\windows-x86_64-normal-server-fastdebug\jdk\bin\java.exe，Command Arguments改为-Djava.class.path=openjdk/debug/src/java com.yuqing.hotspot.HelloDebug，Environment改为JAVA_HOME=openjdk\build\windows-x86_64-normal-server-fastdebug\jdk
Configuration Properties>Linker>Input，在Additional Dependencies添加version.lib ，Kernel32.lib ，Psapi.lib
2)编译调试文件
build/windows-x86_64-normal-server-fastdebug/jdk/bin/javac debug/src/java/com/yuqing/hotspot/HelloDebug.java
3)断点
share/vm/runtime/thread.cpp第3306行是create_vm方法，在该方法内打下断点
4)编译调试
点击Local Windows Debbugger开始调试
### 12.参考
http://www.jianshu.com/p/e85f93cc74cb
