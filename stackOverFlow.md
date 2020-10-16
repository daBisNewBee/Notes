## 为什么64位APP时更容易“StackOverflowError”？

#### 问题描述
```
案例1：
10-15 16:15:10.280 17885 18745 W System.err: java.io.IOException: java.lang.StackOverflowError: stack size 1037KB  URL:http://upload.ximalaya.com/clamper-server/mkblk/
10-15 16:15:10.289 17885 18745 W System.err: 	at com.ximalaya.ting.android.opensdk.httputil.g.a(HttpDNSInterceptor.java:209)
10-15 16:15:10.290 17885 18745 W**** System.err: 	at okhttp3.internal.c.g.a(RealInterceptorChain.java:147)
10-15 16:15:10.290 17885 18745 W System.err: 	at okhttp3.internal.c.g.a(RealInterceptorChain.java:121)
10-15 16:15:10.290 17885 18745 W System.err: 	at com.ximalaya.ting.android.opensdk.httputil.m.a(NetExceptionHandlerInterceptor.java:27)
10-15 16:15:10.290 17885 18745 W System.err: 	at okhttp3.internal.c.g.a(RealInterceptorChain.java:147)
10-15 16:15:10.290 17885 18745 W System.err: 	at okhttp3.internal.c.g.a(RealInterceptorChain.java:121)
10-15 16:15:10.290 17885 18745 W System.err: 	at okhttp3.ad.l(RealCall.java:254)
10-15 16:15:10.290 17885 18745 W System.err: 	at okhttp3.ad.b(RealCall.java:92)
10-15 16:15:10.290 17885 18745 W System.err: 	at com.ximalaya.ting.android.upload.c.h.a(UploadClient.java:309)
10-15 16:15:10.290 17885 18745 W System.err: 	at com.ximalaya.ting.android.upload.c.h.a(UploadClient.java:278)
10-15 16:15:10.290 17885 18745 W System.err: 	at com.ximalaya.ting.android.upload.g.b(ResumeUploader.java:293)

问题：
1. 为什么32位os正常，换到64位就StackOverflowError了？
2. 为什么StackOverflowError时，stack size 是1037KB？

案例2：
    private int count = 0;

    void func() {
        count++;
        try {
            func();
        } catch (StackOverflowError e) {
            System.out.println("count = " + count + " e = " + e.getLocalizedMessage());
        }
    }
    
    ex: 
    64位/32位主线程时：
    System.out: count = 57372 / 111745 e = stack size 8MB / 8MB
    64位/32位主线程时：
    System.out: count = 1226 / 1943 e = stack size 1037KB / 1036KB
    
```

|count |32位 os | 64位 os|
|---|---|---|
| 主线程 | 111745 | 57372 |
| 工作线程 | 1943 | 1226 |

#### 结论
- 64位栈深度比32位小很多，近一半
- 主线程栈深度远大于工作线程

#### 原因

- 32位os与64位os的(函数)指针占用大小不同

```
void MyFunc(){
}

void (*P_FUNC)(void);

test() {
	P_FUNC = MyFunc;
	LOGD("local pointer P_FUNC:%d", sizeof(P_FUNC));
	// 32位：4个字节    64位：8个字节
}
```


- 主线程与工作线程栈大小不同：主线程 8MB，工作线程 1MB不到
	- 虚拟机栈
		- 服务于Java方法
		- Java虚拟机中通过Xss设置，Android虚拟机未提供参数设置
		- 根据**StackOverflowError**日志size猜测：**main下为8MB，work下为1MB不到**(1038KB)
		- adb shell ulimit -s : 得到 "8192". 得到的是主线程栈大小8MB
	- 本地方法栈 
		- 服务端于本地方法
		- 可通过api设置
		- 常见work的stackSIze为**1MB不到**(实际不同Android版本不同，有几KB的区别，见[Android线程栈大小](https://zhuanlan.zhihu.com/p/33562383))
	- 在maps中验证栈大小：

```
cat /proc/$pid/maps |grep stack
7fafb02000-7fafbfd000 rw-p 00000000 00:00 0          [stack:31573]
计算大小为：
7fafbfd000 - 7fafb02000 = 0xFB000 = 1028096 = 1004KB （工作线程栈）

7fc546f000-7fc5c6e000 rw-p 00000000 00:00 0          [stack]
计算大小为：
7fc5c6e000  - 7fc546f000 = 0x7FF000 = 8384512 = 8188KB (主线程栈)
```


#### 其他
- 本地获取线程栈大小：

```
    size_t stacksize;
    pthread_attr_t thread_attr;
    pthread_attr_init(&thread_attr);
    int ret =  pthread_attr_getstacksize(&thread_attr,&stacksize);
    if(ret != 0){
       LOGE("error:%d",ret);
       return;
    }
    LOGD("stacksize:%d", stacksize); 
    // 32位：1040384（32） = 1016KB 
    // 64位：1032192（64） = 1008KB
    
 初始化栈大小过程：
 platform_bionic/libc/bionic/pthread_attr.cpp：
 
 int pthread_attr_init(pthread_attr_t* attr) {
  attr->flags = 0;
  attr->stack_base = nullptr;
  attr->stack_size = PTHREAD_STACK_SIZE_DEFAULT;
  attr->guard_size = PTHREAD_GUARD_SIZE;
  attr->sched_policy = SCHED_NORMAL;
  attr->sched_priority = 0;
  return 0;
}

#if defined(__LP64__)
#define SIGNAL_STACK_SIZE_WITHOUT_GUARD (32 * 1024)
#else
#define SIGNAL_STACK_SIZE_WITHOUT_GUARD (16 * 1024)
#endif

#define PTHREAD_STACK_SIZE_DEFAULT ((1 * 1024 * 1024) - SIGNAL_STACK_SIZE_WITHOUT_GUARD)
```
- [一个有意思的栈溢出crash](https://xionghengheng.github.io/2019/01/06/%E4%B8%80%E4%B8%AA%E6%9C%89%E6%84%8F%E6%80%9D%E7%9A%84%E6%A0%88%E6%BA%A2%E5%87%BAcrash/)

#### 总结
- 64位下，避免过深的递归的调用，或者尽量使用循环代替递归
- 64位下，避免过大、过多的局部变量
- C/C++开发时，扩大线程栈的默认大小(默认1MB不到)




