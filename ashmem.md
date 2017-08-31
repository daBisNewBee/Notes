# 如何基于Android匿名共享内存(Ashmem)实现多进程安全(互斥)
## 问题/需求
### 实例描述：
![](https://github.com/daBisNewBee/Notes/blob/master/pic/ashmem.jpg)
- 假设一个完整的操作Operation由操作A、B、C、D组成。A、B、C、D子操作自带互斥锁，在一个完整的Opea之间，也带有多线程互斥锁。A、B、C、D操作对象为底层硬件设备（类似单片机），只支持单进程单线程下工作。且操作具有记忆，比如单个业务在进行到B操作时，另一个请求从A开始，则会打断原有B的操作，使得原有业务无法继续。
- 图1，展示了单个进程com.ash工作在单线程/多线程模式下，由于子操作ABCD自带互斥锁，所以保证了在一个Opea中间，A、B、C、D能顺序运行；又有多线程互斥锁，保证了即使工作在多线程下，各个Opera操作也能完整独立运行。
- 图2，问题来了。除了进程com.ash，还有一个进程com.ash:service也会进行Opera操作。且service进程发起A操作时，com.ash进程正在执行B操作，由于B带有互斥锁，因此service进程的A操作会等待，直到B操作结束，A开始执行。由于原有设备的操作现场为B，而A操作又需要操作现场A的支持，因此service进程A的乱入导致了原有操作现场B的破坏，service进程也无法继续执行。
- 图3,问题解决了。在图2的基础上，增加了Opera的互斥锁，该锁基于共享内存，因此在多个进程进行ABCD操作时，会判断互斥锁的状态，由于该锁基于共享内存，因此能准确描述Opera的运行状态。service进程中Opera操作使用互斥锁保护，同样的，当service进程A操作请求资源时，发现共享内存互斥锁也被占用，com.ash进程虽然正在执行B，但会直到D操作完，才会释放锁，因此service进程会等待D的结束才开始自己的A操作。当service进程的Opera执行完释放锁，com.ash进程又能继续Opera的执行。
### 问题总结
- 底层硬件设备只能对细粒度的操作互斥，且多为单进程、单线程操作。比如A、B、C、D相互之间互斥，但是Opera不互斥。
- 当多个请求同时访问硬件设备时，易影响硬件设备原有正在进行的正常服务。比如在执行到B时，另一进程进入A操作，此时B的现场被破坏，两个进程都无法继续执行。

### 使用共享内存的互斥锁的好处
- 支持更高粒度的互斥，保护一个完整的业务流程。比如可以基于Opera操作做互斥，防止ABCD流程被中断，保证ABCD流程的完整执行。
- 扩大了硬件资源服务的形式、范围。比如Opera可以支持单进程多线程，又能支持多进程操作。支持了多个进程同时对硬件资源的并发访问。

## Linux下的mmap用法
- 对文件内容的映射、改写，见`mmap_data.c`。
> 将文件映射进内存，对内存的改写即为对文件的改写

- 多进程（有亲缘关系）基于共享内存的通信，见`mmap_addr.c、child_parent_mutex_anony.c`
> 有亲缘关系：可使用匿名fd，不用再打开文件设备
- 多进程（无亲缘关系）基于共享内存的通信，见`child_parent_mutex_fopen.c`
> 需要fopen指定具体文件设备，来作为共享内存的周转
- 多进程(有/无亲缘关系)的互斥吗，见`multi_process_a.c、multi_process_b.c`。
> 第二个进程无需再初始化互斥锁的属性，否则共享失败

## Ashmem(Anonymous Shared Memory)的特性（之与Linux下mmap的差异）
- 共享目录必须为：`/dev/ashmem`
- 多进程必须（获得共享方）通过Binder（AIDL）向被共享方（首次开辟共享目录者）获得`ParcelFileDescriptor`，mmap后才可正常使用Ashmem。（否则，多进程中若直接fopen获得fd，可mmap出内存，但和另一个进程没有实现共享）

## Ashmem用法
- 主进程中，获得Ashmem的fd
```
jint fd = open("/dev/ashmem",O_RDWR);
ioctl(fd,ASHMEM_SET_NAME,"MyAshmemName");
ioctl(fd,ASHMEM_SET_SIZE,4096);
// 系统调用ashmem_create_region对上述操作做了封装
```
- 主进程中，可直接对上述`fd`mmap后，进行内存操作
```
int *map = (int *)mmap(0,4096,PROT_READ|PROT_WRITE,MAP_SHARED,fd,0);
map[0]=99;
map[10]=88;
```
- 其他进程中，需要通过RPC从主进程获得`fd2`,即fd2 = F_rpc("MyAshmemName")，同样，接下来就可mmap出内存操作
```
int *map = (int *)mmap(0,4096,PROT_READ|PROT_WRITE,MAP_SHARED,fd2,0);
LOGD("Mapped data.map[0]:%d	map[10]:%d",map[0],map[10]);
// Mapped data.map[0]:99	map[10]:88
```
- 具体`F_rpc("MyAshmemName")`的过程，可参考[URL](http://www.discoversdk.com/blog/android-shared-memory-ashmem-example)

## 基于Ashmem的多进程互斥实例
- 见本地项目

### 实际生产环境验证

## 参考
- [Android Shared Memory](http://www.discoversdk.com/blog/android-shared-memory-ashmem-example)
- [android共享内存](https://www.bbsmax.com/A/kPzORyM3dx/)
- [Linux下的多进程间共享资源的互斥访问](http://blog.csdn.net/dlutbrucezhang/article/details/8834387)
- [How to use Shared Memory (IPC) in Android
](https://stackoverflow.com/questions/16099904/how-to-use-shared-memory-ipc-in-android)
- [理解mmap](http://www.knowsky.com/1050140.html)
- [Android Binder 分析——匿名共享内存（Ashmem）](http://light3moon.com/2015/01/28/Android%20Binder%20%E5%88%86%E6%9E%90%E2%80%94%E2%80%94%E5%8C%BF%E5%90%8D%E5%85%B1%E4%BA%AB%E5%86%85%E5%AD%98[Ashmem]/#原理概述)
-[Linux内存映射（mmap）  ](http://hubingforever.blog.163.com/blog/static/171040579201246113243361/)
-[android mmap的使用](http://blog.csdn.net/kanwah200/article/details/40047211)




