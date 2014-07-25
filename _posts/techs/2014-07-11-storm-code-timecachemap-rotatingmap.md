---
layout: post
title:  Storm源码-TimeCacheMap & RotatingMap
---

在之前的[常见模式](/storm-docs/common-patterns)中，讲到了Storm有一个数据结构叫做TimeCacheMap，可以保存最近一段时间的数据。当时看名字自然的想到用HashMap的方式来实现,后台加一个线程将过期的数据删除。

下面看看高人是怎么写代码的。

代码的地址在[这里](https://github.com/apache/incubator-storm/blob/master/storm-core/src/jvm/backtype/storm/utils/TimeCacheMap.java)。


#TimeCacheMap

## 代码
类的注释是这样的：

    /**
    * Expires keys that have not been updated in the configured number of seconds.
    * The algorithm used will take between expirationSecs and
    * expirationSecs * (1 + 1 / (numBuckets-1)) to actually expire the message.
    *
    * get, put, remove, containsKey, and size take O(numBuckets) time to run.
    *
    * The advantage of this design is that the expiration thread only locks the object
    * for O(1) time, meaning the object is essentially always available for gets/puts.
    */

主要实现是这样：将所有的数据分为n个桶，每个桶是一个HashMap<K,V>。假设过期时间是s，后台有一个线程每隔s/(n-1)删掉最后一个桶，并在开头放进一个新桶。增删查改的算法分别如下：

+ get 从第一个桶读取，没有往后走
+ put 写到第一个桶，并在其他桶中删除
+ remove 在每个桶中删除
+ containsKey 在每个桶中查找
+ size 遍历

上述接口在实现的时候都要加锁：

    public V get(K key) {
        synchronized(_lock) {
            for(HashMap<K, V> bucket: _buckets) {
                if(bucket.containsKey(key)) {
                    return bucket.get(key);
                }
            }
            return null;
        }
    }


一个kv添加进来后过期时间是(s,s+s/(n-1))。为什么是有一个区间，是因为后台clean线程最快在一个kv被添加后立即运行，最慢要等待s/(n-1)时间。

后台清理线程：

    _cleaner = new Thread(new Runnable() {
            public void run() {
                try {
                    while(true) {
                        Map<K, V> dead = null;
                        Time.sleep(sleepTime);
                        synchronized(_lock) {
                            dead = _buckets.removeLast();
                            _buckets.addFirst(new HashMap<K, V>());
                        }
                        if(_callback!=null) {
                            for(Entry<K, V> entry: dead.entrySet()) {
                                _callback.expire(entry.getKey(), entry.getValue());
                            }
                        }
                    }
                } catch (InterruptedException ex) {

                }
            }
        });
        _cleaner.setDaemon(true);
        _cleaner.start();


这边要说一下这里为什么是间隔s/(n-1)，而不是s/n。因为后者的过期时间是(s-s/n, s)。

## UT

奇怪没有找到这个类的ut。

# RotatingMap

这个类的主体实现和TimeCacheMap一样，除了没有后台定时清理线程。然后暴露了一个rotate接口，用以删除一个桶。

    public Map<K, V> rotate() {
        Map<K, V> dead = _buckets.removeLast();
        _buckets.addFirst(new HashMap<K, V>());
        if(_callback!=null) {
            for(Entry<K, V> entry: dead.entrySet()) {
                _callback.expire(entry.getKey(), entry.getValue());
            }
        }
        return dead;
    }

这样控制桶的到期就交给外部使用者控制了。