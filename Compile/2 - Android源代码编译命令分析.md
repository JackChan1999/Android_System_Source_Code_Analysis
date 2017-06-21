 在前文中，我们分析了[Android](http://lib.csdn.net/base/android)编译环境的初始化过程。[android](http://lib.csdn.net/base/android)编译环境初始化完成后，我们就可以用m/mm/mmm/make命令编译源代码了。当然，这要求每一个模块都有一个Android.mk文件。Android.mk实际上是一个Makefile脚本，用来描述模块编译信息。Android编译系统通过整合Android.mk文件完成编译过程。本文就对Android源代码的编译过程进行详细分析。

老罗的新浪微博：[http://weibo.com/shengyangluo](http://weibo.com/shengyangluo)，欢迎关注！

《Android系统源代码情景分析》一书正在进击的程序员网（[http://0xcc0xcd.com](http://0xcc0xcd.com/)）中连载，点击进入！

​        从前面[Android编译系统环境初始化过程分析](http://blog.csdn.net/luoshengyang/article/details/18928789)这篇文章可以知道，lunch命令其实是定义在build/envsetup.sh文件中的函数lunch提供的。与lunch命令一样，m、mm和mmm命令也分别是由定义在build/envsetup.sh文件中的函数m、mm和mmm提供的，而这三个函数又都是通过make命令来对源代码进行编译的。事实上，命令m就是对make命令的简单封装，并且是用来对整个Android源代码进行编译，而命令mm和mmm都是通过make命令来对Android源码中的指定模块进行编译。接下来我们就先分别介绍一下函数m、mm和mmm的实现，然后进一步分析它们是如何通过make命令来编译代码的。

​        函数m的实现如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/19023609#) [copy](http://blog.csdn.net/luoshengyang/article/details/19023609#)

1. function m()  
2. {  
3. ​    T=$(gettop)  
4. ​    if [ "$T" ]; then  
5. ​        make -C $T $@  
6. ​    else  
7. ​        echo "Couldn't locate the top of the tree.  Try setting TOP."  
8. ​    fi  
9. }  

​       函数m调用函数gettop得到的是Android源代码根目录T。在执行make命令的时候，先通过-C选项指定工作目录为T，即Android源代码根目录，接着又将执行命令m指定的参数$@作为命令make的参数。从这里就可以看出，命令m实际上就是对命令make的简单封装。

​       函数mm的实现如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/19023609#) [copy](http://blog.csdn.net/luoshengyang/article/details/19023609#)

1. function mm()  
2. {  
3. ​    # If we're sitting in the root of the build tree, just do a  
4. ​    # normal make.  
5. ​    if [ -f build/core/envsetup.mk -a -f Makefile ]; then  
6. ​        make $@  
7. ​    else  
8. ​        # Find the closest Android.mk file.  
9. ​        T=$(gettop)  
10. ​        local M=$(findmakefile)  
11. ​        # Remove the path to top as the makefilepath needs to be relative  
12. ​        local M=`echo $M|sed 's:'$T'/::'`  
13. ​        if [ ! "$T" ]; then  
14. ​            echo "Couldn't locate the top of the tree.  Try setting TOP."  
15. ​        elif [ ! "$M" ]; then  
16. ​            echo "Couldn't locate a makefile from the current directory."  
17. ​        else  
18. ​            ONE_SHOT_MAKEFILE=$M make -C $T all_modules $@  
19. ​        fi  
20. ​    fi  
21. }  

​        函数mm首先是判断当前目录是否就是Android源码根目录，即当前目录下是否存在一个build/core/envsetup.mk文件和一个Makefile文件。如果是的话，就将命令mm当作是一个普通的make命令来执行。否则的话，就调用函数findmakefile从当前目录开始一直往上寻找是否存在一个Android.mk文件。如果在寻找的过程中，发现了一个Android.mk文件，那么就获得它的绝对路径，并且停止上述寻找过程。

​        由于接下来执行make命令时，我们需要指定的是要编译的Android.mk文件的相对于Android源码根目录路径，因此函数mm需要将刚才找到的Android.mk绝对文件路径M中与Android源码根目录T相同的那部分路径去掉。这是通过sed命令来实现的，也就是将字符串M前面与字符串T相同的子串删掉。

​        最后，将找到的Android.mk文件的相对路径设置给环境变量ONE_SHOT_MAKE，表示接下来要对它进行编译。另外，函数mm还将make命令目标设置为all_modules。这是什么意思呢？我们知道，一个Android.mk文件同时可以定义多个模块，因此，all_modules就表示要对前面指定的Android.mk文件中定义的所有模块进行编译。

​         函数mmm的实现如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/19023609#) [copy](http://blog.csdn.net/luoshengyang/article/details/19023609#)

1. function mmm()  
2. {  
3. ​    T=$(gettop)  
4. ​    if [ "$T" ]; then  
5. ​        local MAKEFILE=  
6. ​        local MODULES=  
7. ​        local ARGS=  
8. ​        local DIR TO_CHOP  
9. ​        local DASH_ARGS=$(echo "$@" | awk -v RS=" " -v ORS=" " '/^-.*$/')  
10. ​        local DIRS=$(echo "$@" | awk -v RS=" " -v ORS=" " '/^[^-].*$/')  
11. ​        for DIR in $DIRS ; do  
12. ​            MODULES=`echo $DIR | sed -n -e 's/.*:.∗$/\1/p' | sed 's/,/ /'`  
13. ​            if [ "$MODULES" = "" ]; then  
14. ​                MODULES=all_modules  
15. ​            fi  
16. ​            DIR=`echo $DIR | sed -e 's/:.*//' -e 's:/$::'`  
17. ​            if [ -f $DIR/Android.mk ]; then  
18. ​                TO_CHOP=`(cd -P -- $T && pwd -P) | wc -c | tr -d ' '`  
19. ​                TO_CHOP=`expr $TO_CHOP + 1`  
20. ​                START=`PWD= /bin/pwd`  
21. ​                MFILE=`echo $START | cut -c${TO_CHOP}-`  
22. ​                if [ "$MFILE" = "" ] ; then  
23. ​                    MFILE=$DIR/Android.mk  
24. ​                else  
25. ​                    MFILE=$MFILE/$DIR/Android.mk  
26. ​                fi  
27. ​                MAKEFILE="$MAKEFILE $MFILE"  
28. ​            else  
29. ​                if [ "$DIR" = snod ]; then  
30. ​                    ARGS="$ARGS snod"  
31. ​                elif [ "$DIR" = showcommands ]; then  
32. ​                    ARGS="$ARGS showcommands"  
33. ​                elif [ "$DIR" = dist ]; then  
34. ​                    ARGS="$ARGS dist"  
35. ​                elif [ "$DIR" = incrementaljavac ]; then  
36. ​                    ARGS="$ARGS incrementaljavac"  
37. ​                else  
38. ​                    echo "No Android.mk in $DIR."  
39. ​                    return 1  
40. ​                fi  
41. ​            fi  
42. ​        done  
43. ​        ONE_SHOT_MAKEFILE="$MAKEFILE" make -C $T $DASH_ARGS $MODULES $ARGS  
44. ​    else  
45. ​        echo "Couldn't locate the top of the tree.  Try setting TOP."  
46. ​    fi  
47. }  

​        函数mmm的实现就稍微复杂一点，我们详细解释一下。

​        首先，命令mmm可以这样执行：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/19023609#) [copy](http://blog.csdn.net/luoshengyang/article/details/19023609#)

1. $ mmm <dir-1> <dir-2> ... <dir-N>[:module-1,module-2,...,module-M]    

​        其中，dir-1、dir-2、dir-N都是包含有Android.mk文件的目录。在最后一个目录dir-N的后面可以带一个冒号，冒号后面可以通过逗号分隔一系列的模块名称module-1、module-2和module-M，用来表示要编译前面指定的Android.mk中的哪些模块。

​        知道了命令mmm的使用方法之后 ，我们就可以分析函数mmm的执行逻辑了：

​        1. 调用函数gettop获得Android源码根目录。

​        2. 通过命令awk将执行命令mmm时指定的选项参数提取出来，也就是将以横线“-”开头的字符串提取出来，并且保存在变量DASH_ARGS中。

​        3. 通过命令awk将执行命令mmm时指定的非选项参数提取出来，也就是将非以横线“-”开头的字符串提取出来，并且保存在变量DIRS中。这里得到的实际上就是跟在命令mmm后面的字符串“<dir-1> <dir-2> ... <dir-N>[:module-1,module-2,...,module-M]”。

​        4. 变量DIRS保存的字符串可以看成是一系以空格分隔的子字符串，因此，就可以通过一个for循环来对这些子字府串进行遍历。每一个子字符串DIR描述的都是一个包含有Android.mk文件的目录。对每一个目录DIR执行以下操作：

​        4.1 由于目录DIR后面可能会通过冒号指定有模块名称，因此就先通过两个sed命令来获得这些模块名称。第一个sed命令获得的是一系列以逗号分隔的模块名称列表，第二个sed命令用来将前面获得的以逗号分隔的模块名称列表转化为以空格分隔的模块名称列表。最后，获得的以空格分隔的模块名称列表保存在变量MODULES中。由于目录DIR后面也可能不指定有模块名称，因此前面得到的变量MODULES的值就会为空。在这种情况下，需要将变量MODULES的值设置为“all_modules”，表示要编译的是所有模块。

​        4.2 通过两个sed命令获得真正的目录DIR。第一个sed命令将原来DIR字符串后面的冒号以及冒号后面的模块列表字符串删掉。第二个sed命令将执行前面一个sed命令获得的目录后面的"/"斜线去掉，最后就得到一个末尾不带有斜线“/”的路径，并且保存在变量DIR中。

​        4.3 如果变量DIR描述的是一个真正的路径，也就是在该路径下存在一个Android.mk文件，那么就进行以下处理：

​        4.3.1 统计Android源码根目录T包含的字符数，并且将这个字符数加1，得到的值保存在变量TO_CHOP中。

​        4.3.2 通过执行/bin/pwd命令获得当前执行命令mmm的目录START。

​        4.3.3 通过cut命令获得当前目录START相对于Android源码根目录T的路径，并且保存在变量MFILE中。

​        4.3.4 如果变量MFILE的值等于空，就表明是在Android源码根目录T中执行mmm命令，这时候就表明变量DIR描述的就是相对Android源码根目录T的一个目录，这时候指定的Android.mk文件相对于Android源码根目录T的路径就为$DIR/Android.mk。

​        4.3.5 如果变量MFILE的值不等于空，就表明是在Android源码根目录T的某一个子目录中执行mmm命令，这时候$MFILE/$DIR/Android.mk表示的Android.mk文件路径才是相对于Android源码根目录T的。

​        4.3.6 将获得的Android.mk路径MFILE附加在变量MAKEFILE描述的字符串的后面，并且以空格分隔。

​        4.4 如果变量DIR描述的不是一个真正的路径，并且它的值等于"snod"、"showcomands"、“dist”或者“incrementaljavac”，那么它描述的其实是make修饰命令。这四个修饰命令的含义分别如下所示：

​        4.4.1 snod是“systemimage with no dependencies”的意思，表示忽略依赖性地重新打包system.img。

​        4.4.2 showcommands表示显示编译过程中执行的命令。

​        4.4.3 dist表示将编译后产生的发布文件拷贝到out/dist目录中。

​        4.4.4 incrementaljavac表示对[Java](http://lib.csdn.net/base/java)源文件采用增量式编译，也就是如果一个Java文件如果没有修改过，那么就不要重新生成对应的class文件。

​        5. 上面的for循环执行完毕，变量MAKEFILE保存的是要编译的Android.mk文件列表，它们都是相对于Android源码根目录的路径，变量DASH_ARGS保存的是原来执行mmm命令时带的选项参数，变量MODULES保存的是指定要编译的模块名称，变量ARGS保存的是修饰命令。其中，变量MAKEFILE的内容通过环境变量ONE_SHOT_MAKEFILE传递给make命令，而其余变量都是通过参数的形式传递给make命令，并且变量MODULES作为make命令的目标。

​        明白了函数m、mm和mmm的实现之后，我们就可以知道：

​        1. mm和mmm命令是类似的，它们都是用来编译某些模块。

​        2. m命令用来编译所有模块。

​        如果我们理解了mm或者mmm命令的编译过程，那么自然也会明白m命令的编译过程，因为所有模块的编译过程就等于把每一个模块的编译都编译出来，因此，接下来我们就选择具有代表性的、常用的编译命令mmm来分析Android源码的编译过程，如图1所示：

![img](http://img.blog.csdn.net/20140212012436281?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvTHVvc2hlbmd5YW5n/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

图1 mmm命令的编译过程

​       函数mmm在Android源码根目录执行make命令的时候，没有通过-f指定Makefile文件，因此默认就使用Android源码根目录下的Makefile文件，它的内容如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/19023609#) [copy](http://blog.csdn.net/luoshengyang/article/details/19023609#)

1. \### DO NOT EDIT THIS FILE ###  
2. include build/core/main.mk  
3. \### DO NOT EDIT THIS FILE ###  

​       它仅仅是将build/core/main.mk文件加载进来。build/core/main.mk是Android编译系统的入口文件，它通过加载其它的mk文件来对Android源码中的各个模块进行编译，以及将编译出来的文件打包成各种镜像文件。以下就是build/core/main.mk文件的主要内容：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/19023609#) [copy](http://blog.csdn.net/luoshengyang/article/details/19023609#)

1. ......  
2.   
3. \# This is the default target.  It must be the first declared target.  
4. .PHONY: droid  
5. DEFAULT_GOAL := droid  
6. $(DEFAULT_GOAL):  
7. ......  
8.   
9. \# Set up various standard variables based on configuration  
10. \# and host information.  
11. include $(BUILD_SYSTEM)/config.mk  
12. ......  
13.   
14. \# Bring in standard build system definitions.  
15. include $(BUILD_SYSTEM)/definitions.mk  
16. ......  
17.   
18. \# These targets are going to delete stuff, don't bother including  
19. \# the whole directory tree if that's all we're going to do  
20. ifeq ($(MAKECMDGOALS),clean)  
21. dont_bother := true  
22. endif  
23. ifeq ($(MAKECMDGOALS),clobber)  
24. dont_bother := true  
25. endif  
26. ifeq ($(MAKECMDGOALS),dataclean)  
27. dont_bother := true  
28. endif  
29. ifeq ($(MAKECMDGOALS),installclean)  
30. dont_bother := true  
31. endif  
32.   
33. \# Bring in all modules that need to be built.  
34. ifneq ($(dont_bother),true)  
35.   
36. ......  
37.   
38. ifneq ($(ONE_SHOT_MAKEFILE),)  
39. \# We've probably been invoked by the "mm" shell function  
40. \# with a subdirectory's makefile.  
41. include $(ONE_SHOT_MAKEFILE)  
42. ......  
43. else # ONE_SHOT_MAKEFILE  
44.   
45. \#  
46. \# Include all of the makefiles in the system  
47. \#  
48.   
49. \# Can't use first-makefiles-under here because  
50. \# --mindepth=2 makes the prunes not work.  
51. subdir_makefiles := \  
52. ​    $(shell build/tools/findleaves.py --prune=out --prune=.repo --prune=.git $(subdirs) Android.mk)  
53.   
54. include $(subdir_makefiles)  
55.   
56. endif # ONE_SHOT_MAKEFILE  
57.   
58. ......  
59.   
60. \# -------------------------------------------------------------------  
61. \# Define dependencies for modules that require other modules.  
62. \# This can only happen now, after we've read in all module makefiles.  
63. \#  
64. \# TODO: deal with the fact that a bare module name isn't  
65. \# unambiguous enough.  Maybe declare short targets like  
66. \# APPS:Quake or HOST:SHARED_LIBRARIES:libutils.  
67. \# BUG: the system image won't know to depend on modules that are  
68. \# brought in as requirements of other modules.  
69. define add-required-deps  
70. $(1): $(2)  
71. endef  
72. $(foreach m,$(ALL_MODULES), \  
73.   $(eval r := $(ALL_MODULES.$(m).REQUIRED)) \  
74.   $(if $(r), \  
75. ​    $(eval r := $(call module-installed-files,$(r))) \  
76. ​    $(eval $(call add-required-deps,$(ALL_MODULES.$(m).INSTALLED),$(r))) \  
77.    ) \  
78.  )  
79. ......  
80.   
81. modules_to_install := $(sort \  
82. ​    $(ALL_DEFAULT_INSTALLED_MODULES) \  
83. ​    $(product_FILES) \  
84. ​    $(foreach tag,$(tags_to_install),$($(tag)_MODULES)) \  
85. ​    $(call get-tagged-modules, shell_$(TARGET_SHELL)) \  
86. ​    $(CUSTOM_MODULES) \  
87.   )  
88. ......  
89.   
90. \# build/core/Makefile contains extra stuff that we don't want to pollute this  
91. \# top-level makefile with.  It expects that ALL_DEFAULT_INSTALLED_MODULES  
92. \# contains everything that's built during the current make, but it also further  
93. \# extends ALL_DEFAULT_INSTALLED_MODULES.  
94. ALL_DEFAULT_INSTALLED_MODULES := $(modules_to_install)  
95. include $(BUILD_SYSTEM)/Makefile  
96. modules_to_install := $(sort $(ALL_DEFAULT_INSTALLED_MODULES))  
97. ALL_DEFAULT_INSTALLED_MODULES :=  
98.   
99. endif # dont_bother  
100.   
101. ......  
102.   
103. \# -------------------------------------------------------------------  
104. \# This is used to to get the ordering right, you can also use these,  
105. \# but they're considered undocumented, so don't complain if their  
106. \# behavior changes.  
107. .PHONY: prebuilt  
108. prebuilt: $(ALL_PREBUILT)  
109. ......  
110.   
111. \# All the droid stuff, in directories  
112. .PHONY: files  
113. files: prebuilt \  
114. ​        $(modules_to_install) \  
115. ​        $(modules_to_check) \  
116. ​        $(INSTALLED_ANDROID_INFO_TXT_TARGET)  
117. ......  
118.   
119. \# Build files and then package it into the rom formats  
120. .PHONY: droidcore  
121. droidcore: files \  
122. ​    systemimage \  
123. ​    $(INSTALLED_BOOTIMAGE_TARGET) \  
124. ​    $(INSTALLED_RECOVERYIMAGE_TARGET) \  
125. ​    $(INSTALLED_USERDATAIMAGE_TARGET) \  
126. ​    $(INSTALLED_CACHEIMAGE_TARGET) \  
127. ​    $(INSTALLED_FILES_FILE)  
128.   
129. ......  
130.   
131. \# Dist for droid if droid is among the cmd goals, or no cmd goal is given.  
132. ifneq ($(filter droid,$(MAKECMDGOALS))$(filter ||,|$(filter-out $(INTERNAL_MODIFIER_TARGETS),$(MAKECMDGOALS))|),)  
133.   
134. ifneq ($(TARGET_BUILD_APPS),)  
135.   # If this build is just for apps, only build apps and not the full system by default.  
136. ......  
137.   
138. .PHONY: apps_only  
139. apps_only: $(unbundled_build_modules)  
140.   
141. droid: apps_only  
142.   
143. else # TARGET_BUILD_APPS  
144. ......  
145.   
146. \# Building a full system-- the default is to build droidcore  
147. droid: droidcore dist_files  
148.   
149. endif # TARGET_BUILD_APPS  
150. endif # droid in $(MAKECMDGOALS)  
151. ......  
152.   
153. \# phony target that include any targets in $(ALL_MODULES)  
154. .PHONY: all_modules  
155. all_modules: $(ALL_MODULES)  
156.   
157. ......  

​       接下来我们就先对build/core/main.mk文件的核心逻辑进行分析，然后再进一步对其中涉及到的关键点进行分析。

​        build/core/main.mk文件的执行过程如下所示：

​        1. 定义默认make目标为droid。目标droid根据不同的情形有不同的依赖关系。如果在初始化编译环境时，指定了TARGET_BUILD_APPS环境变量，那么就表示当前只编译特定的模块，这些特定的模块保存在变量unbundled_build_modules中，这时候目标droid就透过另外一个伪目标app_only依赖它们。如果在初始化编译环境时没有指定TARGET_BUILD_APPS环境变量，那么目标droid就依赖于另外两个文件droidcore和dist_files。droidcore是一个make伪目标，它依赖于各种预编译文件，以及system.img、boot.img、recovery.img和userdata.img等镜像文件。dist_files也是一个make伪目标，用来指定一些需要在编译后拷贝到out/dist目录的文件。也就是说，当我们在Android源码目录中执行不带目标的make命令时，默认就会对目标droid进行编译，也就是会将整个Android系统编译出来。

​        2. 加载build/core/config.mk文件。从前面[Android编译系统环境初始化过程分析](http://blog.csdn.net/luoshengyang/article/details/18928789)一文可以知道，在加载build/core/config.mk文件的过程中，会在执行make命令的进程中完成对Android编译环境的初始化过程，也就是会指定好目标设备以及编译类型。

​        3. 加载build/croe/definitions.mk文件。该文件定义了很多在编译过程中要用到的宏，相当于就是定义了很多通用函数，供编译过程调用。

​        4. 如果在执行make命令时，指定的不是清理文件相关的目标，也就是不是clean、clobber、dataclean和installclean等目标，那么就会将变量dont_bother的值设置为true，表示接下来要执行的是编译命令。

​        5. 在变量dont_bother的值等于true的情况下，如果环境变量ONE_SHOT_MAKEFILE的值不等于空，也就是我们执行的是mm或者mmm命令，那么就表示要编译的是特定的模块。这些指定要编译的模块的Android.mk文件路径就保存在环境变量ONE_SHOT_MAKEFILE中，因此直接将这些Android,mk文件加载进来就获得相应的编译规则。另一方面，如果环境变量ONE_SHOT_MAKEFILE的值等于空，那么就说明我们执行的是m或者make命令，那么就表示要对Android源代码中的所有模块进行编译，这时候就通过build/tools/findleaves.py脚本获得Android源代码工程下的所有Android.mk文件的路径列表，并且将这些Android.mk文件加载进行获得相应的编译规则。

​        6. 上一步指定的Android.mk文件加载完成之后，变量ALL_MODULES就包含了所有要编译的模块的名称，这些模块名称以空格来分隔形成成一个列表。

​        7. 生成模块依赖规则。每一个模块都可以通过LOCAL_REQUIRED_MODULES来指定它所依赖的其它模块，也就是说当一个模块被安装时，它所依赖的其它模块也同样会被安装。每一个模块m依赖的所有模块都会被保存在ALL_MODULES.$(m).REQUIRED变量中。对于每一个被依赖模块r，我们需要获得它的安装文件，也就是最终生成的模块文件的文件路径，以便可以生成相应的编译规则。获得一个模块m的安装文件是通过调用函数module-installed-files来实现的，实质上就是保存在$(ALL_MODULES.$(m).INSTALLED变量中。知道了一个模块m的所依赖的模块的安装文件路径之后，我们就可以通过函数add-required-deps来指定它们之间的依赖关系了。注意，这里实际上指定的是模块m的安装文件与它所依赖的模块r的安装文件的依赖关系。

​        8. 将所有要安装的模块都保存在变量ALL_DEFAULT_INSTALLED_MODULES中，并且将build/core/Makefie文件加载进来。 build/core/Makefie文件会根据要安装的模块产成system.img、boot.img和recovery.img等镜像文件的生成规则。

​        9. 前面提到，当执行mm命令时，make目标指定为all_moudles。另外，当执行mmm命令时，默认的make目标也指定为all_moudles。因此，我们需要指定目标all_modules的编译规则，实际上只要将它依赖于当前要编译的所有模块就行了，也就是依赖于由变量ALL_MODULES所描述的模块。

​       在上述过程中，最核心的就是第5步和第8步。由于本文只关心Android源码的编译过程，因此我们只分析第5步的执行过程。在接下来一篇文章中分析Android镜像文件的生成过程时，我们再分析第8步的执行过程。

​       第5步实际上就是将指定模块的Android.mk文件加载进来。一个典型的Android.mk文件如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/19023609#) [copy](http://blog.csdn.net/luoshengyang/article/details/19023609#)

1. LOCAL_PATH:= $(call my-dir)  
2. include $(CLEAR_VARS)  
3.   
4. LOCAL_MODULE_TAGS := optional  
5. LOCAL_MODULE    := libdis  
6.   
7. LOCAL_SHARED_LIBRARIES := \  
8. ​    liblog \  
9. ​    libdl  
10.   
11. LOCAL_SRC_FILES := \  
12. ​    dispatcher.cpp \  
13. ​    ../common/common.cpp  
14.   
15. include $(BUILD_SHARED_LIBRARY)  

​       以LOCAL开头的变量都是属于模块局部变量，也就是说，一个模块在开始编译之前，必须要先对它们进行清理，然后再进行初始化。Android编译系统定义了非常多的模块局部变量，因此我们不可能手动地一个一个清理，需要加载一个由变量CLEAR_VARS指定的Makefile脚本来帮我们自动清理。变量CLEAR_VARS的值定义在build/core/config.mk文件，它的值等于build/core/clear_vars.mk。

​       Android.mk文件中还有一个重要的变量LOCAL_PATH，用来指定当前正在编译的模块的目录，我们可以通过调用宏my-dir来获得。宏my-dir定义在build/core/definitions.mk文件，它实际上就是将当前正在加载的Android.mk文件路径的目录名提取出来。

​       Android.mk文件接下来就是通过其它的LOCAL变量定义模块名称、源文件，以及所要依赖的各种库文件等等。例如，在我们这个例子，模块名称定义为libdis，参与编译的源文件为dispatcher.cpp和common.cpp、依赖的库文件为liblog和libdl。

​       最后，Android文件通过加载一个模板文件来告诉编译系统它所要编译的模块的类型。例如，在我们这个例子中，就是通过加载由变量BUILD_SHARED_LIBRARY指定的模板文件来告诉编译系统我们要编译的模块是一个动态链接库。变量BUILD_SHARED_LIBRARY的值定义在build/core/config.mk文件，它的值等于build/core/shared_library.mk。

​       Android编译系统定义了非常多的模板文件，每一个模板文件都对应一种类型的模块，例如除了我们上面的动态链接库模板文件之外，还有：

​       **BUILD_PACKAGE**：指向build/core/package.mk，用来编译APK文件。

​       **BUILD_JAVA_LIBRARY**：指向build/core/java_library.mk，用来编译Java库文件。

​       **BUILD_STATIC_JAVA_LIBRARY**：指向build/core/tatic_java_library.mk，用来编译Java静态库文件。

​       **BUILD_STATIC_LIBRARY**：指向build/core/static_library.mk，用来编译静态库文件。也就是.a文件。

​       **BUILD_EXECUTABLE**：指向build/core/executable.mk，用来编译可执行文件。

​       **BUILD_PREBUILT**：指向build/core/prebuilt.mk。用来编译已经预编译好的第三方库文件，实际上是将这些预编译好的第三方库文件拷贝到合适的位置去，以便可以让其它模块引用。

​       不管编译何种类型的模块，都是主要完成以下的工作：

​       1. 制定好相应的依赖规则

​       2. 调用合适的命令进行编译

​       为了简单起见，接下来我们就以动态链接库（即.so文件）的编译过程为例来说明Android编译命令mmm的执行过程。

​       在分析动态链接库的编译过程之前，我们首先看一看使用mmm命令来编译上述的Android.mk文件时得到的输出，如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/19023609#) [copy](http://blog.csdn.net/luoshengyang/article/details/19023609#)

1. target thumb C++: libdis <= external/si/dispatcher/dispatcher.cpp  
2. target thumb C++: libdis <= external/si/dispatcher/../common/common.cpp  
3. target SharedLib: libdis (out/target/product/generic/obj/SHARED_LIBRARIES/libdis_intermediates/LINKED/libdis.so)  
4. target Symbolic: libdis (out/target/product/generic/symbols/system/lib/libdis.so)  
5. target Strip: libdis (out/target/product/generic/obj/lib/libdis.so)  
6. Install: out/target/product/generic/system/lib/libdis.so  

​       从这些输出我们大体推断出一些文件之间的依赖关系及其生成过程：

​       1. out/target/product/generic/obj/SHARED_LIBRARIES/libdis_intermediates/LINKED/libdis.so文件依赖于external/si/dispatcher/dispatcher.cpp和external/si/dispatcher/../common/common.cpp文件，并且由它们生成。

​       2. out/target/product/generic/symbols/system/lib/libdis.so依赖于out/target/product/generic/obj/SHARED_LIBRARIES/libdis_intermediates/LINKED/libdis.so文件，并且由它生成。

​       3. out/target/product/generic/obj/lib/libdis.so依赖于out/target/product/generic/symbols/system/lib/libdis.so文件，并且由它生成。

​       4. out/target/product/generic/system/lib/libdis.so依赖于out/target/product/generic/obj/lib/libdis.so文件，并且由它生成。

​       回忆前面的分析，我们提到，当执行mmm命令时，默认的make目标是all_modules，并且它依赖于变量ALL_MODULES指向的文件或者目标，因此，我们可以继续推断出，变量ALL_MODULES指向的文件或者目标一定会与文件out/target/product/generic/system/lib/libdis.so有依赖关系，这样才能够从make目标all_modules开始链式地生成上述文件。在接下来的分析中，我们就按照抓住上述文件的依赖关系进行逆向分析。

​      从上面的分析可以知道，在编译动态链接库文件的过程中，文件build/core/shared_library.mk会被加载，它的核心内容如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/19023609#) [copy](http://blog.csdn.net/luoshengyang/article/details/19023609#)

1. ......  
2.   
3. ifeq ($(strip $(LOCAL_MODULE_CLASS)),)  
4. LOCAL_MODULE_CLASS := SHARED_LIBRARIES  
5. endif  
6. ifeq ($(strip $(LOCAL_MODULE_SUFFIX)),)  
7. LOCAL_MODULE_SUFFIX := $(TARGET_SHLIB_SUFFIX)  
8. endif  
9.   
10. ......  
11.   
12. include $(BUILD_SYSTEM)/dynamic_binary.mk  
13.   
14. ......  
15.   
16. $(linked_module): $(all_objects) $(all_libraries) \  
17. ​                  $(LOCAL_ADDITIONAL_DEPENDENCIES) \  
18. ​                  $(my_target_crtbegin_so_o) $(my_target_crtend_so_o)  
19. ​    $(transform-o-to-shared-lib)  

​       LOCAL_MODULE_CLASS用来描述模块文件的类型。对于动态链接库文件来说，如果我们没有对它进行设置的话，它的默认值就等于SHARED_LIBRARIES。

​       LOCAL_MODULE_SUFFIX用来描述生成的模块文件的后缀名。对于动态链接库文件来说，如果我们没有对它进行设置的话，它的默认值就等于TARGET_SHLIB_SUFFIX。即.so。

​       上述两个变量限定了生成的动态链接库文件的完整文件名以及保存位置。

​       接下来，build/core/shared_library.mk文件加载了另外一个文件build/core/dynamic_binary.mk文件，并且为变量linked_module指向的文件制定了一个依赖规则，这个依赖规则由函数transform-o-to-shared-lib来执行。从函数transform-o-to-shared-lib就可以知道，它是根据一系列的中间编译文件（object文件）以及依赖库文件生成指定的动态链库文件的，主要就是由变量all_objects和all_libraries所描述的文件。现在，变量linked_module、all_objects和all_libraries所指向的文件是我们所要关心的。

​       我们接着分析文件build/core/dynamic_binary.mk文件的加载过程，它的内容如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/19023609#) [copy](http://blog.csdn.net/luoshengyang/article/details/19023609#)

1. ......  
2.   
3. LOCAL_UNSTRIPPED_PATH := $(strip $(LOCAL_UNSTRIPPED_PATH))  
4. ifeq ($(LOCAL_UNSTRIPPED_PATH),)  
5.   ifeq ($(LOCAL_MODULE_PATH),)  
6. ​    LOCAL_UNSTRIPPED_PATH := $(TARGET_OUT_$(LOCAL_MODULE_CLASS)_UNSTRIPPED)  
7.   else  
8. ​    # We have to figure out the corresponding unstripped path if LOCAL_MODULE_PATH is customized.  
9. ​    LOCAL_UNSTRIPPED_PATH := $(TARGET_OUT_UNSTRIPPED)/$(patsubst $(PRODUCT_OUT)/%,%,$(LOCAL_MODULE_PATH))  
10.   endif  
11. endif  
12.   
13. LOCAL_MODULE_STEM := $(strip $(LOCAL_MODULE_STEM))  
14. ifeq ($(LOCAL_MODULE_STEM),)  
15.   LOCAL_MODULE_STEM := $(LOCAL_MODULE)  
16. endif  
17. LOCAL_INSTALLED_MODULE_STEM := $(LOCAL_MODULE_STEM)$(LOCAL_MODULE_SUFFIX)  
18. LOCAL_BUILT_MODULE_STEM := $(LOCAL_INSTALLED_MODULE_STEM)  
19.   
20. \# base_rules.make defines $(intermediates), but we need its value  
21. \# before we include base_rules.  Make a guess, and verify that  
22. \# it's correct once the real value is defined.  
23. guessed_intermediates := $(call local-intermediates-dir)  
24.   
25. ......  
26.   
27. linked_module := $(guessed_intermediates)/LINKED/$(LOCAL_BUILT_MODULE_STEM)  
28.   
29. ......  
30.   
31. LOCAL_INTERMEDIATE_TARGETS := $(linked_module)  
32.   
33. \###################################  
34. include $(BUILD_SYSTEM)/binary.mk  
35. \###################################  
36.   
37. ......  
38.   
39. \###########################################################  
40. \## Compress  
41. \###########################################################  
42. compress_input := $(linked_module)  
43.   
44. ifeq ($(strip $(LOCAL_COMPRESS_MODULE_SYMBOLS)),)  
45.   LOCAL_COMPRESS_MODULE_SYMBOLS := $(strip $(TARGET_COMPRESS_MODULE_SYMBOLS))  
46. endif  
47.   
48. ifeq ($(LOCAL_COMPRESS_MODULE_SYMBOLS),true)  
49. $(error Symbol compression not yet supported.)  
50. compress_output := $(intermediates)/COMPRESSED-$(LOCAL_BUILT_MODULE_STEM)  
51.   
52. \#TODO: write the real $(STRIPPER) rule.  
53. \#TODO: define a rule to build TARGET_SYMBOL_FILTER_FILE, and  
54. \#      make it depend on ALL_ORIGINAL_DYNAMIC_BINARIES.  
55. $(compress_output): $(compress_input) $(TARGET_SYMBOL_FILTER_FILE) | $(ACP)  
56. ​    @echo "target Compress Symbols: $(PRIVATE_MODULE) ($@)"  
57. ​    $(copy-file-to-target)  
58. else  
59. \# Skip this step.  
60. compress_output := $(compress_input)  
61. endif  
62.   
63. \###########################################################  
64. \## Store a copy with symbols for symbolic debugging  
65. \###########################################################  
66. symbolic_input := $(compress_output)  
67. symbolic_output := $(LOCAL_UNSTRIPPED_PATH)/$(LOCAL_BUILT_MODULE_STEM)  
68. $(symbolic_output) : $(symbolic_input) | $(ACP)  
69. ​    @echo "target Symbolic: $(PRIVATE_MODULE) ($@)"  
70. ​    $(copy-file-to-target)  
71.   
72. \###########################################################  
73. \## Strip  
74. \###########################################################  
75. strip_input := $(symbolic_output)  
76. strip_output := $(LOCAL_BUILT_MODULE)  
77.   
78. ifeq ($(strip $(LOCAL_STRIP_MODULE)),)  
79.   LOCAL_STRIP_MODULE := $(strip $(TARGET_STRIP_MODULE))  
80. endif  
81.   
82. ifeq ($(LOCAL_STRIP_MODULE),true)  
83. \# Strip the binary  
84. $(strip_output): $(strip_input) | $(TARGET_STRIP)  
85. ​    $(transform-to-stripped)  
86. else  
87. ......  
88. endif # LOCAL_STRIP_MODULE  

​       LOCAL_UNSTRIPPED_PATH描述的是带符号的模块文件的输出目录。如果我们没有设置它，并且也没有设置变量LOCAL_MODULE_PATH的值，那么它的默认值就会与当前要编译的产品以及当前要编译的模块文件类型有关。例如，如果我们在执行lunch命令时，选择的是目标产品是模拟器，并且当前要编译的是动态链接库文件，那么得到的LOCAL_UNSTRIPPED_PATH值就为TARGET_OUT_$(LOCAL_MODULE_CLASS)_UNSTRIPPED。将$(LOCAL_MODULE_CLASS)替换为SHARED_LIBRARIES，就得到LOCAL_UNSTRIPPED_PATH的值为TARGET_OUT_SHARED_LIBRARIES_UNSTRIPPED，它值就等于out/target/product/generic/symbols/system/lib。也就是说，我们在为模拟器编译动态链接库模块时，生成的带符号文件都保存在目录out/target/product/generic/symbols/system/lib中。

​       如果我们没有设置 LOCAL_MODULE_STEM的值的话，那么它的默认值就等在我们在Android.mk文件中设置的LOCAL_MODULE的值。在我们的例子中，LOCAL_MODULE_STEM的值就等于LOCAL_MODULE的值，即libdis。

​       LOCAL_INSTALLED_MODULE_STEM和LOCAL_BUILT_MODULE_STEM的值等于LOCAL_MODULE_STEM的值再加上后缀名LOCAL_MODULE_SUFFIX。在我们的例子中，LOCAL_INSTALLED_MODULE_STEM和LOCAL_BUILT_MODULE_STEM的值就等于libdis.so。

​       这里调用函数local-intermediates-dir得到的是动态链接文件的中间输出目录，默认就是out/target/product/generic/obj/SHARED_LIBRARIES/libdis_intermediates了，因此，我们就可以得到变量linked_module的值为out/target/product/generic/obj/SHARED_LIBRARIES/libdis_intermediates/LINKED/libdis.so，这是编译过程要生成的文件之一。

​       LOCAL_INTERMEDIATE_TARGETS的值被设置为linked_module的值，接下来在加载build/core/binary.mk文件时需要用到。

​       接下来会根据out/target/product/generic/obj/SHARED_LIBRARIES/libdis_intermediates/LINKED/libdis.so文件生成另外三个文件：

​       1. 生成带符号压缩的模块文件，前提是LOCAL_COMPRESS_MODULE_SYMBOLS的值等于true。输入是compress_input，即linked_module，输出是compress_output，即$(intermediates)/COMPRESSED-$(LOCAL_BUILT_MODULE_STEM)。注意。目前还不支持此类型的模块文件。因此，当变量LOCAL_COMPRESS_MODULE_SYMBOLS的值等于true时，就会报错。

​       2. 拷贝一份带符号的模块文件到LOCAL_UNSTRIPPED_PATH描述的目录中去，即out/target/product/generic/symbols/system/lib目录。在我们这个情景中，得到的文件即为out/target/product/generic/symbols/system/lib/libdis.so。

​       3. 生成不带符号的模块文件，前提是LOCAL_STRIP_MODULE的值等于true。输入是前面拷贝到out/target/product/generic/symbols/system/lib目录的文件，输出由变量LOCAL_BUILT_MODULE指定。变量LOCAL_BUILT_MODULE的值是在加载文件build/core/binary.mk的过程中指定的。

​       到目前为止，我们就解决前面提到的文件out/target/product/generic/obj/SHARED_LIBRARIES/libdis_intermediates/LINKED/libdis.so和out/target/product/generic/symbols/system/lib/libdis.so的生成过程，还剩下out/target/product/generic/obj/lib/libdis.so和out/target/product/generic/system/lib/libdis.so文件的生成过程未搞清楚。这就需要继续分析文件build/core/binary.mk的加载过程。

​       文件build/core/binary.mk的核心内容如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/19023609#) [copy](http://blog.csdn.net/luoshengyang/article/details/19023609#)

1. ifdef LOCAL_SDK_VERSION  
2.   # Get the list of INSTALLED libraries as module names.  
3.   # We cannot compute the full path of the LOCAL_SHARED_LIBRARIES for  
4.   # they may cusomize their install path with LOCAL_MODULE_PATH  
5.   installed_shared_library_module_names := \  
6. ​      $(LOCAL_SHARED_LIBRARIES)  
7. else  
8.   installed_shared_library_module_names := \  
9. ​      $(LOCAL_SYSTEM_SHARED_LIBRARIES) $(LOCAL_SHARED_LIBRARIES)  
10. endif  
11. \# The real dependency will be added after all Android.mks are loaded and the install paths  
12. \# of the shared libraries are determined.  
13. LOCAL_REQUIRED_MODULES += $(installed_shared_library_module_names)  
14.   
15. \#######################################  
16. include $(BUILD_SYSTEM)/base_rules.mk  
17. \#######################################  
18.   
19. ......  
20.   
21. \###########################################################  
22. \## C++: Compile .cpp files to .o.  
23. \###########################################################  
24.   
25. \# we also do this on host modules, even though  
26. \# it's not really arm, because there are files that are shared.  
27. cpp_arm_sources    := $(patsubst %$(LOCAL_CPP_EXTENSION).arm,%$(LOCAL_CPP_EXTENSION),$(filter %$(LOCAL_CPP_EXTENSION).arm,$(LOCAL_SRC_FILES)))  
28. cpp_arm_objects    := $(addprefix $(intermediates)/,$(cpp_arm_sources:$(LOCAL_CPP_EXTENSION)=.o))  
29.   
30. cpp_normal_sources := $(filter %$(LOCAL_CPP_EXTENSION),$(LOCAL_SRC_FILES))  
31. cpp_normal_objects := $(addprefix $(intermediates)/,$(cpp_normal_sources:$(LOCAL_CPP_EXTENSION)=.o))  
32.   
33. $(cpp_arm_objects):    PRIVATE_ARM_MODE := $(arm_objects_mode)  
34. $(cpp_arm_objects):    PRIVATE_ARM_CFLAGS := $(arm_objects_cflags)  
35. $(cpp_normal_objects): PRIVATE_ARM_MODE := $(normal_objects_mode)  
36. $(cpp_normal_objects): PRIVATE_ARM_CFLAGS := $(normal_objects_cflags)  
37.   
38. cpp_objects        := $(cpp_arm_objects) $(cpp_normal_objects)  
39.   
40. ifneq ($(strip $(cpp_objects)),)  
41. $(cpp_objects): $(intermediates)/%.o: \  
42. ​    $(TOPDIR)$(LOCAL_PATH)/%$(LOCAL_CPP_EXTENSION) \  
43. ​    $(yacc_cpps) $(proto_generated_headers) $(my_compiler_dependencies) \  
44. ​    $(LOCAL_ADDITIONAL_DEPENDENCIES)  
45. ​    $(transform-$(PRIVATE_HOST)cpp-to-o)  
46. -include $(cpp_objects:%.o=%.P)  
47. endif  
48.   
49. ......  
50.   
51. \###########################################################  
52. \## C: Compile .c files to .o.  
53. \###########################################################  
54.   
55. c_arm_sources    := $(patsubst %.c.arm,%.c,$(filter %.c.arm,$(LOCAL_SRC_FILES)))  
56. c_arm_objects    := $(addprefix $(intermediates)/,$(c_arm_sources:.c=.o))  
57.   
58. c_normal_sources := $(filter %.c,$(LOCAL_SRC_FILES))  
59. c_normal_objects := $(addprefix $(intermediates)/,$(c_normal_sources:.c=.o))  
60.   
61. $(c_arm_objects):    PRIVATE_ARM_MODE := $(arm_objects_mode)  
62. $(c_arm_objects):    PRIVATE_ARM_CFLAGS := $(arm_objects_cflags)  
63. $(c_normal_objects): PRIVATE_ARM_MODE := $(normal_objects_mode)  
64. $(c_normal_objects): PRIVATE_ARM_CFLAGS := $(normal_objects_cflags)  
65.   
66. c_objects        := $(c_arm_objects) $(c_normal_objects)  
67.   
68. ifneq ($(strip $(c_objects)),)  
69. $(c_objects): $(intermediates)/%.o: $(TOPDIR)$(LOCAL_PATH)/%.c $(yacc_cpps) $(proto_generated_headers) \  
70. ​    $(my_compiler_dependencies) $(LOCAL_ADDITIONAL_DEPENDENCIES)  
71. ​    $(transform-$(PRIVATE_HOST)c-to-o)  
72. -include $(c_objects:%.o=%.P)  
73. endif  
74.   
75. ......  
76.   
77. \# some rules depend on asm_objects being first.  If your code depends on  
78. \# being first, it's reasonable to require it to be assembly  
79. all_objects := \  
80. ​    $(asm_objects) \  
81. ​    $(cpp_objects) \  
82. ​    $(gen_cpp_objects) \  
83. ​    $(gen_asm_objects) \  
84. ​    $(c_objects) \  
85. ​    $(gen_c_objects) \  
86. ​    $(objc_objects) \  
87. ​    $(yacc_objects) \  
88. ​    $(lex_objects) \  
89. ​    $(proto_generated_objects) \  
90. ​    $(addprefix $(TOPDIR)$(LOCAL_PATH)/,$(LOCAL_PREBUILT_OBJ_FILES))  
91.   
92. ......  
93.   
94. ifdef LOCAL_SDK_VERSION  
95. built_shared_libraries := \  
96. ​    $(addprefix $($(my_prefix)OUT_INTERMEDIATE_LIBRARIES)/, \  
97. ​      $(addsuffix $(so_suffix), \  
98. ​        $(LOCAL_SHARED_LIBRARIES)))  
99.   
100. my_system_shared_libraries_fullpath := \  
101. ​    $(my_ndk_stl_shared_lib_fullpath) \  
102. ​    $(addprefix $(my_ndk_version_root)/usr/lib/, \  
103. ​        $(addsuffix $(so_suffix), $(LOCAL_SYSTEM_SHARED_LIBRARIES)))  
104.   
105. built_shared_libraries += $(my_system_shared_libraries_fullpath)  
106. LOCAL_SHARED_LIBRARIES += $(LOCAL_SYSTEM_SHARED_LIBRARIES)  
107. else  
108. LOCAL_SHARED_LIBRARIES += $(LOCAL_SYSTEM_SHARED_LIBRARIES)  
109. built_shared_libraries := \  
110. ​    $(addprefix $($(my_prefix)OUT_INTERMEDIATE_LIBRARIES)/, \  
111. ​      $(addsuffix $(so_suffix), \  
112. ​        $(LOCAL_SHARED_LIBRARIES)))  
113. endif  
114.   
115. built_static_libraries := \  
116. ​    $(foreach lib,$(LOCAL_STATIC_LIBRARIES), \  
117. ​      $(call intermediates-dir-for, \  
118. ​        STATIC_LIBRARIES,$(lib),$(LOCAL_IS_HOST_MODULE))/$(lib)$(a_suffix))  
119.   
120. ifdef LOCAL_SDK_VERSION  
121. built_static_libraries += $(my_ndk_stl_static_lib)  
122. endif  
123.   
124. built_whole_libraries := \  
125. ​    $(foreach lib,$(LOCAL_WHOLE_STATIC_LIBRARIES), \  
126. ​      $(call intermediates-dir-for, \  
127. ​        STATIC_LIBRARIES,$(lib),$(LOCAL_IS_HOST_MODULE))/$(lib)$(a_suffix))  
128.   
129. ......  
130.   
131. \###########################################################  
132. \# Define library dependencies.  
133. \###########################################################  
134. \# all_libraries is used for the dependencies on LOCAL_BUILT_MODULE.  
135. all_libraries := \  
136. ​    $(built_shared_libraries) \  
137. ​    $(built_static_libraries) \  
138. ​    $(built_whole_libraries)  
139.   
140. ......  

​        文件build/core/binary.mk的加载逻辑如下所示：

​        1. 获得当前编译的模块所依赖的动态链接库，也就是我们在Android.mk文件中通过LOCAL_SHARED_LIBRARIES变量引用的动态链接库。注意，如果定义了变量LOCAL_SDK_VERSION，那么就表示是在SDK环境下编译，这时候是不可以使用一些隐藏的系统动态链接库。这些隐藏的系统动态链接库由变量LOCAL_SYSTEM_SHARED_LIBRARIES描述。最终获得的依赖动态链接库保存在变量LOCAL_REQUIRED_MODULES中。前面我们在分析build/core/main.mk的加载过程时提到，Android编译系统会为当前编译的模块所依赖的每一个模块都生成一个依赖规则，用来保证编译出来的当前模块是最新的。

​        2. 加载另外一个脚本文件build/core/base_rules.mk，用来计算一些基本变量的值，以及创建一些基本的依赖规则。

​        3. 根据LOCAL_SRC_FILES和LOCAL_CPP_EXTENSION定义的C++文件制定对应的C++目标文件的依赖规则，并且通过函数transform-$(PRIVATE_HOST)cpp-to-o执行这些规则，实际上就是调用gcc来编译相应的C++源文件。注意，当我们是为目标机器编译模块时，变量PRIVATE_HOST的值为空，因此，这时候实际上是调用transform-cpp-to-o来将.cpp源文件编译成.o目标文件。

​       4.  同样被制定依赖规则的还包括在LOCAL_SRC_FILES中引用的C文件、汇编文件、YACC文件和LEX文件等等。最终得到的所有目标文件都保存在变量all_objects中。

​       5.  获得LOCAL_SHARED_LIBRARIES定义的各个动态依赖库的文件路径，并且保存在变量built_shared_libraries中。

​       6.  获得LOCAL_STATIC_LIBRARIES定义的各个静态依赖库的文件路径，并且保存在变量built_static_libraries中。

​       7.  获得LOCAL_WHOLE_STATIC_LIBRARIES定义的各个要完全静态链入当前编译模块的依赖库的文件路径，并且保存在变量built_whole_libraries中。

​       8.  将所有获得的静态和动态依赖库文件路径保存在变量all_libraries中。

​       至此，变量all_objects和all_libraries就描述了当前模块依赖的所有目标文件和库文件。前面在分析build/core/shared_library.mk的加载过程时提到一个依赖规则，也就是变量linked_module定义的文件依赖于变量all_objects和all_libraries的文件。也就是说，在文件build/core/binary.mk文件加载完成之后，我们就可以获得由变量linked_module所定义的文件。在我们这个情景中，变量linked_module定义的文件就是out/target/product/generic/obj/SHARED_LIBRARIES/libdis_intermediates/LINKED/libdis.so。

​       在我们这个情景中，前面提到的out/target/product/generic/obj/lib/libdis.so和out/target/product/generic/system/lib/libdis.so文件的生过过程我们依然还没有搞清楚，这就需要进一步分析build/core/base_rules.mk文件。

​       文件build/core/base_rules.mk的相关内容如下所示：

**[plain]** [view plain](http://blog.csdn.net/luoshengyang/article/details/19023609#) [copy](http://blog.csdn.net/luoshengyang/article/details/19023609#)

1. ......  
2.   
3. LOCAL_MODULE_PATH := $(strip $(LOCAL_MODULE_PATH))  
4. ifeq ($(LOCAL_MODULE_PATH),)  
5.   LOCAL_MODULE_PATH := $($(my_prefix)OUT$(partition_tag)_$(LOCAL_MODULE_CLASS))  
6.   ifeq ($(strip $(LOCAL_MODULE_PATH)),)  
7. ​    $(error $(LOCAL_PATH): unhandled LOCAL_MODULE_CLASS "$(LOCAL_MODULE_CLASS)")  
8.   endif  
9. endif  
10. ......  
11.   
12. LOCAL_INSTALLED_MODULE_STEM := $(LOCAL_MODULE_STEM)$(LOCAL_MODULE_SUFFIX)  
13. ......  
14.   
15. intermediates := $(call local-intermediates-dir)  
16. ......  
17.   
18. \# OVERRIDE_BUILT_MODULE_PATH is only allowed to be used by the  
19. \# internal SHARED_LIBRARIES build files.  
20. OVERRIDE_BUILT_MODULE_PATH := $(strip $(OVERRIDE_BUILT_MODULE_PATH))  
21. ifdef OVERRIDE_BUILT_MODULE_PATH  
22.   ifneq ($(LOCAL_MODULE_CLASS),SHARED_LIBRARIES)  
23. ​    $(error $(LOCAL_PATH): Illegal use of OVERRIDE_BUILT_MODULE_PATH)  
24.   endif  
25.   built_module_path := $(OVERRIDE_BUILT_MODULE_PATH)  
26. else  
27.   built_module_path := $(intermediates)  
28. endif  
29. LOCAL_BUILT_MODULE := $(built_module_path)/$(LOCAL_BUILT_MODULE_STEM)  
30. ......  
31.   
32. ifneq (true,$(LOCAL_UNINSTALLABLE_MODULE))  
33.   LOCAL_INSTALLED_MODULE := $(LOCAL_MODULE_PATH)/$(LOCAL_INSTALLED_MODULE_STEM)  
34. endif  
35. ......  
36.   
37. \# Provide a short-hand for building this module.  
38. \# We name both BUILT and INSTALLED in case  
39. \# LOCAL_UNINSTALLABLE_MODULE is set.  
40. .PHONY: $(LOCAL_MODULE)  
41. $(LOCAL_MODULE): $(LOCAL_BUILT_MODULE) $(LOCAL_INSTALLED_MODULE)  
42. ......  
43.   
44. ifndef LOCAL_UNINSTALLABLE_MODULE  
45.   # Define a copy rule to install the module.  
46.   # acp and libraries that it uses can't use acp for  
47.   # installation;  hence, LOCAL_ACP_UNAVAILABLE.  
48. ifneq ($(LOCAL_ACP_UNAVAILABLE),true)  
49. $(LOCAL_INSTALLED_MODULE): $(LOCAL_BUILT_MODULE) | $(ACP)  
50. ​    @echo "Install: $@"  
51. ​    $(copy-file-to-new-target)  
52. else  
53. $(LOCAL_INSTALLED_MODULE): $(LOCAL_BUILT_MODULE)  
54. ​    @echo "Install: $@"  
55. ​    $(copy-file-to-target-with-cp)  
56. endif  
57. ......  
58.   
59. ALL_MODULES += $(LOCAL_MODULE)  
60. ......  

​       文件build/core/base_rules.mk的加载过程如下所示：

​       1. 如果我们没有在Android.mk文件中定义LOCAL_MODULE_PATH，那么LOCAL_MODULE_PATH的值就设置为$($(my_prefix)OUT$(partition_tag)_$(LOCAL_MODULE_CLASS))。它的具体值与执行lunch命令时选择的目标产品和当前要编译的模块类型有关，例如，当我们编译的是动态链接库，并且目标机器是模拟器时，它的值就等于out/target/product/generic/system/lib。

​        2. 前面在加载文件build/core/dynamic_binary.mk时提到，LOCAL_MODULE_STEM的值默认等于LOCAL_MODULE的值，而LOCAL_MODULE_SUFFIX的值等于.so，因此，这里得到的LOCAL_INSTALLED_MODULE_STEM的值就等于当前要编译的模块名称再加上其对应的后缀名。

​        3. 函数local-intermediates-dir的返回值执行lunch命令时选择的目标产品和当前要编译的模块类型有关。例如，当我们编译的是动态链接库，并且目标机器是模拟器时，它的值就等于out/target/product/generic/obj/lib，也就是intermediates的值等于out/target/product/generic/obj/lib。

​        4. 如果我们没有在Android.mk文件中定义OVERRIDE_BUILT_MODULE_PATH的值，那么就表示要将生成的不带符号的模块文件保存在intermediates指定的目录中。

​        5. 经过上面的准备工作之后，我们就得到LOCAL_BUILT_MODULE的值等于$(built_module_path)/$(LOCAL_BUILT_MODULE_STEM)。也就是它描述的是当前模块在编译的过程中产生的不带符号模块文件路径。在我们这个情景中，实际上就是out/target/product/generic/obj/lib/libdis.so。

​        6. 如果我们没有在Android.mk文件中将LOCAL_UNINSTALLABLE_MODULE的值设置为true，那么就表示我们需要将最终的不带符号的模块文件拷贝到变量LOCAL_MODULE_PATH所描述的目录中，并且文件名为LOCAL_INSTALLED_MODULE_STEM。在我们这个情景中，实际上拷贝得到的文件就是out/target/product/generic/system/lib/libdis.so，并且通过变量LOCAL_INSTALLED_MODULE来描述。

​        7. 制定伪目标LOCAEL_MODULE的依赖规则，即它依赖于变量LOCAL_BUILT_MODULE和LOCAL_INSTALLED_MODULE定义的文件。在我们这个情景中，实际上就是定义了一个伪目标libdis，它依赖于out/target/product/generic/obj/lib/libdis.so和out/target/product/generic/system/lib/libdis.so文件。

​        8. 前面提到，如果我们没有在Android.mk文件中定义LOCAL_UNINSTALLABLE_MODULE，那么就表示要将最终的不带符号的模块文件拷贝到变量LOCAL_MODULE_PATH所描述的目录中，即将LOCAL_BUILT_MODULE描述的文件拷贝为LOCAL_INSTALLED_MODULE描述的文件。这时候我们还需要制定一个LOCAL_INSTALLED_MODULE依赖LOCAL_BUILT_MODULE的规则。

​        9. 将表示当前模块名称的LOCAL_MODULE值附加到ALL_MODULES后面去。

​        通过第5步我们确定了LOCAL_BUILT_MODULE的值。前面对build/core/dynamic_binary.mk文件的加载过程分析中提到，LOCAL_BUILT_MODULE描述的文件依赖于out/target/product/generic/symbols/system/lib/libdis.so文件。也就是说，out/target/product/generic/obj/lib/libdis.so文件依赖于out/target/product/generic/symbols/system/lib/libdis.so文件。同样，通过第6步我们确定了out/target/product/generic/system/lib/libdis.so依赖于out/target/product/generic/obj/lib/libdis.so文件。

​       至此，我们就搞清楚了前面提到的四个文件out/target/product/generic/obj/SHARED_LIBRARIES/libdis_intermediates/LINKED/libdis.so、out/target/product/generic/symbols/system/lib/libdis.so、out/target/product/generic/obj/lib/libdis.so和out/target/product/generic/system/lib/libdis.so之间从右到左的依赖关系以及生成过程。

​       然而，我们前面又提到，在执行mmm命令时，默认的make目标为all_modules，并且该目标依赖于变量ALL_MODULES所描述的伪目标，而变量ALL_MODULES所描述的伪目标又等于当前要编译的模块名称，即LOCAL_MODULE。最后，LOCAL_MODULE也是一个伪目标，它依赖于LOCAL_BUILT_MODULE和LOCAL_INSTALLED_MODULE所定义的文件，也就是out/target/product/generic/obj/lib/libdis.so和out/target/product/generic/system/lib/libdis.so文件。

​       这样，我们就可以得到在我们所分析的情景中，上述各个make目标或者文件的之间的链式依赖关系：

​       1. all_modules依赖于ALL_MODULES；

​       2. ALL_MODULES依赖于LOCAL_MODULE；

​       3. LOCAL_MODULE依赖于out/target/product/generic/obj/lib/libdis.so和out/target/product/generic/system/lib/libdis.so；

​       4. out/target/product/generic/system/lib/libdis.so依赖于out/target/product/generic/obj/lib/libdis.so；

​       5. out/target/product/generic/obj/lib/libdis.so依赖于out/target/product/generic/symbols/system/lib/libdis.so；

​       6. out/target/product/generic/symbols/system/lib/libdis.so依赖于out/target/product/generic/obj/SHARED_LIBRARIES/libdis_intermediates/LINKED/libdis.so；

​       7. out/target/product/generic/obj/SHARED_LIBRARIES/libdis_intermediates/LINKED/libdis.so依赖于all_objects和all_libraries;

​       8. all_objects依赖于在Android.mk文件中通过LOCAL_SRC_FILES定义的文件dispatcher.cpp和common.cpp文件；

​       9. all_libraries依赖于在Android.mk文件中通过LOCAL_SHARED_LIBRARIES定义的文件libdl.so和liblog.so文件。

​       通过调用gcc等编译工具，就可以从最后一步开始，一步步向上编译出各个模块文件，并且保存在合适的位置中。

​       至此，我们就分析完成了使用mmm命令来编译一个动态链接库的过程。使用mmm命令来编译其它类型的模块，例如APK文件、EXE文件和JAVA库，过程也是差不多的，无非都是建立各个文件之间的依赖关系，以及调用相应的编译工具来进行编译。弄懂了mmm命令的编译过程之后， 另外的三个编译命令m、mm和make也可以一目了然了。

​       当在Android源码中定义的各个模块都编译好之后，我们还需要将编译得到的文件打包成相应的镜像文件，例如system.img、boot.img和recorvery.img等，这样我们才可以将这些镜像烧录到目标设备去运行。在接下来的一篇文章中，我们就将继续分析Android镜像文件的打包过程，敬请关注！更多信息可以关注老罗的新浪微博：[http://weibo.com/shengyangluo](http://weibo.com/shengyangluo)