---
title: "Critical Section in Asynchronous Environment"
date: 2020-06-26T21:42:42+08:00
draft: false
---

__Note: This article is automatically translated, please turn to the Chinese version for more accurate expression if possible.__

Recently, I encountered an interesting problem when writing the check-in bot in the quiz group, so let's talk about it.

After a punch-in dialogue is completed, the program needs to write the data into the file, and then it will call the NodeJS API (JavaScript):

```javasctipt
class XXX {
  async saveData() {
    await fs.promises.writeFile(...)
    return
  }
}
```

Since the program is asynchronous and allows two people to check in at the same time, as a result, a check-in dialogue between two people ends at the same time, and the two dialogue functions are executed simultaneously`await saveData()` 。

The consequence of this is that the first`writeFile` The operation returns normally. The person who executed the function first punched in and the function ended normally, and the second one`writeFile` The operation never ends. the second `writeFile` The operation is called during the first operation, and there is no mutual exclusion, which may cause problems in the internal processing of NodeJS, because`writeFile` NodeJS will be responsible for the creation and recycling of file handles.

So, how to solve this problem? This is a typical code key area in lower-level languages, and we have to mutually exclusive the code that accesses file resources. In C++, mutual exclusion is achieved through semaphores. The first thread that accesses the resource is locked, and the threads that access later can only wait, and this waiting process is blocked, and the thread is suspended while waiting. However, in the JavaScript environment, most of the APIs provided by Node are non-blocking, and we cannot and should not solve this problem by calling blocking file writing functions.

The asynchronous nature makes`saveData` The function may be called at any time, and the same may be the last time`saveData` Called before returning. In order to achieve `saveData` The internal code is mutually exclusive, we can use the queue to record all`saveData` Call (that is, the Promise object returned by the flag function). In each call, first check whether there is an unfinished call in the queue. If there is, wait for the last function to execute before continuing to execute the key area logic. At the same time Put the Promise object you want to return into the queue before execution, and remove your Promise object from the queue after the execution ends. The implementation code is as follows:

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

Thanks to ES6's promise object, it greatly simplifies the solution to this problem.
