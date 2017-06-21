[Android](http://lib.csdn.net/base/android)源代码在编译之前，要先对编译环境进行初始化，其中最主要就是指定编译的类型和目标设备的型号。[android](http://lib.csdn.net/base/android)的编译类型主要有eng、userdebug和user三种，而支持的目标设备型号则是不确定的，它们由当前的源码配置情况所决定。为了确定源码支持的所有目标设备型号，Android编译系统在初始化的过程中，需要在特定的目录中加载特定的配置文件。接下来本文就对上述的初始化过程进行详细分析。

老罗的新浪微博：[http://weibo.com/shengyangluo](http://weibo.com/shengyangluo)，欢迎关注！

《Android系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

对Android编译环境进行初始化很简单，分为两步。第一步是打开一个终端，并且将build/envsetup.sh加载到该终端中：

```
$ . ./build/envsetup.sh   
including device/asus/grouper/vendorsetup.sh  
including device/asus/tilapia/vendorsetup.sh  
including device/generic/armv7-a-neon/vendorsetup.sh  
including device/generic/armv7-a/vendorsetup.sh  
including device/generic/mips/vendorsetup.sh  
including device/generic/x86/vendorsetup.sh  
including device/lge/mako/vendorsetup.sh  
including device/samsung/maguro/vendorsetup.sh  
including device/samsung/manta/vendorsetup.sh  
including device/samsung/toroplus/vendorsetup.sh  
including device/samsung/toro/vendorsetup.sh  
including device/ti/panda/vendorsetup.sh  
including sdk/bash_completion/adb.bash  
```
从命令的输出可以知道，文件build/envsetup.sh在加载的过程中，又会在device目录中寻找那些名称为vendorsetup.sh的文件，并且也将它们加载到当前终端来。另外，在sdk/bash_completion目录下的adb.bash文件也会加载到当前终端来，它是用来实现adb命令的bash completion功能的。也就是说，加载了该文件之后，我们在运行adb相关的命令的时候，通过按tab键就可以帮助我们自动完成命令的输入。关于bash completion的知识，可以参考官方文档： 

http://www.gnu.org/s/bash/manual/bash.html#Programmable-Completion

。

第二步是执行命令lunch，如下所示：

```
$ lunch  

You're building on Linux  

Lunch menu... pick a combo:  
     1. full-eng  
     2. full_x86-eng  
     3. vbox_x86-eng  
     4. full_mips-eng  
     5. full_grouper-userdebug  
     6. full_tilapia-userdebug  
     7. mini_armv7a_neon-userdebug  
     8. mini_armv7a-userdebug  
     9. mini_mips-userdebug  
     10. mini_x86-userdebug  
     11. full_mako-userdebug  
     12. full_maguro-userdebug  
     13. full_manta-userdebug  
     14. full_toroplus-userdebug  
     15. full_toro-userdebug  
     16. full_panda-userdebug  
```
Which would you like? [full-eng]   

我们看到lunch命令输出了一个Lunch菜单，该菜单列出了当前Android源码支持的所有设备型号及其编译类型。例如，第一项“full-eng”表示的设备“full”即为模拟器，并且编译类型为“eng”即为工程机。

当我们选定了一个Lunch菜单项序号(1-16)之后，按回车键，就可以完成Android编译环境的初始化过程。例如，我们选择1，可以看到以下输出：

```
Which would you like? [full-eng] 1  

============================================  
PLATFORM_VERSION_CODENAME=REL  
PLATFORM_VERSION=4.2  
TARGET_PRODUCT=full  
TARGET_BUILD_VARIANT=eng  
TARGET_BUILD_TYPE=release  
TARGET_BUILD_APPS=  
TARGET_ARCH=arm  
TARGET_ARCH_VARIANT=armv7-a  
HOST_ARCH=x86  
HOST_OS=linux  
HOST_OS_EXTRA=Linux-3.8.0-31-generic-x86_64-with-Ubuntu-13.04-raring  
HOST_BUILD_TYPE=release  
BUILD_ID=JOP40C  
OUT_DIR=out  
============================================  
```
我们可以看到，lunch命令帮我们设置好了很多环境变量。通过设置这些环境变量，就配置好了Android编译环境。

通过图1我们就可以直观地看到Android编译环境初始化完成后，我们所获得的东西：

![img](http://img.blog.csdn.net/20140208122926593?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

图1 Android编译环境初始化完成之后

总体来说，Android编译环境初始化完成之后，获得了以下三样东西：

将vendor和device目录下的vendorsetup.sh文件加载到了当前终端；

新增了lunch、m、mm和mmm等命令；

通过执行lunch命令设置好了TARGET_PRODUCT、TARGET_BUILD_VARIANT、TARGET_BUILD_TYPE和TARGET_BUILD_APPS等环境变量。  

接下来我们就主要分析build/envsetup.sh文件的加载过程以及lunch命令的执行过程。

一. 文件build/envsetup.sh的加载过程

文件build/envsetup.sh是一个bash shell脚本，从它里面定义的函数hmm可以知道，它提供了lunch、m、mm和mmm等命令供我们初始化编译环境或者编译Android源码。

函数hmm的实现如下所示：

```
function hmm() {  
cat <<EOF  
Invoke ". build/envsetup.sh" from your shell to add the following functions to your environment:  
- lunch:   lunch <product_name>-<build_variant>  
- tapas:   tapas [<App1> <App2> ...] [arm|x86|mips] [eng|userdebug|user]  
- croot:   Changes directory to the top of the tree.  
- m:       Makes from the top of the tree.  
- mm:      Builds all of the modules in the current directory.  
- mmm:     Builds all of the modules in the supplied directories.  
- cgrep:   Greps on all local C/C++ files.  
- jgrep:   Greps on all local Java files.  
- resgrep: Greps on all local res/*.xml files.  
- godir:   Go to the directory containing a file.  

Look at the source to view more functions. The complete list is:  
EOF  
    T=$(gettop)  
    local A  
    A=""  
    for i in `cat $T/build/envsetup.sh | sed -n "/^function /s/function [a−z]∗.*/\1/p" | sort`; do  
A="$A $i"  
    done  
    echo $A  
}  
```
我们在当前终端中执行hmm命令即可以看到函数hmm的完整输出。

函数hmm主要完成三个工作：

调用另外一个函数gettop获得Android源码的根目录T。 

通过cat命令显示一个[Here Document](http://tldp.org/LDP/abs/html/here-docs.html)，说明$T/build/envsetup.sh文件加载到当前终端后所提供的主要命令。

通过sed命令解析$T/build/envsetup.sh文件，并且获得在里面定义的所有函数的名称，这些函数名称就是$T/build/envsetup.sh文件加载到当前终端后提供的所有命令。

注意，sed命令是一个强大的文本分析工具，它以行为单位为执行文本替换、删除、新增和选取等操作。函数hmm通过执行以下的sed命令来获得在$T/build/envsetup.sh文件定义的函数的名称：

```
sed -n "/^function /s/function [a−z]∗.*/\1/p"  
```
它表示对所有以“function ”开头的行，如果紧接在“function ”后面的字符串仅由字母a-z和下横线(_)组成，那么就将这个字符串提取出来。这正好就对应于shell脚本里面函数的定义。

文件build/envsetup.sh除了定义一堆函数之外，还有一个重要的代码段，如下所示：

```
# Execute the contents of any vendorsetup.sh files we can find.  
for f in `/bin/ls vendor/*/vendorsetup.sh vendor/*/*/vendorsetup.sh device/*/*/vendorsetup.sh 2> /dev/null`  
do  
    echo "including $f"  
    . $f  
done  
unset f  
```
这个for循环遍历vendor目录下的一级子目录和二级子目录以及device目录下的二级子目录中的vendorsetup.sh文件，并且通过source命令(.)将它们加载当前终端来。vendor和device相应子目录下的vendorsetup.sh文件的实现很简单，它们主要就是添加相应的设备型号及其编译类型支持到Lunch菜单中去。

例如，device/samsung/maguro目录下的vendorsetup.sh文件的实现如下所示：

```
add_lunch_combo full_maguro-userdebug  
```
它调用函数add_lunch_combo添加一个名称为“full_maguro-userdebug”的菜单项到Lunch菜单去。

函数add_lunch_combo定义在build/envsetup.sh文件中，它的实现如下所示：

```
function add_lunch_combo()  
{  
    local new_combo=$1  
    local c  
    for c in ${LUNCH_MENU_CHOICES[@]} ; do  
 if [ "$new_combo" = "$c" ] ; then  
     return  
 fi  
    done  
    LUNCH_MENU_CHOICES=(${LUNCH_MENU_CHOICES[@]} $new_combo)  
}  
```
传递给函数add_lunch_combo的参数保存在位置参数$1中，接着又保存在一个本地变量new_combo中，用来表示一个要即将要添加的Lunch菜单项。函数首先是在数组LUNCH_MENU_CHOICES中检查要添加的菜单项是否已经存在。只有在不存在的情况下，才会将它添加到数组LUNCH_MENU_CHOICES中去。注意，${LUNCH_MENU_CHOICES[@]}表示数组LUNCH_MENU_CHOICES的所有元素。

数组LUNCH_MENU_CHOICES是定义在文件build/envsetup.sh的一个全局变量，当文件build/envsetup.sh被加载的时候，这个数组会被初始化为化full-eng、full_x86-eng、vbox_x86-eng和full_mips-eng，如下所示：

```
# add the default one here  
add_lunch_combo full-eng  
add_lunch_combo full_x86-eng  
add_lunch_combo vbox_x86-eng  
add_lunch_combo full_mips-eng  
```
这样当文件build/envsetup.sh加载完成之后，数组LUNCH_MENU_CHOICES就包含了当前源码支持的所有设备型号及其编译类型，于是当接下来我们执行lunch命令的时候，就可以通过数组LUNCH_MENU_CHOICES看到一个完整的Lunch藤蔓。

二. lunch命令的执行过程

lunch命令实际上是定义在文件build/envsetup.sh的一个函数，它的实现如下所示：

```
function lunch()  
{  
    local answer  

    if [ "$1" ] ; then  
 answer=$1  
    else  
 print_lunch_menu  
 echo -n "Which would you like? [full-eng] "  
 read answer  
    fi  

    local selection=  

    if [ -z "$answer" ]  
    then  
 selection=full-eng  
    elif (echo -n $answer | grep -q -e "^[0-9][0-9]*$")  
    then  
 if [ $answer -le ${#LUNCH_MENU_CHOICES[@]} ]  
 then  
     selection=${LUNCH_MENU_CHOICES[$(($answer-1))]}  
 fi  
    elif (echo -n $answer | grep -q -e "^[^\-][^\-]*-[^\-][^\-]*$")  
    then  
 selection=$answer  
    fi  

    if [ -z "$selection" ]  
    then  
 echo  
 echo "Invalid lunch combo: $answer"  
 return 1  
    fi  

    export TARGET_BUILD_APPS=  

    local product=$(echo -n $selection | sed -e "s/-.*$//")  
    check_product $product  
    if [ $? -ne 0 ]  
    then  
 echo  
 echo "** Don't have a product spec for: '$product'"  
 echo "** Do you have the right repo manifest?"  
 product=  
    fi  

    local variant=$(echo -n $selection | sed -e "s/^[^\-]*-//")  
    check_variant $variant  
    if [ $? -ne 0 ]  
    then  
 echo  
 echo "** Invalid variant: '$variant'"  
 echo "** Must be one of ${VARIANT_CHOICES[@]}"  
 variant=  
    fi  

    if [ -z "$product" -o -z "$variant" ]  
    then  
 echo  
 return 1  
    fi  

    export TARGET_PRODUCT=$product  
    export TARGET_BUILD_VARIANT=$variant  
    export TARGET_BUILD_TYPE=release  

    echo  

    set_stuff_for_environment  
    printconfig  
}  
```
函数lunch的执行逻辑如下所示：

1. 检查是否带有参数，即位置参数$1是否等于空。如果不等于空的话，就表明带有参数，并且该参数是用来指定要编译的设备型号及其编译类型的。如果等于空的话，那么就调用另外一个函数print_lunch_menu来显示Lunch菜单项，并且通过调用read函数来等待用户输入。无论通过何种方式，最终变量answer的值就保存了用户所指定的备型号及其编译类型。

2. 对变量answer的值的合法性进行检查。如果等于空的话，就将它设置为默认值“full-eng”。如果不等于空的话，就分为三种情况考虑。第一种情况是值为数字，那么就需要确保该数字的大小不能超过Lunch菜单项的个数。在这种情况下，会将输入的数字索引到数组LUNCH_MENU_CHOICES中去，以便获得一个用来表示设备型号及其编译类型的文本。第二种情况是非数字文本，那么就需要确保该文本符合<product>-<variant>的形式，其中<product>表示设备型号，而<variant>表示编译类型 。第三种情况是除了前面两种情况之外的所有情况，这是非法的。经过合法性检查后，变量selection代表了用户所指定的备型号及其编译类型，如果它的值是非法的，即它的值等于空，那么函数lunch就不往下执行了。

3. 接下来是解析变量selection的值，也就是通过sed命令将它的<product>和<variant>值提取出来，并且分别保存在变量product和variant中。提取出来的product和variant值有可能是不合法的，因此需要进一步通过调用函数check_product和check_variant来检查。一旦检查失败，也就是函数check_product和check_variant的返回值$?等于非0，那么函数lunch就不往下执行了。

4. 通过以上合法性检查之后，就将变量product和variant的值保存在环境变量TARGET_PRODUCT和TARGET_BUILD_VARIANT中。此外，另外一个环境变量TARGET_BUILD_TYPE的值会被设置为"release"，表示此次编译是一个release版本的编译。另外，前面还有一个环境变量TARGET_BUILD_APPS，它的值被函数lunch设置为空，用来表示此次编译是对整个系统进行编译。如果环境变量TARGET_BUILD_APPS的值不等于空，那么就表示此次编译是只对某些APP模块进行编译，而这些APP模块就是由环境变量TARGET_BUILD_APPS来指定的。

5. 调用函数set_stuff_for_environment来配置环境，例如设置[Java ](http://lib.csdn.net/base/java)SDK路径和交叉编译工具路径等。

6. 调用函数printfconfig来显示已经配置好的编译环境参数。

在上述执行过程中，函数check_product、check_variant和printconfig是比较关键的，因此接下来我们就继续分析它们的实现。

函数check_product定义在文件build/envsetup.sh中，它的实现如下所示：

```
# check to see if the supplied product is one we can build  
function check_product()  
{  
    T=$(gettop)  
    if [ ! "$T" ]; then  
 echo "Couldn't locate the top of the tree.  Try setting TOP." >&2  
 return  
    fi  
    CALLED_FROM_SETUP=true BUILD_SYSTEM=build/core \  
 TARGET_PRODUCT=$1 \  
 TARGET_BUILD_VARIANT= \  
 TARGET_BUILD_TYPE= \  
 TARGET_BUILD_APPS= \  
 get_build_var TARGET_DEVICE > /dev/null  
    # hide successful answers, but allow the errors to show  
}  
```
函数gettop用来返回Android源代码工程的根目录。函数check_product需要在Android源代码工程根目录或者子目录下调用。否则的话，函数check_product就出错返回。

接下来函数check_product设置几个环境变量，其中最重要的是前面三个CALLED_FROM_SETUP、BUILD_SYSTEM和TARGET_PRODUCT。环境变量CALLED_FROM_SETUP的值等于true表示接下来执行的make命令是用来初始化Android编译环境的。环境变量BUILD_SYSTEM用来指定Android编译系统的核心目录，它的值被设置为build/core。环境变量TARGET_PRODUCT用来表示要检查的产品名称（也就是我们前面说的设备型号），它的值被设置为$1，即函数check_product的调用参数。

最后函数check_product调用函数get_build_var来检查由环境变量TARGET_PRODUCT指定的产品名称是否合法，注意，它的调用参数为TARGET_DEVICE。

函数get_build_var定义在文件build/envsetup.sh中，它的实现如下所示：

```
# Get the exact value of a build variable.  
function get_build_var()  
{  
    T=$(gettop)  
    if [ ! "$T" ]; then  
 echo "Couldn't locate the top of the tree.  Try setting TOP." >&2  
 return  
    fi  
    CALLED_FROM_SETUP=true BUILD_SYSTEM=build/core \  
make --no-print-directory -C "$T" -f build/core/config.mk dumpvar-$1  
}  
```
这里就可以看到，函数get_build_var实际上就是通过make命令在Android源代码工程根目录中执行build/core/config.mk文件，并且将make目标设置为dumpvar-$1，也就是dumpvar-TARGET_DEVICE。

文件build/core/config.mk的内容比较多，这里我们只关注与产品名称合法性检查相关的逻辑，这些逻辑也基本上涵盖了Android编译系统初始化的逻辑，如下所示：

```
......  

# ---------------------------------------------------------------  
# Define most of the global variables.  These are the ones that  
# are specific to the user's build configuration.  
include $(BUILD_SYSTEM)/envsetup.mk  

# Boards may be defined under $(SRC_TARGET_DIR)/board/$(TARGET_DEVICE)  
# or under vendor/*/$(TARGET_DEVICE).  Search in both places, but  
# make sure only one exists.  
# Real boards should always be associated with an OEM vendor.  
board_config_mk := \  
    $(strip $(wildcard \  
 $(SRC_TARGET_DIR)/board/$(TARGET_DEVICE)/BoardConfig.mk \  
 device/*/$(TARGET_DEVICE)/BoardConfig.mk \  
 vendor/*/$(TARGET_DEVICE)/BoardConfig.mk \  
    ))  
ifeq ($(board_config_mk),)  
$(error No config file found for TARGET_DEVICE $(TARGET_DEVICE))  
endif  
ifneq ($(words $(board_config_mk)),1)  
$(error Multiple board config files for TARGET_DEVICE $(TARGET_DEVICE): $(board_config_mk))  
endif  
include $(board_config_mk)  

......  

include $(BUILD_SYSTEM)/dumpvar.mk  
```
上述代码主要就是将envsetup.mk、BoardConfig,mk和dumpvar.mk三个Makefile片段文件加载进来。其中，envsetup.mk文件位于$(BUILD_SYSTEM)目录中，也就是build/core目录中，BoardConfig.mk文件的位置主要就是由环境变量TARGET_DEVICE来确定，它是用来描述目标产品的硬件模块信息的，例如CPU体系结构。环境变量TARGET_DEVICE用来描述目标设备，它的值是在envsetup.mk文件加载的过程中确定的。一旦目标设备确定后，就可以在$(SRC_TARGET_DIR)/board/$(TARGET_DEVICE)、device/*/$(TARGET_DEVICE)和vendor/*/$(TARGET_DEVICE)目录中找到对应的BoradConfig.mk文件。注意，变量SRC_TARGET_DIR的值等于build/target。最后，dumpvar.mk文件也是位于build/core目录中，它用来打印已经配置好的编译环境信息。

接下来我们就通过进入到build/core/envsetup.mk文件来分析变量TARGET_DEVICE的值是如何确定的：

```
# Read the product specs so we an get TARGET_DEVICE and other  
# variables that we need in order to locate the output files.  
include $(BUILD_SYSTEM)/product_config.mk  
```
它通过加载另外一个文件build/core/product_config.mk文件来确定变量TARGET_DEVICE以及其它与目标产品相关的变量的值。

文件build/core/product_config.mk的内容很多，这里我们只关注变量TARGET_DEVICE设置相关的逻辑，如下所示：

```
......  

ifneq ($(strip $(TARGET_BUILD_APPS)),)  
# An unbundled app build needs only the core product makefiles.  
all_product_configs := $(call get-product-makefiles,\  
    $(SRC_TARGET_DIR)/product/AndroidProducts.mk)  
else  
# Read in all of the product definitions specified by the AndroidProducts.mk  
# files in the tree.  
all_product_configs := $(get-all-product-makefiles)  
endif  

# all_product_configs consists items like:  
# <product_name>:<path_to_the_product_makefile>  
# or just <path_to_the_product_makefile> in case the product name is the  
# same as the base filename of the product config makefile.  
current_product_makefile :=  
all_product_makefiles :=  
$(foreach f, $(all_product_configs),\  
    $(eval _cpm_words := $(subst :,$(space),$(f)))\  
    $(eval _cpm_word1 := $(word 1,$(_cpm_words)))\  
    $(eval _cpm_word2 := $(word 2,$(_cpm_words)))\  
    $(if $(_cpm_word2),\  
 $(eval all_product_makefiles += $(_cpm_word2))\  
 $(if $(filter $(TARGET_PRODUCT),$(_cpm_word1)),\  
     $(eval current_product_makefile += $(_cpm_word2)),),\  
 $(eval all_product_makefiles += $(f))\  
 $(if $(filter $(TARGET_PRODUCT),$(basename $(notdir $(f)))),\  
     $(eval current_product_makefile += $(f)),)))  
_cpm_words :=  
_cpm_word1 :=  
_cpm_word2 :=  
current_product_makefile := $(strip $(current_product_makefile))  
all_product_makefiles := $(strip $(all_product_makefiles))  

ifneq (,$(filter product-graph dump-products, $(MAKECMDGOALS)))  
# Import all product makefiles.  
$(call import-products, $(all_product_makefiles))  
else  
# Import just the current product.  
ifndef current_product_makefile  
$(error Cannot locate config makefile for product "$(TARGET_PRODUCT)")  
endif  
ifneq (1,$(words $(current_product_makefile)))  
$(error Product "$(TARGET_PRODUCT)" ambiguous: matches $(current_product_makefile))  
endif  
$(call import-products, $(current_product_makefile))  
endif  # Import all or just the current product makefile  

......  

# Convert a short name like "sooner" into the path to the product  
# file defining that product.  
#  
INTERNAL_PRODUCT := $(call resolve-short-product-name, $(TARGET_PRODUCT))  
ifneq ($(current_product_makefile),$(INTERNAL_PRODUCT))  
$(error PRODUCT_NAME inconsistent in $(current_product_makefile) and $(INTERNAL_PRODUCT))  
endif  
current_product_makefile :=  
all_product_makefiles :=  
all_product_configs :=  

# Find the device that this product maps to.  
TARGET_DEVICE := $(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_DEVICE)  

......  
```
上述代码的执行逻辑如下所示：

检查环境变量TARGET_BUILD_APPS的值是否等于空。如果不等于空，那么就说明此次编译不是针对整个系统，因此只要将核心的产品相关的Makefile文件加载进来就行了，否则的话，就要将所有与产品相关的Makefile文件加载进来的。核心产品Makefile文件在$(SRC_TARGET_DIR)/product/AndroidProducts.mk文件中指定，也就是在build/target/product/AndroidProducts.mk文件，通过调用函数get-product-makefiles可以获得。所有与产品相关的Makefile文件可以通过另外一个函数get-all-product-makefiles获得。无论如何，最终获得的产品Makefie文件列表保存在变量all_product_configs中。

遍历变量all_product_configs所描述的产品Makefile列表，并且在这些Makefile文件中，找到名称与环境变量TARGET_PRODUCT的值相同的文件，保存在另外一个变量current_product_makefile中，作为需要为当前指定的产品所加载的Makefile文件列表。在这个过程当中，上一步找到的所有的产品Makefile文件也会保存在变量all_product_makefiles中。注意，环境变量TARGET_PRODUCT的值是在我们执行lunch命令的时候设置并且传递进来的。

如果指定的make目标等于product-graph或者dump-products，那么就将所有的产品相关的Makefile文件加载进来，否则的话，只加载与目标产品相关的Makefile文件。从前面的分析可以知道，此时的make目标为dumpvar-TARGET_DEVICE，因此接下来只会加载与目标产品，即$(TARGET_PRODUCT)，相关的Makefile文件，这是通过调用另外一个函数import-products实现的。

调用函数resolve-short-product-name解析环境变量TARGET_PRODUCT的值，将它变成一个Makefile文件路径。并且保存在变量INTERNAL_PRODUCT中。这里要求变量INTERNAL_PRODUCT和current_product_makefile的值相等，否则的话，就说明用户指定了一个非法的产品名称。

找到一个名称为PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_DEVICE的变量，并且将它的值保存另外一个变量TARGET_DEVICE中。变量PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_DEVICE是在加载产品Makefile文件的过程中定义的，用来描述当前指定的产品的名称。

上述过程主要涉及到了get-all-product-makefiles、import-products和resolve-short-product-name三个关键函数，理解它们的执行过程对理解Android编译系统的初始化过程很有帮助，接下来我们分别分析它们的实现。

函数get-all-product-makefiles定义在文件build/core/product.mk中，如下所示：

```
#  
# Returns the sorted concatenation of all PRODUCT_MAKEFILES  
# variables set in all AndroidProducts.mk files.  
# $(call ) isn't necessary.  
#  
define get-all-product-makefiles  
$(call get-product-makefiles,$(_find-android-products-files))  
endef  
```
它首先是调用函数_find-android-products-files来找到Android源代码目录中定义的所有AndroidProducts.mk文件，然后再调用函数get-product-makefiles获得在这里AndroidProducts.mk文件里面定义的产品Makefile文件。

函数_find-android-products-files也是定义在文件build/core/product.mk中，如下所示：

```
#  
# Returns the list of all AndroidProducts.mk files.  
# $(call ) isn't necessary.  
#  
define _find-android-products-files  
$(shell test -d device && find device -maxdepth 6 -name AndroidProducts.mk) \  
$(shell test -d vendor && find vendor -maxdepth 6 -name AndroidProducts.mk) \  
$(SRC_TARGET_DIR)/product/AndroidProducts.mk  
endef  
```
从这里就可以看出，Android源代码目录中定义的所有AndroidProducts.mk文件位于device、vendor或者build/target/product目录或者相应的子目录（最深是6层）中。

函数get-product-makefiles也是定义在文件build/core/product.mk中，如下所示：

```
#  
# Returns the sorted concatenation of PRODUCT_MAKEFILES  
# variables set in the given AndroidProducts.mk files.  
# $(1): the list of AndroidProducts.mk files.  
#  
define get-product-makefiles  
$(sort \  
$(foreach f,$(1), \  
    $(eval PRODUCT_MAKEFILES :=) \  
    $(eval LOCAL_DIR := $(patsubst %/,%,$(dir $(f)))) \  
    $(eval include $(f)) \  
    $(PRODUCT_MAKEFILES) \  
) \  
$(eval PRODUCT_MAKEFILES :=) \  
$(eval LOCAL_DIR :=) \  
)  
endef  
```
这个函数实际上就是遍历参数$1所描述的AndroidProucts.mk文件列表，并且将定义在这些AndroidProucts.mk文件中的变量PRODUCT_MAKEFILES的值提取出来，形成一个列表返回给调用者。

例如，在build/target/product/AndroidProducts.mk文件中，变量PRODUCT_MAKEFILES的值如下所示：

```
# Unbundled apps will be built with the most generic product config.  
ifneq ($(TARGET_BUILD_APPS),)  
PRODUCT_MAKEFILES := \  
    $(LOCAL_DIR)/full.mk \  
    $(LOCAL_DIR)/full_x86.mk \  
    $(LOCAL_DIR)/full_mips.mk  
else  
PRODUCT_MAKEFILES := \  
    $(LOCAL_DIR)/core.mk \  
    $(LOCAL_DIR)/generic.mk \  
    $(LOCAL_DIR)/generic_x86.mk \  
    $(LOCAL_DIR)/generic_mips.mk \  
    $(LOCAL_DIR)/full.mk \  
    $(LOCAL_DIR)/full_x86.mk \  
    $(LOCAL_DIR)/full_mips.mk \  
    $(LOCAL_DIR)/vbox_x86.mk \  
    $(LOCAL_DIR)/sdk.mk \  
    $(LOCAL_DIR)/sdk_x86.mk \  
    $(LOCAL_DIR)/sdk_mips.mk \  
    $(LOCAL_DIR)/large_emu_hw.mk  
endif  
```
这里列出的每一个文件都对应于一个产品。

我们再来看函数import-products的实现，它定义在文件build/core/product.mk中，如下所示：

```
#  
# $(1): product makefile list  
#  
#TODO: check to make sure that products have all the necessary vars defined  
define import-products  
$(call import-nodes,PRODUCTS,$(1),$(_product_var_list))  
endef  
```
它调用另外一个函数import-nodes来加载由参数$1所指定的产品Makefile文件，并且指定了另外两个参数PRODUCTS和$(_product_var_list)。其中，变量_product_var_list也是定义在文件build/core/product.mk中，它的值如下所示：

```
_product_var_list := \  
    PRODUCT_NAME \  
    PRODUCT_MODEL \  
    PRODUCT_LOCALES \  
    PRODUCT_AAPT_CONFIG \  
    PRODUCT_AAPT_PREF_CONFIG \  
    PRODUCT_PACKAGES \  
    PRODUCT_PACKAGES_DEBUG \  
    PRODUCT_PACKAGES_ENG \  
    PRODUCT_PACKAGES_TESTS \  
    PRODUCT_DEVICE \  
    PRODUCT_MANUFACTURER \  
    PRODUCT_BRAND \  
    PRODUCT_PROPERTY_OVERRIDES \  
    PRODUCT_DEFAULT_PROPERTY_OVERRIDES \  
    PRODUCT_CHARACTERISTICS \  
    PRODUCT_COPY_FILES \  
    PRODUCT_OTA_PUBLIC_KEYS \  
    PRODUCT_EXTRA_RECOVERY_KEYS \  
    PRODUCT_PACKAGE_OVERLAYS \  
    DEVICE_PACKAGE_OVERLAYS \  
    PRODUCT_TAGS \  
    PRODUCT_SDK_ADDON_NAME \  
    PRODUCT_SDK_ADDON_COPY_FILES \  
    PRODUCT_SDK_ADDON_COPY_MODULES \  
    PRODUCT_SDK_ADDON_DOC_MODULES \  
    PRODUCT_DEFAULT_WIFI_CHANNELS \  
    PRODUCT_DEFAULT_DEV_CERTIFICATE \  
    PRODUCT_RESTRICT_VENDOR_FILES \  
    PRODUCT_VENDOR_KERNEL_HEADERS \  
    PRODUCT_FACTORY_RAMDISK_MODULES \  
    PRODUCT_FACTORY_BUNDLE_MODULES  
```
它描述的是在产品Makefile文件中定义在各种变量。

函数import-nodes定义在文件build/core/node_fns.mk中，如下所示：

```
#  
# $(1): output list variable name, like "PRODUCTS" or "DEVICES"  
# $(2): list of makefiles representing nodes to import  
# $(3): list of node variable names  
#  
define import-nodes  
$(if \  
$(foreach _in,$(2), \  
    $(eval _node_import_context := _nic.$(1).[[$(_in)]]) \  
    $(if $(_include_stack),$(eval $(error ASSERTION FAILED: _include_stack \  
         should be empty here: $(_include_stack))),) \  
    $(eval _include_stack := ) \  
    $(call _import-nodes-inner,$(_node_import_context),$(_in),$(3)) \  
    $(call move-var-list,$(_node_import_context).$(_in),$(1).$(_in),$(3)) \  
    $(eval _node_import_context :=) \  
    $(eval $(1) := $($(1)) $(_in)) \  
    $(if $(_include_stack),$(eval $(error ASSERTION FAILED: _include_stack \  
         should be empty here: $(_include_stack))),) \  
) \  
,)  
endef  
```
这个函数主要是做了三件事情：

调用函数_import-nodes-inner将参数$2描述的每一个产品Makefile文件加载进来。

调用函数move-var-list将定义在前面所加载的产品Makefile文件里面的由参数$3指定的变量的值分别拷贝到另外一组独立的变量中。

将参数$2描述的每一个产品Makefile文件路径以空格分隔保存在参数$1所描述的变量中，也就是保存在变量PRODUCTS中。

上述第二件事情需要进一步解释一下。由于当前加载的每一个文件都会定义相同的变量，为了区分这些变量，我们需要在这些变量前面加一些前缀。例如，假设加载了build/target/product/full.mk这个产品Makefile文件，它里面定义了以下几个变量：

```
# Overrides  
PRODUCT_NAME := full  
PRODUCT_DEVICE := generic  
PRODUCT_BRAND := Android  
PRODUCT_MODEL := Full Android on Emulator  
```
当调用了函数move-var-list对它进行解析后，就会得到以下的新变量：

```
PRODUCTS.build/target/product/full.mk.PRODUCT_NAME := full  
PRODUCTS.build/target/product/full.mk.PRODUCT_DEVICE := generic  
PRODUCTS.build/target/product/full.mk.PRODUCT_BRAND := Android  
PRODUCTS.build/target/product/full.mk.PRODUCT_MODEL := Full Android on Emulator
```
正是由于调用了函数move-var-list，我们在build/core/product_config.mk文件中可以通过PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_DEVICE来设置变量TARGET_DEVICE的值。

回到build/core/config.mk文件中，接下来我们再看BoardConfig.mk文件的加载过程。前面提到，当前要加载的BoardConfig.mk文件由变量TARGET_DEVICE来确定。例如，假设我们在运行lunch命令时，输入的文本为full-eng，那么build/target/product/full.mk就会被加载，并且我们得到TARGET_DEVICE的值就为generic，接下来加载的BoradConfig.mk文件就会在build/target/board/generic目录中找到。

BoardConfig.mk文件定义的信息可以参考build/target/board/generic/BoardConfig.mk文件的内容，如下所示：

```
# config.mk  
#  
# Product-specific compile-time definitions.  
#  

# The generic product target doesn't have any hardware-specific pieces.  
TARGET_NO_BOOTLOADER := true  
TARGET_NO_KERNEL := true  
TARGET_ARCH := arm  

# Note: we build the platform images for ARMv7-A _without_ NEON.  
#  
# Technically, the emulator supports ARMv7-A _and_ NEON instructions, but  
# emulated NEON code paths typically ends up 2x slower than the normal C code  
# it is supposed to replace (unlike on real devices where it is 2x to 3x  
# faster).  
#  
# What this means is that the platform image will not use NEON code paths  
# that are slower to emulate. On the other hand, it is possible to emulate  
# application code generated with the NDK that uses NEON in the emulator.  
#  
TARGET_ARCH_VARIANT := armv7-a  
TARGET_CPU_ABI := armeabi-v7a  
TARGET_CPU_ABI2 := armeabi  
ARCH_ARM_HAVE_TLS_REGISTER := true  

HAVE_HTC_AUDIO_DRIVER := true  
BOARD_USES_GENERIC_AUDIO := true  

# no hardware camera  
USE_CAMERA_STUB := true  

# Enable dex-preoptimization to speed up the first boot sequence  
# of an SDK AVD. Note that this operation only works on Linux for now  
ifeq ($(HOST_OS),linux)  
ifeq ($(WITH_DEXPREOPT),)  
    WITH_DEXPREOPT := true  
endif  
endif  

# Build OpenGLES emulation guest and host libraries  
BUILD_EMULATOR_OPENGL := true  

# Build and enable the OpenGL ES View renderer. When running on the emulator,  
# the GLES renderer disables itself if host GL acceleration isn't available.  
USE_OPENGL_RENDERER := true  
```
它描述了产品的Boot Loader、Kernel、CPU体系结构、CPU ABI和Opengl加速等信息。

再回到build/core/config.mk文件中，它最后加载build/core/dumpvar.mk文件。加载build/core/dumpvar.mk文件是为了生成make目标，以便可以对这些目标进行操作。例如，在我们这个情景中，我们要执行的make目标是dumpvar-TARGET_DEVICE，因此在加载build/core/dumpvar.mk文件的过程中，就会生成dumpvar-TARGET_DEVICE目标。

文件build/core/dumpvar.mk的内容也比较多，这里我们只关注生成make目标相关的逻辑：

```
......  

# The "dumpvar" stuff lets you say something like  
#  
#     CALLED_FROM_SETUP=true \  
#       make -f config/envsetup.make dumpvar-TARGET_OUT  
# or  
#     CALLED_FROM_SETUP=true \  
#       make -f config/envsetup.make dumpvar-abs-HOST_OUT_EXECUTABLES  
#  
# The plain (non-abs) version just dumps the value of the named variable.  
# The "abs" version will treat the variable as a path, and dumps an  
# absolute path to it.  
#  
dumpvar_goals := \  
    $(strip $(patsubst dumpvar-%,%,$(filter dumpvar-%,$(MAKECMDGOALS))))  
ifdef dumpvar_goals  

ifneq ($(words $(dumpvar_goals)),1)  
    $(error Only one "dumpvar-" goal allowed. Saw "$(MAKECMDGOALS)")  
endif  

# If the goal is of the form "dumpvar-abs-VARNAME", then  
# treat VARNAME as a path and return the absolute path to it.  
absolute_dumpvar := $(strip $(filter abs-%,$(dumpvar_goals)))  
ifdef absolute_dumpvar  
    dumpvar_goals := $(patsubst abs-%,%,$(dumpvar_goals))  
    ifneq ($(filter /%,$($(dumpvar_goals))),)  
DUMPVAR_VALUE := $($(dumpvar_goals))  
    else  
DUMPVAR_VALUE := $(PWD)/$($(dumpvar_goals))  
    endif  
    dumpvar_target := dumpvar-abs-$(dumpvar_goals)  
else  
    DUMPVAR_VALUE := $($(dumpvar_goals))  
    dumpvar_target := dumpvar-$(dumpvar_goals)  
endif  

.PHONY: $(dumpvar_target)  
$(dumpvar_target):  
    @echo $(DUMPVAR_VALUE)  

endif # dumpvar_goals  

......  
```
我们在执行make命令时，指定的目示会经由MAKECMDGOALS变量传递到Makefile中，因此通过变量MAKECMDGOALS可以获得make目标。

上述代码的逻辑很简单，例如，在我们这个情景中，指定的make目标为dumpvar-TARGET_DEVICE，那么就会得到变量DUMPVAR_VALUE的值为$(TARGET_DEVICE)。TARGET_DEVICE的值在前面已经被设置为“generic”，因此变量DUMPVAR_VALUE的值就等于“generic”。此外，变量dumpvar_target的被设置为“dumpvar-TARGET_DEVICE”。最后我们就可以得到以下的make规则：

```
.PHONY dumpvar-TARGET_DEVICE  
dumpvar-TARGET_DEVICE:  
    @echo generic  
```
至此，在build/envsetup.sh文件中定义的函数check_product就分析完成了。看完了之后，小伙伴们可能会问，前面不是说这个函数是用来检查用户输入的产品名称是否合法的吗？但是这里没看出哪一段代码给出了true或者false的答案啊。实际上，在前面分析的build/core/config.mk和build/core/product_config.mk等文件的加载过程中，如果发现输入的产品名称是非法的，也就是找不到相应的产品Makefile文件，那么就会通过调用error函数来产生一个错误，这时候函数check_product的返回值$?就会等于非0值。

接下来我们还要继续分析在build/envsetup.sh文件中定义的函数check_variant的实现，如下所示：

```
VARIANT_CHOICES=(user userdebug eng)  

# check to see if the supplied variant is valid  
function check_variant()  
{  
    for v in ${VARIANT_CHOICES[@]}  
    do  
 if [ "$v" = "$1" ]  
 then  
     return 0  
 fi  
    done  
    return 1  
}  
```
这个函数的实现就简单多了。合法的编译类型定义在数组VARIANT_CHOICES中，并且它只有三个值user、userdebug和eng。其中，user表示发布版本，userdebug表示带调试信息的发布版本，而eng表标工程机版本。

最后，我们再来分析在build/envsetup.sh文件中定义的函数printconfig的实现，如下所示：

```
function printconfig()  
{  
    T=$(gettop)  
    if [ ! "$T" ]; then  
 echo "Couldn't locate the top of the tree.  Try setting TOP." >&2  
 return  
    fi  
    get_build_var report_config  
}  
```
对比我们前面对函数check_product的分析，就会发现函数printconfig的实现与这很相似，都是通过调用get_build_var来获得相关的信息，但是这里传递给函数get_build_var的参数为report_config。

我们跳过前面build/core/config.mk和build/core/envsetup.mk等文件对目标产品Makefile文件的加载，直接跳到build/core/dumpvar.mk文件来查看与report_config这个make目标相关的逻辑：

```
......  

dumpvar_goals := \  
    $(strip $(patsubst dumpvar-%,%,$(filter dumpvar-%,$(MAKECMDGOALS))))  
.....  

ifneq ($(dumpvar_goals),report_config)  
PRINT_BUILD_CONFIG:=  
endif  

......  

ifneq ($(PRINT_BUILD_CONFIG),)  
HOST_OS_EXTRA:=$(shell python -c "import platform; print(platform.platform())")  
$(info ============================================)  
$(info   PLATFORM_VERSION_CODENAME=$(PLATFORM_VERSION_CODENAME))  
$(info   PLATFORM_VERSION=$(PLATFORM_VERSION))  
$(info   TARGET_PRODUCT=$(TARGET_PRODUCT))  
$(info   TARGET_BUILD_VARIANT=$(TARGET_BUILD_VARIANT))  
$(info   TARGET_BUILD_TYPE=$(TARGET_BUILD_TYPE))  
$(info   TARGET_BUILD_APPS=$(TARGET_BUILD_APPS))  
$(info   TARGET_ARCH=$(TARGET_ARCH))  
$(info   TARGET_ARCH_VARIANT=$(TARGET_ARCH_VARIANT))  
$(info   HOST_ARCH=$(HOST_ARCH))  
$(info   HOST_OS=$(HOST_OS))  
$(info   HOST_OS_EXTRA=$(HOST_OS_EXTRA))  
$(info   HOST_BUILD_TYPE=$(HOST_BUILD_TYPE))  
$(info   BUILD_ID=$(BUILD_ID))  
$(info   OUT_DIR=$(OUT_DIR))  
$(info ============================================)  
endif  
```
变量PRINT_BUILD_CONFIG定义在文件build/core/envsetup.mk中，默认值设置为true。当make目标为report-config的时候，变量PRINT_BUILD_CONFIG的值就会被设置为空。因此，接下来就会打印一系列用来描述编译环境配置的变量的值，也就是我们执行lunch命令后看到的输出。注意，这些环境配置相关的变量量都是在加载build/core/config.mk和build/core/envsetup.mk文件的过程中设置的，就类似于前面我们分析的TARGET_DEVICE变量的值的设置过程。

至此，我们就分析完成Android编译系统环境的初始化过程了。从分析的过程可以知道，Android编译系统环境是由build/core/config.mk、build/core/envsetup.mk、build/core/product_config.mk、AndroidProducts.mk和BoardConfig.mk等文件来完成的。这些mk文件涉及到非常多的细节，而我们这里只提供了一个大体的骨架和脉络，希望能够起到抛砖引玉的作用。

有了Android编译系统环境的初始化过程知识之后，在接下来的一篇文章中，老罗将继续分析Android编译系统提供的m/mm/mmm编译命令，敬请关注！更多信息也可以关注老罗的新浪微博：[http://weibo.com/shengyangluo](http://weibo.com/shengyangluo)