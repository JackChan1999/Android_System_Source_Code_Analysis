 [Android](http://lib.csdn.net/base/android)源代码在编译之前，要先对编译环境进行初始化，其中最主要就是指定编译的类型和目标设备的型号。[android](http://lib.csdn.net/base/android)的编译类型主要有eng、userdebug和user三种，而支持的目标设备型号则是不确定的，它们由当前的源码配置情况所决定。为了确定源码支持的所有目标设备型号，Android编译系统在初始化的过程中，需要在特定的目录中加载特定的配置文件。接下来本文就对上述的初始化过程进行详细分析。

老罗的新浪微博：[http://weibo.com/shengyangluo](http://weibo.com/shengyangluo)，欢迎关注！

《Android系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

​       对Android编译环境进行初始化很简单，分为两步。第一步是打开一个终端，并且将build/envsetup.sh加载到该终端中：

**[html]** [view plain](http://blog.csdn.net/luoshengyang/article/details/18928789#) [copy](http://blog.csdn.net/luoshengyang/article/details/18928789#)

1. $ . ./build/envsetup.sh   
2. including device/asus/grouper/vendorsetup.sh  
3. including device/asus/tilapia/vendorsetup.sh  
4. including device/generic/armv7-a-neon/vendorsetup.sh  
5. including device/generic/armv7-a/vendorsetup.sh  
6. including device/generic/mips/vendorsetup.sh  
7. including device/generic/x86/vendorsetup.sh  
8. including device/lge/mako/vendorsetup.sh  
9. including device/samsung/maguro/vendorsetup.sh  
10. including device/samsung/manta/vendorsetup.sh  
11. including device/samsung/toroplus/vendorsetup.sh  
12. including device/samsung/toro/vendorsetup.sh  
13. including device/ti/panda/vendorsetup.sh  
14. including sdk/bash_completion/adb.bash  

​      从命令的输出可以知道，文件build/envsetup.sh在加载的过程中，又会在device目录中寻找那些名称为vendorsetup.sh的文件，并且也将它们加载到当前终端来。另外，在sdk/bash_completion目录下的adb.bash文件也会加载到当前终端来，它是用来实现adb命令的bash completion功能的。也就是说，加载了该文件之后，我们在运行adb相关的命令的时候，通过按tab键就可以帮助我们自动完成命令的输入。关于bash completion的知识，可以参考官方文档： 

http://www.gnu.org/s/bash/manual/bash.html#Programmable-Completion

。

​      第二步是执行命令lunch，如下所示：

**[html]** [view plain](http://blog.csdn.net/luoshengyang/article/details/18928789#) [copy](http://blog.csdn.net/luoshengyang/article/details/18928789#)

1. $ lunch  
2.   
3. You're building on Linux  
4.   
5. Lunch menu... pick a combo:  
6. ​     1. full-eng  
7. ​     2. full_x86-eng  
8. ​     3. vbox_x86-eng  
9. ​     4. full_mips-eng  
10. ​     5. full_grouper-userdebug  
11. ​     6. full_tilapia-userdebug  
12. ​     7. mini_armv7a_neon-userdebug  
13. ​     8. mini_armv7a-userdebug  
14. ​     9. mini_mips-userdebug  
15. ​     10. mini_x86-userdebug  
16. ​     11. full_mako-userdebug  
17. ​     12. full_maguro-userdebug  
18. ​     13. full_manta-userdebug  
19. ​     14. full_toroplus-userdebug  
20. ​     15. full_toro-userdebug  
21. ​     16. full_panda-userdebug  
22.   
23. Which would you like? [full-eng]   

​       我们看到lunch命令输出了一个Lunch菜单，该菜单列出了当前Android源码支持的所有设备型号及其编译类型。例如，第一项“full-eng”表示的设备“full”即为模拟器，并且编译类型为“eng”即为工程机。

​       当我们选定了一个Lunch菜单项序号(1-16)之后，按回车键，就可以完成Android编译环境的初始化过程。例如，我们选择1，可以看到以下输出：

**[html]** [view plain](http://blog.csdn.net/luoshengyang/article/details/18928789#) [copy](http://blog.csdn.net/luoshengyang/article/details/18928789#)

1. Which would you like? [full-eng] 1  
2.   
3. ============================================  
4. PLATFORM_VERSION_CODENAME=REL  
5. PLATFORM_VERSION=4.2  
6. TARGET_PRODUCT=full  
7. TARGET_BUILD_VARIANT=eng  
8. TARGET_BUILD_TYPE=release  
9. TARGET_BUILD_APPS=  
10. TARGET_ARCH=arm  
11. TARGET_ARCH_VARIANT=armv7-a  
12. HOST_ARCH=x86  
13. HOST_OS=linux  
14. HOST_OS_EXTRA=Linux-3.8.0-31-generic-x86_64-with-Ubuntu-13.04-raring  
15. HOST_BUILD_TYPE=release  
16. BUILD_ID=JOP40C  
17. OUT_DIR=out  
18. ============================================  

​       我们可以看到，lunch命令帮我们设置好了很多环境变量。通过设置这些环境变量，就配置好了Android编译环境。

​       通过图1我们就可以直观地看到Android编译环境初始化完成后，我们所获得的东西：

![img](http://img.blog.csdn.net/20140208122926593?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

图1 Android编译环境初始化完成之后

​       总体来说，Android编译环境初始化完成之后，获得了以下三样东西：

​       1. 将vendor和device目录下的vendorsetup.sh文件加载到了当前终端；

​       2. 新增了lunch、m、mm和mmm等命令；

​       3. 通过执行lunch命令设置好了TARGET_PRODUCT、TARGET_BUILD_VARIANT、TARGET_BUILD_TYPE和TARGET_BUILD_APPS等环境变量。  

​       接下来我们就主要分析build/envsetup.sh文件的加载过程以及lunch命令的执行过程。

​       一. 文件build/envsetup.sh的加载过程

​       文件build/envsetup.sh是一个bash shell脚本，从它里面定义的函数hmm可以知道，它提供了lunch、m、mm和mmm等命令供我们初始化编译环境或者编译Android源码。

​       函数hmm的实现如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/18928789#) [copy](http://blog.csdn.net/luoshengyang/article/details/18928789#)

1. function hmm() {  
2. cat <<EOF  
3. Invoke ". build/envsetup.sh" from your shell to add the following functions to your environment:  
4. - lunch:   lunch <product_name>-<build_variant>  
5. - tapas:   tapas [<App1> <App2> ...] [arm|x86|mips] [eng|userdebug|user]  
6. - croot:   Changes directory to the top of the tree.  
7. - m:       Makes from the top of the tree.  
8. - mm:      Builds all of the modules in the current directory.  
9. - mmm:     Builds all of the modules in the supplied directories.  
10. - cgrep:   Greps on all local C/C++ files.  
11. - jgrep:   Greps on all local Java files.  
12. - resgrep: Greps on all local res/*.xml files.  
13. - godir:   Go to the directory containing a file.  
14.   
15. Look at the source to view more functions. The complete list is:  
16. EOF  
17. ​    T=$(gettop)  
18. ​    local A  
19. ​    A=""  
20. ​    for i in `cat $T/build/envsetup.sh | sed -n "/^function /s/function [a−z]∗.*/\1/p" | sort`; do  
21. ​      A="$A $i"  
22. ​    done  
23. ​    echo $A  
24. }  

​       我们在当前终端中执行hmm命令即可以看到函数hmm的完整输出。

​       函数hmm主要完成三个工作：

​       1. 调用另外一个函数gettop获得Android源码的根目录T。 

​       2. 通过cat命令显示一个[Here Document](http://tldp.org/LDP/abs/html/here-docs.html)，说明$T/build/envsetup.sh文件加载到当前终端后所提供的主要命令。

​       3. 通过sed命令解析$T/build/envsetup.sh文件，并且获得在里面定义的所有函数的名称，这些函数名称就是$T/build/envsetup.sh文件加载到当前终端后提供的所有命令。

​       注意，sed命令是一个强大的文本分析工具，它以行为单位为执行文本替换、删除、新增和选取等操作。函数hmm通过执行以下的sed命令来获得在$T/build/envsetup.sh文件定义的函数的名称：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/18928789#) [copy](http://blog.csdn.net/luoshengyang/article/details/18928789#)

1. sed -n "/^function /s/function [a−z]∗.*/\1/p"  

​       它表示对所有以“function ”开头的行，如果紧接在“function ”后面的字符串仅由字母a-z和下横线(_)组成，那么就将这个字符串提取出来。这正好就对应于shell脚本里面函数的定义。

​       文件build/envsetup.sh除了定义一堆函数之外，还有一个重要的代码段，如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/18928789#) [copy](http://blog.csdn.net/luoshengyang/article/details/18928789#)

1. \# Execute the contents of any vendorsetup.sh files we can find.  
2. for f in `/bin/ls vendor/*/vendorsetup.sh vendor/*/*/vendorsetup.sh device/*/*/vendorsetup.sh 2> /dev/null`  
3. do  
4. ​    echo "including $f"  
5. ​    . $f  
6. done  
7. unset f  

​        这个for循环遍历vendor目录下的一级子目录和二级子目录以及device目录下的二级子目录中的vendorsetup.sh文件，并且通过source命令(.)将它们加载当前终端来。vendor和device相应子目录下的vendorsetup.sh文件的实现很简单，它们主要就是添加相应的设备型号及其编译类型支持到Lunch菜单中去。

​        例如，device/samsung/maguro目录下的vendorsetup.sh文件的实现如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/18928789#) [copy](http://blog.csdn.net/luoshengyang/article/details/18928789#)

1. add_lunch_combo full_maguro-userdebug  

​        它调用函数add_lunch_combo添加一个名称为“full_maguro-userdebug”的菜单项到Lunch菜单去。

​        函数add_lunch_combo定义在build/envsetup.sh文件中，它的实现如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/18928789#) [copy](http://blog.csdn.net/luoshengyang/article/details/18928789#)

1. function add_lunch_combo()  
2. {  
3. ​    local new_combo=$1  
4. ​    local c  
5. ​    for c in ${LUNCH_MENU_CHOICES[@]} ; do  
6. ​        if [ "$new_combo" = "$c" ] ; then  
7. ​            return  
8. ​        fi  
9. ​    done  
10. ​    LUNCH_MENU_CHOICES=(${LUNCH_MENU_CHOICES[@]} $new_combo)  
11. }  

​        传递给函数add_lunch_combo的参数保存在位置参数$1中，接着又保存在一个本地变量new_combo中，用来表示一个要即将要添加的Lunch菜单项。函数首先是在数组LUNCH_MENU_CHOICES中检查要添加的菜单项是否已经存在。只有在不存在的情况下，才会将它添加到数组LUNCH_MENU_CHOICES中去。注意，${LUNCH_MENU_CHOICES[@]}表示数组LUNCH_MENU_CHOICES的所有元素。

​        数组LUNCH_MENU_CHOICES是定义在文件build/envsetup.sh的一个全局变量，当文件build/envsetup.sh被加载的时候，这个数组会被初始化为化full-eng、full_x86-eng、vbox_x86-eng和full_mips-eng，如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/18928789#) [copy](http://blog.csdn.net/luoshengyang/article/details/18928789#)

1. \# add the default one here  
2. add_lunch_combo full-eng  
3. add_lunch_combo full_x86-eng  
4. add_lunch_combo vbox_x86-eng  
5. add_lunch_combo full_mips-eng  

​       这样当文件build/envsetup.sh加载完成之后，数组LUNCH_MENU_CHOICES就包含了当前源码支持的所有设备型号及其编译类型，于是当接下来我们执行lunch命令的时候，就可以通过数组LUNCH_MENU_CHOICES看到一个完整的Lunch藤蔓。

​       二. lunch命令的执行过程

​       lunch命令实际上是定义在文件build/envsetup.sh的一个函数，它的实现如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/18928789#) [copy](http://blog.csdn.net/luoshengyang/article/details/18928789#)

1. function lunch()  
2. {  
3. ​    local answer  
4.   
5. ​    if [ "$1" ] ; then  
6. ​        answer=$1  
7. ​    else  
8. ​        print_lunch_menu  
9. ​        echo -n "Which would you like? [full-eng] "  
10. ​        read answer  
11. ​    fi  
12.   
13. ​    local selection=  
14.   
15. ​    if [ -z "$answer" ]  
16. ​    then  
17. ​        selection=full-eng  
18. ​    elif (echo -n $answer | grep -q -e "^[0-9][0-9]*$")  
19. ​    then  
20. ​        if [ $answer -le ${#LUNCH_MENU_CHOICES[@]} ]  
21. ​        then  
22. ​            selection=${LUNCH_MENU_CHOICES[$(($answer-1))]}  
23. ​        fi  
24. ​    elif (echo -n $answer | grep -q -e "^[^\-][^\-]*-[^\-][^\-]*$")  
25. ​    then  
26. ​        selection=$answer  
27. ​    fi  
28.   
29. ​    if [ -z "$selection" ]  
30. ​    then  
31. ​        echo  
32. ​        echo "Invalid lunch combo: $answer"  
33. ​        return 1  
34. ​    fi  
35.   
36. ​    export TARGET_BUILD_APPS=  
37.   
38. ​    local product=$(echo -n $selection | sed -e "s/-.*$//")  
39. ​    check_product $product  
40. ​    if [ $? -ne 0 ]  
41. ​    then  
42. ​        echo  
43. ​        echo "** Don't have a product spec for: '$product'"  
44. ​        echo "** Do you have the right repo manifest?"  
45. ​        product=  
46. ​    fi  
47.   
48. ​    local variant=$(echo -n $selection | sed -e "s/^[^\-]*-//")  
49. ​    check_variant $variant  
50. ​    if [ $? -ne 0 ]  
51. ​    then  
52. ​        echo  
53. ​        echo "** Invalid variant: '$variant'"  
54. ​        echo "** Must be one of ${VARIANT_CHOICES[@]}"  
55. ​        variant=  
56. ​    fi  
57.   
58. ​    if [ -z "$product" -o -z "$variant" ]  
59. ​    then  
60. ​        echo  
61. ​        return 1  
62. ​    fi  
63.   
64. ​    export TARGET_PRODUCT=$product  
65. ​    export TARGET_BUILD_VARIANT=$variant  
66. ​    export TARGET_BUILD_TYPE=release  
67.   
68. ​    echo  
69.   
70. ​    set_stuff_for_environment  
71. ​    printconfig  
72. }  

​        函数lunch的执行逻辑如下所示：

​        1. 检查是否带有参数，即位置参数$1是否等于空。如果不等于空的话，就表明带有参数，并且该参数是用来指定要编译的设备型号及其编译类型的。如果等于空的话，那么就调用另外一个函数print_lunch_menu来显示Lunch菜单项，并且通过调用read函数来等待用户输入。无论通过何种方式，最终变量answer的值就保存了用户所指定的备型号及其编译类型。

​        2. 对变量answer的值的合法性进行检查。如果等于空的话，就将它设置为默认值“full-eng”。如果不等于空的话，就分为三种情况考虑。第一种情况是值为数字，那么就需要确保该数字的大小不能超过Lunch菜单项的个数。在这种情况下，会将输入的数字索引到数组LUNCH_MENU_CHOICES中去，以便获得一个用来表示设备型号及其编译类型的文本。第二种情况是非数字文本，那么就需要确保该文本符合<product>-<variant>的形式，其中<product>表示设备型号，而<variant>表示编译类型 。第三种情况是除了前面两种情况之外的所有情况，这是非法的。经过合法性检查后，变量selection代表了用户所指定的备型号及其编译类型，如果它的值是非法的，即它的值等于空，那么函数lunch就不往下执行了。

​        3. 接下来是解析变量selection的值，也就是通过sed命令将它的<product>和<variant>值提取出来，并且分别保存在变量product和variant中。提取出来的product和variant值有可能是不合法的，因此需要进一步通过调用函数check_product和check_variant来检查。一旦检查失败，也就是函数check_product和check_variant的返回值$?等于非0，那么函数lunch就不往下执行了。

​        4. 通过以上合法性检查之后，就将变量product和variant的值保存在环境变量TARGET_PRODUCT和TARGET_BUILD_VARIANT中。此外，另外一个环境变量TARGET_BUILD_TYPE的值会被设置为"release"，表示此次编译是一个release版本的编译。另外，前面还有一个环境变量TARGET_BUILD_APPS，它的值被函数lunch设置为空，用来表示此次编译是对整个系统进行编译。如果环境变量TARGET_BUILD_APPS的值不等于空，那么就表示此次编译是只对某些APP模块进行编译，而这些APP模块就是由环境变量TARGET_BUILD_APPS来指定的。

​        5. 调用函数set_stuff_for_environment来配置环境，例如设置[Java ](http://lib.csdn.net/base/java)SDK路径和交叉编译工具路径等。

​        6. 调用函数printfconfig来显示已经配置好的编译环境参数。

​        在上述执行过程中，函数check_product、check_variant和printconfig是比较关键的，因此接下来我们就继续分析它们的实现。

​        函数check_product定义在文件build/envsetup.sh中，它的实现如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/18928789#) [copy](http://blog.csdn.net/luoshengyang/article/details/18928789#)

1. \# check to see if the supplied product is one we can build  
2. function check_product()  
3. {  
4. ​    T=$(gettop)  
5. ​    if [ ! "$T" ]; then  
6. ​        echo "Couldn't locate the top of the tree.  Try setting TOP." >&2  
7. ​        return  
8. ​    fi  
9. ​    CALLED_FROM_SETUP=true BUILD_SYSTEM=build/core \  
10. ​        TARGET_PRODUCT=$1 \  
11. ​        TARGET_BUILD_VARIANT= \  
12. ​        TARGET_BUILD_TYPE= \  
13. ​        TARGET_BUILD_APPS= \  
14. ​        get_build_var TARGET_DEVICE > /dev/null  
15. ​    # hide successful answers, but allow the errors to show  
16. }  

​        函数gettop用来返回Android源代码工程的根目录。函数check_product需要在Android源代码工程根目录或者子目录下调用。否则的话，函数check_product就出错返回。

​        接下来函数check_product设置几个环境变量，其中最重要的是前面三个CALLED_FROM_SETUP、BUILD_SYSTEM和TARGET_PRODUCT。环境变量CALLED_FROM_SETUP的值等于true表示接下来执行的make命令是用来初始化Android编译环境的。环境变量BUILD_SYSTEM用来指定Android编译系统的核心目录，它的值被设置为build/core。环境变量TARGET_PRODUCT用来表示要检查的产品名称（也就是我们前面说的设备型号），它的值被设置为$1，即函数check_product的调用参数。

​        最后函数check_product调用函数get_build_var来检查由环境变量TARGET_PRODUCT指定的产品名称是否合法，注意，它的调用参数为TARGET_DEVICE。

​        函数get_build_var定义在文件build/envsetup.sh中，它的实现如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/18928789#) [copy](http://blog.csdn.net/luoshengyang/article/details/18928789#)

1. \# Get the exact value of a build variable.  
2. function get_build_var()  
3. {  
4. ​    T=$(gettop)  
5. ​    if [ ! "$T" ]; then  
6. ​        echo "Couldn't locate the top of the tree.  Try setting TOP." >&2  
7. ​        return  
8. ​    fi  
9. ​    CALLED_FROM_SETUP=true BUILD_SYSTEM=build/core \  
10. ​      make --no-print-directory -C "$T" -f build/core/config.mk dumpvar-$1  
11. }  

​        这里就可以看到，函数get_build_var实际上就是通过make命令在Android源代码工程根目录中执行build/core/config.mk文件，并且将make目标设置为dumpvar-$1，也就是dumpvar-TARGET_DEVICE。

​        文件build/core/config.mk的内容比较多，这里我们只关注与产品名称合法性检查相关的逻辑，这些逻辑也基本上涵盖了Android编译系统初始化的逻辑，如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/18928789#) [copy](http://blog.csdn.net/luoshengyang/article/details/18928789#)

1. ......  
2.   
3. \# ---------------------------------------------------------------  
4. \# Define most of the global variables.  These are the ones that  
5. \# are specific to the user's build configuration.  
6. include $(BUILD_SYSTEM)/envsetup.mk  
7.   
8. \# Boards may be defined under $(SRC_TARGET_DIR)/board/$(TARGET_DEVICE)  
9. \# or under vendor/*/$(TARGET_DEVICE).  Search in both places, but  
10. \# make sure only one exists.  
11. \# Real boards should always be associated with an OEM vendor.  
12. board_config_mk := \  
13. ​    $(strip $(wildcard \  
14. ​        $(SRC_TARGET_DIR)/board/$(TARGET_DEVICE)/BoardConfig.mk \  
15. ​        device/*/$(TARGET_DEVICE)/BoardConfig.mk \  
16. ​        vendor/*/$(TARGET_DEVICE)/BoardConfig.mk \  
17. ​    ))  
18. ifeq ($(board_config_mk),)  
19.   $(error No config file found for TARGET_DEVICE $(TARGET_DEVICE))  
20. endif  
21. ifneq ($(words $(board_config_mk)),1)  
22.   $(error Multiple board config files for TARGET_DEVICE $(TARGET_DEVICE): $(board_config_mk))  
23. endif  
24. include $(board_config_mk)  
25.   
26. ......  
27.   
28. include $(BUILD_SYSTEM)/dumpvar.mk  

​       上述代码主要就是将envsetup.mk、BoardConfig,mk和dumpvar.mk三个Makefile片段文件加载进来。其中，envsetup.mk文件位于$(BUILD_SYSTEM)目录中，也就是build/core目录中，BoardConfig.mk文件的位置主要就是由环境变量TARGET_DEVICE来确定，它是用来描述目标产品的硬件模块信息的，例如CPU体系结构。环境变量TARGET_DEVICE用来描述目标设备，它的值是在envsetup.mk文件加载的过程中确定的。一旦目标设备确定后，就可以在$(SRC_TARGET_DIR)/board/$(TARGET_DEVICE)、device/*/$(TARGET_DEVICE)和vendor/*/$(TARGET_DEVICE)目录中找到对应的BoradConfig.mk文件。注意，变量SRC_TARGET_DIR的值等于build/target。最后，dumpvar.mk文件也是位于build/core目录中，它用来打印已经配置好的编译环境信息。

​        接下来我们就通过进入到build/core/envsetup.mk文件来分析变量TARGET_DEVICE的值是如何确定的：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/18928789#) [copy](http://blog.csdn.net/luoshengyang/article/details/18928789#)

1. \# Read the product specs so we an get TARGET_DEVICE and other  
2. \# variables that we need in order to locate the output files.  
3. include $(BUILD_SYSTEM)/product_config.mk  

​       它通过加载另外一个文件build/core/product_config.mk文件来确定变量TARGET_DEVICE以及其它与目标产品相关的变量的值。

​       文件build/core/product_config.mk的内容很多，这里我们只关注变量TARGET_DEVICE设置相关的逻辑，如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/18928789#) [copy](http://blog.csdn.net/luoshengyang/article/details/18928789#)

1. ......  
2.   
3. ifneq ($(strip $(TARGET_BUILD_APPS)),)  
4. \# An unbundled app build needs only the core product makefiles.  
5. all_product_configs := $(call get-product-makefiles,\  
6. ​    $(SRC_TARGET_DIR)/product/AndroidProducts.mk)  
7. else  
8. \# Read in all of the product definitions specified by the AndroidProducts.mk  
9. \# files in the tree.  
10. all_product_configs := $(get-all-product-makefiles)  
11. endif  
12.   
13. \# all_product_configs consists items like:  
14. \# <product_name>:<path_to_the_product_makefile>  
15. \# or just <path_to_the_product_makefile> in case the product name is the  
16. \# same as the base filename of the product config makefile.  
17. current_product_makefile :=  
18. all_product_makefiles :=  
19. $(foreach f, $(all_product_configs),\  
20. ​    $(eval _cpm_words := $(subst :,$(space),$(f)))\  
21. ​    $(eval _cpm_word1 := $(word 1,$(_cpm_words)))\  
22. ​    $(eval _cpm_word2 := $(word 2,$(_cpm_words)))\  
23. ​    $(if $(_cpm_word2),\  
24. ​        $(eval all_product_makefiles += $(_cpm_word2))\  
25. ​        $(if $(filter $(TARGET_PRODUCT),$(_cpm_word1)),\  
26. ​            $(eval current_product_makefile += $(_cpm_word2)),),\  
27. ​        $(eval all_product_makefiles += $(f))\  
28. ​        $(if $(filter $(TARGET_PRODUCT),$(basename $(notdir $(f)))),\  
29. ​            $(eval current_product_makefile += $(f)),)))  
30. _cpm_words :=  
31. _cpm_word1 :=  
32. _cpm_word2 :=  
33. current_product_makefile := $(strip $(current_product_makefile))  
34. all_product_makefiles := $(strip $(all_product_makefiles))  
35.   
36. ifneq (,$(filter product-graph dump-products, $(MAKECMDGOALS)))  
37. \# Import all product makefiles.  
38. $(call import-products, $(all_product_makefiles))  
39. else  
40. \# Import just the current product.  
41. ifndef current_product_makefile  
42. $(error Cannot locate config makefile for product "$(TARGET_PRODUCT)")  
43. endif  
44. ifneq (1,$(words $(current_product_makefile)))  
45. $(error Product "$(TARGET_PRODUCT)" ambiguous: matches $(current_product_makefile))  
46. endif  
47. $(call import-products, $(current_product_makefile))  
48. endif  # Import all or just the current product makefile  
49.   
50. ......  
51.   
52. \# Convert a short name like "sooner" into the path to the product  
53. \# file defining that product.  
54. \#  
55. INTERNAL_PRODUCT := $(call resolve-short-product-name, $(TARGET_PRODUCT))  
56. ifneq ($(current_product_makefile),$(INTERNAL_PRODUCT))  
57. $(error PRODUCT_NAME inconsistent in $(current_product_makefile) and $(INTERNAL_PRODUCT))  
58. endif  
59. current_product_makefile :=  
60. all_product_makefiles :=  
61. all_product_configs :=  
62.   
63. \# Find the device that this product maps to.  
64. TARGET_DEVICE := $(PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_DEVICE)  
65.   
66. ......  

​       上述代码的执行逻辑如下所示：

​       1. 检查环境变量TARGET_BUILD_APPS的值是否等于空。如果不等于空，那么就说明此次编译不是针对整个系统，因此只要将核心的产品相关的Makefile文件加载进来就行了，否则的话，就要将所有与产品相关的Makefile文件加载进来的。核心产品Makefile文件在$(SRC_TARGET_DIR)/product/AndroidProducts.mk文件中指定，也就是在build/target/product/AndroidProducts.mk文件，通过调用函数get-product-makefiles可以获得。所有与产品相关的Makefile文件可以通过另外一个函数get-all-product-makefiles获得。无论如何，最终获得的产品Makefie文件列表保存在变量all_product_configs中。

​       2. 遍历变量all_product_configs所描述的产品Makefile列表，并且在这些Makefile文件中，找到名称与环境变量TARGET_PRODUCT的值相同的文件，保存在另外一个变量current_product_makefile中，作为需要为当前指定的产品所加载的Makefile文件列表。在这个过程当中，上一步找到的所有的产品Makefile文件也会保存在变量all_product_makefiles中。注意，环境变量TARGET_PRODUCT的值是在我们执行lunch命令的时候设置并且传递进来的。

​       3.  如果指定的make目标等于product-graph或者dump-products，那么就将所有的产品相关的Makefile文件加载进来，否则的话，只加载与目标产品相关的Makefile文件。从前面的分析可以知道，此时的make目标为dumpvar-TARGET_DEVICE，因此接下来只会加载与目标产品，即$(TARGET_PRODUCT)，相关的Makefile文件，这是通过调用另外一个函数import-products实现的。

​       4. 调用函数resolve-short-product-name解析环境变量TARGET_PRODUCT的值，将它变成一个Makefile文件路径。并且保存在变量INTERNAL_PRODUCT中。这里要求变量INTERNAL_PRODUCT和current_product_makefile的值相等，否则的话，就说明用户指定了一个非法的产品名称。

​       5. 找到一个名称为PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_DEVICE的变量，并且将它的值保存另外一个变量TARGET_DEVICE中。变量PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_DEVICE是在加载产品Makefile文件的过程中定义的，用来描述当前指定的产品的名称。

​       上述过程主要涉及到了get-all-product-makefiles、import-products和resolve-short-product-name三个关键函数，理解它们的执行过程对理解Android编译系统的初始化过程很有帮助，接下来我们分别分析它们的实现。

​        函数get-all-product-makefiles定义在文件build/core/product.mk中，如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/18928789#) [copy](http://blog.csdn.net/luoshengyang/article/details/18928789#)

1. \#  
2. \# Returns the sorted concatenation of all PRODUCT_MAKEFILES  
3. \# variables set in all AndroidProducts.mk files.  
4. \# $(call ) isn't necessary.  
5. \#  
6. define get-all-product-makefiles  
7. $(call get-product-makefiles,$(_find-android-products-files))  
8. endef  

​       它首先是调用函数_find-android-products-files来找到Android源代码目录中定义的所有AndroidProducts.mk文件，然后再调用函数get-product-makefiles获得在这里AndroidProducts.mk文件里面定义的产品Makefile文件。

​       函数_find-android-products-files也是定义在文件build/core/product.mk中，如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/18928789#) [copy](http://blog.csdn.net/luoshengyang/article/details/18928789#)

1. \#  
2. \# Returns the list of all AndroidProducts.mk files.  
3. \# $(call ) isn't necessary.  
4. \#  
5. define _find-android-products-files  
6. $(shell test -d device && find device -maxdepth 6 -name AndroidProducts.mk) \  
7.   $(shell test -d vendor && find vendor -maxdepth 6 -name AndroidProducts.mk) \  
8.   $(SRC_TARGET_DIR)/product/AndroidProducts.mk  
9. endef  

​      从这里就可以看出，Android源代码目录中定义的所有AndroidProducts.mk文件位于device、vendor或者build/target/product目录或者相应的子目录（最深是6层）中。

​      函数get-product-makefiles也是定义在文件build/core/product.mk中，如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/18928789#) [copy](http://blog.csdn.net/luoshengyang/article/details/18928789#)

1. \#  
2. \# Returns the sorted concatenation of PRODUCT_MAKEFILES  
3. \# variables set in the given AndroidProducts.mk files.  
4. \# $(1): the list of AndroidProducts.mk files.  
5. \#  
6. define get-product-makefiles  
7. $(sort \  
8.   $(foreach f,$(1), \  
9. ​    $(eval PRODUCT_MAKEFILES :=) \  
10. ​    $(eval LOCAL_DIR := $(patsubst %/,%,$(dir $(f)))) \  
11. ​    $(eval include $(f)) \  
12. ​    $(PRODUCT_MAKEFILES) \  
13.    ) \  
14.   $(eval PRODUCT_MAKEFILES :=) \  
15.   $(eval LOCAL_DIR :=) \  
16.  )  
17. endef  

​       这个函数实际上就是遍历参数$1所描述的AndroidProucts.mk文件列表，并且将定义在这些AndroidProucts.mk文件中的变量PRODUCT_MAKEFILES的值提取出来，形成一个列表返回给调用者。

​       例如，在build/target/product/AndroidProducts.mk文件中，变量PRODUCT_MAKEFILES的值如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/18928789#) [copy](http://blog.csdn.net/luoshengyang/article/details/18928789#)

1. \# Unbundled apps will be built with the most generic product config.  
2. ifneq ($(TARGET_BUILD_APPS),)  
3. PRODUCT_MAKEFILES := \  
4. ​    $(LOCAL_DIR)/full.mk \  
5. ​    $(LOCAL_DIR)/full_x86.mk \  
6. ​    $(LOCAL_DIR)/full_mips.mk  
7. else  
8. PRODUCT_MAKEFILES := \  
9. ​    $(LOCAL_DIR)/core.mk \  
10. ​    $(LOCAL_DIR)/generic.mk \  
11. ​    $(LOCAL_DIR)/generic_x86.mk \  
12. ​    $(LOCAL_DIR)/generic_mips.mk \  
13. ​    $(LOCAL_DIR)/full.mk \  
14. ​    $(LOCAL_DIR)/full_x86.mk \  
15. ​    $(LOCAL_DIR)/full_mips.mk \  
16. ​    $(LOCAL_DIR)/vbox_x86.mk \  
17. ​    $(LOCAL_DIR)/sdk.mk \  
18. ​    $(LOCAL_DIR)/sdk_x86.mk \  
19. ​    $(LOCAL_DIR)/sdk_mips.mk \  
20. ​    $(LOCAL_DIR)/large_emu_hw.mk  
21. endif  

​       这里列出的每一个文件都对应于一个产品。

​       我们再来看函数import-products的实现，它定义在文件build/core/product.mk中，如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/18928789#) [copy](http://blog.csdn.net/luoshengyang/article/details/18928789#)

1. \#  
2. \# $(1): product makefile list  
3. \#  
4. \#TODO: check to make sure that products have all the necessary vars defined  
5. define import-products  
6. $(call import-nodes,PRODUCTS,$(1),$(_product_var_list))  
7. endef  

​       它调用另外一个函数import-nodes来加载由参数$1所指定的产品Makefile文件，并且指定了另外两个参数PRODUCTS和$(_product_var_list)。其中，变量_product_var_list也是定义在文件build/core/product.mk中，它的值如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/18928789#) [copy](http://blog.csdn.net/luoshengyang/article/details/18928789#)

1. _product_var_list := \  
2. ​    PRODUCT_NAME \  
3. ​    PRODUCT_MODEL \  
4. ​    PRODUCT_LOCALES \  
5. ​    PRODUCT_AAPT_CONFIG \  
6. ​    PRODUCT_AAPT_PREF_CONFIG \  
7. ​    PRODUCT_PACKAGES \  
8. ​    PRODUCT_PACKAGES_DEBUG \  
9. ​    PRODUCT_PACKAGES_ENG \  
10. ​    PRODUCT_PACKAGES_TESTS \  
11. ​    PRODUCT_DEVICE \  
12. ​    PRODUCT_MANUFACTURER \  
13. ​    PRODUCT_BRAND \  
14. ​    PRODUCT_PROPERTY_OVERRIDES \  
15. ​    PRODUCT_DEFAULT_PROPERTY_OVERRIDES \  
16. ​    PRODUCT_CHARACTERISTICS \  
17. ​    PRODUCT_COPY_FILES \  
18. ​    PRODUCT_OTA_PUBLIC_KEYS \  
19. ​    PRODUCT_EXTRA_RECOVERY_KEYS \  
20. ​    PRODUCT_PACKAGE_OVERLAYS \  
21. ​    DEVICE_PACKAGE_OVERLAYS \  
22. ​    PRODUCT_TAGS \  
23. ​    PRODUCT_SDK_ADDON_NAME \  
24. ​    PRODUCT_SDK_ADDON_COPY_FILES \  
25. ​    PRODUCT_SDK_ADDON_COPY_MODULES \  
26. ​    PRODUCT_SDK_ADDON_DOC_MODULES \  
27. ​    PRODUCT_DEFAULT_WIFI_CHANNELS \  
28. ​    PRODUCT_DEFAULT_DEV_CERTIFICATE \  
29. ​    PRODUCT_RESTRICT_VENDOR_FILES \  
30. ​    PRODUCT_VENDOR_KERNEL_HEADERS \  
31. ​    PRODUCT_FACTORY_RAMDISK_MODULES \  
32. ​    PRODUCT_FACTORY_BUNDLE_MODULES  

​       它描述的是在产品Makefile文件中定义在各种变量。

​       函数import-nodes定义在文件build/core/node_fns.mk中，如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/18928789#) [copy](http://blog.csdn.net/luoshengyang/article/details/18928789#)

1. \#  
2. \# $(1): output list variable name, like "PRODUCTS" or "DEVICES"  
3. \# $(2): list of makefiles representing nodes to import  
4. \# $(3): list of node variable names  
5. \#  
6. define import-nodes  
7. $(if \  
8.   $(foreach _in,$(2), \  
9. ​    $(eval _node_import_context := _nic.$(1).[[$(_in)]]) \  
10. ​    $(if $(_include_stack),$(eval $(error ASSERTION FAILED: _include_stack \  
11. ​                should be empty here: $(_include_stack))),) \  
12. ​    $(eval _include_stack := ) \  
13. ​    $(call _import-nodes-inner,$(_node_import_context),$(_in),$(3)) \  
14. ​    $(call move-var-list,$(_node_import_context).$(_in),$(1).$(_in),$(3)) \  
15. ​    $(eval _node_import_context :=) \  
16. ​    $(eval $(1) := $($(1)) $(_in)) \  
17. ​    $(if $(_include_stack),$(eval $(error ASSERTION FAILED: _include_stack \  
18. ​                should be empty here: $(_include_stack))),) \  
19.    ) \  
20. ,)  
21. endef  

​       这个函数主要是做了三件事情：

​       1. 调用函数_import-nodes-inner将参数$2描述的每一个产品Makefile文件加载进来。

​       2. 调用函数move-var-list将定义在前面所加载的产品Makefile文件里面的由参数$3指定的变量的值分别拷贝到另外一组独立的变量中。

​       3. 将参数$2描述的每一个产品Makefile文件路径以空格分隔保存在参数$1所描述的变量中，也就是保存在变量PRODUCTS中。

​       上述第二件事情需要进一步解释一下。由于当前加载的每一个文件都会定义相同的变量，为了区分这些变量，我们需要在这些变量前面加一些前缀。例如，假设加载了build/target/product/full.mk这个产品Makefile文件，它里面定义了以下几个变量：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/18928789#) [copy](http://blog.csdn.net/luoshengyang/article/details/18928789#)

1. \# Overrides  
2. PRODUCT_NAME := full  
3. PRODUCT_DEVICE := generic  
4. PRODUCT_BRAND := Android  
5. PRODUCT_MODEL := Full Android on Emulator  

​       当调用了函数move-var-list对它进行解析后，就会得到以下的新变量：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/18928789#) [copy](http://blog.csdn.net/luoshengyang/article/details/18928789#)

1. PRODUCTS.build/target/product/full.mk.PRODUCT_NAME := full  
2. PRODUCTS.build/target/product/full.mk.PRODUCT_DEVICE := generic  
3. PRODUCTS.build/target/product/full.mk.PRODUCT_BRAND := Android  
4. PRODUCTS.build/target/product/full.mk.PRODUCT_MODEL := Full Android on Emulator  

​       正是由于调用了函数move-var-list，我们在build/core/product_config.mk文件中可以通过PRODUCTS.$(INTERNAL_PRODUCT).PRODUCT_DEVICE来设置变量TARGET_DEVICE的值。

​       回到build/core/config.mk文件中，接下来我们再看BoardConfig.mk文件的加载过程。前面提到，当前要加载的BoardConfig.mk文件由变量TARGET_DEVICE来确定。例如，假设我们在运行lunch命令时，输入的文本为full-eng，那么build/target/product/full.mk就会被加载，并且我们得到TARGET_DEVICE的值就为generic，接下来加载的BoradConfig.mk文件就会在build/target/board/generic目录中找到。

​       BoardConfig.mk文件定义的信息可以参考build/target/board/generic/BoardConfig.mk文件的内容，如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/18928789#) [copy](http://blog.csdn.net/luoshengyang/article/details/18928789#)

1. \# config.mk  
2. \#  
3. \# Product-specific compile-time definitions.  
4. \#  
5.   
6. \# The generic product target doesn't have any hardware-specific pieces.  
7. TARGET_NO_BOOTLOADER := true  
8. TARGET_NO_KERNEL := true  
9. TARGET_ARCH := arm  
10.   
11. \# Note: we build the platform images for ARMv7-A _without_ NEON.  
12. \#  
13. \# Technically, the emulator supports ARMv7-A _and_ NEON instructions, but  
14. \# emulated NEON code paths typically ends up 2x slower than the normal C code  
15. \# it is supposed to replace (unlike on real devices where it is 2x to 3x  
16. \# faster).  
17. \#  
18. \# What this means is that the platform image will not use NEON code paths  
19. \# that are slower to emulate. On the other hand, it is possible to emulate  
20. \# application code generated with the NDK that uses NEON in the emulator.  
21. \#  
22. TARGET_ARCH_VARIANT := armv7-a  
23. TARGET_CPU_ABI := armeabi-v7a  
24. TARGET_CPU_ABI2 := armeabi  
25. ARCH_ARM_HAVE_TLS_REGISTER := true  
26.   
27. HAVE_HTC_AUDIO_DRIVER := true  
28. BOARD_USES_GENERIC_AUDIO := true  
29.   
30. \# no hardware camera  
31. USE_CAMERA_STUB := true  
32.   
33. \# Enable dex-preoptimization to speed up the first boot sequence  
34. \# of an SDK AVD. Note that this operation only works on Linux for now  
35. ifeq ($(HOST_OS),linux)  
36.   ifeq ($(WITH_DEXPREOPT),)  
37. ​    WITH_DEXPREOPT := true  
38.   endif  
39. endif  
40.   
41. \# Build OpenGLES emulation guest and host libraries  
42. BUILD_EMULATOR_OPENGL := true  
43.   
44. \# Build and enable the OpenGL ES View renderer. When running on the emulator,  
45. \# the GLES renderer disables itself if host GL acceleration isn't available.  
46. USE_OPENGL_RENDERER := true  

​       它描述了产品的Boot Loader、Kernel、CPU体系结构、CPU ABI和Opengl加速等信息。

​       再回到build/core/config.mk文件中，它最后加载build/core/dumpvar.mk文件。加载build/core/dumpvar.mk文件是为了生成make目标，以便可以对这些目标进行操作。例如，在我们这个情景中，我们要执行的make目标是dumpvar-TARGET_DEVICE，因此在加载build/core/dumpvar.mk文件的过程中，就会生成dumpvar-TARGET_DEVICE目标。

​       文件build/core/dumpvar.mk的内容也比较多，这里我们只关注生成make目标相关的逻辑：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/18928789#) [copy](http://blog.csdn.net/luoshengyang/article/details/18928789#)

1. ......  
2.   
3. \# The "dumpvar" stuff lets you say something like  
4. \#  
5. \#     CALLED_FROM_SETUP=true \  
6. \#       make -f config/envsetup.make dumpvar-TARGET_OUT  
7. \# or  
8. \#     CALLED_FROM_SETUP=true \  
9. \#       make -f config/envsetup.make dumpvar-abs-HOST_OUT_EXECUTABLES  
10. \#  
11. \# The plain (non-abs) version just dumps the value of the named variable.  
12. \# The "abs" version will treat the variable as a path, and dumps an  
13. \# absolute path to it.  
14. \#  
15. dumpvar_goals := \  
16. ​    $(strip $(patsubst dumpvar-%,%,$(filter dumpvar-%,$(MAKECMDGOALS))))  
17. ifdef dumpvar_goals  
18.   
19.   ifneq ($(words $(dumpvar_goals)),1)  
20. ​    $(error Only one "dumpvar-" goal allowed. Saw "$(MAKECMDGOALS)")  
21.   endif  
22.   
23.   # If the goal is of the form "dumpvar-abs-VARNAME", then  
24.   # treat VARNAME as a path and return the absolute path to it.  
25.   absolute_dumpvar := $(strip $(filter abs-%,$(dumpvar_goals)))  
26.   ifdef absolute_dumpvar  
27. ​    dumpvar_goals := $(patsubst abs-%,%,$(dumpvar_goals))  
28. ​    ifneq ($(filter /%,$($(dumpvar_goals))),)  
29. ​      DUMPVAR_VALUE := $($(dumpvar_goals))  
30. ​    else  
31. ​      DUMPVAR_VALUE := $(PWD)/$($(dumpvar_goals))  
32. ​    endif  
33. ​    dumpvar_target := dumpvar-abs-$(dumpvar_goals)  
34.   else  
35. ​    DUMPVAR_VALUE := $($(dumpvar_goals))  
36. ​    dumpvar_target := dumpvar-$(dumpvar_goals)  
37.   endif  
38.   
39. .PHONY: $(dumpvar_target)  
40. $(dumpvar_target):  
41. ​    @echo $(DUMPVAR_VALUE)  
42.   
43. endif # dumpvar_goals  
44.   
45. ......  

​      我们在执行make命令时，指定的目示会经由MAKECMDGOALS变量传递到Makefile中，因此通过变量MAKECMDGOALS可以获得make目标。

​      上述代码的逻辑很简单，例如，在我们这个情景中，指定的make目标为dumpvar-TARGET_DEVICE，那么就会得到变量DUMPVAR_VALUE的值为$(TARGET_DEVICE)。TARGET_DEVICE的值在前面已经被设置为“generic”，因此变量DUMPVAR_VALUE的值就等于“generic”。此外，变量dumpvar_target的被设置为“dumpvar-TARGET_DEVICE”。最后我们就可以得到以下的make规则：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/18928789#) [copy](http://blog.csdn.net/luoshengyang/article/details/18928789#)

1. .PHONY dumpvar-TARGET_DEVICE  
2. dumpvar-TARGET_DEVICE:  
3. ​    @echo generic  

​       至此，在build/envsetup.sh文件中定义的函数check_product就分析完成了。看完了之后，小伙伴们可能会问，前面不是说这个函数是用来检查用户输入的产品名称是否合法的吗？但是这里没看出哪一段代码给出了true或者false的答案啊。实际上，在前面分析的build/core/config.mk和build/core/product_config.mk等文件的加载过程中，如果发现输入的产品名称是非法的，也就是找不到相应的产品Makefile文件，那么就会通过调用error函数来产生一个错误，这时候函数check_product的返回值$?就会等于非0值。

​       接下来我们还要继续分析在build/envsetup.sh文件中定义的函数check_variant的实现，如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/18928789#) [copy](http://blog.csdn.net/luoshengyang/article/details/18928789#)

1. VARIANT_CHOICES=(user userdebug eng)  
2.   
3. \# check to see if the supplied variant is valid  
4. function check_variant()  
5. {  
6. ​    for v in ${VARIANT_CHOICES[@]}  
7. ​    do  
8. ​        if [ "$v" = "$1" ]  
9. ​        then  
10. ​            return 0  
11. ​        fi  
12. ​    done  
13. ​    return 1  
14. }  

​       这个函数的实现就简单多了。合法的编译类型定义在数组VARIANT_CHOICES中，并且它只有三个值user、userdebug和eng。其中，user表示发布版本，userdebug表示带调试信息的发布版本，而eng表标工程机版本。

​       最后，我们再来分析在build/envsetup.sh文件中定义的函数printconfig的实现，如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/18928789#) [copy](http://blog.csdn.net/luoshengyang/article/details/18928789#)

1. function printconfig()  
2. {  
3. ​    T=$(gettop)  
4. ​    if [ ! "$T" ]; then  
5. ​        echo "Couldn't locate the top of the tree.  Try setting TOP." >&2  
6. ​        return  
7. ​    fi  
8. ​    get_build_var report_config  
9. }  

​       对比我们前面对函数check_product的分析，就会发现函数printconfig的实现与这很相似，都是通过调用get_build_var来获得相关的信息，但是这里传递给函数get_build_var的参数为report_config。

​       我们跳过前面build/core/config.mk和build/core/envsetup.mk等文件对目标产品Makefile文件的加载，直接跳到build/core/dumpvar.mk文件来查看与report_config这个make目标相关的逻辑：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/18928789#) [copy](http://blog.csdn.net/luoshengyang/article/details/18928789#)

1. ......  
2.   
3. dumpvar_goals := \  
4. ​    $(strip $(patsubst dumpvar-%,%,$(filter dumpvar-%,$(MAKECMDGOALS))))  
5. .....  
6.   
7. ifneq ($(dumpvar_goals),report_config)  
8. PRINT_BUILD_CONFIG:=  
9. endif  
10.   
11. ......  
12.   
13. ifneq ($(PRINT_BUILD_CONFIG),)  
14. HOST_OS_EXTRA:=$(shell python -c "import platform; print(platform.platform())")  
15. $(info ============================================)  
16. $(info   PLATFORM_VERSION_CODENAME=$(PLATFORM_VERSION_CODENAME))  
17. $(info   PLATFORM_VERSION=$(PLATFORM_VERSION))  
18. $(info   TARGET_PRODUCT=$(TARGET_PRODUCT))  
19. $(info   TARGET_BUILD_VARIANT=$(TARGET_BUILD_VARIANT))  
20. $(info   TARGET_BUILD_TYPE=$(TARGET_BUILD_TYPE))  
21. $(info   TARGET_BUILD_APPS=$(TARGET_BUILD_APPS))  
22. $(info   TARGET_ARCH=$(TARGET_ARCH))  
23. $(info   TARGET_ARCH_VARIANT=$(TARGET_ARCH_VARIANT))  
24. $(info   HOST_ARCH=$(HOST_ARCH))  
25. $(info   HOST_OS=$(HOST_OS))  
26. $(info   HOST_OS_EXTRA=$(HOST_OS_EXTRA))  
27. $(info   HOST_BUILD_TYPE=$(HOST_BUILD_TYPE))  
28. $(info   BUILD_ID=$(BUILD_ID))  
29. $(info   OUT_DIR=$(OUT_DIR))  
30. $(info ============================================)  
31. endif  

​       变量PRINT_BUILD_CONFIG定义在文件build/core/envsetup.mk中，默认值设置为true。当make目标为report-config的时候，变量PRINT_BUILD_CONFIG的值就会被设置为空。因此，接下来就会打印一系列用来描述编译环境配置的变量的值，也就是我们执行lunch命令后看到的输出。注意，这些环境配置相关的变量量都是在加载build/core/config.mk和build/core/envsetup.mk文件的过程中设置的，就类似于前面我们分析的TARGET_DEVICE变量的值的设置过程。

​       至此，我们就分析完成Android编译系统环境的初始化过程了。从分析的过程可以知道，Android编译系统环境是由build/core/config.mk、build/core/envsetup.mk、build/core/product_config.mk、AndroidProducts.mk和BoardConfig.mk等文件来完成的。这些mk文件涉及到非常多的细节，而我们这里只提供了一个大体的骨架和脉络，希望能够起到抛砖引玉的作用。

​       有了Android编译系统环境的初始化过程知识之后，在接下来的一篇文章中，老罗将继续分析Android编译系统提供的m/mm/mmm编译命令，敬请关注！更多信息也可以关注老罗的新浪微博：[http://weibo.com/shengyangluo](http://weibo.com/shengyangluo)