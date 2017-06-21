> 视频地址：http://v.youku.com/v_show/id_XMjUzMjM1NjY4MA==.html

## 视频大纲

### 日志调试分析法

与远程gdb调试分析法相比，日志调试分析法有以下优势：

1. 日志记录不会中断程序的执行

2. 日志记录可以收集更多的上下文信息

3. 在多线程环境下，日志记录能更完整的展现程序当时的状态

4. Android的日志系统对调试提供了完善的支持，时间可以精确到毫秒，同时还记录了进程和线程信息

### 获取Java函数调用栈

```
try {
    throw new Exception("Call Stack Trace");
} catch (Exception e) {
    Log.i("shyluo@gmail.com", "xxx", e);
}
```

### 获取C++函数调用栈

Android.mk:

```
LOCAL_SHARED_LIBRARIES := libutils
```
`*.cpp:`
```c
#include <utils/CallStack.h>
 
CallStack stack;
stack.update();
stack.log("shyluo@gmail.com");
```

函数地址转换为函数名称：

```
arm-eabi-addr2line -e out/target/product/generic/symbols/system/lib/libxxx.so -f -C <addr> 
```