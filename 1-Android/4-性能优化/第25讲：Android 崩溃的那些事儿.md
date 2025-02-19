关键词：

异常捕获、异常处理、UncaughtExceptionHandler 、Bugly/XCrash/Sentry



如果要给 crash 日志进行分类，可以分成 2 大类：JVM 异常（Exception）堆栈信息、native 代码崩溃日志。

#### JVM 异常堆栈信息

Java 中异常（Exception）分两种：检查异常 checked Exception 和非检查异常 unchecked Exception。

所谓检查异常就是在代码编译时期，Android Studio 就会提示代码有错误，无法通过编译，比如 IOException。如果我们没有在代码中将这些异常 catch，而是直接抛出，最终也有可能导致程序崩溃。

非检查异常包括 error 和运行时异常（RuntimeException），AS 并不会在编译时期提示这些异常信息，而是在程序运行时期因为代码错误而直接导致程序崩溃，比如 OOM 或者空指针异常（NPE）。

##### Java 异常

对于上述两种异常我们都可以使用 UncaughtExceptionHandler 来进行捕获操作，它是 Thread 的一个内部接口，定义如下：

![image](images/第25讲：Android 崩溃的那些事儿/Ciqc1F7ofWiAV2aBAAEGhcbtn98977.png)

从官方对其介绍能够看出，对于传入的 Thread，如果因为“未捕获”异常而导致被终止，uncaughtException 则会被调用。我们可以借助它来间接捕获程序异常，并进行异常信息的记录工作，或者给出更友好的异常提示信息。

#### 自定义异常处理类

自定义类实现 UncaughtExceptionHandler 接口，并实现 uncaughtException 方法：

![image](images/第25讲：Android 崩溃的那些事儿/Ciqc1F7ofXaAcZ7MAAK-ubwIhZk797.png)

需要注意的几点：

在自定义异常处理类中需要持有线程默认异常处理类，如图中红框处所示。这样做的目的是在自定义异常处理类无法处理或者处理异常失败时，还可以将异常交给系统做默认处理。

如果自定义异常处理类成功处理异常，需要进行页面跳转，或者将程序进程“杀死”。否则程序会一直卡死在崩溃界面，并弹出无响应对话框。如上图所示，我使用 CrashDisplayActivity 页面来展示所捕获的异常详细信息。

一般情况下，在 handlerException 方法中需要做以下几件事情。

1.收集 crash 现场的相关信息，如下面方法获取当前 App 的版本信息，以及所有手机设备的相关信息。

![image](images/第25讲：Android 崩溃的那些事儿/CgqCHl7ofYSAeFGcAAJnMJd645o410.png)

实际上，除了上述信息，还可以添加额外的自定义信息。

2.日志的记录工作，将收集到的信息保存在本地，比如以文件的方式保存。

![image](images/第25讲：Android 崩溃的那些事儿/Ciqc1F7ofYuAEnnPAAGeBy97jW8168.png)

从图中可以看出，除了我们自己收集的日志，还需要将系统抛出的异常信息也保存到文件中，方便后续开发人员分析问题原因。

### 使用自定义异常处理类

LagouExceptionHandler 定义好之后，就可以将其初始化，并将主线程注册到 LagouExceptionHandler 中。如下：

![image](images/第25讲：Android 崩溃的那些事儿/CgqCHl7ofZmAVDhnAACUY1wtWYQ757.png)

最终保存的 crash 日志信息格式下图所示：

![image](images/第25讲：Android 崩溃的那些事儿/Ciqc1F7ofaCAV-N0AAK1uzObDPg259.png)

> 需要注意的是，因为使用了文件写操作，所以需要动态申请文件操作的权限。

## native 异常

当程序中的 native 代码发生崩溃时，系统会在 /data/tombstones/ 目录下保存一份详细的崩溃日志信息。如果一个 native crash 是必现的，不妨在模拟器上重现 bug，并将 /data/tombstones 中的崩溃日志拉到本地电脑中加以分析。

比如创建一个模拟 native crash 的项目 LagouNativeCrash，项目结构如下：

![image](images/第25讲：Android 崩溃的那些事儿/Ciqc1F7ofa2AOQEoAABoHjKPFO4484.png)

MainActivity 中有一个点击按钮 Button，当点击此按钮时，会调用 crash() 方法触发 native 代码崩溃，如下：

![image](images/第25讲：Android 崩溃的那些事儿/Ciqc1F7ofbWACu1SAABHFSC0bhA498.png)

native crash 在 native-lib.cpp 文件中声明，如下：

![image](images/第25讲：Android 崩溃的那些事儿/CgqCHl7ofbyAMpx-AADl8pNCN4o838.png)

当点击 Button，触发 native 崩溃之后，系统会在 /data/tombstones 目录下生成 tombstone 日志文件，可以在此日志文件中，查看详细的报错信息。如下所示：

![image](images/第25讲：Android 崩溃的那些事儿/Ciqc1F7ofcWAPr9kAASbO90geqo412.png)

但是如果 native crash 是偶发现象，并且在模拟器上一时难以复现，那么就需要将工作交给测试工程师在真机上尝试复现。那就需要一种机制，将 native crash 现场的日志信息保存到我们可以访问的手机目录中。目前比较成熟，使用也比较广泛的就是谷歌的 BreakPad。

Breakpad 是一个跨平台的开源库，我们也可以在其 [Breakpad Github]()https://github.com/google/breakpad 上下载自己编译。并通过 JNI 的方式引入到项目中。

### NDK 导入 BreakPad

在 Breakpad GitHub 官网上，有一个 README Android 的介绍文件。这个文件专门介绍了如何在 Android 项目中导入 BreakPad。我们可以直接使用 CMake 方式将其编译为一个静态库。

![image](images/第25讲：Android 崩溃的那些事儿/CgqCHl7ofdmAexkGAAQt9jC9U2g259.png)

在捕获 native crash 之前，需要初始化 Breakpad，主要是设置 BreakPad 保存 crash 日志的路径，如下所示：

![image](images/第25讲：Android 崩溃的那些事儿/Ciqc1F7ofeOAOmfAAAI9QS6RESI036.png)

图中传入的 path 就是 Breakpad 保存日志的文件目录，一般情况下保存在外置 SD 卡目录。

初始化好之后，就可以在我们自己的 native 业务层模拟一个崩溃现场，Breakpad 会自动捕获这次 crash，并将生成的 crash 信息保存在所设置的目录中。

![image](images/第25讲：Android 崩溃的那些事儿/CgqCHl7ofeuAJlINAADteaIIElo908.png)

breakpad 生成的文件是 .dmp 文件，需要将其行转换，如下所示：

> minidump_stackwalk 生成的 crash.dmp > result.txt
>

上述命令是将 breakpad 生成的 dmp 文件转化为 .txt 文件，然后使用文本编辑器打开此 txt 文件，就可以看出代码中是哪一行发生崩溃。

对于程序崩溃信号机制的介绍，可以参考腾讯的这篇文章：[Android 平台 Native 代码的崩溃捕获机制及实现](https://mp.weixin.qq.com/s/g-WzYF3wWAljok1XjPoo7w?)。

### 线上崩溃日志获取

上面介绍的 Java 和 Native 崩溃的捕获都是基于现场能够复现 bug 的前提下。但是对于线上的用户，这种操作方式是不太现实的，首先我们不可能将一个 debug 版本的 APK 安装到终端用户的手机上；另外也不太可能要求用户操作一遍手机并将某一目录下的文件发送给开发人员。

对于大多数公司来说，针对线上版本，没有必要自己实现一个抓取 log 的平台系统。最快速的实现方式就是集成第三方 SDK。目前比较成熟，采用也比较多的就是腾讯的 Bugly。Bugly 基本能够满足线上版本捕获 crash 的所有需求，包括 Java 层和 Native 层的 crash 都可以获取相应的日志。并且每天 Bugly 都会邮件通知上一天的崩溃日志，方便测试和开发统计 bug 的分布以及崩溃率。

除了 Bugly 之外，还有一些其他的 crash 上报工具。比如 XCrash 和 Sentry。这两者比 Bugly 好的地方就是除了自动拦截界面崩溃事件，还可以主动上报错误信息。以 XCrash 为例，基本使用如下所示：

![image](images/第25讲：Android 崩溃的那些事儿/CgqCHl7offuAb-QwAANxaVWOCd8915.png)

可以看出 XCrash 的使用更加灵活，工程师的掌控性更高。可以通过设置不同的过滤方式，针对性地上报相应的 crash 日志。并且在捕获到 crash 之后，可以加入自定义的操作，比如本地保存日志或者直接进行网络上传等。

> 另外：Sentry 还有一个好处就是可以通过设置过滤，来判断是否上报 crash 日志。这对于 SDK 的开发人员是很有用的。比如一些 SDK 的开发商只是想收集自身 SDK 引入的 crash，对于用户的其他操作导致的 crash 进行过滤，这种情况就可以考虑集成 Sentry。

### 总结

本文主要介绍了 Android 崩溃的相关知识。对 Android 工程师来说，crash 可以分 2 类：Java 层和 Native 层。针对这 2 者进行捕获的方式也不尽相同。针对 Java 层一般通过自定义 UncaughtExceptionHandler 进行异常拦截；针对 Native 层可以考虑集成谷歌的 breakpad 进行捕获，并保存日志在本地。最后介绍了几个线上捕获 crash 的工具，实际上这几个工具的实现原理都是基于上文介绍的基础知识。



# 参考

1、[拉勾教育 Android 工程师进阶 34 讲](https://kaiwu.lagou.com/course/courseInfo.htm?courseId=67#/content?courseId=67&isShowSections=true)