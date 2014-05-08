---
layout: post
title: "深入理解Grand Central Dispatch:第一部分"
date: 2014-04-17 18:29:08 +0800
comments: true
categories: 翻译
---

尽管Grand Central Dispatch（简称GCD） 很早就已经使用，但并不是每个人都了解如何充分使用它们。棘手的并发处理和一堆基于C指针的GCD开发文文档，似乎与Objective-C通常表达形式格格不入。本篇教程将分两部分更深一步学习GCD。

第一部分主要是学习什么是GCD，了解GCD的一些基本功能。第二部分我们将学习GCD提供的更为高级的功能。

###什么是GCD

GCD是**libdispatch**（是苹果公司支持并发执行的多核处理机器中iOS和OS X系统上的一个类库）的别称。它具有以下特性：

- GCD可以显著提高程序的响应速度，使那些消耗较多资源的计算在后台运行。
- GCD提供了一种比锁和线程更加容易处理的并发模型，并且有效地避免了并发错误。
- GCD可以通过一些原语性的通用模型来提高我们的代码质量，如单例模式。

本篇教程假定您已经对**Block**和**GCD**有了基本的了解，如果还不是那么清楚，可以阅读下这篇教程[Multithreading and Grand Central Dispatch on iOS for Beginners](http://www.raywenderlich.com/4295/multithreading-and-grand-central-dispatch-on-ios-for-beginners-tutorial)

###GCD相关术语

要想理解GCD，你需要对线程和并发相关概念相当熟悉。所以在深入学习GCD之前我们先重温下关于线程和并发的概念。

####串行和并行

串行和并行描述了不同任务在执行时的一种相互关系。多任务可以顺序执行，也可以同一时间同时执行。

尽管串行和并行已经有了广泛的应用，但是在本篇教程我们把一个任务当成**Objective-C block**或许会更好理解。更多块语法相关知识请查看这篇文章[How to Use Blocks in iOS 5 Tutorial](http://www.raywenderlich.com/9328/creating-a-diner-app-using-blocks-part-1)。事实上，你可以把GCD当成一种函数指针，但是大多数情况GCD比Block在实际运用中会更加麻烦。

####同步与异步

在GCD中，同步和异步描述一个函数的完成是依赖于其中使用GCD来完成任务的情况。一个同步执行的函数只有在任务按顺序完成时才会返回。

对于异步函数，不会等到顺序任务完成之后返回，而是会立即返回。因此，一个异步函数不会阻塞当前进程而回继续执行下一个函数。

请注意，当你读到同步函数阻塞当前进程时，不要混淆该函数是个“***blocking***”函数还是**blocking（阻塞）**操作。前面的**blocks**是描述这个函数是否阻塞当前线程，而和**block**语法没有什么关系（这里的block语法是指**Objective-C**中定义的匿名函数和提交给GCD来执行的一个任务。也就是说block在英语中有两种意思一种是阻塞的意思，另外一个就是Objective-C Block的专有名词称为块语法）。

####临界区

每个线程中访问临界资源的那段代码称为临界区，也就是说这段代不能被两个线程同时访问。这通常是由于代码共享同一资源引起的，例如并发进程访问同一个变量，可能会使这个数据变为脏数据。

####死锁

不同线程因相互等待对方完成而进行下一步操作的现象称为死锁。第一个线程无法完成是因为它在等待第二个线程的完成，而第二个线程却在等待第一个线程的完成。

####线程安全

线程安全是指代码可以被不同线程或者并发任务唤起而不会造成任何问题（如数据损坏，程序崩溃等）。非线程安全是指代码只允许同一时间在同一个上下文执行。例如`NSDictionary`就是线程安全的，你可以在同一时间的多个线程调用而不会产生任何问题。然而对于`NSMutableDictionary`却不是线程安全的，在每一时刻只能运行在一个线程中。

####上下文切换

上下文切换是指在一个单独进程中进行不同线程的切换，存储和恢复之前执行的一种状态的过程。这个过程在多任务处理的程序中是相当普遍的，但是会消耗额外的资源。


###并发与并行

并发与并行经常会被同时提起，所以有必要对这两个概念进行区分。

并发代码的不同部分可以被同时执行，这是由系统决定的。

多核设备可以在同一时间并行执行多个线程，然而在单核系统上只能执行单个线程，所以为了达到这种效果，只能通过上下文切换来执行另外一个线程，这通常是发生在极短时间内的，所以会给人一种并发的假象，如下图所示：

![Concurrency vs Parallelism](/images/GCD/Concurrency_vs_Parallelism.png)

尽管你可以通过GCD来实现并发执行，但是会有多少并行却是由GCD来决定的。并行决定着并发，但是并发却不能保证并行。

从深层次来看，并发是架构层面的。当你使用GCD来架构代码时，你可以对代码进行并发处理，也可以不那么做。如果想更深入了解这方面的知识请浏览[this excellent talk by Rob Pike](http://vimeo.com/49718712).

###队列

GCD提供**调度队列（dispatch queues）**来处理代码块，这些队列管理着你提供给GCD的任务并按照先进先出（**FIFO**）的顺序执行。这就保证了添加到队列中的任务是按照顺序执行的。

所有的调度队列都是线程安全的，所以你可以同时在多个线程执行调度队列。当你明白了调度队列如何保证你的代码线程安全时，你会发现GCD是很好用的。把你的任务提交到队列的关键是选择合适的调度队列和正确的调度功能。

在这部分教程中，你将了解到两种由GCD支持的队列调度方式，然后通过一些例子了解如何通过GCD将任务添加到调度队列中。

####串行队列

在串行队列每次只执行一个任务，每个任务只有在前一个任务结束时才能够执行。同时，你无法确定这个块结束和下一个开始之间的时长，如下图所示：

![Serial Queues](/images/GCD/Serial-Queue.png)

这些任务的执行时间是由GCD来控制的，唯一能够确定的是GCD每次只执行一个任务并且会按照进队顺序执行。

由于串行队列中两个任务不能同时运行，所以没有同时访问临界区的风险。这就可以保证这些任务避免资源竞争。所以如果访问临界区的唯一途径就是将任务提交到调度队列中，那么你就可以保证临界区是安全的。

####并发队列

并发进程能够保证任务按照进队顺序执行，但无法保证按照进队顺序结束的，同时也不知道什么时候执行下一个快或者每个块执行所需时间。因为这些都是由GCD决定的。

下图展示了在GCD下四个并发任务执行的情况：

![Concurrent-Queue](/images/GCD/Concurrent-Queue.png)

上图描述了一个块的开始是由GCD来决定的。如果一个块的执行时间与另外一个重叠，那么GCD会决定是在不同的内核上运行，还是通过上下文切换在同一个可用内核上并发运行。

为了变得更有趣，GCD提供了至少5种队列类型以供选择。

####队列类型

首先，系统提供了一个被称为**主队列(main queue)**的特殊队列，像串行队列一样，每次只执行一个任务。然而，主队列会保证每个任务是在主线程中执行的，主线程是唯一一个允许你执行界面更新的一个线程。这个线程可以给**UIViews**发送消息或者通知。

系统同样提供了几个并发队列。这些队列被称作**全局调度队列（Global Dispatch Queues）**。目前有四个不同级别的全局队列分别为：**background**, **low**, **default**, 和 **high**。请注意苹果的API也会使用这些队列，所以这些队列中不会仅仅只有你添加的任务。

最后，你也可以创建属于你自己的串行或者并行队列。这就意味着你至少可以使用五种不同类型的队列：主队列和四种不同类型的全局队列，另外还有更多自定义的队列可供选择。

使用GCD的关键是将任务添加到合适的队列调度中来执行。你可以通过下面建议的通常方法来获得最佳的实践经验。

###开始

这篇教程的目的是通过GCD在不同的线程安全调用所写的代码，我们以一个完成的项目**GooglyPuff**来开始下面的教程。

GooglyPuff 是一个未经过优化的，线程不安全的应用程序，这个程序主要通过**Core Image API**检测人脸,并在眼睛上放个曲棍球。对于所检测的图片既可以从本地图片库选取也可以从网络上下载。

[点击这里下载应用源码。](http://cdn4.raywenderlich.com/wp-content/uploads/2014/01/GooglyPuff_Start_1.zip)

下载源码，打开并运行程序会如下图所示：

![Workflow](/images/GCD/Workflow1.png)

当你点击**Le Internet**下载图片时，一个**UIAlertView**弹出框会过早的弹出。在第二部分教程我们将修复这个问题。

在这个项目中有四个类：

* **PhotoCollectionViewController**:这是应用程序启动时的第一个视图控制器，它通过缩略图展示了所有选定的图片。
* **PhotoDetailViewController**:这个类执行将类似曲棍球的眼睛添加到图像上相关逻辑，并展示到UIScrollView上面。
* **Photo**:这是一个与**NSURL**和**ALAsset**实例相关的图片类簇。这个类提供图像，缩略图和从网络获取图片的下载状态。
* **PhotoManager**:这个类管理与**Photo**相关的所有实例。

###通过dispatch_sync处理后台任务

回到应用程序，从图片库添加一些图片或者使用**Le Internet**下载一些图片。

请注意在**PhotoCollectionViewController**中点击**UICollectionViewCell**时需要花费较长时间来实例化一个新的**PhotoDetailViewController**。尤其在一个速度慢的机器上打开一张较大的图片时会有明显的滞后。

这很容易加载**UIViewController’s viewDidLoad**一些冗余的东西。这就导致在页面出现之前会有一个明显的停滞问题。如果可以的话最好在后台完成一些在页面加载时不必要的操作。

打开**PhotoDetailViewController**并用下面的代码替换**viewDidLoad**中的内容

```
- (void)viewDidLoad
{   
    [super viewDidLoad];
    NSAssert(_image, @"Image not set; required to use view controller");
    self.photoImageView.image = _image;
    //Resize if neccessary to ensure it's not pixelated
    if (_image.size.height <= self.photoImageView.bounds.size.height &&
        _image.size.width <= self.photoImageView.bounds.size.width) {
        [self.photoImageView setContentMode:UIViewContentModeCenter];
    } 
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{ // 1
        UIImage *overlayImage = [self faceOverlayImageFromImage:_image];
        dispatch_async(dispatch_get_main_queue(), ^{ // 2
            [self fadeInNewImage:overlayImage]; // 3
        });
    });
}
```

为什么这样修改：

1. 首先把主线程中的操作放到全局队列中。因为这是一个**dispatch_async()**，所以异步提交这个块意味着使用线程来继续执行。这使**viewDidLoad**在主线程更快返回，使页面加载更顺畅。同时，人脸检测操作会在稍后开始并完成。
2. 此时，人脸检测完成并生成一张新的图片。既然想把这张新的图片更新到**UIImageView**，那么就需要在主线程添加一个新的块操作。请记住必须在主线程操作UI Kit classes。
3. 最后，你需要通过**fadeInNewImage:**更新界面来渐变显示一张带有曲棍球眼睛的图片。

编译并运行程序，选择一张图片你会发现视图控制器明显加载更快了，在短时间延迟后添加上了曲棍球眼睛。照片前后不同的显示带来了不错的视觉冲击。

还有，如果你尝试加载一张巨大的图片，应用程序不会在加载过程中挂起，这就使应用能够进行更好的进行扩展了。

如上所述，**dispatch_async**附加一个块到队列中并立即返回。该任务会由GCD来决定何时执行。使用**dispatch_async**在后台执行基于网络的或者CPU密集型任务而不会去阻塞主线程。

以下是在何时及如何使用**dispatch_async**各种队列类型的快速指南：

* **Custom Serial Queue**:当需要在后台跟踪不断执行的任务时可以使用此类型队列。因为一次只执行一个任务，所以就避免了资源竞争。需要注意的是，如果你需要获取一个方法的返回值，你必须内嵌另外一个块去返回数据或者考虑使用**dispatch_sync**。
* **Main Queue (Serial)**:当一个任务在并发线程完成时，通常会选择主线程去更新页面显示，为了做到这一点你需要将一个块嵌到另外一个中。还有，如果在线程中调用了**dispatch_async**，你需要保证这个异步任务会在主线程完成之后不久去执行。
* **Concurrent Queue**:通常在后台去执行不需要更新页面的任务时会使用并发线程。

###使用dispatch_after延迟任务执行

考虑一下你的应用的用户体验。用户在第一次打开程序的时候是否对如何使用这个应用而感到困惑。

如果在**PhotoManager**没有找到任何图片，这时给用户一个提醒或许是个好主意。不过，你同时需要注意，如果提醒时间过短的话，用户可能会因为浏览导航界面的其他部分而错过这个提醒。

当用户第一次浏览应用时，延迟一秒去显示提醒或许已经足够引起用户注意。

将下列代码添加到**PhotoCollectionViewController.m**文件的**showOrHideNavPrompt**方法中:

```
- (void)showOrHideNavPrompt
{
    NSUInteger count = [[PhotoManager sharedManager] photos].count;
    double delayInSeconds = 1.0;
    dispatch_time_t popTime = dispatch_time(DISPATCH_TIME_NOW, (int64_t)(delayInSeconds * NSEC_PER_SEC)); // 1 
    dispatch_after(popTime, dispatch_get_main_queue(), ^(void){ // 2 
        if (!count) {
            [self.navigationItem setPrompt:@"Add photos with faces to Googlyify them!"];
        } else {
            [self.navigationItem setPrompt:nil];
        }
    });
}
```
**showOrHideNavPrompt**方法在**viewDidLoad**中执行并会在**UICollectionView**加载时执行。如下列步骤所示：

1. 首先声明指定时间延迟的变量。
2. 然后等待**delayInSeconds**一段时间后将同步代码块添加到主线程中。

编译并运行程序，等待一会儿会出现一个用户如何进行操作的提示。

**dispatch_after**就像一个**dispatch_async**延迟操作。但是仍然无法获取这个块执行的时间并且没办法在**dispatch_after**返回时取消相应操作。

思考下在何种情况下使用**dispatch_after**较好？

* **Custom Serial Queue**:在自定义队列使用**dispatch_after**要小心，最好将这个操作在主队列执行。
* **Main Queue (Serial)**:推荐在主队列中使用**dispatch_after**，Xcode有一个很好的宏定义来使用此方法。
* **Concurrent Queue**:在并发队列中使用**dispatch_after**要十分注意，需要始终在主队列中进行相关操作。

###确保单例线程安全

单例，不管喜欢还是讨厌，在iOS中就是这么流行。

一个经常担心的问题是单例是不是线程安全的。单例通常会在同一时间被不同的视图控制器访问。

单例的线程安全问题包括初始化，读取和写入数据。

**PhotoManager**就是通过单例来实现的——它同样会受到上述问题影响。你将很快看到问题的出现。

切换到**PhotoManager.m**类中，找到**sharedManager**方法，如下面代码所示

```
+ (instancetype)sharedManager    
{
    static PhotoManager *sharedPhotoManager = nil;
    if (!sharedPhotoManager) {
        sharedPhotoManager = [[PhotoManager alloc] init];
        sharedPhotoManager->_photosArray = [NSMutableArray array];
    }
    return sharedPhotoManager;
}
```

这段代码看起来相当简单，创建一个单例，并初始化一个**NSMutableArray**类型的私有变量**photosArray**。

然而，**if**分支不是线程安全的。如果你多次调用这个方法，很有可能A线程在进入**if**代码块中并在**sharedPhotoManager**初始化前，此时B线程进入**if**中并进行了初始化，然后退出。

当系统切换到A线线程时，B线程依然会继续生成另外一个单例，然后退出。这时系统就会同时拥有两个单例而不清楚到底使用哪一个。

为了实现这种效果，使用下面的带面替换**PhotoManager.m**类中的**sharedManager**方法：

```
+ (instancetype)sharedManager  
{
    static PhotoManager *sharedPhotoManager = nil;
    if (!sharedPhotoManager) {
        [NSThread sleepForTimeInterval:2];
        sharedPhotoManager = [[PhotoManager alloc] init];
        NSLog(@"Singleton has memory address at: %@", sharedPhotoManager);
        [NSThread sleepForTimeInterval:2];
        sharedPhotoManager->_photosArray = [NSMutableArray array];
    }
    return sharedPhotoManager;
}
```

在上边代码中你需要使用**NSThread’s sleepForTimeInterval:**方法来强制进行上下文切换。

打开**AppDelegate.m**文件并将下面的代码添加到**application:didFinishLaunchingWithOptions:**方法中：

```
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
    [PhotoManager sharedManager];
});
dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
    [PhotoManager sharedManager];
});
```

这将创建多个异步并发线程，同时调用单例初始化方法并出现上边所述的竞争关系。

编译并运行项目，查看控制窗口输出，你会看到多个不同的单例，如下图所示：

![NSLog-Race-Condition-700x90](/images/GCD/NSLog-Race-Condition-700x90.png)

上图不同地址的单例是不是已经违背了单例的初衷了呢；

上边的输出显示本应该只执行一次的临界区域被访问了多次。我们强制了这种情况的发生，但是你可以想象一下这种情况在现实中是如何发生的。

>注意:上述**NSLogs**是在特定条件下展示的，基于其他系统时间是不可控的，所以当发生线程问题时是很难追踪并重现该问题的。

为了避免这种情况的发生，初始化代码必须只执行一次并且阻止其他情况下的初始化行为，所以**if**条件的修改是关键。这正是**dispatch_once**所应该做的。

使用**dispatch_once**来代替**if**语句的初始化如下面代码所示：

```
+ (instancetype)sharedManager
{
    static PhotoManager *sharedPhotoManager = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        [NSThread sleepForTimeInterval:2];
        sharedPhotoManager = [[PhotoManager alloc] init];
        NSLog(@"Singleton has memory address at: %@", sharedPhotoManager);
        [NSThread sleepForTimeInterval:2];
        sharedPhotoManager->_photosArray = [NSMutableArray array];
    });
    return sharedPhotoManager;
}
```
编译并运行程序，查看输出窗口你会发现只有一个初始化地址的单例，这正是我们所想要的单例。

现在你明白了防止资源竞争的重要性了，在**AppDelegate.m**去掉**dispatch_async**代码，并用下边的代码初始化**PhotoManager‘s**单例：

```
+ (instancetype)sharedManager
{
    static PhotoManager *sharedPhotoManager = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedPhotoManager = [[PhotoManager alloc] init];
        sharedPhotoManager->_photosArray = [NSMutableArray array];
    });
    return sharedPhotoManager;
}
```
**dispatch_once()**以一种线程安全的方式执行且只执行一次代码块。当不同的线程尝试访问这个临界区时，此时临界区中的线程会阻止其他线程访问，并且直到完成初始化操作之后才会允许其他线程操作。

![Highlander_dispatch_once-480x274](/images/GCD/Highlander_dispatch_once-480x274.png)

值得注意的是我们仅仅让共享的实例线程安全而已，而并没有使整个类线程安全。在这个类中肯定还有其他的临界区，例如同步访问数据的地方，这些地方也应该是线程安全的。接下来我们将学习这一部分。

###处理读与写的问题

线程安全不仅仅是处理单例时的问题。如果在单例属性中有个可变的对象，那么同样也需要考虑这个对象的线程安全问题。

系统自带的类库中的类是否会有线程安全问题，答案是可能会有。[苹果文档](https://developer.apple.com/library/mac/documentation/cocoa/conceptual/multithreading/ThreadSafetySummary/ThreadSafetySummary.html)中列出了很多非线程安全的基础类，例如上面提到的**NSMutableArray**类型。

尽管很多线程可以同时读取**NSMutableArray**而不会产生什么问题，但是如果一个线程读，一个线程写入可能就会造成线程的不安全。你当前的单例并没有预防这种情况的发生。

打开**PhotoManager.m**，查看**addPhoto:**方法：

```
- (void)addPhoto:(Photo *)photo
{
    if (photo) {
        [_photosArray addObject:photo];
        dispatch_async(dispatch_get_main_queue(), ^{
            [self postContentAddedNotification];
        });
    }
}
```

这是一个写操作，改变了私有数组的值。

现在看下**photos**方法：

```
- (NSArray *)photos
{
  return [NSArray arrayWithArray:_photosArray];
}
```

这是一个从可变数组读取数据的方法。这个方法为调用者生成一个不可变的副本以防止对可变数组不恰当的访问。但是该方式并没有阻止一个线程在**addPhoto:**写入，另外一个并发线程从**photos**读取数据。

这是一个经典的读者写者问题[Readers-Writers Problem](http://en.wikipedia.org/wiki/Readers–writers_problem)。GCD提供了一种优雅的解决方案--使用**dispatch barriers**来创建读写索[Readers-writer lock](http://en.wikipedia.org/wiki/Read/write_lock_pattern)。

**Dispatch barriers**是把一组并发队列操作的函数当做一个串行的阻塞（**serial-style bottleneck**）。使用GCD的阻塞API是为了确保提交到指定队列的代码块是在特定时间执行的唯一一个任务。这意味所有提交到队列中的任务必须在当前代码块执行完毕时才能够调用。

当代轮到下一个代码块执行时，阻塞队列会执行该块并确保队列此时没有执行其他的任务。一旦完成，队列就会返回到默认设置。GCD提供了同步和异步阻塞功能。

下图展示了阻塞功能在异步操作时的效果：

![Dispatch-Barrier-480x272](/images/GCD/Dispatch-Barrier-480x272.png)

值得注意的是在正常情况下队列操作像一个并发队列，当阻塞任务执行时，此时队列更像个串行队列。当前仅仅只有阻塞任务能够执行。当阻塞任务完成时，队列又回到了正常的并发队列中。

阻塞功能在什么场合下使用更为恰当：

* **Custom Serial Queue**:串行队列下使用并不是一个好的选择，串行操作本身就是一次只能执行一个任务，所以阻塞操作在这里没有起什么作用。
* **Global Concurrent Queue**:在全局并发队列中使用不是个好方法，因为系统操作也可能会使用到这个队列然而你并不希望阻塞系统相关的操作。
* **Custom Concurrent Queue**:对于原子性或者临界区的操作，任何需要线程安全的实例或者操作，阻塞线程都是一个好的选择。

既然作为唯一比较好的选择是自定义的并发队列，那么我们就需要自己创建一个处理阻塞的函数并区分读和写的功能。并发队列允许同时进行多个读操作。

打卡**PhotoManager.m**，把下面的私有函数添加到实现类中：

```
@interface PhotoManager ()
@property (nonatomic,strong,readonly) NSMutableArray *photosArray;
@property (nonatomic, strong) dispatch_queue_t concurrentPhotoQueue; ///< Add this
@end
```
找到**addPhoto**函数用下面的代码来实现：

```
- (void)addPhoto:(Photo *)photo
{
    if (photo) { // 1
        dispatch_barrier_async(self.concurrentPhotoQueue, ^{ // 2 
            [_photosArray addObject:photo]; // 3
            dispatch_async(dispatch_get_main_queue(), ^{ // 4
                [self postContentAddedNotification]; 
            });
        });
    }
}
```

下面步骤描述了该函数是如何工作的：

1. 在执行其他操作之前先检查下是否有可用的图片资源。
2. 添加写操作到自定义的队列中，当临界区在执行一个任务时，这个任务会是该队列中唯一一个在执行的。
3. 这段代码是将对象添加到数组中。因为这是一个阻塞块，所以该块永远不会在**concurrentPhotoQueue**队列中与其他块同时运行。
4. 最后当你添加完图片后发送了一个通知，因为要更新UI界面，所以该通知需要在主线程进行操作。因此这里就在主线程中异步调取了通知操作。

这只是个写操作，你同时需要实现**photos**读操作和**concurrentPhotoQueue**的实例化。

为了确保同写操作的相关任务的线程安全，你需要在**concurrentPhotoQueue**队列中执行读操作。并从函数中返回且不能异步调度队列，因为写操作未必会在进行读操作前返回。

在这种情况下**dispatch_sync**会是一个很好的选择。

**dispatch_sync()**同步提交操作并且会等当前任务完成之后再返回。使用**dispatch_sync**来跟踪阻塞调度任务，或者在你需要等待相关任务操作完成之后再进行后续操作，此时可以使用**dispatch_sync**来处理数据。如果是第二种情况，有时你会在**dispatch_sync**外看到一个**__block**变量，这个变量是用来处理**dispatch_sync**函数返回的数据。

值得注意的是，如果你针对当前一个正在运行的串行队列调用**dispatch_sync**，这将会造成死锁。因为这个操作会等待当前块执行完毕，但当前正在执行的任务又在等待**dispatch_sync**完成。这就迫使你需要考虑**dispatch_sync**是从哪个队列中调取的，同时应该放到哪个队列里执行。

下面是**dispatch_sync**使用的环境：

* **Custom Serial Queue**:在这种情况下调用要十分小心，如果你针对一个正在运行的队列调用**dispatch_sync**，这一定会造成死锁的。
* **Main Queue (Serial)**:和上边的同样的理由，这种情况也可能会造成潜在的死锁现象。
* **Concurrent Queue**:通过阻塞调度同步执行任务或者等待相关任务完成之后执行后续任务，在这种情况下调用**dispatch_sync**是完全可以的。

打开**PhotoManager.m**用下面的代码替换**photos**方法：

```
- (NSArray *)photos
{
    __block NSArray *array; // 1
    dispatch_sync(self.concurrentPhotoQueue, ^{ // 2
        array = [NSArray arrayWithArray:_photosArray]; // 3
    });
    return array;
}
```

根据序号查看上面代码的相关解释：

1. **__block**关键字表示允许对象在块中是可变的，如果没有这个关键字**array**数组变量在块中是只读属性，根本无法通过编译。
2. 在**concurrentPhotoQueue**队列中同步调度读取数据。
3. 将图片存储到**array**中并返回数据。

最后你需要初始化**concurrentPhotoQueue**队列属性，如下面代码所示：

```
+ (instancetype)sharedManager
{
    static PhotoManager *sharedPhotoManager = nil;
    static dispatch_once_t onceToken;
    dispatch_once(&onceToken, ^{
        sharedPhotoManager = [[PhotoManager alloc] init];
        sharedPhotoManager->_photosArray = [NSMutableArray array];
        // ADD THIS:
        sharedPhotoManager->_concurrentPhotoQueue = dispatch_queue_create("com.selander.GooglyPuff.photoQueue",
                                                    DISPATCH_QUEUE_CONCURRENT); 
    });
    return sharedPhotoManager;
}
```

**concurrentPhotoQueue**的初始化使用的是**dispatch_queue_create**创建的并发队列。第一个参数是域名地址的方向命名，确保命名描述的清晰性，因为这在调试中会非常有用。第二个参数是指定你的队列是并发的还是串行的。

> 注意：当在网络上搜索例子时，你经常会看到使用**dispatch_queue_create**方法创建对队列时第二个参数往往传递的是**0**或者**NULL**，这是创建串行队列的一种过时方法，通常是传递更为清晰具体的参数。

Congratulations--**PhotoManager**单例现在线程安全了，不管你什么时候或者什么地方读取或者写入图片，你完全可以相信这种操作是十分安全的。

###回顾队列相关知识点

并不是100%确定掌握了GCD的要领？为了确保你已经掌握了基本的要点，请使用GCD函数创建个简单的例子，然后根据断点和**NSLog**输出，来确保你的确明白了函数在执行过程中发生了什么。

这里提供了两个GIF图片以帮助你巩固对于**dispatch_async**和**dispatch_sync**的理解。每个GIF左边显示了代码执行的步骤，右边显示了队列当前的状态。

####dispatch_sync知识点再现

```
- (void)viewDidLoad
{
  [super viewDidLoad]; 
dispatch_sync(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
      NSLog(@"First Log");
  });
  NSLog(@"Second Log");
}
```
![dispatch_sync_in_action](/images/GCD/dispatch_sync_in_action.gif)

1. 主队列中是按顺序执行的任务，将要执行的任务是包含**viewDidLoad**的**UIViewController**的实例。
2. **viewDidLoad**在主队列中执行。
3. 主线程进入到了**viewDidLoad**中而且即将执行**dispatch_sync**。
4. **dispatch_sync**添加到了全局队列中并于稍后执行，主线程中的任务停止执行直到**dispatch_sync**完成操作。同时，全局队列并行处理任务，出现顺序将按照先进先出（FIFO）的顺序，但是执行时会按照并发处理。
5. 全局队列处理在**dispatch_sync**之前添加到队列中的任务。
6. 最后轮到执行**dispatch_sync**块。
7. 当**dispatch_sync**执行完毕，主线程就可以继续执行了。
8. **viewDidLoad**执行完毕后，主线程将继续操作后续的任务。

**dispatch_sync**添加一个任务到队列中并且直到任务执行完毕才会返回。**dispatch_async**除了不需要等待任务执行完毕才能够执行继续后续任务外，其他流程和上面类似。

####dispatch_async知识点再现

```
- (void)viewDidLoad
{
  [super viewDidLoad]; dispatch_async(dispatch_get_global_queue(DISPATCH_QUEUE_PRIORITY_HIGH, 0), ^{
      NSLog(@"First Log");
  });
  NSLog(@"Second Log");
}
```
![dispatch_async_in_action](/images/GCD/dispatch_async_in_action.gif)

1. 主线程按顺序执行，下一个将要执行的是含有**viewDidLoad**方法的**UIViewController**的一个实例。
2. **viewDidLoad**在主线程中执行。
3. 现在主线程在**viewDidLoad**中执行并且稍后执行**dispatch_async**方法块。
4. **dispatch_async**方法块添加到了全局队列中并且稍后会被执行。
5. **viewDidLoad**中的断点会继续移动，**dispatch_async**添加到全局队列中后主队列依然会继续后续任务的执行。同时全局队列也会并发处理相关任务，记住各个块会以先进先出的顺序出现在全局队列中，但是会并发的执行这些任务。
6. 添加到**dispatch_async**代码现在开始执行。
7. **dispatch_async**执行完毕并且打印相关信息。

在上面这个例子中，第二个**NSLog**信息出现在第一个**NSLog**之后。这是由硬件决定在那一刻到底谁先执行，而你是无法确定的，第一个**NSLog**可能在某些调用中会第一个执行并打印出信息。

###下一部分我们将会学习什么呢

在这部分教程中，我们学习到了如何使代码线程安全，并且在执行CPU密集型任务时保持主线程的响应能力。

你可以下载这个改进后的[GooglyPuff Project](http://cdn2.raywenderlich.com/wp-content/uploads/2014/01/GooglyPuff_End_1.zip)项目。在第二部分教程中我们将继续改进这个项目。

如果你正在准备优化你的APP，强烈建议使用**Instruments**中的**Time Profile**模版分析你的程序。使用此工具已超出本篇教程的范围，但是可以查看这篇文章[How to Use Instruments](http://www.raywenderlich.com/23037/how-to-use-instruments-in-xcode)学习下如何使用。

同时确保是使用真机进行测试的，因为在模拟器上分析的数据是相当不准确的。

接下来的教程我们将更深入的了解GCD相关的API并做更多更有趣的事。


> 本篇教程翻译自：[Grand Central Dispatch In-Depth: Part 1/2](http://www.raywenderlich.com/60749/grand-central-dispatch-in-depth-part-1)

