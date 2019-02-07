---
layout: post
title:  Using Semephore for synchronizing application state
#subtitle: Each post also has a subtitle
#image: /img/hello_world.jpeg
tags: [threads, semephore, locks]
---
I have a multi threded application. The application has multi-phase processiing model. A submitted request goes through a validation phhase first. Post the validation the request is routed to a processing engine for fetching data. The application validtion has static data validation , to check for null data as well as dynamic check to throttle back user submitted execution requests. Specifically in my case the *throtlling* validator must know the state of executions happening (performed by the processing engine) at the moment by a particular user. Basically the *validation phase* will pass a request. This request, if successful, then reaches the *processing phase* which then adds it to the execution list.

Now this design required a state synchroniztion model. Our server makes a thread pool for every phase. So our model should be such that it works with diffrent threads. A new request by a user must wait to know if it is still within the prescribed limits. This could only be accomplished when we synchronize states between the *validation* and *processing* phases.

So I started to look at `java.util.concurrent.locks.Lock`. I wrote the following test case to simulate our usercase :
```
class Value {
    Lock lock = new ReentrantLock();

    void lock() {
        lock.lock();
    }

    void release() {
        lock.unlock();
    }
}

public class TestTH {
    @Test
    public void test1() throws Exception {
        Value res = new Value();
        CountDownLatch l1 = new CountDownLatch(1);
        new Thread(() -> {
            res.lock();
            l1.countDown();
        }).start();

        CountDownLatch l2 = new CountDownLatch(1);
        new Thread(() -> {
            try {
                l1.await();
            } catch (InterruptedException e) {
                e.printStackTrace();
            }
            res.release();
            l2.countDown();
        }).start();
        l2.await();
    }
}
```

When I ran the test it failed with the following exception :

```
Exception in thread "Thread-1" java.lang.IllegalMonitorStateException
	at java.util.concurrent.locks.ReentrantLock$Sync.tryRelease(ReentrantLock.java:151)
	at java.util.concurrent.locks.AbstractQueuedSynchronizer.release(AbstractQueuedSynchronizer.java:1261)
	at java.util.concurrent.locks.ReentrantLock.unlock(ReentrantLock.java:457)
	at com.assessmatic.test.service.Value.release(TestTH.java:17)
	at com.assessmatic.test.service.TestTH.lambda$test1$1(TestTH.java:38)
	at java.lang.Thread.run(Thread.java:748)
```
So I looked at the lock API again. I found out that *lock* can be acquired by a single thread. The thread acquiring it must also release the thread. Any other thread trying to do so will lead to exception.

We tred to hack our way by creating new lock in the release method as follows. But there were couple of issues with this. :
- Waiting threads will never get notification.
- There is a big leak in locking solution !
```
class Value {
    Lock lock = new ReentrantLock();

    void lock() {
        lock.lock();
    }

    void release() {
        lock = new ReentrantLock ();
    }
}
```

I looked a little further and found `java.util.concurrent.Semaphore`. Semaphore can be ues to impleamnt mutex scenarios betwenn different threads. the api has terminology of `permits`. A thread can consumer the permits while the other thread can work towards restoring the permit.
```
class Value {
    Semaphore lock = new Semaphore(1);

    void lock() {
        try {
            lock.acquire();
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
    }

    void release() {
        lock.release();
    }
}
```
Now I ran my testcase without any change. The testcase passes with a green bar. Adapting the same model I locked the permit in validator and released the permit in my processing engine.
