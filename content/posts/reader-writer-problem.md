---
title: 读者-写者问题
date: '2018-06-03'
dateformat: 'Y-m-d H:i'
tags:
    - 多线程
    - 课程
categories:
    - 课程
---

这是操作系统课的一次实验，利用信号量机制实现读者-写者问题。使用n个线程表示n个读者与写者。

<!-- more -->

读者-写者问题的操作限制：

1. 写-写互斥
2. 读-写互斥
3. 读-读允许

读者优先的附加限制：如果一个读者申请进行读操作时已有另一读者正在进行读操作，则该读者可直接开始读操作。

写者优先的附加限制：如果一个读者申请进行读操作时已有另一写者在等待访问共享资源，则该读者必须等到没有写者处于等待状态后才能开始读操作。

运行结果显示要求：要求在每个线程创建、发出读写操作申请、开始读写操作和结束读写操作时分别显示一行提示信息，以确信所有处理都遵守相应的写操作限制。

测试数据文件格式：

id、读/写者、开始时间、持续时长，以空格隔开，单位为毫秒。

```
1 R 3000 5000
2 W 4000 5000
3 R 5000 2000
4 R 6100 5000
5 W 5100 3000
6 R 6100 5100
```


布置实验的时候提供了参考源码，不过是用winAPI实现的，看起来极为丑陋（虽然深入了解一下也能体会到其设计的精妙）也不能向Linux移植。正好没有具体的实验要求，就决定自己利用C++11的新的库来具体实现一个读者写者问题，尽量使代码清晰易读，且能够跨平台编译。

C++11提供的多线程相关的库有`thread`、`mutex`、`condition_variable`，却没有`semaphore`。参考网上的解决方案，我先利用`mutex`与`condition_variable`类实现了一个`semaphore`类（具体实现也很简单），之后利用`semaphore`实现了两类读者-写者问题。

![img](https://sine-img-bed.oss-cn-beijing.aliyuncs.com/autoup/681873-20151022180551474-1329293072_rmbg.png)

`mutex`用于实现代码关键区互斥，`condition_varibale`用于实现信号同步机制（PV原语），而C++官方给的条件变量使用说明中总是与互斥锁结合使用的，没有直接提供信号量可能是为了增加灵活性吧。

封装一个semaphore类很简单：

```cpp
#include <mutex>
#include <condition_variable>

class semaphore {
public:
    semaphore (int count_ = 0) : count(count_) {}

    inline void notify() {
        std::unique_lock<std::mutex> lock(mtx);
        count++;
        cv.notify_one();
    }

    inline void wait() {
        std::unique_lock<std::mutex> lock(mtx);
        cv.wait(lock, [this]() { return count > 0; });
        count--;
    }

private:
    std::mutex mtx;
    std::condition_variable cv;
    int count;
};

// https://stackoverflow.com/questions/4792449/c0x-has-no-semaphores-how-to-synchronize-threads
```

后面就可以用这个高级一点的东西来做题了。

定义的变量有读者和写者的计数器以及对应的计数器信号量，还有读-写（文件）互斥以及写-写互斥的信号量。如下：

```cpp
int reader_count = 0, writer_count = 0;
semaphore reader_count_mtx(1);
semaphore writer_count_mtx(1);

semaphore file(1);  // Read & Write Mutex
semaphore writer_sem(1); // Writer Mutex
```

之后的读者与写者线程函数分为读者优先与写者优先两个命名空间，以下是声明：

```cpp
namespace writer_priority {
    void reader(int id, int pre_time, int exec_time);
    void writer(int id, int pre_time, int exec_time);
}

namespace reader_priority {
    void reader(int id, int pre_time, int exec_time);
    void writer(int id, int pre_time, int exec_time);
}
```

参数分别是操作者id、发出请求的时间、执行操作所需的时间。它们的定义在main函数的下方。

主函数负责从文件读取并创建线程以及询问问题的类型（何者优先）。

读者优先与写者优先的写者函数是一样的，而写者优先的读者函数比读者优先的多两行，其余内容都一样。先考虑读者优先问题。

我们把同一时间段内霸占文件的读者看作一个连续的整体，在这个整体出现以前，文件要么为空，要么被写者占有，此时的reader_count变量必然为0。在读者提出请求并发现自己是第一个读者（reader_count==1）的时候，它要为这个整体获取文件锁（读-写互斥锁）。这之后的其他读者不需要获取文件锁便可以访问文件，因为此时文件属于读者们的整体。在读者读完并发现自己时整体中的最后一个读者（reader_count==0）的时候，它需要释放文件锁以供写者使用。写者也是同样的思路。

下面考虑写者优先。在读者优先情况下，读者发出请求后如果已经有人在读那么无需等待，而现在则要检查一下是否已有写者在等待，有写者的话要先等写者完成。如何表明有写者在等待呢？如果有写者发现自己是第一个写者，那么它会等待获取文件锁，而这时写者计数器已经被它加锁了，因此只需要在写者进入刚才的流程前等待一下写者的计数器锁就行了。具体实现请参考代码。

```cpp
/*
 * Reader-Writer Problem
 * author: Liu Siyao
 * compile:
 *   - Linux: g++ reader-writer.cc -std=c++11 -lpthread -o reader-writer
 *   - Windows(VS2015 Native Tools environment): cl reader-writer.cc
 */

#include <iostream>
#include <string>
#include <vector>
#include <fstream>
#include <thread>
#include <chrono>
#include "semaphore.h"

using namespace std;

int reader_count = 0, writer_count = 0;
semaphore reader_count_mtx(1);
semaphore writer_count_mtx(1);

semaphore file(1);  // Read & Write Mutex
semaphore writer_sem(1); // Writer Mutex

namespace writer_priority {
    void reader(int id, int pre_time, int exec_time);
    void writer(int id, int pre_time, int exec_time);
}

namespace reader_priority {
    void reader(int id, int pre_time, int exec_time);
    auto writer = writer_priority::writer;
    // the writer logic of the two kinds are identical
}


int main() {
    vector<thread> threads;

    // process the input and spawn all the processes
    ifstream f("./thread.dat");
    char kind;
    int id, pre_time, exec_time;

    int choice;
    cout << "Choose the competition type of this program:\n"
         << "1. writer priority\n"
         << "2. reader priority\n"
         << "your choice: ";
    cin >> choice;

    while(f) {
        f >> id >> kind >> pre_time >> exec_time;
        f.get();

        if(choice == 1) {
            auto func = kind == 'R' ?
                        writer_priority::reader : writer_priority::writer;
            threads.push_back(thread(func, id, pre_time, exec_time));
        } else {
            auto func = kind == 'R' ?
                        reader_priority::reader : reader_priority::writer;
            threads.push_back(thread(func, id, pre_time, exec_time));
        }
        
        cout << kind << " " << pre_time << " " << exec_time << " loaded!\n";
    }

    // wait for all the process to be terminated
    for(int i = 0; i < threads.size(); i++)
        threads[i].join();

    return 0;
}


namespace writer_priority {
    void reader(int id, int pre_time, int exec_time) {
        this_thread::sleep_for(chrono::milliseconds(pre_time));
        cout << "reader " << id << " sends the read request\n";

        // wait for waiting writers
        writer_count_mtx.wait();
        writer_count_mtx.notify();

        reader_count_mtx.wait();
        reader_count++;
        if(reader_count == 1) file.wait();
        reader_count_mtx.notify();

        cout << "reader " << id << " starts reading\n";
        this_thread::sleep_for(chrono::milliseconds(pre_time));
        cout << "reader " << id << " has finished reading\n";

        reader_count_mtx.wait();
        reader_count--;
        if(reader_count == 0) file.notify();
        reader_count_mtx.notify();

        return;
    }

    void writer(int id, int pre_time, int exec_time) {
        this_thread::sleep_for(chrono::milliseconds(pre_time));
        cout << "writer " << id << " sends the write request\n";

        writer_count_mtx.wait();
        writer_count++;
        if(writer_count == 1) file.wait();
        writer_count_mtx.notify();

        writer_sem.wait();

        cout << "writer " << id << " starts writing\n";
        this_thread::sleep_for(chrono::milliseconds(pre_time));
        cout << "writer " << id << " has finished writing\n";

        writer_sem.notify();

        writer_count_mtx.wait();
        writer_count--;
        if(writer_count == 0) file.notify();
        writer_count_mtx.notify();

        return;
    }
}

namespace reader_priority {
    void reader(int id, int pre_time, int exec_time) {
        this_thread::sleep_for(chrono::milliseconds(pre_time));
        cout << "reader " << id << " sends the read request\n";

        reader_count_mtx.wait();
        reader_count++;
        if(reader_count == 1) file.wait();
        reader_count_mtx.notify();

        cout << "reader " << id << " starts reading\n";
        this_thread::sleep_for(chrono::milliseconds(pre_time));
        cout << "reader " << id << " has finished reading\n";

        reader_count_mtx.wait();
        reader_count--;
        if(reader_count == 0) file.notify();
        reader_count_mtx.notify();

        return;
    }

    // the writer logic of the two kinds are identical
    // auto writer = writer_priority::writer;
}
```
