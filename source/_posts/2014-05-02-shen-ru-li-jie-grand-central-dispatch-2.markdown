---
layout: post
title: "深入理解Grand Central Dispatch:第二部分"
date: 2014-05-02 16:18:04 +0800
comments: true
categories: 翻译
---

欢迎来到第二部分教程，本篇教程我们将更加深入的理解GCD。

在前一篇教程我们学习了并发，线程和GCD是如何运行的。我们使用**dispatch_once**初始化**PhotoManager**单例来确保线程安全，通过使用**dispatch_barrier_async**和**dispatch_sync**来确保读写**Photos**时线程安全。

除此之外，我们使用**dispatch_after**控制提示时间，增强了应用的用户体验。同时通过使用**dispatch_async**来执行CPU密集型任务来提高生成一个视图控制器实例效率。

如果你按步骤学习了上篇教程，你可以重新打开在第一篇教程中的示例工程文件。如果没有的话可以从[点击这里下载](http://cdn2.raywenderlich.com/wp-content/uploads/2014/01/GooglyPuff_End_1.zip)。

###纠正下载提醒框弹出时间

或许你已经注意到，当你通过**Le Internet**来添加图片时，在图片还未下载完时就弹出了提醒框，如下图所示：

![Screen-Shot-2014-01-17-at-5.49.51-PM-308x500](/images/GCD/Screen-Shot-2014-01-17-at-5.49.51-PM-308x500.png)

这个错误出现在**PhotoManagers**类中的**downloadPhotoWithCompletionBlock:**方法中，如下面代码所示：

```
- (void)downloadPhotosWithCompletionBlock:(BatchPhotoDownloadingCompletionBlock)completionBlock
{
    __block NSError *error;
    for (NSInteger i = 0; i < 3; i++) {
        NSURL *url;
        switch (i) {
            case 0:
                url = [NSURL URLWithString:kOverlyAttachedGirlfriendURLString];
                break;
            case 1:
                url = [NSURL URLWithString:kSuccessKidURLString];
                break;
            case 2:
                url = [NSURL URLWithString:kLotsOfFacesURLString];
                break;
            default:
                break;
        }
        Photo *photo = [[Photo alloc] initwithURL:url
                              withCompletionBlock:^(UIImage *image, NSError *_error) {
                                  if (_error) {
                                      error = _error;
                                  }
                              }];
        [[PhotoManager sharedManager] addPhoto:photo];
    }
    if (completionBlock) {
        completionBlock(error);
    }
}
```

在你认为所有图片已经下载完的地方即在上面这个函数的结尾处调用了**completionBlock**。关键问题是，你没办法确定所有图片的确是在这个时候下载完的。

当**Photo**类的实例调用该方法开始下载图片时，在下载完成之前该方法就已经返回了。换句话说，**downloadPhotoWithCompletionBlock:**在方法结尾处调用**completionBlock**就好比该函数中的代码都是线性运行的，每个方法都会在前一个方法完成后运行。

然而，**-[Photo initWithURL:withCompletionBlock:]**是异步的并且会立即返回，所以上面的方法是行不通的。

所以，**downloadPhotoWithCompletionBlock:**应该在所有图片调用他们自己的下载完成的块方法中调用**completionBlock**。现在的问题是，你如何监控并发的异步事件的，你不知道他们什么时候完成，因为他们可以按任意顺序完成。

也许你可以使用很多的**BOOLs**跟踪每一个下载，但这是一种相当烂的代码并且非常不易于扩展。

幸运的是，**dispatch groups**正是为监控多个异步任务完成的情况所设计的。

####Dispatch Groups

**Dispatch groups**会在组内所有的任务完成时通知你。这些任务可以是异步的或者同步的甚至可以监控来自不同的队列的任务。当监控不同队列中任务时，**dispatch_group_t**将会在不同队列中跟踪的不同任务。

当组内的任务完成时，GCD的API提供了两种方式来进行监听。

第一种方式是**dispatch_group_wait**，这个方法是通过阻塞你当前的线程，并且直到组内所有任务完成或者发生超时才会通知你。这种方法或许正是我们所需要的。

打开**PhotoManager.m**，使用下面的代码来替换**downloadPhotosWithCompletionBlock:**：

```
- (void)downloadPhotosWithCompletionBlock:(BatchPhotoDownloadingCompletionBlock)completionBlock
{
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{ // 1 
        __block NSError *error;
        dispatch_group_t downloadGroup = dispatch_group_create(); // 2
        for (NSInteger i = 0; i < 3; i++) {
            NSURL *url;
            switch (i) {
                case 0:
                    url = [NSURL URLWithString:kOverlyAttachedGirlfriendURLString];
                    break;
                case 1:
                    url = [NSURL URLWithString:kSuccessKidURLString];
                    break;
                case 2:
                    url = [NSURL URLWithString:kLotsOfFacesURLString];
                    break;
                default:
                    break;
            }
            dispatch_group_enter(downloadGroup); // 3
            Photo *photo = [[Photo alloc] initwithURL:url
                                  withCompletionBlock:^(UIImage *image, NSError *_error) {
                                      if (_error) {
                                          error = _error;
                                      }
                                      dispatch_group_leave(downloadGroup); // 4
                                  }];
            [[PhotoManager sharedManager] addPhoto:photo];
        }
        dispatch_group_wait(downloadGroup, DISPATCH_TIME_FOREVER); // 5
        dispatch_async(dispatch_get_main_queue(), ^{ // 6
            if (completionBlock) { // 7
                completionBlock(error);
            }
        });
    });
}
```

来看下下面对每段代码的解释：

1. 当你使用同步方法**dispatch_group_wait**来阻塞当前进程时，你需要使用**dispatch_async**来将整个方法放入到后台队列中执行以确保没有阻塞当前的主线程。
2. 这段代码是创建一个调度组好比一个包含很多未完成任务的容器。
3. **dispatch_group_enter**手动通知一组任务已经开始，**dispatch_group_enter**和**dispatch_group_leave**必须成对调用，否则会导致程序的崩溃。
4. 该处是手动通知组内任务已经完成，同时你需要平衡**dispatch_group_enter**和**dispatch_group_leave**的数量。
5. **dispatch_group_wait**会在所有任务完成或者超时后执行。如果在所有任务完成前发生了超时，那么这个方法将会返回一个非零值。你可以把这部分放到一个条件块中去检查是否超时，但是在这里我么可以使用**DISPATCH_TIME_FOREVER**，因为这些图片的下载总是会完成的。
6. 此时你可以确保所有的图片都已经下载完了。然后回到主线程调用该方法的**completionBlock**。主线程会在稍后执行该部分代码。
7. 最后检查**completionBlock**是否为空，如果不为空就返回。

编译并运行程序，尝试下载多张图片并注意观察程序是如何处理图片下载完成后的操作的。

> 注意：如果你在真机上运行而且网络速度很快以至于没有观察到完成后的操作，你可以通过调节在设置中的开发者选项，选择非常糟糕的网速来进行查看。
如果你是在模拟器上运行，可以通过这个（[network link conditioner](http://nshipster.com/network-link-conditioner/)）来改变网速。这是个很好的工具并且会迫使你去考虑网速不好的情况下程序运行状态。

这个解决方案已经很不错了，但是通常在允许的情况下我们会尽量避免阻塞线程的。你下面的任务是重写这个方法并在所有的下载任务完成时进行异步通知。

在我们开始使用另外一种**dispatch groups**时，先简要说明下在各种队列中调用**dispatch groups**相关指南：

* **Custom Serial Queue**:当组内任务完成时，在该队列中进行通知是个很好的选择。
* **Main Queue (Serial)**:这也是个不错的选择，但是在同步等待任务完成时你应该小心避免阻塞主线程。然而在等待例如网络连接一些长任务完成时，异步模型对于更新UI界面是个不错的选择。
* **Concurrent Queue**:这个对于调用**dispatch groups**并获取完成时的通知同样是个不错的选择。

####第二种Dispatch groups方法

上一个方法已经很好了，但是有些麻烦我们不得不异步调度到另一个队列中然后使用**dispatch_group_wait**阻塞线程。这里还有另外一个办法：

使用下面的代码替换**PhotoManager.m**文件中的**downloadPhotosWithCompletionBlock:**方法

```
- (void)downloadPhotosWithCompletionBlock:(BatchPhotoDownloadingCompletionBlock)completionBlock
{
    // 1
    __block NSError *error;
    dispatch_group_t downloadGroup = dispatch_group_create(); 
    for (NSInteger i = 0; i < 3; i++) {
        NSURL *url;
        switch (i) {
            case 0:
                url = [NSURL URLWithString:kOverlyAttachedGirlfriendURLString];
                break;
            case 1:
                url = [NSURL URLWithString:kSuccessKidURLString];
                break;
            case 2:
                url = [NSURL URLWithString:kLotsOfFacesURLString];
                break;
            default:
                break;
        }
        dispatch_group_enter(downloadGroup); // 2
        Photo *photo = [[Photo alloc] initwithURL:url
                              withCompletionBlock:^(UIImage *image, NSError *_error) {
                                  if (_error) {
                                      error = _error;
                                  }
                                  dispatch_group_leave(downloadGroup); // 3
                              }];
        [[PhotoManager sharedManager] addPhoto:photo];
    }
    dispatch_group_notify(downloadGroup, dispatch_get_main_queue(), ^{ // 4
        if (completionBlock) {
            completionBlock(error);
        }
    });
}
```

下面是代码运行步骤说明：

1. 在这个新方法中你不需要把代码放到异步调用块中，因为你不需要阻塞主线程。
2. 这个是**enter**方法，和上个方法调用的一样。
3. 这个是**leave**方法，也和之前的一样。
4. **dispatch_group_notify**作为最终的异步完成块进行调用。这部分代码是在调度组内任务都完成后才运行**completion block**的。你也可以选择在其他队列中调用完成块代码，这里是在主线程运行的。

这个方法更简明易懂而且没有阻塞任何线程。

###大量并发的危险性

这些新方法都掌握后，你是不是想在项目各个地方都使用线程呢？

![Thread_All_The_Code_Meme](/images/GCD/Thread_All_The_Code_Meme.jpg)

再看下*PhotoManager*中的*downloadPhotosWithCompletionBlock*方法，你可能注意到有个*for*循环，通过循环三次来下载三张不同的图片。你现在的任务时能否使这个*for*循环并发执行并提高它的运行速度。

*dispatch_apply*这时就该上场了。

*dispatch_apply*就像*for*循环一样可以并发执行不同的循环。这个方法是同步的，所以就像一个正常的*for*循环一样，只有在所有的任务都完成时*dispatch_apply*才会返回。

值得注意的是，对于在块中的任务我们需要指定明确适当的循环次数，因为多次循环的少量任务会导致很大的开销而并没有因并发调用而获取什么好处。被称作**striding**（跨越式）的技术可以帮助你，这里就是你在每个循环需要做的任务。----*该处翻译有问题请查看原文*

什么时候调用*dispatch_apply*更合适呢？

* **Custom Serial Queue**:串行队列是完全不能适合使用*dispatch_apply*的，使用*for*循环就可以了。
* **Main Queue (Serial)**:如上文所示，在串行队列中使用*dispatch_apply*并不是个好主意，使用*for*循环即可。
* **Concurrent Queue**:在并发循环更适合调用，尤其是当你跟踪任务任务进程时。

回到**downloadPhotosWithCompletionBlock**方法中并用下列代码替换：

```
- (void)downloadPhotosWithCompletionBlock:(BatchPhotoDownloadingCompletionBlock)completionBlock
{
    __block NSError *error;
    dispatch_group_t downloadGroup = dispatch_group_create();
    dispatch_apply(3, dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^(size_t i) {
        NSURL *url;
        switch (i) {
            case 0:
                url = [NSURL URLWithString:kOverlyAttachedGirlfriendURLString];
                break;
            case 1:
                url = [NSURL URLWithString:kSuccessKidURLString];
                break;
            case 2:
                url = [NSURL URLWithString:kLotsOfFacesURLString];
                break;
            default:
                break;
        }
        dispatch_group_enter(downloadGroup);
        Photo *photo = [[Photo alloc] initwithURL:url
                              withCompletionBlock:^(UIImage *image, NSError *_error) {
                                  if (_error) {
                                      error = _error;
                                  }
                                  dispatch_group_leave(downloadGroup);
                              }];
         [[PhotoManager sharedManager] addPhoto:photo];
    });
    dispatch_group_notify(downloadGroup, dispatch_get_main_queue(), ^{
        if (completionBlock) {
            completionBlock(error);
        }
    });
}
```
现在的循环就是并发调用了，在上面的代码中在调用**dispatch_apply**时，第一个参数需要指定循环次数，第二个参数指定在那个队列中执行，第三个参数执行循环任务。

尽管添加图片的代码是线程安全的，但是图片添加完成顺序则取决于相关线程完成顺序。

编译并运行，使用Le Internet添加图片，是否注意到有什么不同么？

有时在设备上运行这段代码会感觉更快，但是真的值得这样做么？

实际上，在上面这个例子中并不建议这样使用：

* 相对于*for*而言，使用并发线程运行可能会产生很多开销。*dispatch_apply*应该在遍历非常大的集合而且集合中的任务不是那么复杂的情况下使用。
* 启动程序的时间是有限的，不要在你不了解的代码上浪费时间去进行优化。如果你要优化的代码的好处是显而易见的，那么就值得你去优化。通过运行*Instruments*来分析你的应用找出最耗费资源的方法。通过这篇文章了解更多[How to Use Instruments in Xcode](http://www.raywenderlich.com/23037/how-to-use-instruments-in-xcode)。
* 通常来讲优化代码使你或者后来看你代码的人会感觉更为复杂，请确保程序中添加并发处理的确有必要而且是有意义的。

记住不要疯狂的对代码进行无意义的优化，这样只会使你或者后来读你代码的人感到更加困惑。

###其他有趣的GCD方法

但是等等，这里还有更多另辟蹊径的方法，尽管你不会经常使用这些方法，但是在适当情况下使用会对你的程序有很大的帮助。

####阻塞-正确的方式

这听起来像个疯狂的想法，但是你知道Xcode有测试功能么？我知道有时我们并不经常使用他们，但是对于复杂的逻辑代码写测试用例还是很有必要的。

通过运行Xcode上的**XCTestCase**子类文件中以**test**开头的方法来进行测试。测试方法是在主线程中执行的，这样你就可以假设所有的测试方法是以串行方式运行的。

一旦一个给定的测试方法执行完毕，XCTest就会认为这条测试已经完成并且会继续执行下一条测试。这就意味着在执行下条测试时上条测试的异步代码仍在继续执行。

网络连接相关的代码通常是异步的，如果在执行网络连接时你没有阻塞阻塞主线程，那就意味着当测试方法执行完时很难再对网络相关的代码进行测试。除非你在测试方法中阻塞主线程直到网络连接完成。

> 有些人或许认为这种测试方法并不属于集成测试中的优先选择，一些人同意，一些人则不同意。但是如果这样做是有用的，那就可以使用这种方法。

![Gandalf_Semaphore](/images/GCD/Gandalf_Semaphore.png)

回到**GooglyPuffTests.m**文件中，找到**downloadImageURLWithString**方法，使用下面代码进行替换：

```
- (void)downloadImageURLWithString:(NSString *)URLString
{
    NSURL *url = [NSURL URLWithString:URLString];
    __block BOOL isFinishedDownloading = NO;
    __unused Photo *photo = [[Photo alloc]
                             initwithURL:url
                             withCompletionBlock:^(UIImage *image, NSError *error) {
                                 if (error) {
                                     XCTFail(@"%@ failed. %@", URLString, error);
                                 }
                                 isFinishedDownloading = YES;
                             }];
    while (!isFinishedDownloading) {}
}
```
这是一种很幼稚的方法来测试异步网络代码。在方法末尾的*while*循环直到**isFinishedDownloading**布尔值变为*true*时即执行*completion block*时才会退出。让我们来看看这样做会什么样的影响。

点击菜单栏**Product / Test**或者**⌘+U**来运行测试代码。

在运行测试时，注意观察调试导航栏中显示的CPU的运行情况。这种不当的实现方式被称作自旋锁（**spinlock**）。这种实现方式是不合理的，你不仅在浪费CPU宝贵的时间-在等待*while*循环的结束，也无法很好的对这段代码进行扩展。

你或许可以使用前面我们介绍过的网络连接适配器-*network link conditioner*来进行调试，你会更容易发现这个问题。如果你的网络速度足够快卡顿也只会发生一会儿。

你需要一个更优化的，可扩展的方法来阻塞线程直到资源变为有效。信号量-**Semaphores**或许是个不错的选择。

####Semaphores-信号量

信号量在学校时教授Edsger W. Dijkstra相关理论时的一种线程概念。信号量是个复杂的概念，因为他们是建立在复杂的操作系统功能之上的。

如果你想了解关于信号量的更多知识，可以点击这里[link which discusses semaphore theory in more detail](http://greenteapress.com/semaphores/)。如果你是学术派，那肯定了解使用信号量解决的一个经典的软件开发问题-哲学家就餐问题[Dining Philosophers Problem](http://en.wikipedia.org/wiki/Dining_philosophers_problem)。

你可以通过信号量来控值多个消费者访问有限的资源。例如，如果你创建了一个拥有两个资源池的信号量，通常情况下只能够使两个线程同时访问这个临界区，另外想使用资源的线程必须等待，这就是先进先出队列。

来让我们一起使用信号量。

打开**GooglyPuffTests.m**文件并使用下列代码替换**downloadImageURLWithString**函数：

```
- (void)downloadImageURLWithString:(NSString *)URLString
{
    // 1
    dispatch_semaphore_t semaphore = dispatch_semaphore_create(0);
    NSURL *url = [NSURL URLWithString:URLString];
    __unused Photo *photo = [[Photo alloc]
                             initwithURL:url
                             withCompletionBlock:^(UIImage *image, NSError *error) {
                                 if (error) {
                                     XCTFail(@"%@ failed. %@", URLString, error);
                                 }
                                 // 2
                                 dispatch_semaphore_signal(semaphore);
                             }];
     // 3
    dispatch_time_t timeoutTime = dispatch_time(DISPATCH_TIME_NOW,kDefaultTimeoutLengthInNanoSeconds);
    if (dispatch_semaphore_wait(semaphore, timeoutTime)) {
        XCTFail(@"%@ timed out", URLString);
    }
}
```

下边是对上面代码执行步骤的解释：

1. 创建信号量。传递的参数是用来指定信号量开始时间，这个参数值可以通过信号量来控制，而不必去管理这个数字的递增。（注意信号量的递增即为发送信号）
2. 在*completion block*中你告诉信号量不需要资源了，这就增加了信号量的计数并告诉其他信号量这个资源是可用的。
3. 在给定的时间内等待信号量的返回，这段代码将阻塞线程直到接收到信号为止。但不为0的数字返回时就意味着超时了。在这个例子中，如果测试用例失败可能是因为网络超过10s才返回。

重新运行测试，只要网络正常应该就能够测试成功。注意查看CPU消耗情况并和前面的进行对比。

断开网络连接并再次进行测试，如果是在真机上运行，直接调成飞行模式就可以了。在模拟器上则需要关掉网络，在10s之后这个测试失败了。

这些都是相当繁琐的测试，但是如果你在一个团队中工作，这些基本的测试能够指出网络发生问题是到底谁应该负这个责任。

####使用Dispatch Sources

GCD一个特别有趣的功能就是**Dispatch Sources**，这是一个非常底层的函数主要是用来帮助你监听Unix信号，文件描述，Mach端口，VFS节点等一些晦涩难懂的东西。所有的这些都已经超过了这篇教程的范围，但是你可以通过一种特别的方式来浅层次调用*dispatch source*对象。

第一次使用*dispatch sources*的用户可能对如何调用感到很疑惑，所以首先你要明白**dispatch_source_create**是如何工作的，下面是创建的函数原型：

```
dispatch_source_t dispatch_source_create(
   dispatch_source_type_t type,
   uintptr_t handle,
   unsigned long mask,
   dispatch_queue_t queue);
```

**dispatch_source_type_t**是个很重要的参数，他决定了使用什么句柄和掩码参数。你需要到[开发文档](https://developer.apple.com/library/mac/documentation/Performance/Reference/GCD_libdispatch_Ref/Reference/reference.html#//apple_ref/doc/constant_group/Dispatch_Source_Type_Constants)中去查看**dispatch_source_type_t**每种类型的含义。

你将监听**DISPATCH_SOURCE_TYPE_SIGNAL**参数类型，如[文档中描述](https://developer.apple.com/library/mac/documentation/Performance/Reference/GCD_libdispatch_Ref/Reference/reference.html#//apple_ref/c/macro/DISPATCH_SOURCE_TYPE_SIGNAL")

一个调度源监听当前进程的信号，句柄是一个信号编号（int）。掩码没有使用（传0值）。

这些Unix信号列表可以在[**signal.h**](http://www.opensource.apple.com/source/xnu/xnu-1456.1.26/bsd/sys/signal.h)文件中查看。在文件头部有一串**#define**。在这些信号中，你将监听**SIGSTOP**信号，当收到一个无法回避的挂起指令时，该信号将会发出。当你使用*LLDB*调试程序时该信号同样会被发出。

到**PhotoCollectionViewController.m**文件中将下列方法添加到**viewDidLoad**中：

```
- (void)viewDidLoad
{
  [super viewDidLoad];
  // 1
  #if DEBUG
      // 2
      dispatch_queue_t queue = dispatch_get_main_queue();
      // 3
      static dispatch_source_t source = nil;
      // 4
      __typeof(self) __weak weakSelf = self;
      // 5
      static dispatch_once_t onceToken;
      dispatch_once(&onceToken, ^{
          // 6
          source = dispatch_source_create(DISPATCH_SOURCE_TYPE_SIGNAL, SIGSTOP, 0, queue);
          // 7
          if (source)
          {
              // 8
              dispatch_source_set_event_handler(source, ^{
                  // 9
                  NSLog(@"Hi, I am: %@", weakSelf);
              });
              dispatch_resume(source); // 10
          }
      });
  #endif
   // The other stuff
```

这段代码有点繁琐，所以通过注释来看看代码是如何执行的：

1. 最好在DEBUG模式下来编译这段代码。
2. 通过**dispatch_queue_t**来创建实例变量而不是直接在在参数部分提供这个函数，当代码变得很长时，分开赋值可能更具有可读性。
3. **source**会在方法外重复调用，所以你需要个静态变量。
4. 使用**weakSelf**来确保不会陷入循环引用。对于**PhotoCollectionViewController**并不是完全有必要的，因为在app的整个生病周期中它是始终驻守在内存中的。但是如果有什么类被销毁了，仍需这样做来确保循环引用。
5. 使用**dispatch_once**对调度源只创建一次。
6. 这是source变量初始化方法，你需要指定对信号监听感兴趣并且给第二个参数赋值**SIGSTOP**，另外需要使用主队列来接收事件。
7. 如果你提供的参数格式不正确*dispatch source*对象是不会被创建的，所以你在调用之前你要确保该对象是可用的。
8. 当你监听到你接收到的信号时**dispatch_source_set_event_handler**就会被调用。然后在block参数中写入相应的逻辑。
9. 这是在终端打印类型相关的信息。
10. 默认情况下，所有创建的资源是挂起状态。所以当你在监听事件时，需要告诉源对象恢复状态。

编译并运行程序，在调试部分暂停了一会儿，但是立刻又恢复了app运行状态。查看终端输出，你会发现函数确实运行了，如下面代码所示：

2014-03-29 17:41:30.610 GooglyPuff[8181:60b] Hi, I am: 

但是在app运行周期应该如何调用呢？

每当您恢复应用程序状态时可以使用此方法来调试相关对象并打印信息。当恶意攻击者将调试器附加到你的应用上时，你也可以调用自定义安全逻辑来保护你的程序。

一个很有趣的想法就是通过使用此方法来跟踪堆栈相关信息，并发现你在调试器中操纵的对象。

![What_Meme](/images/GCD/What_Meme.jpg)

想一想这种情况，当你在断点外运行时，你几乎看不到你所需要的堆栈信息。现在你可以在任何时间调试，并在你设计的地方执行相关代码。相对于调试器繁琐操作而言，对于在程序某个地方用此方法进行调试是非常有用的。

![I_See_What_You_Did_Meme](/images/GCD/I_See_What_You_Did_Meme.png)

在**viewDidLoad**句柄中的**NSLog**处添加断点。程序在DEBUG处暂停，然后再次运行时，程序会暂停到断点处。此时已经运行到了**PhotoCollectionViewController**类的底层，你可以访问**PhotoCollectionViewController**类的核心方法了。

> 如果你还不知道哪个线程在调试器中，现在就看看。在*libdispatch*之后主线程总会是第一个出现，GCD线程会在第二个出现。当程序遇到断点时，线程数和线程保留数依赖于硬件当时在做什么。

在调试器中，输入下面的代码

**po [[weakSelf navigationItem] setPrompt:@"WOOT!"]**

然后继续执行程序，会看到如下图所示：

![Dispatch_Sources_Xcode_Breakpoint_Console-650x500](/images/GCD/Dispatch_Sources_Xcode_Breakpoint_Console-650x500.png)

![Dispatch_Sources_Debugger_Updating_UI-308x500](/images/GCD/Dispatch_Sources_Debugger_Updating_UI-308x500.png)

通过这种方法，你不仅可以更新UI，而且可以查询类的属性，甚至执行方法。所有这些操作都不需要通过重新启动App来获取当前更改后的状态。真的是非常好用。

###接下来学习什么

你可点击这个地址下载[示例程序](http://cdn3.raywenderlich.com/wp-content/uploads/2014/01/GooglyPuff-Final.zip)。

我很讨厌再次重复之前强调的东西，但是你真的应该去学习这篇教程[How to Use Instruments](http://www.raywenderlich.com/23037/how-to-use-instruments-in-xcode)。如果你准备对你的程序进行优化，这个工具是必不可少的。*Instruments*很擅长分析不同代码执行的效率：对比不同代码之间花费时长。如果想知道某个方法的确切执行时间，那就需要你自己写方法来完成这个计算。

同时查看这篇教程[ How to Use NSOperations and NSOperationQueues](http://www.raywenderlich.com/19788/how-to-use-nsoperations-and-nsoperationqueues)，是建立在GCD之上的一种并发处理技术。通常情况下对于简单的并发任务使用GCD已经足够了，但是对于大量并发操作的实现和使用一种更接近于面向对象编程模式，*NSOperations*会更好。

请记住，除非你有特别的需要才能会去使用底层API，否则就要坚持使用更高层的API。只有进入苹果的黑魔法领域，你才能够学到更多或者做到更多有趣的事。

>翻译自[Grand Central Dispatch In-Depth: Part 2/2](http://www.raywenderlich.com/63338/grand-central-dispatch-in-depth-part-2)