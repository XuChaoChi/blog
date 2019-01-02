## 使用
    


## 传入std::unique_lock
因为condition_variable的唤醒往往是其他线程的，为了保证线程安全需要使用锁，当wait的时候锁又被解开避免死锁。


## 虚假唤醒

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
### 理解

为什么wait()的时候用if会导致虚假唤醒？就好比几个人(线程)抢一个凳子，收到开始信号(notify_one())后大家开始抢，有的人座下了一半，有的人开始座，当有一个人坐下后其他人就不能坐下去了，如果还是继续坐下去(if的过程)就只能坐地上了，需要重新等待位置空出来才可以(while的过程)。

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