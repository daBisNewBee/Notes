### 记一次multiDex下的“java.lang.NoClassDefFoundError”的解决过程
#### 现象：
- 报错：

```
E/dalvikvm(25256): Could not find class 'com.ximalaya.ting.android.host.util.timelogger.TimeNoteData', referenced from method com.ximalaya.ting.android.host.util.timelogger.AppStartUpTimeLogger.printLog
E/AndroidRuntime(25256): FATAL EXCEPTION: main
E/AndroidRuntime(25256): Process: com.ximalaya.ting.android, PID: 25256
E/AndroidRuntime(25256): java.lang.RuntimeException: Unable to instantiate application com.ximalaya.ting.android.xmloader.XMApplication: java.lang.RuntimeException: java.lang.reflect.InvocationTargetException
E/AndroidRuntime(25256): 	at android.app.LoadedApk.makeApplication(LoadedApk.java:509)
E/AndroidRuntime(25256): 	at android.app.ActivityThread.handleBindApplication(ActivityThread.java:4413)
E/AndroidRuntime(25256): 	at android.app.ActivityThread.access$1500(ActivityThread.java:141)
E/AndroidRuntime(25256): 	at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1272)
E/AndroidRuntime(25256): 	at android.os.Handler.dispatchMessage(Handler.java:102)
E/AndroidRuntime(25256): 	at android.os.Looper.loop(Looper.java:136)
E/AndroidRuntime(25256): 	at android.app.ActivityThread.main(ActivityThread.java:5113)
E/AndroidRuntime(25256): 	at java.lang.reflect.Method.invokeNative(Native Method)
E/AndroidRuntime(25256): 	at java.lang.reflect.Method.invoke(Method.java:515)
E/AndroidRuntime(25256): 	at com.android.internal.os.ZygoteInit$MethodAndArgsCaller.run(ZygoteInit.java:796)
E/AndroidRuntime(25256): 	at com.android.internal.os.ZygoteInit.main(ZygoteInit.java:612)
E/AndroidRuntime(25256): 	at dalvik.system.NativeStart.main(Native Method)
E/AndroidRuntime(25256): Caused by: java.lang.RuntimeException: java.lang.reflect.InvocationTargetException
E/AndroidRuntime(25256): 	at com.ximalaya.ting.android.xmloader.LoaderApplication.attachBaseContext(LoaderApplication.java:71)
E/AndroidRuntime(25256): 	at android.app.Application.attach(Application.java:184)
E/AndroidRuntime(25256): 	at android.app.Instrumentation.newApplication(Instrumentation.java:991)
E/AndroidRuntime(25256): 	at android.app.Instrumentation.newApplication(Instrumentation.java:975)
E/AndroidRuntime(25256): 	at android.app.LoadedApk.makeApplication(LoadedApk.java:504)
E/AndroidRuntime(25256): 	... 11 more
E/AndroidRuntime(25256): Caused by: java.lang.reflect.InvocationTargetException
E/AndroidRuntime(25256): 	at java.lang.reflect.Method.invokeNative(Native Method)
E/AndroidRuntime(25256): 	at java.lang.reflect.Method.invoke(Method.java:515)
E/AndroidRuntime(25256): 	at com.ximalaya.ting.android.xmloader.LoaderApplication.attachBaseContext(LoaderApplication.java:69)
E/AndroidRuntime(25256): 	... 15 more
E/AndroidRuntime(25256): Caused by: java.lang.NoClassDefFoundError: com.ximalaya.ting.android.host.util.timelogger.TimeNoteData
E/AndroidRuntime(25256): 	at com.ximalaya.ting.android.host.util.timelogger.AppStartUpTimeLogger.markStartTime(AppStartUpTimeLogger.java:49)
E/AndroidRuntime(25256): 	at com.ximalaya.ting.android.host.MainApplication.attachBaseContext(MainApplication.java:130)
E/AndroidRuntime(25256): 	... 18 more
W/ActivityManager(  905):   Force finishing activity com.ximalaya.ting.android/.host.activity.WelComeActivity

```

- 主要关注：

E/dalvikvm(25256): Could not find class 'com.ximalaya.ting.android.host.util.timelogger.TimeNoteData', referenced from method 


#### 问题背景
- 运行环境:
	- Android 4.4 
- 类的依赖情况
	- MainApplication 中直接引用了 AppStartUpTimeLogger
	- AppStartUpTimeLogger 中的方法markStartTime 直接引用了 TimeNoteData
	- MainApplication 间接引用了 TimeNoteData
- MultiDex使用情况
	- gradle配置
		- "multiDexEnabled true"
		- multiDexKeepProguard file('multidex-config.pro')
		- compile 'com.android.support:multidex:1.0.1' 
	- 代码初始化
		- MainApplication 如下：

```
public void attachBaseContext(Context base) {
	AppStartUpTimeLogger.markStartTime("MainApplication_attachBaseContext",false);
	....
	MultiDex.install(realApplication);
	
}
```
#### 解决办法
- 在多dex加载之前引用到的新增类(即：间接引用类，区别于直接引用类)，均需要在“分包规则”中指定。
	- 即：在“multidex-config.pro”中添加：
```
-keep class com.ximalaya.ting.android.host.util.timelogger.TimeNoteData
```
- 验证：

```
public void attachBaseContext(Context base) {
	try {
              AppStartUpTimeLogger.markStartTime("MainApplication_attachBaseContext",false);
                    Log.v("bb","555 load FuckA success." + obj);
                }catch (Exception e){
                    Log.v("bb", "555 Exception :" + e.getMessage());
                }
                MultiDex.install(realApplication); 
                /*
                以此为时间点!
                在这之前的调用需要遵循“分包规则”！
                在这之后的调用可以任意!因为已经加载过dex了!
                */
                try {
                    Log.v("bb","666 load AppStartUpTimeLogger.");
                    AppStartUpTimeLogger.markStartTime("MainApplication_attachBaseContext",false);
                    Log.v("bb","666 load AppStartUpTimeLogger success.");
                }catch (Exception e){
                    Log.v("bb", "666 Exception :" + e.getMessage());
                }
	
}
```

#### 原因
- 直接原因
	- AppStartUpTimeLogger class 位于主dex（classes.dex），TimeNoteData 位于第二个dex(classes2.dex)
	- 在对AppStartUpTimeLogger调用时，MultiDex未进行初始化，即默认支持加载主dex中的类文件（Android 5.0的Dalvik限制）
- 根本原因
	- "爆包"：dexOpt时，保存方法数的字段是一个short类型，即容量为65535(64K)
	- multiDex对原dex拆包具有随机性，相互引用的类若不在同一个dex，很有可能出错！

#### 需要注意的地方
- 分包规则：
	- 在keep中直接引用的类，可以不用再分包规则中指定，间接引用类需要指定 
	- 比如：由于指定了“-keep class com.ximalaya.ting.android.host.MainApplication”，因此就不用再指定“AppStartUpTimeLogger”了，因为其直接引用，但是需要指定“TimeNoteData”，因为是间接引用
	- 原因：gradle会自动将直接引用的类添加到分包规则中
	- 分包规则文件最终生成位置：
		- /Users/xmly/gitlab/app/TingMainHost/Application/build/intermediates/multi-dex/and-f5-ocpa/release/maindexlist.txt 
	- **TODO**: 分包规则的生成过程

- 在“Analyze APK”中反编译的dex类中，并不表示都该dex中定义了，其中“斜线”的类只是引用了。

```
"Toggle Show all referenced methods or fields  to show or hide referenced packages, classes, methods, and fields. In the tree view, italicized nodes are references that do not have a definition in the selected DEX file."
```
- JD-GUI,在“open gui”终端下打开时，每次打开jar看到的类不准确，需要重启jd-gui
	- 原因：**TODO**
- MultiDex.install与插件化原理的区别？
	- 相同：都对PathList中的dexElements进行处理：将新增的dex文件路径添加进去
	- 区别：multidex下，调用的是makeDexElements，其中进行了loadDex操作（dexOpt）
- 使用时，"MultiDex.install"与继承"MultiDexApplication"二选一即可
	- 其实MultiDexApplication中调用的也是MultiDex.install

#### 参考
- [Multidex Android DEX手动拆包](https://www.jianshu.com/p/a5d554b614aa)
- [Analyze your build with APK Analyzer](https://developer.android.com/studio/build/apk-analyzer)
- 官方说明：[配置方法数超过 64K 的应用](https://developer.android.google.cn/studio/build/multidex.html#about)
- 对原理解释的比较好：[Android MultiDex实现原理解析](http://allenfeng.com/2016/11/17/principle-analysis-on-multidex/)
- 对“间接引用”问题解释的比较好：[Android拆分与加载Dex的多种方案对比](https://mp.weixin.qq.com/s?__biz=MzAwNDY1ODY2OQ==&mid=207151651&idx=1&sn=9eab282711f4eb2b4daf2fbae5a5ca9a&3rd=MzA3MDU4NTYzMw==&scene=6#rd)
- 对异步加载dex，dexopt解释的比较好：[其实你不知道MultiDex到底有多坑](http://www.cnblogs.com/tonny-li/p/7839306.html)

#### 引申
- multiDex加载dex时间过长，容易导致的主进程ANR：
	- 解决方案：在子进程进行“MultiDex.install”并提供界面，主进程阻塞等待，使用sp文件做同步
