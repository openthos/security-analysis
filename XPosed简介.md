# XPosed简介
## Xposed构成
Xposed由两大块构成，一个是Rovo89的Xposed框架，另一大块就是基于这个框架的各种Xpose插件或是应用。
### Rovo89的Xposed框架
Rovo89的Xposed框架，由Xposed、android_art、XposedBridge、XposedTools及XposedInstaller组成。
下面分别描述各部分的作用。
#### 1. Xposed  
主要用来完成对系统程序app_process的修改，使在调用fork zygote并调用app之前，能加将XPosedBridge
进行载入，以便相应的插件或是应用可以通过对XPosedBridge的继续来发挥作用  
链接[Xposed https://github.com/rovo89/Xposed](https://github.com/rovo89/Xposed)  
其主要由如下文件构成  
```c
ART.mk	                 //SDK_VERSION>=21时使用的Makefile，由Android.mk调用
Android.mk	             //Makefile
Dalvik.mk	               //SDK_VERSION<21时使用的Makefile，由Android.mk调用
MODULE_LICENSE_APACHE2	 //指示本MODULE使用APACHE2License
NOTICE	                 //Apache2 License
app_main.cpp	           //SDK_VERSION<21时生成xposed版app_process使用的app_main.cpp
app_main2.cpp	           //SDK_VERSION>=21时生成xposed版app_process使用的app_main.cpp
fd_utils-inl.h	         
libxposed_art.cpp           //与下面的libxposed_common.cpp一起编译生成了libxposed_art.so
libxposed_common.cpp	
libxposed_common.h	
libxposed_dalvik.cpp	   //与上面的libxposed_common.cpp一起编译生成了libxposed_dalvik.so
libxposed_dalvik.h	
xposed.cpp	             //生成xposed版app_process使用
xposed.h
xposed_logcat.cpp	       //生成xposed版app_process使用
xposed_logcat.h
xposed_offsets.h
xposed_safemode.cpp	     //生成xposed版app_process使用
xposed_safemode.h
xposed_service.cpp	     //生成xposed版app_process使用
xposed_service.h
xposed_shared.h  
```

下面的Android.mk说明了如何生成xposed版本的app_process和libxposed_art.so  
```Makefile  
##########################################################
# Customized app_process executable
##########################################################

LOCAL_PATH:= $(call my-dir)
include $(CLEAR_VARS)

ifeq (1,$(strip $(shell expr $(PLATFORM_SDK_VERSION) \>= 21)))
  LOCAL_SRC_FILES := app_main2.cpp
  LOCAL_MULTILIB := both
  LOCAL_MODULE_STEM_32 := app_process32_xposed
  LOCAL_MODULE_STEM_64 := app_process64_xposed
else
  LOCAL_SRC_FILES := app_main.cpp
  LOCAL_MODULE_STEM := app_process_xposed
endif

LOCAL_SRC_FILES += \
  xposed.cpp \
  xposed_logcat.cpp \
  xposed_service.cpp \
  xposed_safemode.cpp

LOCAL_SHARED_LIBRARIES := \
  libcutils \
  libutils \
  liblog \
  libbinder \
  libandroid_runtime \
  libdl

LOCAL_CFLAGS += -Wall -Werror -Wextra -Wunused
LOCAL_CFLAGS += -DPLATFORM_SDK_VERSION=$(PLATFORM_SDK_VERSION)

ifeq (1,$(strip $(shell expr $(PLATFORM_SDK_VERSION) \>= 17)))
  LOCAL_SHARED_LIBRARIES += libselinux
  LOCAL_CFLAGS += -DXPOSED_WITH_SELINUX=1
endif

ifeq (1,$(strip $(shell expr $(PLATFORM_SDK_VERSION) \>= 22)))
  LOCAL_WHOLE_STATIC_LIBRARIES := libsigchain
  LOCAL_LDFLAGS := -Wl,--version-script,art/sigchainlib/version-script.txt -Wl,--export-dynamic
endif

LOCAL_MODULE := xposed
LOCAL_MODULE_TAGS := optional
LOCAL_STRIP_MODULE := keep_symbols

# Always build both architectures (if applicable)
ifeq ($(TARGET_IS_64_BIT),true)
  $(LOCAL_MODULE): $(LOCAL_MODULE)$(TARGET_2ND_ARCH_MODULE_SUFFIX)
endif

include $(BUILD_EXECUTABLE)

##########################################################
# Library for Dalvik-/ART-specific functions
##########################################################
ifeq (1,$(strip $(shell expr $(PLATFORM_SDK_VERSION) \>= 21)))
  include frameworks/base/cmds/xposed/ART.mk
else
  include frameworks/base/cmds/xposed/Dalvik.mk
endif
```

#### 2. android_art  
链接[android_art https://github.com/rovo89/android_art](https://github.com/rovo89/android_art)  
这个project是对AOSP/art的patch.  
其具体的作用应该是通过在art中打了一些桩函数，供XposedBridge来使用（暂时只是推测，后续仍需要分析代码
明确实际 用途）。
#### 3. XposedBridge
链接[XposedBridge https://github.com/rovo89/XposedBridge](https://github.com/rovo89/XposedBridge)  
这个子项目生成了XposedBridge.jar  
XposedBridge既是Xpose的插件开发或是基于Xposed的APP开发使用的框架，也是各插件或是app的加载器。  
#### 4. XposedTools  
链接[Xpose的Tools https://github.com/rovo89/XposedTools](https://github.com/rovo89/XposedTools)  
Xposed和XposedBridge编译依赖于Android源码，而且还有一些定制化的东西。
所以XposedTools就是用来帮助我们编译Xposed和XposedBridge的。  
#### 5. XposedInstaller
[XposedInstaller]https://github.com/rovo89/XposedInstaller  
之个子项目编译生成了Xpose的Installer.apk。是Xposed的插件管理和功能控制APP，也就是说Xposed整体管控功能就是由这个APP来完成的，
它包括启用Xposed插件功能，下载和启用指定插件APP，还可以禁用Xposed插件功能等。注意，这个app要正常无误得运行必须能拿到root权限。
