## 一次APP内存泄露的解决记录

### 内存泄露 OOM（out of memory）
- 特征：**可达，无用**
    - 可达
    - 无用：创建了但是不再使用之后没有释放。
    - 能重用但是却创建了新的对象
- 本质：GC只能回收垃圾对象、软引用、弱引用，不能回收强引用的对象。当需要分配的堆内存大于系统分配的剩余可用堆内存时，便会报错OOM。
- 解释：有个引用指向一个不再被使用的对象，导致该对象无法被GC（垃圾回收器）回收。如果这个对象有个引用指向一个包括很多其他对象的集合，就会导致这些对象都不会被垃圾回收。

### 问题还原
- log
```
V/kl.cds  (  689): ############################　setTypeface
D/LockPatternUtils(  689): file pathcds.log
E/filemap (  689): mmap(0,11252549) failed: Out of memory
W/asset   (  689): create map from entry failed
E/kl.cds  (  689): 加载第三方字体失败。
D/LockPatternUtils(  689): file pathcds.log
W/libc    (  689): pthread_create failed: couldn't allocate 1064960-byte stack: Out of memory
E/art     (  689): Throwing OutOfMemoryError "pthread_create (1040KB stack) failed: Try again"
D/AndroidRuntime(  689): Shutting down VM
E/AndroidRuntime(  689): FATAL EXCEPTION: main
E/AndroidRuntime(  689): Process: kl.cds, PID: 689
E/AndroidRuntime(  689): java.lang.OutOfMemoryError: pthread_create (1040KB stack) failed: Try again
E/AndroidRuntime(  689): 	at java.lang.Thread.nativeCreate(Native Method)
E/AndroidRuntime(  689): 	at java.lang.Thread.start(Thread.java:1063)
```

- code
```
public class FontTextView extends TextView{

    private static Typeface TEXT_TYPE = null;

    public FontTextView(Context context){
        super(context);
    }

    public FontTextView(Context context, AttributeSet attrs){
        super(context,attrs);
        setTypeface(context);
    }

    public FontTextView(Context context, AttributeSet attrs,int defStyle){
        super(context,attrs,defStyle);
        setTypeface(context);
    }

    private void setTypeface(Context context){
        // 如果自定义typeface初始化失败，就用原生的typeface
        try {
            LogHelper.v("############################　setTypeface");
            TEXT_TYPE = Typeface.createFromAsset(context.getAssets(), "fonts/wqwmh.ttc");
        } catch (Exception e) {
            LogHelper.e( "加载第三方字体失败。");
            TEXT_TYPE = getTypeface();
        }
        setTypeface(TEXT_TYPE) ;
    }
}

```
- fix
```
    private static Typeface TEXT_TYPE = null;
    private void setTypeface(Context context){
        // 如果自定义typeface初始化失败，就用原生的typeface
        try {
            if (TEXT_TYPE == null){
                TEXT_TYPE = Typeface.createFromAsset(context.getAssets(), "fonts/wqwmh.ttc");
            }
        } catch (Exception e) {
            LogHelper.e( "加载第三方字体失败。");
            TEXT_TYPE = getTypeface();
        }
        setTypeface(TEXT_TYPE) ;
    }
```


### 使用MAT（Memory Analysis Tool）分析Java堆
1. 在AS中Dump Java堆内存文件
> 点击“Dump Java Heap”，获得kl.cds_2017.09.18_20.14.hprof
2. 堆转储hprof,转成标准、MAT可识别的dump文件
> hprof-conv kl.cds_2017.09.18_20.14.hprof standard.hprof
3. 在Eclipse中安装MAT
> http://www.eclipse.org/mat/downloads.php
4. standard.hprof导入MAT

### MAT常用的几个步骤
- list objects -> with incoming references
> 显示该对象**被**哪些对象持有。（如果一个类有很多不需要的实例，那么可以找到哪些对象持有该对象，让这个对象没法被回收）
- Path to GC Roots -> exclude weak/soft references
> 显示选中对象到GC根节点的引用路径，排除了弱/软引用。显示最终的引用关系
- adb shell dumpsys meminfo kl.cds
> 显示kl.cds的内存占用情况,详细说明见[调查 RAM 使用情况](https://developer.android.com/studio/profile/investigate-ram.html#ViewingAllocations)。

### 几个泄露的例子
- 静态属性持有非静态内部类的实例
```
public class MainActivity extends Activity {
  //静态属性持有非静态内部类的实例--这么做非常糟糕
  static MyLeakedClass leakInstance = null;
 
  @Override
  public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
 
    // Static field initialization
    if (leakInstance == null)
      leakInstance = new MyLeakedClass();
 
    ImageView mView = new ImageView(this);
    mView.setBackgroundResource(R.drawable.leak_background);
 
    setContentView(mView);
  }
 
  /*
   *非静态内部类
   */
  class MyLeakedClass {
    int someInt;
  }
}
```
- 将Activity作为application context传递给一个单例模式的类
```
public class MainActivity2 extends Activity {
  SingletonClass mSingletonClass = null;
 
  @Override
  public void onCreate(Bundle savedInstanceState) {
    super.onCreate(savedInstanceState);
    mSingletonClass = SingletonClass.getInstance(this);
  }
}
 
class SingletonClass {
  private Context mContext = null;
  private static SingletonClass mInstance;
 
  private SingletonClass(Context context) {
    mContext = context;
  }
 
  public static SingletonClass getInstance(Context context) {
    if (mInstance == null) {
      mInstance = new SingletonClass(context);
    }
    return mInstance;
  }
}
```


### 相关概念


### 参考
- [调查 RAM 使用情况](https://developer.android.com/studio/profile/investigate-ram.html#ViewingAllocations)
- [管理App内存](http://www.jianshu.com/p/d061fa36a0d9)
- [MAT使用进阶](http://www.jianshu.com/p/c8e0f8748ac0)  
- [Android 内存剖析 – 发现潜在问题](http://www.importnew.com/2433.html)
