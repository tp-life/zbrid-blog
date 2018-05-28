---
title: GO并发实战学习笔记 锁
date: 2018-02-06T04:14:13.000Z
lastmod: 2018-02-06T04:14:13.000Z
draft: true
keywords: ["GO","并发","锁"]
description: ''
tags: ["GO","并发","锁"]
categories: []
author: ''
comment: false
toc: true
autoCollapseToc: false
contentCopyright: false
reward: false
mathjax: false
style: candy
---

**锁** 是传统并发程序对共享资源访问的主要手段，GO语言锁相关的API主要有两个：互斥锁、读写锁。

<!--more-->

## 互斥锁
> 顾名思义：互斥的概念表示两个相对排斥，同时只有一个进行。于此来保证并发安全性。在使用锁过后，必须进行及时性的解锁，以免程序造成阻塞。

GO语言中互斥锁由标准库：`sync` 中的 `Mutex` 结构体进行表示。`sync.Mutex` 类型只有两个公开的指针方法——`Lock`和`Unlock`。
`sync.Mutex`结构的零值便是未被锁定的互斥量。对其声明如下：
```golang
var mutex sync.Mutex
```
对锁的使用，必须在锁定之后执行完相关代码后对其进行解锁。也就是说，加锁与解锁总是成对出现的。
```golang
var mutex sync.Mutex

func write(){
	mutex.Lock() //加锁
	defer mutex.Unlock() //解除锁定
}
```
思考一下一下代码：
```golang
package main

import (
		"sync"
		"fmt"
	)

func main(){
	var mutex sync.Mutex
	mutex.Lock()
	fmt.Println("test")
	mutex.Lock()
	mutex.Unlock()
	mutex.Unlock()
}

//panic()
```
上面的代码讲触发一个panic！**_重复的加锁与重复的解锁都将触发一个不可逆的panic_**。因此，在使用过程中，强烈建议：**将锁放在同一代码块里面使用**。

## 读写锁

> 读写锁其实就是针对雨读与写的互斥锁。它与互斥锁的区别在于：互斥锁是所有锁之间都是互斥的；而读写锁是分别对读操作和写操作进行锁定与解锁的操作。读写锁之间多个写操作之间是互斥的，并且读操作与写操作也是互斥的，但是读操作之间却不存在互斥的关系。这可以有效的避免多个读之间性能的消耗。

读写锁由结构体`sync.RWMutex`表示。此类型包含两对方法：
```golang
//写操作
func (*RWMutex) Lock()
func (*RWMutex) Unlock()

//读操作
func (*RWMutex) RLock()
func (*RWMutex) RUnlock()
```
写解锁会唤醒所有因欲进行读锁定而阻塞的goroutine，而读解锁只有在无任何读锁定的情况下唤醒一个因写锁定的goroutine。同样，对一个未锁定的读锁定或者写锁定进行解锁都将产生一个不可恢复的panic。

关于读写锁的特性，来看一个例子：
```golang
package main
import (
	"fmt"
	"sync"
	"time"
)

func main(){
	var rwm sync.RWMutex
	for i:=0;i<3;i++{
		go func(i int){
			fmt.Printf("Try to lock for reading...[%d]\n",i)
			rwm.RLock()
			fmt.Printf("Locked for reading.[%d]\n",i)
			time.Sleep(2 * time.Second)
			fmt.Printf("Try to unlock for reading...[%d]\n",i)
			rwm.RUnlock()
			fmt.Printf("Unlocked for reading.[%d]\n",i)
		}(i)
	}

	time.Sleep(time.Millisecond * 100)
	fmt.Printf("Try to lock for writing...")
	rwm.Lock()
	fmt.Println("locked for writing .")
}

```
输出：
```
Try to lock for reading....[1]
Try to lock for reading....[2]
Locked for reading .[1]
Try to lock for reading....[0]
Locked for reading .[2]
Locked for reading .[0]
Try to lock for writing....
Try to unlock for reading....[1]
Try to unlock for reading....[2]
Try to unlock for reading....[0]
Unlocked for reading .[1]
Unlocked for reading .[2]
Unlocked for reading .[0]
Locked for writing.
```

从输出可以便可以看出读锁与写锁之间的关系了。写锁在经过2s之后，go函数中的读锁都解锁之后，main函数中的写操作函数才会成功。


除了上述两个方法，`sync.REMutex`还有一个指针方法——`RLocker`，该方法调用会返回一个`sync.Locker`接口类型的值。sync.Locker 接口包含两个方法：Locker和Unlock。其实`*sync.Mutex` 和 `*sync.RWMutex`都是该类型的实现。而在调用读写锁RLocker之后，得到的结果本身就是读写锁本身，只不过调用这个结果值的Lock方法和Unlock方法对应的是读写锁的Rlock和RUnlock。这样做的实际意义在于，可以使我们在以后以相同的方式操作该读写锁中的写锁与读锁。


## 完整实例
```golang
package sync   
import (
 "os"
 "sync" 
 "errors" 
 "io" )

//用于表示数据的类型 
type Data []byte   

//用于表示数据文件的接口类型 
type DataFile interface {
   //读取一个数据块
  Read() (rsn int64, d Data, err error)
   //写入一个数据块
  Write(d Data) (wsn int64, err error)
   //获取最后读取的数据块序列
  RSN() int64
  //获取最后写入的数据块序列
  WSN() int64
  //获取数据块长度
  DataLen() uint32
  //关闭数据文件
  Close() error 
}

//用于表示数据文件的实现类型 
type myDataFile struct {
  f *os.File //文件
  fmutex sync.RWMutex //用于文件的读写锁
  woffset int64 //写操作需要用到的偏移量
  roffset int64 //读操作需要用到的偏移量
  wmutex sync.Mutex //写操作需要用到的互斥锁  更新woffset
  rmutex sync.Mutex //读操作需要用到的互斥锁  更新roffset
  datalen uint32 // 数据块长度 
}

//新建一个数据文件的实例 
func NewDataFile(path string, datalen uint32) (DataFile, error) {
   f, err := os.Create(path)
   if err != nil {
      return nil, err
  }
   if datalen == 0 {
      return nil, errors.New("Invaild data length")
   }
   df := &myDataFile{f: f, datalen: datalen}
   return df, nil 
}

//该版本Read 无法解决read goroutine大于write goroutine时，导致数据漏读的情况。
//roffset 大于 woffset时 d.f.ReadAt 返回io.EOF，但此时roffset的值已经增加了
//func (df *myDataFile) Read() (int64, Data, error) { 
// var offset int64 
// df.rmutex.Lock() 
// offset = df.roffset 
// df.roffset += int64(offset) 
// df.rmutex.Unlock()
 
//读取一个数据块 
// rsn := offset / int64(df.datalen) 
// df.fmutex.RLock() 
// defer df.fmutex.RUnlock() 
// bytes := make([]byte, df.datalen) 
// _, err := df.f.ReadAt(bytes, rsn) 
// if err != nil { 
//    return offset, nil, err 
// } 
// return offset, bytes, nil 
//}   

//第二版 更优化的方案在后期给出 
func (df *myDataFile) Read() (int64, Data, error) {
  var offset int64
  df.rmutex.Lock()
  offset = df.roffset
  df.roffset += int64(offset)
  df.rmutex.Unlock()

   //读取一个数据块
  rsn := offset / int64(df.datalen)
   bytes := make([]byte, df.datalen)
   for {
	//在for循环中进行读取数据，确保一定读取到数据为止，
	// 重复使用df.fmutex.RUnlock目的在于，每次循环比较解锁后，写操作方能成功，否则会一直因为未解锁而阻塞  df.fmutex.RLock()
      _, err := df.f.ReadAt(bytes, rsn)
      if err != nil {
         if err == io.EOF {
            df.fmutex.RUnlock()
            continue
  }
         df.fmutex.RUnlock()
         return offset, nil, err
  }
      df.fmutex.RUnlock()
      return offset, bytes, nil
  }

}

func (df *myDataFile) Write(d Data) (wsn int64, err error) {
   //读取并更新偏移量
  var offset int64
  df.wmutex.Lock()
  offset = df.woffset
  df.woffset += int64(offset)
  df.wmutex.Unlock()

   //写入数据
  wsn = offset / int64(df.datalen)
  var bytes []byte
  if len(d) > int(df.datalen) {
      bytes = d[0:df.datalen]
   } else {
      bytes = d
  }
   df.fmutex.Lock()
   defer df.fmutex.Unlock()
   _, err = df.f.Write(bytes)
   return   
}

func (df *myDataFile) RSN() int64 {
   df.rmutex.Lock()
   defer df.rmutex.Unlock()
   return df.roffset / int64(df.datalen)
}

func (df *myDataFile) WSN() int64 {
   df.wmutex.Lock()
   defer df.wmutex.Unlock()
   return df.woffset / int64(df.datalen)
}

func (df *myDataFile) DataLen() uint32 {
   return df.datalen }

func (df *myDataFile) Close() error {
   return df.f.Close()
}
```


