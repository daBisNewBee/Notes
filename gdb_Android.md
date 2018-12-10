
# gdb在Android Native下的使用
## 1. gdb单步调试Android ELF（C/C++）
### 1.1 gdbserver
- adb push gdbserver /system/bin/

> "gdbserver"位于host的**android-ndk-r9d\prebuilt\android-arm\gdbserver\gdbserver**

- adb push E:\github\hello-jni\libs\armeabi /data/local/armeabi

> armeabi目录下包括二进制程序及依赖的动态库，如main，libhello-jni.so

- adb forward tcp:7777 tcp:7777
- adb shell 
    - cd /data/local/armeabi
    - chmod 777 ./main
    - export LD_LIBRARY_PATH=./
    - gdbserver :7777 ./main

```
root@msm8974:/data/local/armeabi # gdbserver :7777 main
gdbserver :7777 main
Process main created; pid = 2291
Listening on port 7777
```

### 1.2 gdbclient
- cd D:\Installer\android-ndk-r9d\toolchains\arm-linux-androideabi-4.6\prebuilt\windows\bin

> toolchains版本默认为“arm-linux-androideabi-4.6”

- arm-linux-androideabi-gdb.exe -q
E:\github\hello-jni\obj\local\armeabi\main (指定的本地可执行文件与target上一致；且必须位于obj目录下，带调试信息)
    - (gdb) target remote :7777
    - (gdb) b main
    - (gdb) continue (不可以用 run 或者start。main程序已经执行，只不过处于挂起状态)
    - (gdb) i sharedlibrary

    ```
    From        To          Syms Read   Shared Object Library
                        No          /system/bin/linker
                        No          libc.so
                        No          libstdc++.so
                        No          libm.so
                        No          liblog.so
                        No          libcutils.so
                        No          libNimsWrap.so
                        No          libhello-jni.so
    ```
    - (gdb) set solib-search-path E:\github\hello-jni\obj\local\armeabi
    - (gdb) i sharedlibrary
    
    ```
    From        To          Syms Read   Shared Object Library
    0xb6f84a60  0xb6f8fac0  Yes (*)     E:/github/hello-jni/obj/local/armeabi/linker
    0xb6f2e180  0xb6f5ecec  Yes (*)     E:/github/hello-jni/obj/local/armeabi/libc.so
                            No          libstdc++.so
                            No          libm.so
                            No          liblog.so
                            No          libcutils.so
                            No          libNimsWrap.so
    0xb6eecbec  0xb6eee0a0  Yes         E:/github/hello-jni/obj/local/armeabi/libhello-jni.so
    ```
    - (gdb) next\step\break xxx\......

## 2. gdb快速定位Android APK crash堆栈现场
```
 jstring
Java_com_example_hellojni_HelloJni_stringFromJNI( JNIEnv* env,
                                                  jobject thiz )
{
	printf("111111111111111\n");
	printf("222222222222222222\n");
	dynamicFunc();
    return (*env)->NewStringUTF(env, "Hello from JNI !  Compiled with ABI " ABI ".");
}
void dynamicFunc()
{
	printf("This called from dynamicFunc\n");
	char *p=NULL;
	*p = "123";   // signal SIGSEGV
}
比如点击按钮运行“stringFromJNI”，在调用到dynamicFunc()中存在错误的内存使用，产生段错误。
```
### 2.1 gdbserver
- 点击应用，运行apk服务
- adb shell
    - ps |grep "com.example.hellojni"

    ```
    root@msm8974:/data/local/armeabi # ps|grep "com.example.hellojni"
	u0_a208   25704 277   1013352 27136 ffffffff 4010b8e8 S com.example.hellojni
    ```
    
    - gdbserver :7777 --attach 25704

    ```
    root@msm8974:/data/local/armeabi # gdbserver :7777 --attach 25704
	gdbserver :7777 --attach 25704
	Attached; pid = 25704
	Listening on port 7777
    ```

### 2.2 gdbclient
- arm-linux-androideabi-gdb.exe -q
    - (gdb) target remote :7777
    - (gdb) set solib-search-path E:\github\hello-jni\obj\local\armeabi
    - (gdb) c
    - 点击按钮，触发crash

    ```
    (gdb) c
    Continuing.
    
    Program received signal SIGSEGV, Segmentation fault.
    0x7c3f2c30 in dynamicFunc () at E:/github/hello-jni//jni/hello-jni.c:75
    75              *p = "123";
    ```
    - (gdb) bt  // 查看堆栈信息

    ```
    (gdb) bt
    #0  0x7c3f2c30 in dynamicFunc () at E:/github/hello-jni//jni/hello-jni.c:75
    #1  0x7c3f2c5c in Java_com_example_hellojni_HelloJni_stringFromJNI (env=0x41b91fc8, thiz=<optimized out>)
        at E:/github/hello-jni//jni/hello-jni.c:64
    #2  0x41667768 in ?? ()
    #3  0x4160a0d4 in ?? ()
    #4  0x4160a0d4 in ?? ()
    Backtrace stopped: previous frame identical to this frame (corrupt stack?)
    ```

## 3. 常见问题
### ~~3.1 无法**单步**调试APK下的动态库~~ `见3.6`
```
(gdb) next

Program received signal SIGILL, Illegal instruction.
0x7c3f1ba8 in ?? () from E:/github/hello-jni/obj/local/armeabi/libhello-jni.so
```
- 程序在指定的断点停下后，单步执行，会报如上错误。
- 原因：
    - [root cause](https://e2e.ti.com/support/embedded/android/f/509/t/374574).
    
    > It seems that the root cause is an address offset calculation are done twice in different places. I checked the rowboat kernel code in rowboat/rowboat-am335x-kernel-3.2 branch, the code isn't fixed yet.

    - 安全红线问题

- 解决方法：
    - 添加多个断点，使用**continue**操作。使用该方法代替单步。
    
>
Afaict openssl probes the capabilities of the user's CPU by trying to do things and trapping the illegal instruction errors. So a couple of sigills during startup is normal. When using a debugger in order to find the real failure in your application you must **continue** past the startup sigills. 14. Use GDB commands to debug

### 3.2 ndk-gdb的使用

> ndk-gdb --start --verbose --force --nowait

- 这个辅助脚本集成gdbserver、gdbclient等一系列操作，这是调试apk应用的最佳方式。 

### 3.3 堆栈线程信息未知

```
(gdb) bt
#0  0x400f08e8 in ?? ()
#1  0x4016a642 in ?? ()
#2  0x4016a642 in ?? ()

(gdb) i threads
  Id   Target Id         Frame
  11   Thread 10442      0x400f0ab0 in ?? ()
  10   Thread 10432      0x400ef734 in ?? ()
  9    Thread 10431      0x400ef734 in ?? ()
  8    Thread 10429      0x400f0ab0 in ?? ()
  7    Thread 10428      0x400f0ab0 in ?? ()
  6    Thread 10427      0x400f0ab0 in ?? ()
  5    Thread 10426      0x400f0ab0 in ?? ()
  4    Thread 10425      0x400f0584 in ?? ()
  3    Thread 10424      0x400f031c in ?? ()
  2    Thread 10423      0x400f0ab0 in ?? ()
* 1    Thread 10419      0x400f08e8 in ?? ()
```

- adb pull /system/lib/libc.so E:\github\hello-jni\obj\local\armeabi
- (gdb) set solib-search-path E:\github\hello-jni\obj\local\armeabi
- i sharedlibrary

```
(gdb) i threads
  Id   Target Id         Frame
  11   Thread 10442      0x400f0ab0 in __futex_syscall3 () from E:/github/hello-jni/obj/local/armeabi/libc.so
  10   Thread 10432      0x400ef734 in __ioctl () from E:/github/hello-jni/obj/local/armeabi/libc.so
  9    Thread 10431      0x400ef734 in __ioctl () from E:/github/hello-jni/obj/local/armeabi/libc.so
  8    Thread 10429      0x400f0ab0 in __futex_syscall3 () from E:/github/hello-jni/obj/local/armeabi/libc.so
  7    Thread 10428      0x400f0ab0 in __futex_syscall3 () from E:/github/hello-jni/obj/local/armeabi/libc.so
  6    Thread 10427      0x400f0ab0 in __futex_syscall3 () from E:/github/hello-jni/obj/local/armeabi/libc.so
  5    Thread 10426      0x400f0ab0 in __futex_syscall3 () from E:/github/hello-jni/obj/local/armeabi/libc.so
  4    Thread 10425      0x400f0584 in recvmsg () from E:/github/hello-jni/obj/local/armeabi/libc.so
  3    Thread 10424      0x400f031c in __rt_sigtimedwait () from E:/github/hello-jni/obj/local/armeabi/libc.so
  2    Thread 10423      0x400f0ab0 in __futex_syscall3 () from E:/github/hello-jni/obj/local/armeabi/libc.so
* 1    Thread 10419      0x400f08e8 in epoll_wait () from E:/github/hello-jni/obj/local/armeabi/libc.so
(gdb) bt
#0  0x400f08e8 in epoll_wait () from E:/github/hello-jni/obj/local/armeabi/libc.so
#1  0x4016a642 in ?? ()
#2  0x4016a642 in ?? ()
```

- 相关动态库的信息就打印出来了。原因就在于host上不存在target上的动态库或者版本不一致。需要将target上实际使用的动态库同步到本地

### 3.4 同步必要的手机(target)系统lib至本地(host)
- 在host上，创建本地目录，专门存放该型号手机下的系统lib

```
mkdir ~/HUAWEI_P10
mkidr ~/HUAWEI_P10/system_lib
mkidr ~/HUAWEI_P10/vendor_lib
```
- 同步target的多个目录、文件至host

```
cd ~/Android/system_lib/
adb pull /system/lib

cd ~/Android/vendor_lib/
adb pull /vendor/lib

cd ~/Android
# 在64位系统 /system/bin/app_process32 和 /system/bin/app_process64
adb pull /system/bin/app_process

cd ~/Android
# 在64位系统 /system/bin/linker 和 /system/bin/linker64
adb pull /system/bin/linker         
```

### 3.5 如何设置动态库的多路径

```
set solib-search-path J:\git\SDK-Project\TF-SDK\src\main\obj\local\armeabi;H:\Users\liuwb\HUAWEI_P10;H:\Users\liuwb\HUAWEI_P10\system_lib;H:\Users\liuwb\HUAWEI_P10\vendor_lib
```

其中：
- 多路径之间必须用`";"`分割！！（windows环境下）
- [原因](http://infocenter.arm.com/help/index.jsp?topic=/com.arm.doc.dui0452a/CIHJHBDE.html)：

```
Multiple directories can be specified but must be separated with either:
    a colon (Unix)
    a semi-colon (Windows).
```

- 也可参考host本身的`PATH`,同样使用`;`分割

```
(gdb) PATH
Executable and object file path: D:\Android\android-ndk-r9d\prebuilt\windows\bin;H:\WINDOWS\system32;H:\WINDOWS;H:\WINDOWS\System32\Wbem;H:\WINDOWS\System32\WindowsPowerShell\v1.0\;H:\Program Files (x86)\Java\jdk1.7.0_55\bin;
```

### 3.6 在非root设备中，gdb单步调试的普通APP

> NDK在非root手机上调试APP的原理：gdbserver通过run-as获得与目标进程相同的UID，然后就可以ptrace到目标进程去调试了。

- APP开启debuggable："android:debuggable="true""
- 推送cmd:gdbserver至手机中某目录下

```
adb push D:\Android\android-ndk-r9d\prebuilt\android-arm\gdbserver\gdbserver /sdcard/
```

- 进入APP沙盒内（本质使得接下来的操作获得与目标APP同样的UID。原因：由于gdbserver使用ptrace注入目标进程，权限检查包括gdbserver和目标进程的uid、gid一致，而run-as可以修改gdbserver进程的uid）

```
adb shell
run-as km.ssl
```

- 获得gdbserver命令行，拷贝至APP目录下

```
HWVKY:/data/data/km.ssl $ pwd
/data/data/km.ssl

cp /sdcard/gdbserver ./
chmod 777 ./gdbserver
```
- 运行APP，获得APP进程号，attach后进行调试。（与上相同）

```
HWVKY:/sdcard $ ps|grep km.ssl
ps|grep km.ssl
u0_a343   16940 516   1592224 109824          0 0000000000 S km.ssl

HWVKY:/data/data/km.ssl $ ./gdbserver :7777 --attach 16940
./gdbserver :7777 --attach 16940
Attached; pid = 16940
Listening on port 7777

```

### 3.7 TODO:gdb无法调试运行中进程
- gdbserver 使用socket启动失败，报错：

```
130|a6pltechn:/data/data/com.ximalaya.mediaprocessor $ ./gdbserver :7777 --attach 26229                                                                                     
 Can't open socket: Permission denied.
Exiting
```

- gdbserver 使用管道代替，启动成功:

```
1|a6pltechn:/data/data/com.ximalaya.mediaprocessor $ ./gdbserver +debug-pipe --attach 26229                                                                                 
Attached; pid = 26229
Listening on Unix socket debug-pipe

```
- 此时，host下的设置需要做相应调整，表示已经建立本地10000端口到server端pipe的映射：

```
adb forward --remove-all

adb forward tcp:10000 local:/data/data/com.ximalaya.mediaprocessor/debug-pipe

adb forward --list
8a329736 tcp:10000 local:/data/data/com.ximalaya.mediaprocessor/debug-piped

```

- 此时，gdbclient无法连接到remote上，调试中断

```
(gdb) target remote :10000
Remote debugging using :10000
Remote communication error.  Target disconnected.: Connection reset by peer.
```

- `arm-linux-androideabi-gdb`在新版本ndk中已经废除，需要从老版本r10e中找。



## 4. 参考
- [大家怎么调试android c/c++?](https://www.zhihu.com/question/31993785)
- [Android逆向系列之动态调试(五)–gdb调试](http://www.tasfa.cn/index.php/2016/06/01/android-re-gdb/)
- [gdb 远程调试android进程](http://blog.csdn.net/xinfuqizao/article/details/7955346)
- [Android debugging with remote GDB](https://github.com/mapbox/mapbox-gl-native/wiki/Android-debugging-with-remote-GDB)
- [NDK调试之ndk-gdb](https://my.oschina.net/wolfcs/blog/527317)
- [从NDK在非Root手机上的调试原理探讨Android的安全机制](http://blog.csdn.net/dj0379/article/details/43554761)
- [非root Android 设备用gdbserver进行native 调试的方法](http://blog.csdn.net/ly890700/article/details/53104773)
- [Android NDK：为什么arm-linux-androideabi-gdb.exe消失了？](https://stackoverrun.com/cn/q/9975180)
