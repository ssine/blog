---
title: "异步环境下的代码临界区"
date: 2020-06-26T21:42:42+08:00
draft: false
---

最近在写刷题群里的打卡 bot 时遇到了一个比较有意思的问题，因此来聊一下。

在一段打卡对话完成之后，程序需要把数据写入文件，这时会调用 NodeJS 的 API （JavaScript）:

```javasctipt
class XXX {
  async saveData() {
    await fs.promises.writeFile(...)
    return
  }
}
```

由于程序是异步的，并且允许两个人同时打卡，结果导致有一次两个人的打卡对话同时结束，两个对话函数同时执行了 `await saveData()` 。

这样的后果是，第一个 `writeFile` 操作正常返回了，先执行该函数的人打卡函数正常结束，而第二个 `writeFile` 操作却永远没有结束。 第二个 `writeFile` 操作是在第一个操作进行过程中被调用的，并且没有做互斥，这可能导致 NodeJS 内部的处理出现了问题，因为 `writeFile` 将由 NodeJS 负责文件句柄的创建与回收。

那么，要如何解决这个问题呢？ 这在更底层的语言中属于典型的代码关健区，我们要对访问文件资源的代码做互斥。 在 C++ 中，互斥是通过信号量实现的，第一个访问资源的线程加锁，后面访问的线程只能等待，而这个等待的过程是阻塞的，等待时该线程被挂起。 然而在 JavaScript 环境中， Node 提供的多数 API 是非阻塞的，我们不能也不应当通过调用阻塞写文件函数的方式来解决这个问题。

异步特性使得 `saveData` 函数在任意时刻都有可能被调用，同样有可能在上一次 `saveData` 返回之前被调用。 为了实现 `saveData` 内部的代码的互斥，我们可以使用队列来记录下所有的 `saveData` 调用（也就是标志函数返回的 Promise 对象），在每次调用时，先检查队列中是否有上一次未完成的调用，如果有，就等待上一次的函数执行完再继续执行关键区逻辑，同时在执行之前将自己要返回的 Promise 对象放入队列，并在执行结束后把自己的 Promise 对象从队列中移除。 实现代码如下：

```javascript
let queue = []

async function serialized_async_function() {
  const promise = new Promise(async (resolve, reject) => {
    if (queue.length > 0)
      await queue[queue.length - 1]

    await fs.promises.writeFile(...) // any critical section logic

    resolve()
    queue.splice(0, 1)
  })
  queue.push(promise)
  return promise
}
```

多亏了 ES6 的 promise 对象，它使得对这一问题的解决方案得到了极大的简化。
