---
title: golang内存缓存库的调研
date: 2021-05-23 04:13:05
tags:
---

它山之石可以攻玉。

自己写某个SDK时，发生了莫名其妙的问题：从内存缓存（一个成员变量）读取数据，偶尔读不到数据。

排查半天看不出什么问题，怀疑是间歇性更新缓存时，出现短暂的数据不可读问题。代码层面找不到任何问题，所以来看看其他缓存库是如何实现的。

## 我遇到的问题

client异步地更新缓存，上层应用永远读取缓存数据。理论上有读写锁的保护，不可能获取不到数据。

实在是费解啊。

```golang
package cache

import (
	"log"
	"sync"
	"time"
)

type client struct {
	cache map[string]string
	lock  sync.RWMutex
}

func (c *client) refresh() {
    // 经过一些复杂的处理逻辑，更新cache
	time.Sleep(time.Microsecond * 100)
	c.lock.Lock()
	c.cache = map[string]string{
		"key": time.Now().String(),
	}
	c.lock.Unlock()
}

func (c *client) get(key string) string {
	c.lock.RLock()
	cache := c.cache
	c.lock.RUnlock()
	return cache[key]
}

func newClient() *client {
	c := &client{
		cache: map[string]string{},
		lock:  sync.RWMutex{},
	}
	c.refresh()
	go func() {
		for range time.NewTicker(time.Second * 2).C {
			c.refresh()
		}
	}()
	return c
}

func myCase() {
	c := newClient()
	begin := time.Now()
	for range time.NewTicker(time.Microsecond * 5).C {
		if time.Since(begin) > time.Second*25 {
			break
		}
		got := c.get("key")
		if got != "" {
			// log.Printf("key got %v", got)
			continue
		}

		log.Fatal("key missed.")// 间歇性的获取不到配置 很离谱
	}
}

```

## patrickmn/go-cache

* https://github.com/patrickmn/go-cache 一个5K星，但是2019年开始没有任何维护的库。

### 实现细节

这个库的原理非常简单，实现细节只有1k行。内部维护一个map+一个rw锁，其他的任何操作都是基于这两个变量的:
* 旁路goroutine，周期性的遍历整个storage，依次驱逐过期的key
* 注册对象gc回调runtime.SetFinalizer，主client被gc时，自动把旁路goroutine停掉
* 驱逐缓存时，提供callback执行毁掉逻辑

**所以说，一款基础功能的内存cache，并不需要多复杂。即使只有1k行代码，也可以有大批量用户**

![](/pics/patrickmn_go_cache.png)

### 评判一番

读起来确实易懂，没有花里胡哨的机制，用起来非常放心。

但是rwlock+map，效率必然不高：
* map容量较大时，如果触发扩容、缩容，都会影响整体效率
* 全局锁过于粗放，读、写操作的不均匀，有可能引起操作阻塞

## allegro/bigcache

## coocood/freecache

## golang/groupcache

## dgraph-io/ristretto