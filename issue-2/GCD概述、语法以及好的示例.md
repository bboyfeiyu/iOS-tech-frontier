GCD概述、语法以及好的示例
---

>* 原文链接 : [Grand Central Dispatch (GCD): Summary, Syntax & Best Practices](http://amattn.com/p/grand_central_dispatch_gcd_summary_syntax_best_practices.html)
* 原文作者 : [amattn](http://amattn.com/p/grand_central_dispatch_gcd_summary_syntax_best_practices.html)
* [译文出自 :  开发技术前线 www.devtf.cn](www.devtf.cn)
* 译者 : [starmier](https://github.com/starmier/) 

##队列

苹果官方最初对GCD(Grand Center Dispatch)的描述是这样的：

1. 线程是复杂的
1. 通过GCD可以使得线程变得简单有趣

这两种看法都是正确的，当然还有一些其他的观点：

   .GCD 并不是一个线程库或者包装线程
   .GCD 可以使用线程，但是开发人员并不能直接控制它
   .GCD 是一个通过FIFO队列方式实现的并发库
   .GCD 只是libdispath的一种通称;#include <dispatch/dispatch.h>
   
##提交block到队列

  GCD的主要机制是通过提交block块到队列，或者响应被弹出队列的事件。有不同的方式可以创建block，也有不同类型的队列，他们中的一些非常有趣。最终，调度执行任务或者执行任务来响应事件。
  
  并发方面需要开发者手动处理。线程管理会自动调谐系统负载。通常的并发的危险是：所有UI更新必须在主线程中完成，可以搜索文件/谷歌来确认NS或UI是否是线程安全的。
 
  这篇文章主要着重“创建block块队列”，但是开发者需要意识到libdispatch底层的机制：
  
  
    -调度组                                 //协调队列中不同的线程组
    
    -信号量                                 //传统的计数信号量
    
    -队列屏障                               //给定并发队列中的的同步任务
    
    -调度源                                 //处理低级别事件的事件
    
    -调度 I/O                               //基于文件说明符的操作
    
    -调度数据缓冲区                        //内存的数据缓冲区


##创建或者获取队列

  
  这是值得重复强调的问题:使用GCD的主要机制是提交任务到队列。

  理解队列的最好方法是要认识到在底层实现中，队列只有两种类型：串行队列和并行队列
  
  串行队列是单行线的，但是不受约束。如果你添加一堆任务到每个串行队列,通过一个线程就可以同时启动这些任务。没有承诺的意思是,串行队列会切换到不同任务的线程。串行队列总是等待一个任务完成再执行下一个。即任务按照FIFO的形式执行。你可以通过dispatch_queue_create创建你需要的串行队列。
  

  主队列是一种特殊的串行队列。不像其他串行队列,没有承诺的,在某一个时间他们只“约会”许多线程中的一个,主队列“嫁给”了主线程，用来执行所有任务。主队列中的任务要在runloop中优雅的运行，这样小的行动不要阻塞UI和其他重要的部分。像所有的串行队列、任务以先进先出的顺序完成。你通过dispatch_get_main_queue可以得到它。


  如果串行队列是一夫一妻制,那么并发队列是滥交。他们将任务提交给任何可用的线程甚至根据系统负载来创建新的线程。同时他们在不同的线程执行不同的任务。任务提交全局队列中的任务是线程安全的和少副作用是非常重要的。任务提交执行的顺序是先进先出,但完成的顺序是没有保证的。
  
  
  在Mac OS X 10.6和iOS 4中,只有三个,内置(全局)并发队列,你不能获取他们,你只能用dispatch_get_global_queue获取它们。Mac OS 10.7和iOS 5,您可以创建他们，通过方法dispatch_queue_create(“label”,DISPATCH_QUEUE_CONCURRENT)。你不能为你自己创建的并发队列设置优先级。在实践中,往往更多的使用设定了优先级的全局并发队列而不是自己创建。
 
　　
  用于创建或获取队列的方法进行了总结:
　　

    dispatch_queue_create       // create a serial or concurrent queue
    dispatch_get_main_queue     // get the one and only main queue
    dispatch_get_global_queue   // get one of the global concurrent queues
    dispatch_get_current_queue  // DEPRECATED
    dispatch_queue_get_label    // get the label of a given queue
    dispatch_get_current_queue快速指南:这个方法已经被弃用,它也并不总是适用每一个案例。如果你的实现需要这个,那么你应该重构实现它。最常见的用例是“运行一些block在正在运行的任何队列上”。重构设计应该传递一个有明确的目标的队列块作为参数或参量,而不是试图依靠运行时确定提交哪些队列。

##添加任务到队列

  如果你有创建你自己的队列,你可以添加任务到队列中。
  

  主要实现机制如下:
　　
    // Asynchronous functions
    dispatch_async
    dispatch_after
    dispatch_apply
    // Synchronous functions
    dispatch_once
    dispatch_sync
    dispatch_async 将提交一个任务队列并立即返回。
    dispatch_after 方法执行会立即返回,但有些延迟，直到指定的时间内完成了任务提交。
    dispatch_apply也立即返回并多次提交任务。
    dispatch_sync将提交一个任务队列,当任务完成时返回。
    dispatch_once将提交一个任务一次,并且在应用程序生命周期中只提交一次,当块任务完成时返回。

  在实践中,我发现自己使用dispatch_async 、dispatch_after和dispatch_once最多。
  
　　
  示例代码:
　　

    // add ui_update_block to the main queue
    dispatch_async(dispatch_get_main_queue(), ui_update_block);

    // add check_for_updates_block to some_queue in 2 seconds
    dispatch_after(dispatch_time(DISPATCH_TIME_NOW, 2 * NSEC_PER_SEC), some_queue, check_for_updates_block);
    // add work_unit_block to some_queue i times.
    dispatch_apply(i, some_queue, work_unit_block);
    // perform the only_once_block once and only once. 
    static dispatch_once_t onceToken = 0; // It is important this is static!  
    // wait for completion
    dispatch_once(&onceToken, only_once_block);
    
    // add blocking_block to background_queue & wait for completion
    dispatch_sync(background_queue, blocking_block);


##队列内存管理

  GCD第一次出现在Mac OS X10.6和iOS 4系统中。当时,GCD对象(队列、信号量、队列屏障等)和CFObjects一样，在使用过程中，需要开发者调用dispatch_release和dispatch_retain进行管理。

  在Mac OS X 10.8和iOS 6中,GCD对象是由自动引用计数进行管理,而手动引用计数则被明确禁止。

　　
  此外,对于ARC有以下几点说明:
　　
1. 如果你在block块中使用的是GCD对象,可能会造成循环引用。使用__weak或显式地破坏对象(通过dispatch_source_cancel等机制)可以很好的解决这个问题。Xcode 4.6中的静态分析器捕获不到这一点。例子:
　　

        // Create a GCD object:
        dispatch_queue_t someQueue = dispatch_queue_create("someQueue", nil);
        // put a block on the queue, the queue retains the block.
        dispatch_async(someQueue, ^{
            // capture the GCD object inside the block,
            // the block retains the queue and BAM! retain cycle!
            const char *label = dispatch_queue_get_label(someQueue);
            NSLog(@"%s", label);
        });
        
        
        // You can use the typical __weak dance to workaround:
        __weak dispatch_queue_t weakQueue = someQueue;
        dispatch_async(someQueue, ^{
            __strong dispatch_queue_t strongQueue = weakQueue;
            const char *label = dispatch_queue_get_label(strongQueue);
            NSLog(@"%s", label);
        });
        

2. 最后,说明淹没珍宝的巨人dispatch_data_create_map。使用GCD方法dispatch_data_create_map和dispatch_data_apply创建内部对象时，需要额外的小心。如果父GCD对象被释放,那么内部对象会被变成野指针,将会有不好的事情发生。__block关键字或objc_precise_lifetime修饰父dispatch_data_t对象，可以帮助保持父对象的生命周期。


     	  // dispatch_data_create_map returns a new GCD data object.
     	  // However, since we are not using it, the object is immediately
        // destroyed by ARC and our buffer is now a dangling pounter!
     	  dispatch_data_create_map(data, &danglingBuffer, &bufferLen);
    	
     		// By stashing the results in a __strong var, our buffer
     		// is no longer dangerous.
     		__strong dispatch_data_t newData = dispatch_data_create_map(data, &okBuffer, &bufferLen);


##队列实例
  队列,和其他强大的工具一样,如果使用不当会导致严重的问题。现实世界中使用需要一些纪律。
  这里有一些常规准则:

　　
  * 使用主队列时，应该限制请求主线程，并且要避开耗时操作，以防止UI卡顿。
  * 创建的每一个串行队列应该有一个目的。
  * 创建的每一个串行队列应该根据目标被合理的命名/标记。
  * 并发队列中执行的任务必须是线程安全的。
  

  上面的描述的第二条应该得到进一步研究。因为队列是轻量级的,你可以创建很多很多。最好是有许多专门的串行队列，而不是填充许多不连续的任务到一个或两个“超级”串行/并行队列。
 
dispatch 实践的典型用法：

        //used for importing into Core Data so we don't block the UI
        dispatch_queue_create("com.yourcompany.CoreDataBackgroundQueue", NULL);
        
        //used to prevent concurrent access to Somefile
        dispatch_queue_create("com.yourcompany.SomeFile.ReadWriteQueue", NULL);
        
        //used to perform long calculations in the the background
        dispatch_queue_create("com.yourcompany.Component.BigLongCalculationQueue", NULL);
    

  实际中，我们通常嵌套使用队列：
         
         dispatch_queue_t background_queue = dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_DEFAULT, NULL);
        dispatch_async(background_queue, ^{
            // do some stuff that takes a long time here...
        
            // follow up with some stuff on the main queue
            dispatch_async(dispatch_get_main_queue(), ^{
                // Typically updating the UI on the main thread.
            });
        });

这时，我们开启了一个在后台执行的任务队列。当任务完成后，我们在主线程中触发UI更新操作。
	  
另外，需要强调的是，这个dispatch方法（dispatch_get_global_queue），不可过度使用。它会影响代码的可读性和可维护性，被认为是比较尖锐的代码。（何为尖锐？就是小心、谨慎地使用，否则出了问题，你根本找不着原因。我的一个项目里就是因为使用了至少3个global_queue，出现了crash，后来，我全改为了自定义dispatch_queue_t。当然，使用环境不一样，也不一定会出现我这样的问题。）

##深入学习
如果你对GCD每个方面(调度组、信号量、队列屏障等)都特别感兴趣，请邮件我，我会分享出来更多内容。

同时，应用通常的知识点、查阅网络文档与通过Xcode或是在WWDC中讨论GCD和块都是一样的



