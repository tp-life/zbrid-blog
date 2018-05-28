---
title: "GO并发实战学习笔记 条件变量"
date: 2018-02-11T15:04:08+08:00
lastmod: 2018-02-11T15:04:08+08:00
draft: true
keywords: ["GO","并发","条件变量"]
description: ""
tags: ["GO","并发","条件变量"]
categories: []
author: ""

# You can also close(false) or open(true) something for this content.
# P.S. comment can only be closed
comment: false
toc: true
autoCollapseToc: false
# You can also define another contentCopyright. e.g. contentCopyright: "This is another copyright."
contentCopyright: false
reward: false
mathjax: false
---
> 上一篇讲到了锁，这一篇主要讲条件变量有关的知识。

## 条件变量概念

条件变量与互斥量不同，条件变量的作用并不是保证在同一时刻仅有一个线程访问某一个共享数据，而是在共享数据的状态发生改变时，通知其他因此而被阻塞的线程。条件变量总是与互斥量组合使用。条件变量一个经典的说法便是通常我们所说的：`生产者-消费者问题`。

<!--more-->


与互斥量不同，使用条件变量前，必须创建和初始化。同样，条件变量的初始化必须要保证唯一性。另外条件变量在真正使用之前，还必须要与某一个互斥量进行绑定，这与它的具体操作有关。条件变量的具体操作有以下3种：

**等待通知**：它的意思是阻塞当前线程，直到该条件变量发来的通知。

**单发通知**：它的意思是让该条件变量向至少一个正在等待它通知的线程发送通知，以表示某个共享数据的状态已经发生改变。

**广播通知**：它的意思是让条件变量给正在等待它通知的所有线程发送通知，以表示某个共享数据的状态已经发生改变。


## GO当中的条件变量

具体的条件变量的内容在本篇就不做多的介绍。主要介绍GO当中与条件变量有关的API。

GO标准库中的`sync.Cond`类型代表了条件变量。与锁不同，条件变量的声明需要用到`sync.NewCond`函数，该函数声明如下：
```golang
func NewCond(l locker) *Cond
```
前面已经讲过，条件变量总要与互斥量组合使用。sync.NewCond 函数的唯一参数是 sync.Locker类型的，而具体的参数既可以是互斥锁，也可以是读写锁。sync.NewCond 函数被调用之后会返回一个 *sync.Cond 类型的结果值，这个值有几个方法给我们调用：`Wait`、`Signal`、`Broadcast`。它们分别代表了：**等待通知**、**单发通知**、**广播通知**。

`Wait` 方法会自动地对与该条件变量关了的锁进行解锁，并且使它所在的goroutine阻塞。一旦收到通知，该方法所在的goroutine就会被唤醒，并且该方法会立即尝试锁定该锁。方法`Signal` 和 `Broadcast` 的作用都是发送通知，以唤醒正在为此阻塞的goroutine。不同的是前者的目标只有一个，而后者的目标是所有。

## 实例

在《锁》的笔记当中讲到 `df.f.ReadAt` 方法可能会返回一个`io.EOF` 的错误，我们使用了for 循环来直到读取到数据为止。现我们用条件变量来进行一次改造。

首先，给`myDataFile` 结构体增加一个类型为`*sync.Cond` 的字段 `rcond`。先不管怎样初始化它。直接先改造Read 方法和 Write 方法：

Read方法：
```golang
func (df *myDataFile) Read()(rsn int64,d Data,err error){
 //读取并更新读偏移量
 //省略若干代码

 //读取一个数据块
 rsn = offset/int64(df.dataLen)
 bytes := make([]byte,df.dataLen)
 df.fmutex.RLock()
 defer df.fmutex.RUnlock()
 for{
  _,err = df.f.ReadAt(bytes,offset)
  if err !=nil{
   if err == io.EOF{
    df.rcond.Wait()
    continue
   }
   return
  }
  d = bytes
  return
 }
}
```
Write 方法：
```golang
func (df *myDataFile) Write(d Data) (wsn int64,err error){
 //省略若干代码
 var bytes []byte
 df.fmutex.Lock()
 defer df.fmutex.Unlock()
 _,err = df.f.Write(bytes)
 df.rcond.Signal()
 return
}
```

这里假设 rcond 与读写锁 fmutex 相关联。当每次写完数据之后 Signal 便会去通知某个为此而等待的 Wait 。那条件变量是怎样与 fmutex 相绑定的呢？在 NewDataFile 函数声明中声明 df 变量的语句后加一句 ：
```golang
df.rcond = sync.NewCond(df.fmutex.RLocker())
```

完整代码：
```golang
package cond   
import (
   "os"
   "sync" 
   "errors" 
   "io" 
)

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
  rcond *sync.Cond //条件变量
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
   df.rcond = sync.NewCond(df.fmutex.RLocker())
   return df, nil 
}

func (df *myDataFile) Read() (rsn int64, d Data, err error) {
  var offset int64
  df.rmutex.Lock()
  offset = df.roffset
  df.roffset += int64(offset)
  df.rmutex.Unlock()

   //读取一个数据块
  rsn = offset / int64(df.datalen)
   bytes := make([]byte, df.datalen)
   df.fmutex.RLock()
   defer df.fmutex.RUnlock()
   for {
      _, err = df.f.ReadAt(bytes, rsn)
      if err != nil {
         if err == io.EOF {
            df.rcond.Wait()
            continue
  }
         return
  }
   d = bytes
   return
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
   df.rcond.Signal()
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
   return df.datalen 
}

func (df *myDataFile) Close() error {

   return df.f.Close()
}
```