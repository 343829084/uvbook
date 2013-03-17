线程
====

等等, 我们为什么会提到线程? 事件循环不应该本身就可以应对高并发网络编程么?
不一定, 线程仍然可以在 CPU 处理任务时作为程序代码的中坚力量来执行子任务,
即使在编程多线程程序中你必须编写大量同步原语, 但它们在很多时候还是可以派上大用场的.

线程通常用于内部模拟系统调用的异步特性<?>(原文:
Threads are used internally to fake the asynchronous nature of all the system
calls). libuv 同样可以利用线程让你异步完成一项本可能阻塞的任务, 通常是创建子线程,
在子线程完成任务后获取其结果.

目前存在两个主流的线程库, Windows 线程库实现和 `pthreads`_. libuv 的线程库 API 与
pthread API 相似, 因此两者具有相同的语义.

值得注意的是, libuv 中的线程库是自包含的. 而其他特性都直接依赖事件循环或者回调.
loop and callback principles, threads are complete agnostic, they block as
required, signal errors directly via return values and, as shown in the
:ref:`first example <thread-create-example>`, don't even require a running
event loop.

libuv 的线程 API 也非常有限, 因为不同平台上线程库的语义和语法也不同,
API 功能的完善程度也不尽相同.

本节This chapter makes the following assumption: **There is only one event loop,
running in one thread (the main thread)**. No other thread interacts
with the event loop (except using ``uv_async_send``). :doc:`multiple` covers
running event loops in different threads and managing them.

线程核心操作(Core thread operations)
------------------------------------

下面的例子程序并没有太多代码, 只是通过 ``uv_thread_create`` 创建了线程, 然后调用
``uv_thread_join`` 等待线程退出.

.. _thread-create-example:

.. rubric:: thread-create/main.c
.. literalinclude:: ../code/thread-create/main.c
    :linenos:
    :lines: 26-37
    :emphasize-lines: 3-7

.. tip::

    在 Unix 平台上 ``uv_thread_t`` 只是 ``pthread_t`` 的别名,但是在实现细节上也避免 but this is an
    implementation detail, avoid depending on it to always be true.

The second parameter is the function which will serve as the entry point for
the thread, the last parameter is a ``void *`` argument which can be used to pass
custom parameters to the thread. The function ``hare`` will now run in a separate
thread, scheduled pre-emptively by the operating system:

.. rubric:: thread-create/main.c
.. literalinclude:: ../code/thread-create/main.c
    :linenos:
    :lines: 6-14
    :emphasize-lines: 2

Unlike ``pthread_join()`` which allows the target thread to pass back a value to
the calling thread using a second parameter, ``uv_thread_join()`` does not. To
send values use :ref:`inter-thread-communication`.

同步原语(Synchronization Primitives)
-------------------------------------

This section is purposely spartan. This book is not about threads, so I only
catalogue any surprises in the libuv APIs here. For the rest you can look at
the pthreads `man pages <pthreads>`_.

互斥量(Mutexes)
~~~~~~~~~~~~~~~

The mutex functions are a **direct** map to the pthread equivalents.

.. rubric:: libuv mutex functions
.. literalinclude:: ../libuv/include/uv.h
    :lines: 1834-1838

The ``uv_mutex_init()`` and ``uv_mutex_trylock()`` functions will return 0 on
success, -1 on error instead of error codes.

If `libuv` has been compiled with debugging enabled, ``uv_mutex_destroy()``,
``uv_mutex_lock()`` and ``uv_mutex_unlock()`` will ``abort()`` on error.
Similarly ``uv_mutex_trylock()`` will abort if the error is anything *other
than* ``EAGAIN``.

Recursive mutexes are supported by some platforms, but you should not rely on
them. The BSD mutex implementation will raise an error if a thread which has
locked a mutex attempts to lock it again. For example, a construct like::

    uv_mutex_lock(a_mutex);
    uv_thread_create(thread_id, entry, (void *)a_mutex);
    uv_mutex_lock(a_mutex);
    // more things here

can be used to wait until another thread initializes some stuff and then
unlocks ``a_mutex`` but will lead to your program crashing if in debug mode, or
return an error in the second call to ``uv_mutex_lock()``.

.. note::

    Mutexes on linux support attributes for a recursive mutex, but the API is
    not exposed via libuv.

锁(Locks)
~~~~~~~~~

Read-write locks are a more granular access mechanism. Two readers can access
shared memory at the same time. A writer may not acquire the lock when it is
held by a reader. A reader or writer may not acquire a lock when a writer is
holding it. Read-write locks are frequently used in databases. Here is a toy
example.

.. rubric:: locks/main.c - simple rwlocks
.. literalinclude:: ../code/locks/main.c
    :linenos:
    :emphasize-lines: 13,16,27,31,42,55

Run this and observe how the readers will sometimes overlap. In case of
multiple writers, schedulers will usually give them higher priority, so if you
add two writers, you'll see that both writers tend to finish first before the
readers get a chance again.

其他
~~~~

libuv also supports semaphores_, `condition variables`_ and barriers_ with APIs
very similar to their pthread counterparts.

In the case of condition variables, libuv also has a timeout on a wait, with
platform specific quirks [#]_.

.. _semaphores: http://en.wikipedia.org/wiki/Semaphore_(programming)
.. _condition variables: http://en.wikipedia.org/wiki/Condition_variable#Waiting_and_signaling
.. _barriers: http://en.wikipedia.org/wiki/Barrier_(computer_science)

In addition, libuv provides a convenience function ``uv_once()`` (not to be
confused with ``uv_run_once()``. Multiple threads can attempt to call
``uv_once()`` with a given guard and a function pointer, **only the first one
will win, the function will be called once and only once**::

    /* Initialize guard */
    static uv_once_t once_only = UV_ONCE_INIT;

    int i = 0;

    void increment() {
        i++;
    }

    void thread1() {
        /* ... work */
        uv_once(once_only, increment);
    }

    void thread2() {
        /* ... work */
        uv_once(once_only, increment);
    }

    int main() {
        /* ... spawn threads */
    }

After all threads are done, ``i == 1``.

.. _libuv-work-queue:

libuv 工作队列
--------------

``uv_queue_work()`` is a convenience function that allows an application to run
a task in a separate thread, and have a callback that is triggered when the
task is done. A seemingly simple function, what makes ``uv_queue_work()``
tempting is that it allows potentially any third-party libraries to be used
with the event-loop paradigm. When you use event loops, it is *imperative to
make sure that no function which runs periodically in the loop thread blocks
when performing I/O or is a serious CPU hog*, because this means the loop slows
down and events are not being dealt with at full capacity.

But a lot of existing code out there features blocking functions (for example
a routine which performs I/O under the hood) to be used with threads if you
want responsiveness (the classic 'one thread per client' server model), and
getting them to play with an event loop library generally involves rolling your
own system of running the task in a separate thread.  libuv just provides
a convenient abstraction for this.

Here is a simple example inspired by `node.js is cancer`_. We are going to
calculate fibonacci numbers, sleeping a bit along the way, but run it in
a separate thread so that the blocking and CPU bound task does not prevent the
event loop from performing other activities.

.. rubric:: queue-work/main.c - lazy fibonacci
.. literalinclude:: ../code/queue-work/main.c
    :linenos:
    :lines: 17-29

The actual task function is simple, nothing to show that it is going to be
run in a separate thread. The ``uv_work_t`` structure is the clue. You can pass
arbitrary data through it using the ``void* data`` field and use it to
communicate to and from the thread. But be sure you are using proper locks if
you are changing things while both threads may be running.

The trigger is ``uv_queue_work``:

.. rubric:: queue-work/main.c
.. literalinclude:: ../code/queue-work/main.c
    :linenos:
    :lines: 31-44
    :emphasize-lines: 40

The thread function will be launched in a separate thread, passed the
``uv_work_t`` structure and once the function returns, the *after* function
will be called, again with the same structure.

For writing wrappers to blocking libraries, a common :ref:`pattern <baton>`
is to use a baton to exchange data.

.. _inter-thread-communication:

线程间通信(Inter-thread communication)
--------------------------------------

Sometimes you want various threads to actually send each other messages *while*
they are running. For example you might be running some long duration task in
a separate thread (perhaps using ``uv_queue_work``) but want to notify progress
to the main thread. This is a simple example of having a download manager
informing the user of the status of running downloads.

.. rubric:: progress/main.c
.. literalinclude:: ../code/progress/main.c
    :linenos:
    :lines: 7-8,34-
    :emphasize-lines: 2,11

The async thread communication works *on loops* so although any thread can be
the message sender, only threads with libuv loops can be receivers (or rather
the loop is the receiver). libuv will invoke the callback (``print_progress``)
with the async watcher whenever it receives a message.

.. warning::

    It is important to realize that the message send is *async*, the callback
    may be invoked immediately after ``uv_async_send`` is called in another
    thread, or it may be invoked after some time. libuv may also combine
    multiple calls to ``uv_async_send`` and invoke your callback only once. The
    only guarantee that libuv makes is -- The callback function is called *at
    least once* after the call to ``uv_async_send``. If you have no pending
    calls to ``uv_async_send``, the callback won't be called. If you make two
    or more calls, and libuv hasn't had a chance to run the callback yet, it
    *may* invoke your callback *only once* for the multiple invocations of
    ``uv_async_send``. Your callback will never be called twice for just one
    event.

.. rubric:: progress/main.c
.. literalinclude:: ../code/progress/main.c
    :linenos:
    :lines: 10-23
    :emphasize-lines: 7-8

In the download function we modify the progress indicator and queue the message
for delivery with ``uv_async_send``. Remember: ``uv_async_send`` is also
non-blocking and will return immediately.

.. rubric:: progress/main.c
.. literalinclude:: ../code/progress/main.c
    :linenos:
    :lines: 30-33

The callback is a standard libuv pattern, extracting the data from the watcher.

Finally it is important to remember to clean up the watcher.

.. rubric:: progress/main.c
.. literalinclude:: ../code/progress/main.c
    :linenos:
    :lines: 25-28
    :emphasize-lines: 3

After this example, which showed the abuse of the ``data`` field, bnoordhuis_
pointed out that using the ``data`` field is not thread safe, and
``uv_async_send()`` is actually only meant to wake up the event loop. Use
a mutex or rwlock to ensure accesses are performed in the right order.

.. warning::

    mutexes and rwlocks **DO NOT** work inside a signal handler, whereas
    ``uv_async_send`` does.

One use case where ``uv_async_send`` is required is when interoperating with
libraries that require thread affinity for their functionality. For example in
node.js, a v8 engine instance, contexts and its objects are bound to the thread
that the v8 instance was started in. Interacting with v8 data structures from
another thread can lead to undefined results. Now consider some node.js module
which binds a third party library. It may go something like this:

1. In node, the third party library is set up with a JavaScript callback to be
   invoked for more information::

    var lib = require('lib');
    lib.on_progress(function() {
        console.log("Progress");
    });

    lib.do();

    // do other stuff

2. ``lib.do`` is supposed to be non-blocking but the third party lib is
   blocking, so the binding uses ``uv_queue_work``.

3. The actual work being done in a separate thread wants to invoke the progress
   callback, but cannot directly call into v8 to interact with JavaScript. So
   it uses ``uv_async_send``.

4. The async callback, invoked in the main loop thread, which is the v8 thread,
   then interacts with v8 to invoke the JavaScript callback.

.. _pthreads: http://man7.org/linux/man-pages/man7/pthreads.7.html

----

.. _node.js is cancer: https://raw.github.com/teddziuba/teddziuba.github.com/master/_posts/2011-10-01-node-js-is-cancer.html
.. _bnoordhuis: https://github.com/bnoordhuis

.. [#] https://github.com/joyent/libuv/blob/master/include/uv.h#L1853
