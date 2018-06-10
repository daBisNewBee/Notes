# 动手写一个O(1)的单元素LRUCache

> Least Recently Used (LRU) cache.

## 需求
- 需要从候选者中选出一个有效对象，候选者包括A、B、C、D、E、F、G...，候选者无排序
- 判断对象有效是一个耗时操作，每次都遍历所有对象时，耗时更长
- 伪代码实现如下：

```
Object[] candidates = new Object[]{A、B、C、D、E、F、G};
for(int i = 0; i< candidates.size(); i++){
	if(isValid(candidates[i]))
		return candidates[i];
	else
		continue;
}

```

## 背景
- **LRUCache** 实现已经添加到Android SDK中：android.util.LRUCache
- 一个常规的LRUCache用法可以参考[LruCache](https://blog.csdn.net/weixin_40290793/article/details/78780767)
- 核心的操作包括：
1. 在初始化时指定缓存大小：
	
```
 
 //获取到最大可用的内存空间
        int maxSize = (int)Runtime.getRuntime().maxMemory()/8;//一般用除以八来表示，具体视APP大小而定
        mLrucache = new LruCache<String,Bitmap>(maxSize){
            @Override
            protected int sizeOf(String key, Bitmap value) {
                return value.getByteCount();
            }
        };
```

2. 在map.put存储对象至缓存中，检查若占用大于指定大小，进行trimToSize

```
public void trimToSize(int maxSize) {
        //死循环
        while (true) {
            K key;
            V value;
            synchronized (this) {
                //如果map为空并且缓存size不等于0或者缓存size小于0，抛出异常
                if (size < 0 || (map.isEmpty() && size != 0)) {
                    throw new IllegalStateException(getClass().getName()
                            + ".sizeOf() is reporting inconsistent results!");
                }
                //如果缓存大小size小于最大缓存，或者map为空，不需要再删除缓存对象，跳出循环
                if (size <= maxSize || map.isEmpty()) {
                    break;
                }
                //迭代器获取第一个对象，即队尾的元素，近期最少访问的元素
                Map.Entry<K, V> toEvict = map.entrySet().iterator().next();
                key = toEvict.getKey();
                value = toEvict.getValue();
                //删除该对象，并更新缓存大小
                map.remove(key);
                size -= safeSizeOf(key, value);
                evictionCount++;
            }
            entryRemoved(true, key, value, null);
        }
    }

```

- 从上不难看出：
	- 指定缓存大小，是为了缓存中占用增加时，用于判断删除无用对象；
	- LinkedHashMap中的双向链表是为了维护缓存的中的数据为最年轻（这里年轻的定义是：最近访问的对象）
	- 在缓存中留存年轻对象，是为了取代代价更高的磁盘、网络IO操作
- 所以现在的问题是
	- 如果想获取，最近一次访问的对象，该怎么办？
	- 又对象为单元素，无key对应时，该如何获取？ 

## 实现
> 为了解决上述问题，这里提出一个**LRUCacheList**。

> 主要思路是：

> 1. 在遍历到某一个候选者后，就不再遍历时，认为当前取到的候选者就是有效的候选者。
> 2. 每次获取对象时，将当前候选者取出，并放到当前数据结构的第一个
> 3. 下次遍历时，直接从有效候选者开始遍历

**实现可参考[LRUCacheList](https://github.com/daBisNewBee/JavaProject/blob/master/src/algorithm/LRUCacheList.java)，几点注意如下：**

- 内部数据结构使用双向链表，可快速查找到当前节点的前、后节点，在进行删除操作时只需要O(1)的time
- LinkedList就基于双向链表实现，因此**LRUCacheList**基于 LinkedList来实现
- 查找数据时，仍是需要O(N/2)的time，原因：

```
    /**
     * Returns the (non-null) Node at the specified element index.
     */
    Node<E> node(int index) {
        // assert isElementIndex(index);

        if (index < (size >> 1)) {
            Node<E> x = first;
            for (int i = 0; i < index; i++)
                x = x.next;
            return x;
        } else {
            Node<E> x = last;
            for (int i = size - 1; i > index; i--)
                x = x.prev;
            return x;
        }
    }
```


## LRUCacheList 与 LRUCache 的 比较

- 获取元素的方式不同。
	- （其实都为key-value，不过index是特殊的key）
	- LRUCache 为双元素缓存，每次必须根据key才能取到为value的元素
	- LRUCacheList 为单元素缓存，可从index = 0处开始遍历
	- LRUCache 取到的元素是key对应值，不是最近访问的值（即：其中维护的双向链表未发挥作用）
 
- 目的（作用）不同。
	- LRUCache 为了提供内存使用率，减少IO。在指定大小的内存中保留被访问最频繁的数据，减少APP从网络或者物理磁盘中IO
	- LRUCacheList 为了每次取到最近一次访问的值；
 
- 底层数据结构不同。
	- "LRUCache"，基于LinkedHashMap；
	- "LRUCacheList" 基于LinkedList

## 应用
- 适用的应用场景有如下特点：
	- 常用于需要从候选者队列中选出一名有效候选者；
	- 判断单个有效候选者过程耗时；
	- 每次重复以上操作
- 实际应用：
	- 多服务器链路选择
	- 当前使用的手机TF加密卡厂商类型的判断 
