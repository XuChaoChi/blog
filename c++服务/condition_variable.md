---
title: c++并发学习--condition_variable
tags:
  - c++并发
  - c11
category: 'c++并发'
keywords: 'condition_variable,竞争条件,c++并发'
abbrlink: 19fc34ac
date: 2019-01-03 20:35:05
---

## c++并发学习--condition_variable

## 介绍
condition_variable 类是同步原语，能用于阻塞一个线程，或同时阻塞多个线程，直至另一线程修改共享变量（条件）并通知 condition_variable 。

## 方法

### notify_one()和wait()
分别是通知一个等待的线程和阻塞当前线程，直到条件变量被唤醒 
<!--more-->
    #include <mutex>
    #include <iostream>
    #include <queue>
    class Test {
    public:
        void test_notify_one() {
            std::unique_lock<std::mutex> lck(m_mutex);
            m_queue.push("test");
            m_cv.notify_one();
        }

        void test_wait() {
            std::unique_lock<std::mutex> lck(m_mutex);
            while (m_queue.empty()) {
                m_cv.wait(lck);
            }
            std::cout << m_queue.front().c_str() << std::endl;
        }
    private:
        std::condition_variable m_cv;
        std::mutex m_mutex;
        std::queue<std::string> m_queue;
    };
    int main() {
        Test t;
        std::thread td([&]() {
            t.test_wait();
        });
        std::thread td2([&]() {
            t.test_wait();
        });
        t.test_notify_one();
        getchar();
        td.join();
        td2.join();
        return 0;
    }

    //结果:test

### notify_all()

通知所有等待的线程 

    #include <mutex>
    #include <iostream>
    #include <queue>
    class Test {
    public:
        void test_notify_all() {
            std::unique_lock<std::mutex> lck(m_mutex);
            m_queue.push("test");
            m_cv.notify_all();
        }

        void test_wait() {
            std::unique_lock<std::mutex> lck(m_mutex);
            while (m_queue.empty()) {
                m_cv.wait(lck);
            }
            std::cout << m_queue.size() << std::endl;
        }
    private:
        std::condition_variable m_cv;
        std::mutex m_mutex;
        std::queue<std::string> m_queue;
    };
    int main() {
        Test t;
        std::thread td([&]() {
            t.test_wait();
        });
        std::thread td2([&]() {
            t.test_wait();
        });
        t.test_notify_all();
        getchar();
        td.join();
        td2.join();
        return 0;
    }

    //结果:1 1

###  wait_for()

阻塞当前线程，直到条件变量被唤醒，或到指定时限时长后 

    #include <mutex>
    #include <iostream>
    #include <queue>
    class Test {
    public:
	void test_wait_for() {
		std::unique_lock<std::mutex> lck(m_mutex);
		while (m_cv.wait_for(lck, std::chrono::seconds(2)) == std::cv_status::timeout) {
			std::cout << "timeout" << std::endl;
		}
		std::cout << __FUNCTION__ << std::endl;
	}
	void notify(){
		std::unique_lock<std::mutex> lck(m_mutex);
		m_cv.notify_one();
	}
    private:
        std::condition_variable m_cv;
        std::mutex m_mutex;
        std::queue<std::string> m_queue;
    };
    int main() {
        Test t;
        std::thread td1([&]() {
            t.test_wait_for();
        });
        
        std::this_thread::sleep_for(std::chrono::seconds(5));
        t.notify();
        getchar();
        td1.join();
        return 0;
    }

    //结果: timeout
            timeout
            Test::test_wait_for

### wait_until()

阻塞当前线程，直到条件变量被唤醒，或直到抵达指定时间点 
    #include <mutex>
    #include <iostream>
    #include <queue>
    class Test {
    public:
        void test_wait_until() {
            std::unique_lock<std::mutex> lck(m_mutex);
            std::chrono::system_clock::time_point time = std::chrono::system_clock::now() + std::chrono::seconds(2);
            while (m_cv.wait_until(lck, time) == std::cv_status::timeout) {
                std::cout << "timeout" << std::endl;
            }
            //也有这样的写法当nIndex=2的时候将阻塞
            //int nIndex = 0;
            //while (m_cv.wait_until(lck, time, [&]() {return nIndex != 2;})//) {
            //     std::cout << "timeout" << std::endl;
            //    nIndex++;
            //}
            std::cout << __FUNCTION__ << std::endl;
        }

        void notify(){
            std::unique_lock<std::mutex> lck(m_mutex);
            m_cv.notify_one();
        }
    private:
        std::condition_variable m_cv;
        std::mutex m_mutex;
        std::queue<std::string> m_queue;
    };
    int main() {
        Test t;
        std::thread td1([&]() {
            t.test_wait_until();
        });
        
        std::this_thread::sleep_for(std::chrono::seconds(5));
        t.notify();
        getchar();
        td1.join();
        return 0;
    }
    //结果:n个timeout，5秒后Test::test_wait_until
## 传入std::unique_lock
因为condition_variable的唤醒往往是其他线程的，为了保证线程安全需要使用锁，当wait的时候锁又被解开避免死锁。


## 虚假唤醒
### 理解

为什么wait()的时候用if会导致虚假唤醒？就好比几个人(线程)抢一个凳子，收到开始信号(notify_one())后大家开始抢，有的人座下了一半，有的人开始座，当有一个人坐下后其他人就不能坐下去了，如果还是继续坐下去(if的过程)就只能坐地上了，需要重新等待位置空出来才可以(while的过程)。

### 例子
下面是一个虚假唤醒的测试代码，运行下面代码会使得get的时候虚假唤醒导致qTest为空的时候front()导致崩溃

    #include <mutex>
    #include <string>
    #include <iostream>
    #include <queue>
    class Test {
    public:
        void push(const std::string strData) {
            std::unique_lock<std::mutex> lck(m_mutex);
            qTest.push(strData);
            m_cv.notify_one();
        }
        void get(std::string &strData) {
            std::unique_lock<std::mutex> lck(m_mutex);
            //使用if出现虚假唤醒
            //应该使用while(qTest.empty())的时候避免虚假唤醒
            if (qTest.empty()) {
                m_cv.wait(lck);
            }
            strData = qTest.front();
            std::cout  <<strData << std::endl;
            qTest.pop();
        }
    private:
        std::condition_variable m_cv;
        std::queue<std::string> qTest;
        std::mutex m_mutex;
    };
    int main() {
        Test t;
        std::thread tdGet([&]() {
            for (int i = 0; i < 100; i++) {
                std::string strData;
                t.get(strData);;
            }	
        });
        std::thread tdGet2([&]() {
            for (int i = 0; i < 100; i++) {
                std::string strData;
                t.get(strData);
            }
            
        });
        std::thread tdPush([&]() {
            for (int i = 0; i < 100; i++) {
                t.push("test"+std::to_string(i));
            }
                
        });
        tdGet.join();
        tdGet2.join();
        tdPush.join();
        getchar();
        return 0;
    }


## 运用
下面举一个std::condition_variable 使用在消息队列中运用的例子get()的时候如果队列为空就进入忙等状态，如果有队列数据就唤醒。贴一个完整[消息队列代码的链接](https://github.com/XuChaoChi/XSvr/blob/master/src/common/utility/XMsgQueue.hpp)

    bool push(const TMsg &msg)
    {
        std::unique_lock<std::mutex> lck(m_mutex);
        if (_isFull())
        {
            return false;
        }
        else
        {
            m_queue[m_nTailIndex] = msg;
            m_nTailIndex = ++m_nTailIndex % m_queue.capacity();
            m_emptyCv.notify_one();
        }
        return true;
    }
    bool get(TMsg &msg)
    {
        std::unique_lock<std::mutex> lck(m_mutex);
        while (_isEmpty())
        {
            m_emptyCv.wait(lck);
        }
        msg = m_queue[m_nFrontIndex];
        m_nFrontIndex = ++m_nTailIndex % m_queue.capacity();
        m_emptyCv.notify_one();
        return true;
    }
