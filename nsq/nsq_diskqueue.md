---
title: nsq源码阅读笔记--diskqueue(磁盘读写消息队列)
tags:
  - nsq
  - go
category: 'nsq'
keywords: 'nsq,go'
date: 2020-02-01 23:32:05
---

# nsq源码阅读笔记--diskqueue(磁盘读写消息队列)

### 代码地址

[go-diskqueue模块阅读](https://github.com/XuChaoChi/nsq_read/blob/master/nsqio/go-diskqueue%40v0.0.0-20180306152900-74cfbc9de839/diskqueue.go)

### 简介

- diskqueue主要处理消息和消息对应的元素据的落地(异步读写防止消息丢失)和读取，通过
	```go
	func New(name string,           //创建文件和元素据文件的名字
			dataPath string,        //数据文件保存路径
			maxBytesPerFile int64,  //单个文件最大存储
			minMsgSize int32,       //消息最小值
			maxMsgSize int32,       //消息最大值
			syncEvery int64,        //写入多少条数据后同步写入多少条数据后同步
			syncTimeout time.Duration, //同步时间间隔
			logf AppLogFunc         //日志函数
			) Interface             //返回队列对象的接口
	```
	<!--more-->
	来创建对象，diskqueue拥有独立的goroute进行消息的处理。diskqueue创建后返回**Interface**的接口，此接口定义和**BackendQueue**的定义相同，结构：
	```go
	type BackendQueue interface {
		Put([]byte) error
		ReadChan() chan []byte // this is expected to be an *unbuffered* channel
		Close() error
		Delete() error
		Depth() int64
		Empty() error
	}
	```

- 消息格式

|消息内容长度（不包含int32的长度）|消息内容|
|-|-|
|int32|[]byte|

- 元素据格式

	格式:%d\n%d,%d\n%d,%d\n

|depth|readFileNum|readPos|writeFileNum|writePos|
|-|-|-|-|-|
|int64|int64|int64|int64|int64|

- 消息文件名

	"%s.diskqueue.%06d.dat",d.name, d.filenum

- 元素据文件名

	"%s.diskqueue.meta.dat",d.name

### 代码分析

#### 结构定义

```go
type diskQueue struct {
	// 64bit atomic vars need to be first for proper alignment on 32bit platforms

	// run-time state (also persisted to disk)
	readPos      int64 //当前文件读取的位置
	writePos     int64 //当前文件写入的位置
	readFileNum  int64 //正在读取的文件索引
	writeFileNum int64 //正在写入的文件索引
	depth        int64 //当前消息的数量(队列的大小)

	sync.RWMutex

	// instantiation time metadata
	name            string		  // 创建文件和元素据文件的名字
	dataPath        string		  // 文件保存的路径
	maxBytesPerFile int64         // currently this cannot change once created 单个文件最大存储
	minMsgSize      int32         //消息最小值
	maxMsgSize      int32         //消息最大值
	syncEvery       int64         // number of writes per fsync  写入多少条数据后同步写入多少条数据后同步
	syncTimeout     time.Duration // duration of time per fsync	 同步时间间隔
	exitFlag        int32		  // 标记是否退出
	needSync        bool //是否需要同步数据

	// keeps track of the position where we have read
	// (but not yet sent over readChan)
	nextReadPos     int64 //下一次读的位置
	nextReadFileNum int64 //下一次读的文件索引

	readFile  *os.File //读文件的对象
	writeFile *os.File //写文件的对象
	reader    *bufio.Reader
	writeBuf  bytes.Buffer

	// exposed via ReadChan()
	readChan chan []byte //读取消息的chan通过ReadChan()暴露

	// internal channels
	writeChan         chan []byte
	writeResponseChan chan error
	emptyChan         chan int
	emptyResponseChan chan error
	exitChan          chan int
	exitSyncChan      chan int

	logf AppLogFunc	//日志函数
}
```

#### 消息循环

- 处理的消息：
	- 消息读取
	- 消息清空
	- 消息写入
	- 定时任务（同步元素据）
	- 退出消息循环

- 消息循环
	```go
	select {
		// the Go channel spec dictates that nil channel operations (read or write)
		// in a select are skipped, we set r to d.readChan only when there is data to read
		//读取消息
		case r <- dataRead:
			count++
			// moveForward sets needSync flag if a file is removed
			//删除已读文件，检测读取状态
			d.moveForward()
		//清空的消息
		case <-d.emptyChan:
			d.emptyResponseChan <- d.deleteAllFiles()
			count = 0
		//写入消息
		case dataWrite := <-d.writeChan:
			count++
			d.writeResponseChan <- d.writeOne(dataWrite)
		//定时器
		case <-syncTicker.C:
			if count == 0 {
			// avoid sync when there's no activity
			//没有写入过直接跳过
				continue
			}
			d.needSync = true
		//收到退出消息
		case <-d.exitChan:
			goto exit
		}
	```

- 消息循环流程图

	![消息循环流程图](/blog/img/nsq/diskqueue_1.png) 

#### 读取

读取通过 **ReadChan() chan []byte** 返回chan，然后通过外层topic的消息循环处理消息

- 外层topic读取

	```go
	if len(chans) > 0 && !t.IsPaused() {
		memoryMsgChan = t.memoryMsgChan
		backendChan = t.backend.ReadChan()
	}

	// main message loop
	// topic的主消息循环
	for {
		select {
		...
		case buf = <-backendChan:
			msg, err = decodeMessage(buf)
			if err != nil {
				t.ctx.nsqd.logf(LOG_ERROR, "failed to decode message - %s", err)
				continue
			}
		...
		}
	}
	```
- diskqueue的消息读取

	```go
	func (d *diskQueue) readOne() ([]byte, error) {
		var err error
		var msgSize int32

		if d.readFile == nil {
			curFileName := d.fileName(d.readFileNum)
			//打开当前读到的文件
			d.readFile, err = os.OpenFile(curFileName, os.O_RDONLY, 0600)
			if err != nil {
				return nil, err
			}

			d.logf(INFO, "DISKQUEUE(%s): readOne() opened %s", d.name, curFileName)
			//移动到要读的位置
			if d.readPos > 0 {
				_, err = d.readFile.Seek(d.readPos, 0)
				if err != nil {
					d.readFile.Close()
					d.readFile = nil
					return nil, err
				}
			}

			d.reader = bufio.NewReader(d.readFile)
		}

		//读取消息的长度(不包含这个uint32)
		err = binary.Read(d.reader, binary.BigEndian, &msgSize)
		if err != nil {
			d.readFile.Close()
			d.readFile = nil
			return nil, err
		}

		//检测文件是否损坏
		if msgSize < d.minMsgSize || msgSize > d.maxMsgSize {
			// this file is corrupt and we have no reasonable guarantee on
			// where a new message should begin
			d.readFile.Close()
			d.readFile = nil
			return nil, fmt.Errorf("invalid message read size (%d)", msgSize)
		}
		//读取消息内容
		readBuf := make([]byte, msgSize)
		_, err = io.ReadFull(d.reader, readBuf)
		if err != nil {
			d.readFile.Close()
			d.readFile = nil
			return nil, err
		}

		//消息和头的总长度
		totalBytes := int64(4 + msgSize)

		// we only advance next* because we have not yet sent this to consumers
		// (where readFileNum, readPos will actually be advanced)
		//更新下次的阅读位置
		d.nextReadPos = d.readPos + totalBytes
		d.nextReadFileNum = d.readFileNum

		// TODO: each data file should embed the maxBytesPerFile
		// as the first 8 bytes (at creation time) ensuring that
		// the value can change without affecting runtime
		//判断是否要读下个文件
		if d.nextReadPos > d.maxBytesPerFile {
			if d.readFile != nil {
				d.readFile.Close()
				d.readFile = nil
			}

			d.nextReadFileNum++
			d.nextReadPos = 0
		}

		return readBuf, nil
	}
	```

- 读取流程图

	![读取流程图](/blog/img/nsq/diskqueue_2.png) 

#### 写入

- 调用函数

	```go 
	//将一个[]byte数据推入队列
	func (d *diskQueue) Put(data []byte) error {
		d.RLock()
		defer d.RUnlock()

		if d.exitFlag == 1 {
			return errors.New("exiting")
		}

		//推送到写channel
		d.writeChan <- data
		//等待返回写的结果
		return <-d.writeResponseChan
	}
	```

- diskqueue的消息写入

	```go
	func (d *diskQueue) writeOne(data []byte) error {
		var err error

		if d.writeFile == nil {
			curFileName := d.fileName(d.writeFileNum)
			//创建新的存储文件
			d.writeFile, err = os.OpenFile(curFileName, os.O_RDWR|os.O_CREATE, 0600)
			if err != nil {
				return err
			}

			d.logf(INFO, "DISKQUEUE(%s): writeOne() opened %s", d.name, curFileName)

			if d.writePos > 0 {
				//根据文件原点偏移
				_, err = d.writeFile.Seek(d.writePos, 0)
				if err != nil {
					d.writeFile.Close()
					d.writeFile = nil
					return err
				}
			}
		}

		dataLen := int32(len(data))
		//判断消息长度的合法性
		if dataLen < d.minMsgSize || dataLen > d.maxMsgSize {
			return fmt.Errorf("invalid message write size (%d) maxMsgSize=%d", dataLen, d.maxMsgSize)
		}

		d.writeBuf.Reset()
		//写入消息长度
		err = binary.Write(&d.writeBuf, binary.BigEndian, dataLen)
		if err != nil {
			return err
		}
		//写入消息内容
		_, err = d.writeBuf.Write(data)
		if err != nil {
			return err
		}

		// only write to the file once
		//只写入一次
		_, err = d.writeFile.Write(d.writeBuf.Bytes())
		if err != nil {
			d.writeFile.Close()
			d.writeFile = nil
			return err
		}

		totalBytes := int64(4 + dataLen)
		d.writePos += totalBytes
		//增加写入条数
		atomic.AddInt64(&d.depth, 1)

		//写入文件大于最大字节后创建新文件
		if d.writePos > d.maxBytesPerFile {
			d.writeFileNum++
			d.writePos = 0

			// sync every time we start writing to a new file
			//创建之前先同步当前的数据
			err = d.sync()
			if err != nil {
				d.logf(ERROR, "DISKQUEUE(%s) failed to sync - %s", d.name, err)
			}

			if d.writeFile != nil {
				d.writeFile.Close()
				d.writeFile = nil
			}
		}

		return err
	}
	```

- 写入流程图

	![写入流程图](/blog/img/nsq/diskqueue_3.png) 
