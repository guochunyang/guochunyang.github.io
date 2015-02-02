---
layout: post
title: 使用Python实现生产者消费者问题
category : C++
tagline: "Supporting tagline"
tags : [C++, Python, Socket, Tornado]
---
之前用C++写过一篇生产者消费者的实现。
  生产者和消费者主要是处理互斥和同步的问题：
     队列作为缓冲区，需要互斥操作
      队列中没有产品，消费者需要等待，直到生产者放入产品并通知它。队列慢的情况类似。
   这里我使用list模拟Python标准库的Queue，这里我设置一个大小限制为5：
  SyncQueue.py
  

```Python
from threading import Lock
from threading import Condition
class Queue():
    def __init__(self):
        self.mutex = Lock()
        self.full = Condition(self.mutex)
        self.empty = Condition(self.mutex)
        self.data = []

    def push(self, element):
        self.mutex.acquire()
        while len(self.data) >= 5:
            self.empty.wait()
            
        self.data.append(element)
        self.full.notify()    
        self.mutex.release()


    def pop(self):
        self.mutex.acquire()
        while len(self.data) == 0:
            self.full.wait()
        data = self.data[0]
        self.data.pop(0)
        self.empty.notify()
        self.mutex.release()

        return data

if __name__ == '__main__':
    q = Queue()
    q.push(10)
    q.push(2)
    q.push(13)

    print q.pop()
    print q.pop()
    print q.pop()
```
		



这是最核心的代码，注意里面判断条件要使用while循环。


接下来是生产者进程，producer.py




```Python
from threading import Thread
from random import randrange
from time import sleep
from SyncQueue import Queue

class ProducerThread(Thread):
    def __init__(self, queue):
        Thread.__init__(self)
        self.queue = queue
    def run(self):
        while True:
            data = randrange(0, 100)
            self.queue.push(data)
            print 'push %d' % (data)
            sleep(1)

if __name__ == '__main__':
    q = Queue()
    t = ProducerThread(q)
    t.start()
    t.join()
```
		



消费者,Condumer.py




```Python
from threading import Thread
from time import sleep
from SyncQueue import Queue

class ConsumerThread(Thread):
    def __init__(self, queue):
        Thread.__init__(self)
        self.queue = queue
    def run(self):
        while True:
            data = self.queue.pop()
            print 'pop %d' % (data)
            sleep(1)

if __name__ == '__main__':
    q = Queue()
    t = ConsumerThread(q)
    t.start()
    t.join()
```
		



最后我们写一个车间类,可以指定线程数量:




```Python
from SyncQueue import Queue
from Producer import ProducerThread
from Consumer import ConsumerThread

class WorkShop():
    def __init__(self, producerNums, consumerNums):
        self.producers = []
        self.consumers = []
        self.queue = Queue()
        self.producerNums = producerNums
        self.consumerNums = consumerNums
    def start(self):
        for i in range(self.producerNums):
            self.producers.append(ProducerThread(self.queue))
        for i in range(self.consumerNums):
            self.consumers.append(ConsumerThread(self.queue))
        for i in range(len(self.producers)):
            self.producers[i].start()
        for i in range(len(self.consumers)):
            self.consumers[i].start()
        for i in range(len(self.producers)):
            self.producers[i].join()
        for i in range(len(self.consumers)):
            self.consumers[i].join()

if __name__ == '__main__':
    w = WorkShop(3, 4)
    w.start()
```
		



最后写一个main模块:




```Python
from WorkShop import WorkShop

if __name__ == '__main__':
    w = WorkShop(2, 3)
    w.start()
```
		
			