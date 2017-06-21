在前面一篇文章中，我们分析了[Android](http://lib.csdn.net/base/android)模块的编译过程。当[android](http://lib.csdn.net/base/android)系统的所有模块都编译好之后，我们就可以对编译出来的模块文件进行打包了。打包结果是获得一系列的镜像文件，例如system.img、boot.img、ramdisk.img、userdata.img和recovery.img等。这些镜像文件最终可以烧录到手机上运行。在本文中，我们就详细分析Android系统的镜像文件的打包过程。

老罗的新浪微博：[http://weibo.com/shengyangluo](http://weibo.com/shengyangluo)，欢迎关注！

《Android系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

Android系统镜像文件的打包工作同样是由Android编译系统来完成的，如图1所示：

![img](http://img.blog.csdn.net/20140305012118562?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

图1 Android系统镜像文件的打包过程

从前面[Android编译系统环境初始化过程分析](http://blog.csdn.net/luoshengyang/article/details/18928789)和[Android源代码编译命令m/mm/mmm/make分析](http://blog.csdn.net/luoshengyang/article/details/19023609)这两篇文章可以知道，Android编译系统在初始化的过程中，会通过根目录下的Makefile脚本加载build/core/main.mk脚本，接着build/core/main.mk脚本又会加载build/core/Makefile脚本，而Android系统镜像文件就是由build/core/Makefile脚本负责打包生成的。

在build/core/main.mk文件中，与Android系统镜像文件打包过程相关的内容如下所示：

```
......  

ifdef FULL_BUILD  
# The base list of modules to build for this product is specified  
# by the appropriate product definition file, which was included  
# by product_config.make.  
product_MODULES := $(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_PACKAGES)  
# Filter out the overridden packages before doing expansion  
product_MODULES := $(filter-out $(foreach p, $(product_MODULES), \  
      $(PACKAGES.$(p).OVERRIDES)), $(product_MODULES))  
$(call expand-required-modules,product_MODULES,$(product_MODULES))  
product_FILES := $(call module-installed-files, $(product_MODULES))  
......  
else  
# We're not doing a full build, and are probably only including  
# a subset of the module makefiles.  Don't try to build any modules  
# requested by the product, because we probably won't have rules  
# to build them.  
product_FILES :=  
endif  
......  

modules_to_install := $(sort \  
    $(ALL_DEFAULT_INSTALLED_MODULES) \  
    $(product_FILES) \  
    $(foreach tag,$(tags_to_install),$($(tag)_MODULES)) \  
    $(call get-tagged-modules, shell_$(TARGET_SHELL)) \  
    $(CUSTOM_MODULES) \  
)  

# Some packages may override others using LOCAL_OVERRIDES_PACKAGES.  
# Filter out (do not install) any overridden packages.  
overridden_packages := $(call get-package-overrides,$(modules_to_install))  
ifdef overridden_packages  
#  old_modules_to_install := $(modules_to_install)  
modules_to_install := \  
      $(filter-out $(foreach p,$(overridden_packages),$(p) %/$(p).apk), \  
   $(modules_to_install))  
endif  
......  

# Install all of the host modules  
modules_to_install += $(sort $(modules_to_install) $(ALL_HOST_INSTALLED_FILES))  

# build/core/Makefile contains extra stuff that we don't want to pollute this  
# top-level makefile with.  It expects that ALL_DEFAULT_INSTALLED_MODULES  
# contains everything that's built during the current make, but it also further  
# extends ALL_DEFAULT_INSTALLED_MODULES.  
ALL_DEFAULT_INSTALLED_MODULES := $(modules_to_install)  
include $(BUILD_SYSTEM)/Makefile  
modules_to_install := $(sort $(ALL_DEFAULT_INSTALLED_MODULES))  
ALL_DEFAULT_INSTALLED_MODULES :=  
......  

.PHONY: ramdisk  
ramdisk: $(INSTALLED_RAMDISK_TARGET)  
......  

.PHONY: userdataimage  
userdataimage: $(INSTALLED_USERDATAIMAGE_TARGET)  
......  

.PHONY: bootimage  
bootimage: $(INSTALLED_BOOTIMAGE_TARGET)  

......  
```
如果定义在FULL_BUILD这个变量，就意味着我们是要对整个系统进行编译，并且编译完成之后 ，需要将编译得到的文件进行打包，以便可以得到相应的镜像文件，否则的话，就仅仅是对某些模块进行编译。

在前面[Android编译系统环境初始化过程分析](http://blog.csdn.net/luoshengyang/article/details/18928789)一篇文章中，我们提到，变量INTERNAL_PRODUCT描述的是执行lunch命令时所选择的产品所对应的产品Makefile文件，而变量PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_PACKAGES描述的是在该产品Makefile文件中通过变量PRODUCT_PACKAGES所定义的模块名称列表。因此，我们得到的变量product_MODULES描述的就是要安装的模块名称列表。

我们知道，Android源码中自带了很多默认的APK模块。如果我们想用自己编写的一个APK来代替某一个系统自带的APK，那么就可以通过变量PACKAGES.<new>.OVERRIDES := <old>来说明。其中，<new>表示用来替换的APK，而<old>表示被替换的APK。在这种情况下，被替换的APK是不应该被打包到系统镜像中去的。因此，我们需要从上一步得到的模块名称列表中剔除那些被替换的APK，这是通过Makefile脚本提供的filter-out函数来完成的。

一个模块可以通过LOCAL_REQUIRED_MODULES变量来指定它所依赖的其它模块，因此，当一个模块被安装的时候，它所依赖的其它模块也会一起被安装。调用函数expand-required-modules获得的就是所有要安装的模块所依赖的其它模块，并且将这些被依赖的模块也一起保存在变量product_MODULES中。

注意，这时候我们得到的product_MODULES描述的仅仅是要安装的模块的名称，但是我们实际需要的这些模块对应的具体文件，因此，需要进一步调用函数module-installed-files来获得要安装的模块所对应的文件，也就是要安装的模块经过编译之后生成的文件，这些文件就保存在变量product_FILES中。

最终需要安装的文件除了变量product_FILES所描述的文件之后， 还包含以下四类模块文件：

变量ALL_DEFAULT_INSTALLED_MODULES描述的文件。

变量CUSTOM_MODULES描述的文件。

与当前编译类型对应的模块文件。例如，如果当前设定的编译类型为debug，那么所有通过将变量LOCAL_MODULE_TAGS将自己设置为debug的模块也会被打包到系统镜像文件中去。

与当前shell名称对应的模块文件。例如，如果当前使用的shell是mksh，那么所有通过将变量LOCAL_MODULE_TAGS将自己设置为shell_mksh的模块也会被打包到系统镜像文件中去。

最后，modules_to_install就描述当前需要打包到系统镜像中去的模块文件。实际上，我们除了可以通过PACKAGES.$(p).OVERRIDES来描述要替换的APK之后，还可以在一个模块中通过LOCAL_OVERRIDES_PACKAGES来描述它所替换的模块。因此，我们需要通过函数get-package-overrides来获得此类被替换的模块文件，并且将它们从modules_to_install中剔除，这样得到的模块文件才是最终需要安装的。

确定了要安装的所有模块文件之后，就可以将build/core/Makefile文件加载进来了。注意，文件build/core/Makefile是通过变量ALL_DEFAULT_INSTALLED_MODULES来获得当前所有要打包到系统镜像文件中去的模块文件的。

文件build/core/Makefile主要就是用来打包各种Android系统镜像文件的，当然它也是通过make规则来执行各种Android系统镜像文件打包命令的。每一个Android镜像文件都对应有一个make伪目标。例如，在build/core/main.mk文件中，就定义了三个make伪目标ramdisk、userdataimage和bootimage，它们分别依赖于变量INSTALLED_USERDATAIMAGE_TARGET、INSTALLED_USERDATAIMAGE_TARGET和INSTALLED_BOOTIMAGE_TARGET所描述的文件，并且它们分别表示的就是ramdisk.img、userdata.img和boot.img文件。

变量INSTALLED_USERDATAIMAGE_TARGET、INSTALLED_USERDATAIMAGE_TARGET和INSTALLED_BOOTIMAGE_TARGET都是在build/core/Makefile文件中定义的。此外，build/core/Makefile文件还定义了另外两个镜像文件system.img和recovery.img的生成规则。接下来，我们就分别对这些镜像文件的打包过程进行分析。

## 1. system.img

system.img镜像文件描述的是设备上的system分区，即/system目录，它是在build/core/Makefile文件中生成的，相关的内容如下所示：

```
# Rules that need to be present for the all targets, even  
# if they don't do anything.  
.PHONY: systemimage  
systemimage:  
......  

INSTALLED_SYSTEMIMAGE := $(PRODUCT_OUT)/system.img  
......  

$(INSTALLED_SYSTEMIMAGE): $(BUILT_SYSTEMIMAGE) $(RECOVERY_FROM_BOOT_PATCH) | $(ACP)  
    @echo "Install system fs image: $@"  
    $(copy-file-to-target)  
    $(hide) $(call assert-max-image-size,$@ $(RECOVERY_FROM_BOOT_PATCH),$(BOARD_SYSTEMIMAGE_PARTITION_SIZE),yaffs)  

systemimage: $(INSTALLED_SYSTEMIMAGE)  

.PHONY: systemimage-nodeps snod  
systemimage-nodeps snod: $(filter-out systemimage-nodeps snod,$(MAKECMDGOALS)) \  
         | $(INTERNAL_USERIMAGES_DEPS)  
    @echo "make $@: ignoring dependencies"  
    $(call build-systemimage-target,$(INSTALLED_SYSTEMIMAGE))  
    $(hide) $(call assert-max-image-size,$(INSTALLED_SYSTEMIMAGE),$(BOARD_SYSTEMIMAGE_PARTITION_SIZE),yaffs)  
```
从这里就可以看出，build/core/Makefile文件定义了两个伪目标来生成system.img：1. systemimg；2. systemimg-nodeps或者snod。伪目标systemimg表示在打包system.img之前，要根据依赖规则重新生成所有要进行打包的文件，而伪目标systemimg-nodeps则不需要根据依赖规则重新生成所有需要打包的文件而直接打包system.img文件。因此，执行systemimg-nodep比执行systemimg要快很多。通常，如果我们在Android源码中修改了某一个模块，并且这个模块不被其它模块依赖，那么对这个模块进行编译之后，就可以简单地执行make systemimg-nodeps来重新打包system.img。但是，如果我们修改的模块会被其它模块引用，例如，我们修改了Android系统的核心模块framework.jar和services.jar，那么就需要执行make systemimg来更新所有依赖于framework.jar和services.jar的模块，那么最后得到的system.img才是正确的镜像。否则的话，会导致Android系统启动失败。

接下来，我们就主要关注伪目标systemimg生成system.img镜像文件的过程。伪目标systemimg依赖于INSTALLED_SYSTEMIMAGE，也就是最终生成的$(PRODUCT_OUT)/system.img文件。INSTALLED_SYSTEMIMAGE又依赖于BUILT_SYSTEMIMAGE、RECOVERY_FROM_BOOT_PATCH和ACP。注意，BUILT_SYSTEMIMAGE、RECOVERY_FROM_BOOT_PATCH和ACP之间有一个管道符号相隔。在一个make规则之中，一个目标的依赖文件可以划分为两类。一个类是普通依赖文件，它们位于管道符号的左则，另一类称为“order-only”依赖文件，它们位于管理符号的右侧。每一个普通依赖文件发生修改后，目标都会被更新。但是"order-only"依赖文件发生修改时，却不一定会导致目标更新。只有当目标文件不存在的情况下，"order-only"依赖文件的修改才会更新目标文件。也就是说，只有在目标文件不存在的情况下，“order-only”依赖文件才会参与到规则的执行过程中去。

ACP描述的是一个Android专用的cp命令，在生成system.img镜像文件的过程中是需要用到的。普通的cp命令在不同的平台（Mac OS X、MinGW/Cygwin和[Linux](http://lib.csdn.net/base/linux)）的实现略有差异，并且可能会导致一些问题，于是Android编译系统就重写了自己的cp命令，使得它在不同平台下执行具有统一的行为，并且解决普通cp命令可能会出现的问题。例如，在[linux](http://lib.csdn.net/base/linux)平台上，当我们把一个文件从NFS文件系统拷贝到本地文件系统时，普通的cp命令总是会认为在NFS文件系统上的文件比在本地文件系统上的文件要新，因为前者的时间戳精度是微秒，而后者的时间戳精度不是微秒。Android专用的cp命令源码可以参考build/tools/acp目录。

RECOVERY_FROM_BOOT_PATCH描述的是一个patch文件，它的依赖规则如下所示：

```
# The system partition needs room for the recovery image as well.  We  
# now store the recovery image as a binary patch using the boot image  
# as the source (since they are very similar).  Generate the patch so  
# we can see how big it's going to be, and include that in the system  
# image size check calculation.  
ifneq ($(INSTALLED_RECOVERYIMAGE_TARGET),)  
intermediates := $(call intermediates-dir-for,PACKAGING,recovery_patch)  
RECOVERY_FROM_BOOT_PATCH := $(intermediates)/recovery_from_boot.p  
$(RECOVERY_FROM_BOOT_PATCH): $(INSTALLED_RECOVERYIMAGE_TARGET) \  
                      $(INSTALLED_BOOTIMAGE_TARGET) \  
          $(HOST_OUT_EXECUTABLES)/imgdiff \  
                  $(HOST_OUT_EXECUTABLES)/bsdiff  
    @echo "Construct recovery from boot"  
    mkdir -p $(dir $@)  
    PATH=$(HOST_OUT_EXECUTABLES):$$PATH $(HOST_OUT_EXECUTABLES)/imgdiff $(INSTALLED_BOOTIMAGE_TARGET) $(INSTALLED_RECOVERYIMAGE_TARGET) $@  
endif  
```
这个patch文件的名称为recovery_from_boot.p，保存在设备上system分区中，描述的是recovery.img与boot.img之间的差异。也就是说，在设备上，我们可以通过boot.img和recovery_from_boot.p文件生成一个recovery.img文件，使得设备可以进入recovery模式。

INSTALLED_SYSTEMIMAGE描述的是system.img镜像所包含的核心文件，它的依赖规则如下所示：

```
systemimage_intermediates := \  
    $(call intermediates-dir-for,PACKAGING,systemimage)  
BUILT_SYSTEMIMAGE := $(systemimage_intermediates)/system.img  

# $(1): output file  
define build-systemimage-target  
@echo "Target system fs image: $(1)"  
@mkdir -p $(dir $(1)) $(systemimage_intermediates) && rm -rf $(systemimage_intermediates)/system_image_info.txt  
$(call generate-userimage-prop-dictionary, $(systemimage_intermediates)/system_image_info.txt)  
$(hide) PATH=$(foreach p,$(INTERNAL_USERIMAGES_BINARY_PATHS),$(p):)$$PATH \  
      ./build/tools/releasetools/build_image.py \  
      $(TARGET_OUT) $(systemimage_intermediates)/system_image_info.txt $(1)  
endef  

$(BUILT_SYSTEMIMAGE): $(FULL_SYSTEMIMAGE_DEPS) $(INSTALLED_FILES_FILE)  
    $(call build-systemimage-target,$@)  
```
INSTALLED_SYSTEMIMAGE描述的是$(systemimage_intermediates)/system.img文件，它依赖于FULL_SYSTEMIMAGE_DEPS和INSTALLED_FILES_FILE，并且是通过调用函数build-systemimage-target来生成，而函数build-systemimage-target实际上又是通过调用[Python](http://lib.csdn.net/base/python)脚本build_image.py来生成system.img文件的。

INSTALLED_FILES_FILE的依赖规则如下所示：

```
# -----------------------------------------------------------------  
# installed file list  
# Depending on anything that $(BUILT_SYSTEMIMAGE) depends on.  
# We put installed-files.txt ahead of image itself in the dependency graph  
# so that we can get the size stat even if the build fails due to too large  
# system image.  
INSTALLED_FILES_FILE := $(PRODUCT_OUT)/installed-files.txt  
$(INSTALLED_FILES_FILE): $(FULL_SYSTEMIMAGE_DEPS)  
    @echo Installed file list: $@  
    @mkdir -p $(dir $@)  
    @rm -f $@  
    $(hide) build/tools/fileslist.py $(TARGET_OUT) > $@  
```
INSTALLED_FILES_FILE描述的是$(PRODUCT_OUT)/installed-files.txt文件，该文件描述了要打包在system.img镜像中去的文件列表，同时它与INSTALLED_SYSTEMIMAGE一样，也依赖于FULL_SYSTEMIMAGE_DEPS。        

 FULL_SYSTEMIMAGE_DEPS的定义如下所示：

```
FULL_SYSTEMIMAGE_DEPS := $(INTERNAL_SYSTEMIMAGE_FILES) $(INTERNAL_USERIMAGES_DEPS)  
```
INTERNAL_USERIMAGES_DEPS描述的是制作system.img镜像所依赖的工具。例如，如果要制作的system.img使用的是yaffs2文件系统，那么对应工具就是mkyaffs2image。

INTERNAL_SYSTEMIMAGE_FILES描述的是用来制作system.img镜像的文件，它的定义如下所示：

```
INTERNAL_SYSTEMIMAGE_FILES := $(filter $(TARGET_OUT)/%, \  
    $(ALL_PREBUILT) \  
    $(ALL_COPIED_HEADERS) \  
    $(ALL_GENERATED_SOURCES) \  
    $(ALL_DEFAULT_INSTALLED_MODULES) \  
    $(PDK_FUSION_SYSIMG_FILES) \  
    $(RECOVERY_RESOURCE_ZIP))  
```
从这里就可以看出，INTERNAL_SYSTEMIMAGE_FILES描述的就是从ALL_PREBUILT、ALL_COPIED_HEADERS、ALL_GENERATED_SOURCES、ALL_DEFAULT_INSTALLED_MODULES、PDK_FUSION_SYSIMG_FILES和RECOVERY_RESOURCE_ZIP中过滤出来的存放在TARGET_OUT目录下的那些文件，即在目标产品输出目录中的system子目录下那些文件。

ALL_PREBUILT是一个过时的变量，用来描述要拷贝到目标设备上去的文件，现在建议使用PRODUCT_COPY_FILES来描述要拷贝到目标设备上去的文件。

ALL_COPIED_HEADERS描述的是要拷贝到目标设备上去的头文件。

ALL_GENERATED_SOURCES描述的是要拷贝到目标设备上去的由工具自动生成的源代码文件。

ALL_DEFAULT_INSTALLED_MODULES描述的就是要安装要目标设备上的模块文件，这些模块文件是在build/core/main.mk文件中设置好并且传递给build/core/Makefile文件使用的。

PDK_FUSION_SYSIMG_FILES是从PDK(Platform Development Kit)提取出来的相关文件。PDK是Google为了解决Android碎片化问题而为手机厂商提供的一个新版本的、还未发布的Android开发包，目的是为了让手机厂商跟上官方新版Android的开发节奏。具体可以参考这篇文章：http://www.xinwengao[.NET](http://lib.csdn.net/base/dotnet)/release/af360/67079.shtml。

RECOVERY_RESOURCE_ZIP描述的是Android的recovery系统要使用的资源文件，对应于/system/etc目录下的recovery-resource.dat文件。

注意，ALL_DEFAULT_INSTALLED_MODULES描述的文件除了在build/core/main.mk文件中定义的模块文件之外，还包含以下这些文件：

1. 通过PRODUCT_COPY_FILES定义的要拷贝到目标设备上去的文件

```
unique_product_copy_files_pairs :=  
$(foreach cf,$(PRODUCT_COPY_FILES), \  
    $(if $(filter $(unique_product_copy_files_pairs),$(cf)),,\  
 $(eval unique_product_copy_files_pairs += $(cf))))  
unique_product_copy_files_destinations :=  
$(foreach cf,$(unique_product_copy_files_pairs), \  
    $(eval _src := $(call word-colon,1,$(cf))) \  
    $(eval _dest := $(call word-colon,2,$(cf))) \  
    $(call check-product-copy-files,$(cf)) \  
    $(if $(filter $(unique_product_copy_files_destinations),$(_dest)), \  
 $(info PRODUCT_COPY_FILES $(cf) ignored.), \  
 $(eval _fulldest := $(call append-path,$(PRODUCT_OUT),$(_dest))) \  
 $(if $(filter %.xml,$(_dest)),\  
     $(eval $(call copy-xml-file-checked,$(_src),$(_fulldest))),\  
     $(eval $(call copy-one-file,$(_src),$(_fulldest)))) \  
 <span style="color:#FF0000;"><strong>$(eval ALL_DEFAULT_INSTALLED_MODULES += $(_fulldest)) \</strong></span>  
 $(eval unique_product_copy_files_destinations += $(_dest))))  
```
2.  由ADDITIONAL_DEFAULT_PROPERTIES定义的属性所组成的default.prop文件

```
INSTALLED_DEFAULT_PROP_TARGET := $(TARGET_ROOT_OUT)/default.prop  
<span style="color:#FF0000;"><strong>ALL_DEFAULT_INSTALLED_MODULES += $(INSTALLED_DEFAULT_PROP_TARGET)</strong></span>  
ADDITIONAL_DEFAULT_PROPERTIES := \  
    $(call collapse-pairs, $(ADDITIONAL_DEFAULT_PROPERTIES))  
ADDITIONAL_DEFAULT_PROPERTIES += \  
    $(call collapse-pairs, $(PRODUCT_DEFAULT_PROPERTY_OVERRIDES))  
ADDITIONAL_DEFAULT_PROPERTIES := $(call uniq-pairs-by-first-component, \  
    $(ADDITIONAL_DEFAULT_PROPERTIES),=)  

$(INSTALLED_DEFAULT_PROP_TARGET):  
    @echo Target buildinfo: $@  
    @mkdir -p $(dir $@)  
    $(hide) echo "#" > $@; \  
     echo "# ADDITIONAL_DEFAULT_PROPERTIES" >> $@; \  
     echo "#" >> $@;  
    $(hide) $(foreach line,$(ADDITIONAL_DEFAULT_PROPERTIES), \  
 echo "$(line)" >> $@;)  
    build/tools/post_process_props.py $@  
```
3. 由ADDITIONAL_BUILD_PROPERTIES等定义的属性所组成的build.prop文件

```
INSTALLED_BUILD_PROP_TARGET := $(TARGET_OUT)/build.prop  
<span style="color:#FF0000;"><strong>ALL_DEFAULT_INSTALLED_MODULES += $(INSTALLED_BUILD_PROP_TARGET)</strong></span>  
ADDITIONAL_BUILD_PROPERTIES := \  
    $(call collapse-pairs, $(ADDITIONAL_BUILD_PROPERTIES))  
ADDITIONAL_BUILD_PROPERTIES := $(call uniq-pairs-by-first-component, \  
    $(ADDITIONAL_BUILD_PROPERTIES),=)  
......  
BUILDINFO_SH := build/tools/buildinfo.sh  
$(INSTALLED_BUILD_PROP_TARGET): $(BUILDINFO_SH) $(INTERNAL_BUILD_ID_MAKEFILE) $(BUILD_SYSTEM)/version_defaults.mk $(wildcard $(TARGET_DEVICE_DIR)/system.prop)  
    @echo Target buildinfo: $@  
    @mkdir -p $(dir $@)  
    $(hide) TARGET_BUILD_TYPE="$(TARGET_BUILD_VARIANT)" \  
     TARGET_DEVICE="$(TARGET_DEVICE)" \  
     PRODUCT_NAME="$(TARGET_PRODUCT)" \  
     PRODUCT_BRAND="$(PRODUCT_BRAND)" \  
     PRODUCT_DEFAULT_LANGUAGE="$(call default-locale-language,$(PRODUCT_LOCALES))" \  
     PRODUCT_DEFAULT_REGION="$(call default-locale-region,$(PRODUCT_LOCALES))" \  
     PRODUCT_DEFAULT_WIFI_CHANNELS="$(PRODUCT_DEFAULT_WIFI_CHANNELS)" \  
     PRODUCT_MODEL="$(PRODUCT_MODEL)" \  
     PRODUCT_MANUFACTURER="$(PRODUCT_MANUFACTURER)" \  
     PRIVATE_BUILD_DESC="$(PRIVATE_BUILD_DESC)" \  
     BUILD_ID="$(BUILD_ID)" \  
     BUILD_DISPLAY_ID="$(BUILD_DISPLAY_ID)" \  
     BUILD_NUMBER="$(BUILD_NUMBER)" \  
     PLATFORM_VERSION="$(PLATFORM_VERSION)" \  
     PLATFORM_SDK_VERSION="$(PLATFORM_SDK_VERSION)" \  
     PLATFORM_VERSION_CODENAME="$(PLATFORM_VERSION_CODENAME)" \  
     BUILD_VERSION_TAGS="$(BUILD_VERSION_TAGS)" \  
     TARGET_BOOTLOADER_BOARD_NAME="$(TARGET_BOOTLOADER_BOARD_NAME)" \  
     BUILD_FINGERPRINT="$(BUILD_FINGERPRINT)" \  
     TARGET_BOARD_PLATFORM="$(TARGET_BOARD_PLATFORM)" \  
     TARGET_CPU_ABI="$(TARGET_CPU_ABI)" \  
     TARGET_CPU_ABI2="$(TARGET_CPU_ABI2)" \  
     TARGET_AAPT_CHARACTERISTICS="$(TARGET_AAPT_CHARACTERISTICS)" \  
     bash $(BUILDINFO_SH) > $@  
    $(hide) if [ -f $(TARGET_DEVICE_DIR)/system.prop ]; then \  
       cat $(TARGET_DEVICE_DIR)/system.prop >> $@; \  
     fi  
    $(if $(ADDITIONAL_BUILD_PROPERTIES), \  
 $(hide) echo >> $@; \  
         echo "#" >> $@; \  
         echo "# ADDITIONAL_BUILD_PROPERTIES" >> $@; \  
         echo "#" >> $@; )  
    $(hide) $(foreach line,$(ADDITIONAL_BUILD_PROPERTIES), \  
 echo "$(line)" >> $@;)  
    $(hide) build/tools/post_process_props.py $@  
```
4. 用来描述event类型日志格式的event-log-tags文件

```
all_event_log_tags_file := $(TARGET_OUT_COMMON_INTERMEDIATES)/all-event-log-tags.txt  

event_log_tags_file := $(TARGET_OUT)/etc/event-log-tags  

# Include tags from all packages that we know about  
all_event_log_tags_src := \  
    $(sort $(foreach m, $(ALL_MODULES), $(ALL_MODULES.$(m).EVENT_LOG_TAGS)))  

# PDK builds will already have a full list of tags that needs to get merged  
# in with the ones from source  
pdk_fusion_log_tags_file := $(patsubst $(PRODUCT_OUT)/%,$(_pdk_fusion_intermediates)/%,$(filter $(event_log_tags_file),$(ALL_PDK_FUSION_FILES)))  

$(all_event_log_tags_file): PRIVATE_SRC_FILES := $(all_event_log_tags_src) $(pdk_fusion_log_tags_file)  
    $(hide) mkdir -p $(dir $@)  
    $(hide) build/tools/merge-event-log-tags.py -o $@ $(PRIVATE_SRC_FILES)  

# Include tags from all packages included in this product, plus all  
# tags that are part of the system (ie, not in a vendor/ or device/  
# directory).  
event_log_tags_src := \  
    $(sort $(foreach m,\  
      $(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_PACKAGES) \  
      $(call module-names-for-tag-list,user), \  
      $(ALL_MODULES.$(m).EVENT_LOG_TAGS)) \  
      $(filter-out vendor/% device/% out/%,$(all_event_log_tags_src)))  

$(event_log_tags_file): PRIVATE_SRC_FILES := $(event_log_tags_src) $(pdk_fusion_log_tags_file)  
$(event_log_tags_file): PRIVATE_MERGED_FILE := $(all_event_log_tags_file)  
$(event_log_tags_file): $(event_log_tags_src) $(all_event_log_tags_file) $(pdk_fusion_log_tags_file)  
    $(hide) mkdir -p $(dir $@)  
    $(hide) build/tools/merge-event-log-tags.py -o $@ -m $(PRIVATE_MERGED_FILE) $(PRIVATE_SRC_FILES)  

event-log-tags: $(event_log_tags_file)  

<strong><span style="color:#FF0000;">ALL_DEFAULT_INSTALLED_MODULES += $(event_log_tags_file)</span></strong>  
```
关于Android系统的event日志，可以参考Android日志系统驱动程序Logger源代码分析、Android应用程序框架层和系统运行库层日志系统源代码分析和Android日志系统Logcat源代码简要分析这个系列的文章。

5. 由于使用了BSD、GPL和Apache许可的模块而必须在发布时附带的Notice文件

```
target_notice_file_txt := $(TARGET_OUT_INTERMEDIATES)/NOTICE.txt  
target_notice_file_html := $(TARGET_OUT_INTERMEDIATES)/NOTICE.html  
target_notice_file_html_gz := $(TARGET_OUT_INTERMEDIATES)/NOTICE.html.gz  
tools_notice_file_txt := $(HOST_OUT_INTERMEDIATES)/NOTICE.txt  
tools_notice_file_html := $(HOST_OUT_INTERMEDIATES)/NOTICE.html  

kernel_notice_file := $(TARGET_OUT_NOTICE_FILES)/src/kernel.txt  
pdk_fusion_notice_files := $(filter $(TARGET_OUT_NOTICE_FILES)/%, $(ALL_PDK_FUSION_FILES))  
......  
# Install the html file at /system/etc/NOTICE.html.gz.  
# This is not ideal, but this is very late in the game, after a lot of  
# the module processing has already been done -- in fact, we used the  
# fact that all that has been done to get the list of modules that we  
# need notice files for.  
$(target_notice_file_html_gz): $(target_notice_file_html) | $(MINIGZIP)  
    $(hide) $(MINIGZIP) -9 < $< > $@  
installed_notice_html_gz := $(TARGET_OUT)/etc/NOTICE.html.gz  
$(installed_notice_html_gz): $(target_notice_file_html_gz) | $(ACP)  
    $(copy-file-to-target)  

# if we've been run my mm, mmm, etc, don't reinstall this every time  
ifeq ($(ONE_SHOT_MAKEFILE),)  
<span style="color:#FF0000;"><strong>ALL_DEFAULT_INSTALLED_MODULES += $(installed_notice_html_gz)</strong></span>  
endif  
```
6. 用来验证OTA更新的证书文件

```
<span style="color:#FF0000;"><strong>ALL_DEFAULT_INSTALLED_MODULES += $(TARGET_OUT_ETC)/security/otacerts.zip</strong></span>  
$(TARGET_OUT_ETC)/security/otacerts.zip: KEY_CERT_PAIR := $(DEFAULT_KEY_CERT_PAIR)  
$(TARGET_OUT_ETC)/security/otacerts.zip: $(addsuffix .x509.pem,$(DEFAULT_KEY_CERT_PAIR))  
    $(hide) rm -f $@  
    $(hide) mkdir -p $(dir $@)  
    $(hide) zip -qj $@ $<  

.PHONY: otacerts  
otacerts: $(TARGET_OUT_ETC)/security/otacerts.zip  
```
## 2. boot.img         

从前面的分析可以知道，build/core/main.mk文件定义了boot.img镜像文件的依赖规则，我们可以通过执行make bootimage命令来执行。其中，bootimage是一个伪目标，它依赖于INSTALLED_BOOTIMAGE_TARGET。

INSTALLED_BOOTIMAGE_TARGET定义在build/core/Makefile文件中，如下所示：

```
ifneq ($(strip $(TARGET_NO_KERNEL)),true)  

# -----------------------------------------------------------------  
# the boot image, which is a collection of other images.  
INTERNAL_BOOTIMAGE_ARGS := \  
    $(addprefix --second ,$(INSTALLED_2NDBOOTLOADER_TARGET)) \  
    --kernel $(INSTALLED_KERNEL_TARGET) \  
    --ramdisk $(INSTALLED_RAMDISK_TARGET)  

INTERNAL_BOOTIMAGE_FILES := $(filter-out --%,$(INTERNAL_BOOTIMAGE_ARGS))  

BOARD_KERNEL_CMDLINE := $(strip $(BOARD_KERNEL_CMDLINE))  
ifdef BOARD_KERNEL_CMDLINE  
INTERNAL_BOOTIMAGE_ARGS += --cmdline "$(BOARD_KERNEL_CMDLINE)"  
endif  

BOARD_KERNEL_BASE := $(strip $(BOARD_KERNEL_BASE))  
ifdef BOARD_KERNEL_BASE  
INTERNAL_BOOTIMAGE_ARGS += --base $(BOARD_KERNEL_BASE)  
endif  

BOARD_KERNEL_PAGESIZE := $(strip $(BOARD_KERNEL_PAGESIZE))  
ifdef BOARD_KERNEL_PAGESIZE  
INTERNAL_BOOTIMAGE_ARGS += --pagesize $(BOARD_KERNEL_PAGESIZE)  
endif  

INSTALLED_BOOTIMAGE_TARGET := $(PRODUCT_OUT)/boot.img  

ifeq ($(TARGET_BOOTIMAGE_USE_EXT2),true)  
tmp_dir_for_image := $(call intermediates-dir-for,EXECUTABLES,boot_img)/bootimg  
INTERNAL_BOOTIMAGE_ARGS += --tmpdir $(tmp_dir_for_image)  
INTERNAL_BOOTIMAGE_ARGS += --genext2fs $(MKEXT2IMG)  
$(INSTALLED_BOOTIMAGE_TARGET): $(MKEXT2IMG) $(INTERNAL_BOOTIMAGE_FILES)  
    $(call pretty,"Target boot image: $@")  
    $(hide) $(MKEXT2BOOTIMG) $(INTERNAL_BOOTIMAGE_ARGS) --output $@  

else # TARGET_BOOTIMAGE_USE_EXT2 != true  

$(INSTALLED_BOOTIMAGE_TARGET): $(MKBOOTIMG) $(INTERNAL_BOOTIMAGE_FILES)  
    $(call pretty,"Target boot image: $@")  
    $(hide) $(MKBOOTIMG) $(INTERNAL_BOOTIMAGE_ARGS) $(BOARD_MKBOOTIMG_ARGS) --output $@  
    $(hide) $(call assert-max-image-size,$@,$(BOARD_BOOTIMAGE_PARTITION_SIZE),raw)  
endif # TARGET_BOOTIMAGE_USE_EXT2  

else    # TARGET_NO_KERNEL  
# HACK: The top-level targets depend on the bootimage.  Not all targets  
# can produce a bootimage, though, and emulator targets need the ramdisk  
# instead.  Fake it out by calling the ramdisk the bootimage.  
# TODO: make the emulator use bootimages, and make mkbootimg accept  
#       kernel-less inputs.  
INSTALLED_BOOTIMAGE_TARGET := $(INSTALLED_RAMDISK_TARGET)  
endif  
```
在介绍boot.img之前，我们首先要介绍BootLoader。我们都知道，PC启动的时候，首先执行的是在BIOS上的代码，然后再由BIOS负责将Kernel加载起来执行。在[嵌入式](http://lib.csdn.net/base/embeddeddevelopment)世界里，BootLoader的作用就相当于PC的BIOS。

BootLoader的启动通常分为两个阶段。第一阶段在固态存储中(Flash)执行，负责初始化硬件，例如设置CPU工作模式、时钟频率以及初始化内存等，并且将第二阶段拷贝到RAM去准备执行。第二阶段就是在RAM中执行，因此速度会更快，主要负责建立内存映像，以及加载Kernel镜像和Ramdisk镜像。

BootLoader的第二阶段执行完成后，就开始启动Kernel了。Kernel负责启动各个子系统，例如CPU调度子系统和内存管理子系统等等。Kernel启动完成之后，就会将Ramdisk镜像安装为根系统，并且在其中找到一个init文件，将其启动为第一个进程。init进程启动就意味着系统进入到用户空间执行了，这时候各种用户空间运行时以及守护进程就会被加载起来。最终完成整个系统的启动过程。

BootLoader的第一阶段是固化在硬件中的，而boot.img的存在就是为BootLoader的第一阶段提供第二阶段、Kernel镜像、Ramdisk镜像，以及相应的启动参数等等。也就是说，boot.img主要包含有BootLoader的第二阶段、Kernel镜像和Ramdisk镜像。当然，BootLoader的第二阶段是可选的。当不存在BootLoader的第二阶段的时候，BootLoader的第一阶段启动完成后，就直接进入到kernel启动阶段。

boot.img镜像的文件格式定义在system/core/mkbootimg/bootimg.h中，如下所示：

```
/*  
** +-----------------+   
** | boot header     | 1 page  
** +-----------------+  
** | kernel          | n pages    
** +-----------------+  
** | ramdisk         | m pages    
** +-----------------+  
** | second stage    | o pages  
** +-----------------+  
**  
** n = (kernel_size + page_size - 1) / page_size  
** m = (ramdisk_size + page_size - 1) / page_size  
** o = (second_size + page_size - 1) / page_size  
**  
** 0. all entities are page_size aligned in flash  
** 1. kernel and ramdisk are required (size != 0)  
** 2. second is optional (second_size == 0 -> no second)  
** 3. load each element (kernel, ramdisk, second) at  
**    the specified physical address (kernel_addr, etc)  
** 4. prepare tags at tag_addr.  kernel_args[] is  
**    appended to the kernel commandline in the tags.  
** 5. r0 = 0, r1 = MACHINE_TYPE, r2 = tags_addr  
** 6. if second_size != 0: jump to second_addr  
**    else: jump to kernel_addr  
*/  
```
它由4部分组成：boot header、kernel、ramdisk和second state。每一个部分的大小都是以页为单位的，其中，boot header描述了kernel、ramdisk、sencond stage的加载地址、大小，以及kernel启动参数等等信息。

boot header的结构同样是定义在system/core/mkbootimg/bootimg.h中，如下所示：

```c
struct boot_img_hdr  
{  
    unsigned char magic[BOOT_MAGIC_SIZE];  

    unsigned kernel_size;  /* size in bytes */  
    unsigned kernel_addr;  /* physical load addr */  

    unsigned ramdisk_size; /* size in bytes */  
    unsigned ramdisk_addr; /* physical load addr */  

    unsigned second_size;  /* size in bytes */  
    unsigned second_addr;  /* physical load addr */  

    unsigned tags_addr;    /* physical addr for kernel tags */  
    unsigned page_size;    /* flash page size we assume */  
    unsigned unused[2];    /* future expansion: should be 0 */  

    unsigned char name[BOOT_NAME_SIZE]; /* asciiz product name */  

    unsigned char cmdline[BOOT_ARGS_SIZE];  

    unsigned id[8]; /* timestamp / checksum / sha1 / etc */  
};  
```
各个成员变量的含义如下所示：

**magic**：魔数，等于“ANDROID!”。

**kernel_size**：Kernel大小，以字节为单位。

**kernel_addr**：Kernel加载的物理地址。

**ramdisk_size**：Ramdisk大小，以字节为单位。

**ramdisk_addr**：Ramdisk加载的物理地址。

**second_size**：BootLoader第二阶段的大小，以字节为单位。

**second_addr**：BootLoader第二阶段加载的物理地址。

**tags_addr**：Kernel启动之前，它所需要的启动参数都会填入到由tags_addr所描述的一个物理地址中去。

**unused**：保留以后使用。

**page_size**：页大小。

**name**：产品名称。

** id**：时间戳、校验码等等。

理解了BootLoader的启动过程，以及boot.img的文件结构之后，就不难理解boot.img文件的生成过程了。

首先检查变量TARGET_NO_KERNEL的值是否等于true。如果等于true的话，那么就表示boot.img不包含有BootLoader第二阶段和Kernel，即它等同于Ramdisk镜像。如果不等于true的话，那么就通过INSTALLED_2NDBOOTLOADER_TARGET、INSTALLED_KERNEL_TARGET和INSTALLED_RAMDISK_TARGET获得BootLoader第二阶段、Kernel和Ramdisk对应的镜像文件，以及通过BOARD_KERNEL_CMDLINE、BOARD_KERNEL_BASE和BOARD_KERNEL_PAGESIZE获得Kernel启动命令行参数、内核基地址和页大小等参数。最后根据TARGET_BOOTIMAGE_USE_EXT2的值来决定是使用genext2fs还是mkbootimg工具来生成boot.img。

## 3. ramdisk.img

从前面的分析可以知道，build/core/main.mk文件定义了ramdisk.img镜像文件的依赖规则，我们可以通过执行make ramdisk命令来执行。其中，ramdisk是一个伪目标，它依赖于INSTALLED_RAMDISK_TARGET。

INSTALLED_RAMDISK_TARGET定义在build/core/Makefile文件中，如下所示：

```
INTERNAL_RAMDISK_FILES := $(filter $(TARGET_ROOT_OUT)/%, \  
    $(ALL_PREBUILT) \  
    $(ALL_COPIED_HEADERS) \  
    $(ALL_GENERATED_SOURCES) \  
    $(ALL_DEFAULT_INSTALLED_MODULES))  

BUILT_RAMDISK_TARGET := $(PRODUCT_OUT)/ramdisk.img  

# We just build this directly to the install location.  
INSTALLED_RAMDISK_TARGET := $(BUILT_RAMDISK_TARGET)  
$(INSTALLED_RAMDISK_TARGET): $(MKBOOTFS) $(INTERNAL_RAMDISK_FILES) | $(MINIGZIP)  
    $(call pretty,"Target ram disk: $@")  
    $(hide) $(MKBOOTFS) $(TARGET_ROOT_OUT) | $(MINIGZIP) > $@  
```
ALL_PREBUILT、ALL_COPIED_HEADERS、ALL_GENERATED_SOURCES和ALL_DEFAULT_INSTALLED_MODULES这几个变量的含义前面分析system.img的生成过程时已经介绍过了。因此，这里我们就很容易知道，ramdisk.img镜像实际上就是由这几个变量所描述的、保存在TARGET_ROOT_OUT目录中的文件所组成。与此相对应的是，system.img由保存在TARGET_OUT目录中的文件组成。

TARGET_ROOT_OUT和TARGET_OUT又分别是指向什么目录呢？假设我们的编译目标产品是模拟器，那么TARGET_ROOT_OUT和TARGET_OUT对应的目录就分别为out/target/product/generic/root和out/target/product/generic/system。

收集好对应的文件之后，就可以通过MKBOOTFS和MINIGZIP这两个变量描述的mkbootfs和minigzip工具来生成一个格式为cpio的ramdisk.img了。mkbootfs和minigzip这两个工具对应的源码分别位于system/core/cpio和external/zlib目录中。

## 4. userdata.img

userdata.img镜像描述的是Android系统的data分区，即/data目录，里面包含了用户安装的APP以及数据等等。

从前面的分析可以知道，build/core/main.mk文件定义了userdata.img镜像文件的依赖规则，我们可以通过执行make userdataimage命令来执行。其中，userdataimage是一个伪目标，它依赖于INSTALLED_USERDATAIMAGE_TARGET。

INSTALLED_USERDATAIMAGE_TARGET定义在build/core/Makefile文件中，如下所示：

```
INTERNAL_USERDATAIMAGE_FILES := \  
    $(filter $(TARGET_OUT_DATA)/%,$(ALL_DEFAULT_INSTALLED_MODULES))  

$(info $(TARGET_OUT_DATA))  

# If we build "tests" at the same time, make sure $(tests_MODULES) get covered.  
ifdef is_tests_build  
INTERNAL_USERDATAIMAGE_FILES += \  
    $(filter $(TARGET_OUT_DATA)/%,$(tests_MODULES))  
endif  

userdataimage_intermediates := \  
    $(call intermediates-dir-for,PACKAGING,userdata)  
BUILT_USERDATAIMAGE_TARGET := $(PRODUCT_OUT)/userdata.img  

define build-userdataimage-target  
$(call pretty,"Target userdata fs image: $(INSTALLED_USERDATAIMAGE_TARGET)")  
@mkdir -p $(TARGET_OUT_DATA)  
@mkdir -p $(userdataimage_intermediates) && rm -rf $(userdataimage_intermediates)/userdata_image_info.txt  
$(call generate-userimage-prop-dictionary, $(userdataimage_intermediates)/userdata_image_info.txt)  
$(hide) PATH=$(foreach p,$(INTERNAL_USERIMAGES_BINARY_PATHS),$(p):)$$PATH \  
      ./build/tools/releasetools/build_image.py \  
      $(TARGET_OUT_DATA) $(userdataimage_intermediates)/userdata_image_info.txt $(INSTALLED_USERDATAIMAGE_TARGET)  
$(hide) $(call assert-max-image-size,$(INSTALLED_USERDATAIMAGE_TARGET),$(BOARD_USERDATAIMAGE_PARTITION_SIZE),yaffs)  
endef  

# We just build this directly to the install location.  
INSTALLED_USERDATAIMAGE_TARGET := $(BUILT_USERDATAIMAGE_TARGET)  
$(INSTALLED_USERDATAIMAGE_TARGET): $(INTERNAL_USERIMAGES_DEPS) \  
                            $(INTERNAL_USERDATAIMAGE_FILES)  
    $(build-userdataimage-target)  
```
INSTALLED_USERDATAIMAGE_TARGET的值等于BUILT_USERDATAIMAGE_TARGET，后者指向的就是userdata.img文件，它依赖于INTERNAL_USERDATAIMAGE_FILES描述的文件，即那些由ALL_DEFAULT_INSTALLED_MODULES描述的、并且位于TARGET_OUT_DATA目录下的文件。假设我们的编译目标产品是模拟器，那么TARGET_OUT_DATA对应的目录就为out/target/product/generic/data。此外，如果我们在make userdataimage的时候，还带有一个额外的tests目标，那么那些将自己的tag设置为tests的模块也会被打包到userdata.img镜像中。

INSTALLED_USERDATAIMAGE_TARGET还依赖于INTERNAL_USERIMAGES_DEPS。前面在分析system.img镜像的生成过程时提到，INTERNAL_USERIMAGES_DEPS描述的是制作userdata.img镜像所依赖的工具。例如，如果要制作的userdata.img使用的是yaffs2文件系统，那么对应工具就是mkyaffs2image。

INSTALLED_USERDATAIMAGE_TARGET规则由函数build-userdataimage-target来执行，该函数通过build_image.py脚本来生成userdata.img镜像文件。

与system.img镜像类似，userdata.img镜像的生成也可以通过一个没有任何依赖的伪目标userdataimage-nodeps生成，如下所示：

```
.PHONY: userdataimage-nodeps  
userdataimage-nodeps: | $(INTERNAL_USERIMAGES_DEPS)  
    $(build-userdataimage-target)  
```
当我们执行make userdataimage-nodeps的时候，函数build-userdataimage-target就会被调用直接生成userdata.img文件。

## 5. recovery.img

recovery.img是设备进入recovery模式时所加载的镜像。recovery模式就是用来更新系统的，我们可以认为我们的设备具有两个系统。一个是正常的系统，它由boot.img、system.img、ramdisk.img和userdata.img等组成，另外一个就是recovery系统，由recovery.img组成。平时我们进入的都是正常的系统，只有当我们需要更新这个正常的系统时，才会进入到recovery系统。因此，我们可以将recovery系统理解为在Linux Kernel之上运行的一个小小的用户空间运行时。这个用户空间运行时可以访问我们平时经常使用的那个系统的文件，从而实现对它的更新。

在build/core/Makefile文件中，定义了一个伪目标recoveryimage，用来生成recovery.img，如下所示：

```
.PHONY: recoveryimage  
recoveryimage: $(INSTALLED_RECOVERYIMAGE_TARGET) $(RECOVERY_RESOURCE_ZIP)  
```
INSTALLED_RECOVERYIMAGE_TARGET描述的是组成recovery系统的模块文件，而RECOVERY_RESOURCE_ZIP描述的是recovery系统使用的资源包，它的定义如下所示：

```
INSTALLED_RECOVERYIMAGE_TARGET := $(PRODUCT_OUT)/recovery.img  

recovery_initrc := $(call include-path-for, recovery)/etc/init.rc  
recovery_kernel := $(INSTALLED_KERNEL_TARGET) # same as a non-recovery system  
recovery_ramdisk := $(PRODUCT_OUT)/ramdisk-recovery.img  
recovery_build_prop := $(INSTALLED_BUILD_PROP_TARGET)  
recovery_binary := $(call intermediates-dir-for,EXECUTABLES,recovery)/recovery  
recovery_resources_common := $(call include-path-for, recovery)/res  
recovery_resources_private := $(strip $(wildcard $(TARGET_DEVICE_DIR)/recovery/res))  
recovery_resource_deps := $(shell find $(recovery_resources_common) \  
$(recovery_resources_private) -type f)  
recovery_fstab := $(strip $(wildcard $(TARGET_DEVICE_DIR)/recovery.fstab))  
# Named '.dat' so we don't attempt to use imgdiff for patching it.  
RECOVERY_RESOURCE_ZIP := $(TARGET_OUT)/etc/recovery-resource.dat  

......  

INTERNAL_RECOVERYIMAGE_ARGS := \  
    $(addprefix --second ,$(INSTALLED_2NDBOOTLOADER_TARGET)) \  
    --kernel $(recovery_kernel) \  
    --ramdisk $(recovery_ramdisk)  

# Assumes this has already been stripped  
ifdef BOARD_KERNEL_CMDLINE  
INTERNAL_RECOVERYIMAGE_ARGS += --cmdline "$(BOARD_KERNEL_CMDLINE)"  
endif  
ifdef BOARD_KERNEL_BASE  
INTERNAL_RECOVERYIMAGE_ARGS += --base $(BOARD_KERNEL_BASE)  
endif  
BOARD_KERNEL_PAGESIZE := $(strip $(BOARD_KERNEL_PAGESIZE))  
ifdef BOARD_KERNEL_PAGESIZE  
INTERNAL_RECOVERYIMAGE_ARGS += --pagesize $(BOARD_KERNEL_PAGESIZE)  
endif  

# Keys authorized to sign OTA packages this build will accept.  The  
# build always uses dev-keys for this; release packaging tools will  
# substitute other keys for this one.  
OTA_PUBLIC_KEYS := $(DEFAULT_SYSTEM_DEV_CERTIFICATE).x509.pem  

# Generate a file containing the keys that will be read by the  
# recovery binary.  
RECOVERY_INSTALL_OTA_KEYS := \  
    $(call intermediates-dir-for,PACKAGING,ota_keys)/keys  
DUMPKEY_JAR := $(HOST_OUT_JAVA_LIBRARIES)/dumpkey.jar  
$(RECOVERY_INSTALL_OTA_KEYS): PRIVATE_OTA_PUBLIC_KEYS := $(OTA_PUBLIC_KEYS)  
$(RECOVERY_INSTALL_OTA_KEYS): extra_keys := $(patsubst %,%.x509.pem,$(PRODUCT_EXTRA_RECOVERY_KEYS))  
$(RECOVERY_INSTALL_OTA_KEYS): $(OTA_PUBLIC_KEYS) $(DUMPKEY_JAR) $(extra_keys)  
    @echo "DumpPublicKey: $@ <= $(PRIVATE_OTA_PUBLIC_KEYS) $(extra_keys)"  
    @rm -rf $@  
    @mkdir -p $(dir $@)  
    java -jar $(DUMPKEY_JAR) $(PRIVATE_OTA_PUBLIC_KEYS) $(extra_keys) > $@  

$(INSTALLED_RECOVERYIMAGE_TARGET): $(MKBOOTFS) $(MKBOOTIMG) $(MINIGZIP) \  
 $(INSTALLED_RAMDISK_TARGET) \  
 $(INSTALLED_BOOTIMAGE_TARGET) \  
 $(recovery_binary) \  
 $(recovery_initrc) $(recovery_kernel) \  
 $(INSTALLED_2NDBOOTLOADER_TARGET) \  
 $(recovery_build_prop) $(recovery_resource_deps) \  
 $(recovery_fstab) \  
 $(RECOVERY_INSTALL_OTA_KEYS)  
    @echo ----- Making recovery image ------  
    $(hide) rm -rf $(TARGET_RECOVERY_OUT)  
    $(hide) mkdir -p $(TARGET_RECOVERY_OUT)  
    $(hide) mkdir -p $(TARGET_RECOVERY_ROOT_OUT)/etc $(TARGET_RECOVERY_ROOT_OUT)/tmp  
    @echo Copying baseline ramdisk...  
    $(hide) cp -R $(TARGET_ROOT_OUT) $(TARGET_RECOVERY_OUT)  
    @echo Modifying ramdisk contents...  
    $(hide) rm -f $(TARGET_RECOVERY_ROOT_OUT)/init*.rc  
    $(hide) cp -f $(recovery_initrc) $(TARGET_RECOVERY_ROOT_OUT)/  
    $(hide) -cp $(TARGET_ROOT_OUT)/init.recovery.*.rc $(TARGET_RECOVERY_ROOT_OUT)/  
    $(hide) cp -f $(recovery_binary) $(TARGET_RECOVERY_ROOT_OUT)/sbin/  
    $(hide) cp -rf $(recovery_resources_common) $(TARGET_RECOVERY_ROOT_OUT)/  
    $(hide) $(foreach item,$(recovery_resources_private), \  
      cp -rf $(item) $(TARGET_RECOVERY_ROOT_OUT)/)  
    $(hide) $(foreach item,$(recovery_fstab), \  
      cp -f $(item) $(TARGET_RECOVERY_ROOT_OUT)/etc/recovery.fstab)  
    $(hide) cp $(RECOVERY_INSTALL_OTA_KEYS) $(TARGET_RECOVERY_ROOT_OUT)/res/keys  
    $(hide) cat $(INSTALLED_DEFAULT_PROP_TARGET) $(recovery_build_prop) \  
     > $(TARGET_RECOVERY_ROOT_OUT)/default.prop  
    $(hide) $(MKBOOTFS) $(TARGET_RECOVERY_ROOT_OUT) | $(MINIGZIP) > $(recovery_ramdisk)  
    $(hide) $(MKBOOTIMG) $(INTERNAL_RECOVERYIMAGE_ARGS) $(BOARD_MKBOOTIMG_ARGS) --output $@  
    $(hide) $(call assert-max-image-size,$@,$(BOARD_RECOVERYIMAGE_PARTITION_SIZE),raw)  
    @echo ----- Made recovery image: $@ --------  

$(RECOVERY_RESOURCE_ZIP): $(INSTALLED_RECOVERYIMAGE_TARGET)  
    $(hide) mkdir -p $(dir $@)  
    $(hide) find $(TARGET_RECOVERY_ROOT_OUT)/res -type f | sort | zip -0qrj $@ -@  
```
由于recovery.img和boot.img都是用来启动系统的，因此，它们的内容是比较类似，例如都包含有Kernel及其启动参数、Ramdisk，以及可选的BootLoader第二阶段。此外，recovery.img还包含有以下的主要内容：

1. 用来设置系统属性的文件build.prop和default.prop；

2. 进入recovery界面时用到的资源文件recovery_resources_common和recovery_resources_private；

3. 描述进入recovery模式时要安装的文件系统脚本recovery.fstab；

4. 在recovery模式执行OTA更新要用到的证书OTA_PUBLIC_KEYS；

5. 负责升级系统的recovery程序，它的源代码位于bootable/recovery目录中。

通过对比Android正常使用时的系统和进行升级时使用的Recovery系统，我们就可以对运行Linux系统的移动设备（或者说嵌入式设备）世界窥一斑而知全豹，实际上我们只需要有一个BootLoader、一个Linux Kernel，以及一个Ramdisk，就可以将它们运行起来。Ramdisk属于用户空间的范畴，它里面主要包含了Linux Kernel启动完成时所要加载的根文件系统。这个根文件系统的作用就是为整个系统提供一个init程序，以便Linux Kernel可以将控制权从内核空间转移到用户空间。系统提供什么样的功能给用户使用主要取决于在用户空间层运行了哪些服务和守护进程。这些服务守护进程都是直接或者间接地由init进程启动的。例如，对于recovery系统来说，它最主要提供的功能就是让用户升级系统，也就是它的主要用户空间服务就是负责执行长级任务的recovery程序。又如，我们正常使用的Android系统，它的主要用户空间服务和守护进程都是由system.img镜像提供的，此外，我们还可以通过userdata.img镜像来安装第三方应用来丰富手机的功能。因此，我们也可以说，init进程启动的服务决定了系统的功能。当然 ，这些功能是建立是硬件的基础之上的。

至此，我们就分析完成Android系统的主要镜像文件system.img、boot.img、ramdisk.img、userdata.img和recovery.img的制作过程了，希望通过这些镜像文件的制作过程以及它们的作用的介绍，使得小伙伴后对Android系统有更进一步的认识。同时，我们也结束了对Android编译系统的学习了。重新学习请参考[Android编译系统简要介绍和学习计划](http://blog.csdn.net/luoshengyang/article/details/18466779)一文。想了解更多Android相关技术，欢迎关注老罗的新浪微博：[http://weibo.com/shengyangluo](http://weibo.com/shengyangluo)。