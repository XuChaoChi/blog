---
title: c++并发学习--std::thread
tags:
  - c++并发
  - c11
category: 'c++并发'
keywords: 'thread,线程,c++并发'
date: 2019-03-05 02:17:05
---

# c++并发学习--std::thread


## 结构体

    typedef unsigned int _Thrd_id_t;
    typedef struct
        {	/* thread identifier for Win32 */
        void *_Hnd;	/* Win32 HANDLE win32句柄*/
        _Thrd_id_t _Id;//线程id
        } _Thrd_imp_t;   

## 方法

### get_id()

    return (_Thr_val(_Thr));

- #define _Thr_val(thr) thr._Id 直接返回线程结构体中的id

<!--more-->

### join_able()

检测线程是否是活跃线程，或者说是正常线程，比如线程已经退出了再join()则会返回

    return (!_Thr_is_null(_Thr));

- _Thr_is_null只检测id是否是0，没有检测句柄

###  native_handle()

直接返回了线程的句柄

    return (_Thr._Hnd);


### hardware_concurrency()

返回支持的线程数

    return (_Thrd_hardware_concurrency());


### join()

阻塞当前线程直到调用join的线程执行完毕。

#### 具体实现

    if (!joinable())
		_Throw_Cpp_error(_INVALID_ARGUMENT);
	const bool _Is_null = _Thr_is_null(_Thr);	// Avoid Clang -Wparentheses-equality
	if (_Is_null)
		_Throw_Cpp_error(_INVALID_ARGUMENT); 
	if (get_id() == _STD this_thread::get_id())
		_Throw_Cpp_error(_RESOURCE_DEADLOCK_WOULD_OCCUR);
	if (_Thrd_join(_Thr, nullptr) != _Thrd_success)
		_Throw_Cpp_error(_NO_SUCH_PROCESS);
	_Thr_set_null(_Thr);

- 如果不能joinable或者线程id是错的则抛出_INVALID_ARGUMENT
- 如果get_id() == this_thread::get_id()则抛出死锁(_RESOURCE_DEADLOCK_WOULD_OCCUR)
- _Thrd_join != _Thrd_success抛出_NO_SUCH_PROCESS
- 最后_Thr_set_null把_Thr结构体中的句柄释放，把id置为0
- 所以重复的join会抛出_INVALID_ARGUMENT


#### 使用
    int main() {
        std::thread th1([]() {
            int nCnt = 3;
            while (nCnt-- ) {
                std::cout << "thread run...." << std::endl;
                std::this_thread::sleep_for(std::chrono::seconds(2));
            }
        });
        //这边不需要用joinable了,因为在join里面已经判断了,看到有些人的代码写了。
        th1.join();
        std::cout << "end" << std::endl;
        getchar();
        return 0;
    }

    //结果：
    //thread run....
    //thread run....
    //thread run....
    //end

### detach()

从 thread 对象分离执行的线程，允许执行独立地持续。一旦线程退出，则释放所有分配的资源。调用 detach 后， 这个线程不再占有任何线程。

#### 实现

    if (!joinable())
			_Throw_Cpp_error(_INVALID_ARGUMENT);
		_Thrd_detachX(_Thr);
		_Thr_set_null(_Thr);

- 如果线程不是活跃线程则抛出_INVALID_ARGUMENT
- 调用_Thrd_detachX分离执行的线程
- 把id置0

#### 使用

    int main() {
        std::thread th1([]() {
            int nCnt = 3;
            while (nCnt-- ) {
                std::cout << "thread run...." << std::endl;
                std::this_thread::sleep_for(std::chrono::seconds(2));
            }
        });

        th1.detach();
        std::cout << "end" << std::endl;
        getchar();
        return 0;
    }

    //结果：
    //thread run....
    //end
    //thread run....
    //thread run....
    
### swap(std::thread& other)

互换二个 thread 对象的底层句柄

#### 实现

    _STD swap(_Thr, _Other._Thr);
- 直接特化了std::swap的模板

#### 使用

    int main() {
        std::thread th1([]() {
            std::this_thread::sleep_for(std::chrono::seconds(2));
        });
        std::thread th2([]() {
            std::this_thread::sleep_for(std::chrono::seconds(2));
        });
        std::cout << "thread1:" << th1.get_id() << " thread2:" << th2.get_id() << std::endl;
        th1.swap(th2);
        std::cout << "thread1:" << th1.get_id() << " thread2:" << th2.get_id() << std::endl;
        th1.join();
        th2.join();
        std::cout << "end" << std::endl;
        getchar();
        return 0;
    }
    //结果：
    //thread1:1012 thread2:16988
    //thread1:16988 thread2:1012
    //end

