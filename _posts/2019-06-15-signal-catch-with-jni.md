---
layout: post
title:  JNI/JNA + Linux的信号量捕捉异常
date:   2019-06-15
categories: 
  - jni
  - linux
comments: false
---

> 继续上次的股市量化项目，由于需要和各种券商对接，而量化项目大多都是c/c++项目,在对接的过程中各种奇葩事情都遇到了,其中最坑爹的就是JNI下的SIGSEGV信号量抛出问题了
 
<!-- more -->

第一次接触JNI应该是在2015年，因为对于Java源代码严格保密的需求，第一次在java中使用JNI，虽然磕磕碰碰，不过好歹最终都达到了预期的结果，参考 [有哪些防止反编译 Java 类库 jar 文件的办法？ - ShadowSong的回答 - 知乎](https://www.zhihu.com/question/19766494/answer/132496394) ,后续在Android编程中也有接触，不过都很顺溜。

#### callback

callback可以大大的减少轮询的消耗，网络编程中经常使用，由于JNI本身支持类似于反射的机制，所以支持callback非常简单：

 - 定义callback的接口规范
 - 设置回调接口，将回调的接口实现传入JNI
 - JNI通过NewGlobalRef创建对java对象的引用
 - JNI通过传入的env的GetObjectClass和GetMethodID能够获取回调的方法反射信息
 - JNI通过CallVoidMethod（或者其他的调用）通过类似于反射的机制调用java方法

其中需要注意的是：

 - 及时回收中间变量，避免内存泄露
 - JNI在Call的时候占用的是JNI的线程，需要尽快回收，否则可能会导致JNI中的线程阻塞

参考样例：

```
//注册回调
JNIEXPORT jint JNICALL Java_com_cli_api_setCallback
  (JNIEnv *env, jobject obj, jlong handle, jobject callback)
{
  int iRetCode = 1;
  // process
  
   callbackHolder->env = env;
   callbackHolder->obj = env->NewGlobalRef(callback);
   callbackHolder->cls = env->GetObjectClass(callbackHolder->obj);
   callbackHolder->mid = env->GetMethodID(callbackHolder->cls, "CallBack", "([B[BI)V");
  
  // other process
  return iRetCode;
}

//执行回调
callbackHolder->env->CallVoidMethod(callbackHolder->obj, callbackHolder->mid, argus);
```

#### Signal

c/c++的信号量处理还是很方便的，主要用来做诊断和优雅退出之类的实现，参考：

```
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <signal.h>
// 创建回调函数
void signal_callback_handler(int signum) {
    printf("Caught signal %d\n", signum);
}

int main() {
    // 设置信号回调函数，运行程序后键入ctrl + C回调signal_callback_handler函数
    signal(SIGINT, signal_callback_handler);
    while (true) {
        printf("Program processing stuff here.\n");
        sleep(1);
    }
    return EXIT_SUCCESS;
}
```

而我们这次就是栽在了这块上面，c++由于有非常严格的编程规范，所以通过信号机制订阅了SIGSEGV以便获取所有的空指针异常并且打印日志，然后退出进程，方便定位异常位置。如果是纯粹的c/c++是完全没有问题的,在windows下也是ok的,然后在Linux+java下面,则GG了.

具体参考文档: [Integrating Signal and Exception Handling](https://docs.oracle.com/javase/7/docs/webnotes/tsg/TSG-VM/html/signals.html) : 在Linux下jvm会在内部空指针的时候,抛出SIGSEGV信号量,即使你在jvm内部通过catch进行了空指针异常的捕获和处理。当然如果不主动进行信号量的处理，也不会出问题，然而我们的业务方均进行了处理，最终的结果就是我们的应用总是在启动之后随机的时间之后莫名退出，退出堆栈参考：

```
##### begin >>> dump signal [11] backtrace <<< begin #####
  [14] frames as follows:
  [00] /***.so(_Z14dump_backtraceP8_IO_FILE+0x1f) [0x7f87937c25af]
  [01] /***.so(_Z19dump_signal_handleri+0xee) [0x7f87937c272e]
  [02] /lib64/libc.so.6(+0x36280) [0x7f87f90b1280]
  [03] /export/service/jdk/jdk1.8.0_211/jre/lib/amd64/server/libjvm.so(+0x3eca75) [0x7f87f847ea75]
  [04] /export/service/jdk/jdk1.8.0_211/jre/lib/amd64/server/libjvm.so(+0x32f82a) [0x7f87f83c182a]
  [05] /export/service/jdk/jdk1.8.0_211/jre/lib/amd64/server/libjvm.so(+0x32fb40) [0x7f87f83c1b40]
  [06] /export/service/jdk/jdk1.8.0_211/jre/lib/amd64/server/libjvm.so(+0x3302c8) [0x7f87f83c22c8]
  [07] /export/service/jdk/jdk1.8.0_211/jre/lib/amd64/server/libjvm.so(+0x48b92c) [0x7f87f851d92c]
  [08] /export/service/jdk/jdk1.8.0_211/jre/lib/amd64/server/libjvm.so(+0x48d568) [0x7f87f851f568]
  [09] /export/service/jdk/jdk1.8.0_211/jre/lib/amd64/server/libjvm.so(+0xa7ba9b) [0x7f87f8b0da9b]
  [10] /export/service/jdk/jdk1.8.0_211/jre/lib/amd64/server/libjvm.so(+0xa7bda1) [0x7f87f8b0dda1]
  [11] /export/service/jdk/jdk1.8.0_211/jre/lib/amd64/server/libjvm.so(+0x90d952) [0x7f87f899f952]
  [12] /lib64/libpthread.so.0(+0x7dd5) [0x7f87f986bdd5]
  [13] /lib64/libc.so.6(clone+0x6d) [0x7f87f9178ead]
#####   end >>> dump signal [11] backtrace <<< end   #####
```

在定位问题的时候真的是让人难受，到处查找原因，最后才发现是这里，而且也很难处理，要么只能放过这个信号量不处理，要么还必须检查堆栈，如果是java抛出来的这个信号量则不处理。