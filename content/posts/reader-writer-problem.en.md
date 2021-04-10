---
title: Reader-Writer Problem
date: '2018-06-03'
dateformat: 'Y-m-d H:i'
tags:
    - Multithreading
---

__Note: This article is automatically translated, please turn to the Chinese version for more accurate expression if possible.__

This is an experiment in the operating system class, using the semaphore mechanism to realize the reader-writer problem. Use n threads to represent n readers and writers.

<!-- more -->

Operational restrictions for reader-writer problems:

1. Write-write mutual exclusion
2. Read-write mutual exclusion
3. Read-read allowed

Additional restrictions for readers priority: If a reader is already reading when another reader is applying for a read operation, the reader can directly start the read operation.

Additional restriction for writer priority: If a reader is applying for a read operation and another writer is waiting to access the shared resource, the reader must wait until no writer is in the waiting state before starting the read operation.

Operation result display requirements: It is required to display a line of prompt information when each thread is created, issue a read-write operation request, start a read-write operation, and end a read-write operation to make sure that all processing complies with the corresponding write operation restrictions.

Test data file format:

id, reader/writer, start time, duration, separated by spaces, in milliseconds.

```
1 R 3000 5000
2 W 4000 5000
3 R 5000 2000
4 R 6100 5000
5 W 5100 3000
6 R 6100 5100
```


The reference source code was provided when setting up the experiment, but it was implemented with winAPI, which looks extremely ugly (although you can understand the subtlety of its design after a deeper understanding) and cannot be ported to Linux. There was no specific experimental requirement, so I decided to use the new library of C++11 to implement a reader-writer problem. Try to make the code clear and easy to read, and it can be compiled across platforms.

The multi-thread-related libraries provided by C++11 are`thread`、`mutex`、`condition_variable`, But not`semaphore`. Refer to the online solution, I first use`mutex`versus`condition_variable`The class implements a`semaphore`Class (the specific implementation is also very simple), and then use`semaphore`Two types of reader-writer problems are realized.

![img](https://sine-img-bed.oss-cn-beijing.aliyuncs.com/autoup/681873-20151022180551474-1329293072_rmbg.png)

`mutex`Used to achieve mutual exclusion in key areas of the code,`condition_varibale`It is used to realize the signal synchronization mechanism (PV primitive), and the condition variable usage instructions given by the C++ official are always used in combination with the mutex. The semaphore is not directly provided to increase flexibility.

Encapsulating a semaphore class is very simple:

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

You can use this advanced thing to do problems later.

The defined variables include reader and writer counters and corresponding counter semaphores, as well as read-write (file) mutual exclusion and write-write mutual exclusion semaphores. as follows:

```cpp
int reader_count = 0, writer_count = 0;
semaphore reader_count_mtx(1);
semaphore writer_count_mtx(1);

semaphore file(1);  // Read & Write Mutex
semaphore writer_sem(1); // Writer Mutex
```

The subsequent reader and writer thread functions are divided into two namespaces, reader first and writer first. The following is the declaration:

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

The parameters are the operator id, the time the request was made, and the time required to perform the operation. Their definition is below the main function.

The main function is responsible for reading from the file and creating threads and asking the type of question (which one takes precedence).

The writer function of reader first and writer first is the same, and the reader function of writer first is two more lines than the reader first, and the rest of the content is the same. Consider the reader's priority first.

We treat the readers who occupy the file in the same time period as a continuous whole. Before this whole appears, the file is either empty or occupied by the writer. At this time, the reader_count variable must be 0. When a reader makes a request and finds that it is the first reader (reader_count==1), it must acquire a file lock (read-write mutex) for the whole. After this, other readers can access the file without obtaining the file lock, because the file belongs to the readers at this time. When the reader finishes reading and finds itself the last reader in the whole (reader_count==0), it needs to release the file lock for the writer to use. The writer has the same idea.

Consider the writer first. In the case of reader priority, after the reader sends a request, if someone is already reading, there is no need to wait, but now it is necessary to check whether there is already a writer waiting, and if there is a writer, wait for the writer to complete it first. How to show that there are writers waiting? If a writer finds that he is the first writer, it will wait to acquire the file lock, and at this time the writer counter has been locked by it, so it only needs to wait for the writer before entering the process just now. The counter lock will do. Please refer to the code for specific implementation.

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
