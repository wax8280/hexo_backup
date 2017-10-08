---
title: 阅读aiosplite源码笔记
date: 2017-09-28 17:35:20
tags:
  - Python
categories:
  - Code
---

在GIthub是看到这样一个[`aiospilte`](https://github.com/jreese/aiosqlite)的库，具体是将sqlite改成异步的。发现源码体积挺小的，遂看一下，看完后发现作者脑路新奇。
作者主要使用了代理模式，代理了`Connection`和`Cursor`这两个`sqlite3`的类。但是作者新奇就在于，其实这是一个披着异步外衣的多线程。作者对于每一个连接都新建一个独立线程。仅仅使用异步作为询问，而实际费IO的操作是在独立的线程中。

作者的`Connection`类继承自`threading.Thread`。

使用Python的上下文管理器：

```python
async def __aenter__(self) -> 'Connection':
    self.start()
    await self._connect()
    return self

async def __aexit__(self, exc_type, exc_val, exc_tb) -> None:
    await self.close()
    self._conn = None
```

可见，每开一个连接，就创建一个新的独立的线程。对于每个独立的线程，执行`run()`方法，`self._tx`是`Queue`，放置着包装好的待执行的函数。可见，所有的实际操作`fn()`是阻塞的。

```python
    def run(self) -> None:
        """Execute function calls on a separate thread."""
        while self._running:
            try:
                fn = self._tx.get(timeout=0.1)
            except Empty:
                continue

            try:
                Log.debug('executing %s', fn)
                result = fn()
                Log.debug('returning %s', result)
                self._rx.put(result)
            except Exception as e:
                Log.exception('returning exception %s', e)
                self._rx.put(e)
```

而所谓的异步的`_execute()`，它仅仅作为询问用，它每隔一段时间看看结果队列`self._rx`是否有结果，有结果即返回。

```python
    async def _execute(self, fn, *args, **kwargs):
        """Queue a function with the given arguments for execution."""
        await self._lock.acquire()

        pt = partial(fn, *args, **kwargs)
        self._tx.put_nowait(pt)

        while True:
            try:
                result = self._rx.get_nowait()
                break

            except Empty:
                await asyncio.sleep(0.1)
                continue

        self._lock.release()

        if isinstance(result, Exception):
            raise result

        return result
```

作者还在这里加了锁。作者这样做是为了防止结果的混乱。举个例子，如下代码，可能因为`await asyncio.sleep(0.1)`，而造成`cursor2`拿到的是`table1`的结果。值得注意的是，如果同一EventLoop里面有其他需要获得锁的协程，可能会导致Bug的出现。

```python
async with aiosqlite.connect(...) as db:
    cursor1 = await db.execute('SELECT * FROM table1')
    cursor2 = await db.execute('SELECT * FROM table2')
```