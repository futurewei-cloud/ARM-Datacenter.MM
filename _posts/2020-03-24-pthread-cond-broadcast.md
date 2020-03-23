---
title: "Understanding pthread_cond_broadcast"
date: 2020-03-23T06:25:34-04:00
author: Rob
categories:
  - QEMU
tags:
  - QEMU Debugging
classes: wide
---
Recently we came across a piece of code in QEMU using the [pthread_cond_broadcast](https://manpages.debian.org/testing/glibc-doc/pthread_cond_broadcast.3.en.html) function.

This method is intended to wake up all threads waiting on a condition variable.  However, the method needs to be used with care.  In particular, it should only be used if you can guarantee:
a) that the waiter is in fact waiting
or
b) that there is another mechanism to wake up the waiter if the broadcast signal arrives when the thread is not waiting

For example, suppose we have the following code:

~~~
pthread_mutex_lock(&first_cpu->lock);
while (first_cpu->stopped) {
    pthread_cond_wait(first_cpu->halt_cond, first_cpu->lock);
    pthread_mutex_unlock(&first_cpu->lock);

    /* process any pending work */
    pending_work();    
    pthread_mutex_lock(&first_cpu->lock);
}
pthread_mutex_unlock(&first_cpu->lock);
~~~
 
Also suppose we have another thread which will call to pthread_cond_broadcast() to wakeup this thread.

If the above thread is waiting in pthread_cond_wait() when it is woken up by pthread_cond_broadcast(), then all is well.

However, if this thread is outside of the pthread_cond_wait() in the loop when pthread_cond_broadcast() is called, then this thread will not be woken up.  In other words, when the thread loops around to pthread_cond_wait() it will *NOT* wait.

This means that either we need to guarantee the thread is waiting when the broadcast is sent OR we need to make sure that there is another way to wakeup the thread.

One other option is to change the pthread_cond_wait() to a pthread_cond_timedwait() to ensure that we will periodically perform this "pending_work(), even if the pthread_cond_brodcast() is missed.
