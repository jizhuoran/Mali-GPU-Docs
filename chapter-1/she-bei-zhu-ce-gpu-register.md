---
description: 内核自举模块载，显卡注册驱动生 （王子旭，2019）
---

# 设备注册: GPU Register

很明显，现在你有一个GPU，你觉得自己可厉害，有一个能算的这么快的玩意儿，但是你得让你的操作系统也知道你有这么个玩意儿，之后你才能通过操作系统来给GPU提交任务。

由于Linux采取了模块化的设计，所以device driver可以写成一个module，然后我们只要把这个模块注册到Linux Kernel里，他就成了操作系统的一部分。所以盘古开天辟地之操作就是调用[**module\_init**](https://www.kernel.org/doc/htmldocs/kernel-hacking/routines-init-again.html)函数把driver初始化函数注册到系统里。如果你跟我一样不太理解module\_init是如何运作的，不如看看这篇[**blog**](https://blog.csdn.net/u013216061/article/details/72511653)。

在代码里，传进module\_init里的那个函数就干了一件正事儿，就是调用一个叫**platform\_driver\_register**的函数。由于GPU是你先插在机器上的（其实我不知道对于Mali这种移动GPU，你强行从芯片上把GPU扣下来，整个设备还能不能用），当你调用**platform\_driver\_register\(platform\_driver\)**之后，在安装驱动程序过程中从总线上遍历各个设备，通过"platform\_driver.driver.name"看驱动程序是否与其相匹配，如果匹配就调用"platform\_driver.probe"。

如果一切顺利，到了这一步，系统已经找到了你的GPU，并把GPU对应的**platform\_device**传给"platform\_driver.probe"。在Mali driver里，这个叫kbase\_platform\_device\_probe的probe函数干了老鼻子事儿了，它凭一己之力完成了整个注册过程。当然这个注册过程无外乎就两个方面：一是处理硬件方面的的注册事宜，另一个是创建一些class（其实这里是structure，但说着方便吧，大家都是成年人了，都懂）的instance，这就是Mali GPU 里面的kbase（kernel side base API）了，这些API提供了对GPU的抽象，无论是对GPU的控制还是状态记录都是通过kbase API进行的。

在硬件层面上，总共干了三件大事儿，一是分配了irqs，二是注册了map，三是初始化了电源控制，要是说还有一件，那就是注册了io history。这一部分固然重要，不然空有驱动而无设备，则政令不达。然而对理解GPU 驱动里面的police并没有啥帮助，所以我不准备详细展开这一部分，而是把笔墨用在另一个部分。当然，这只是我现在的认识，也有可能这部分有用处，那么我就回来再写，这就是电子编辑的好处。

那么在kbase上，便是及其复杂的了。



### kbase\_backend\_early\_init

在这个明显是像我这种钢铁直男才会起这种名的函数里，首先调用了一个平台相关的函数（kbasep\_platform\_device\_init）去初始化硬件，但是看了这个函数的函数体，里面是个NULL。。。接着他又调用了（kbase\_pm\_runtime\_init），这个函数其实就是设置callback的，把关于电源管理的callback都注册到kbdev-&gt;pm.backend上去。完成了电源的callback注册之后，early\_init通过一个叫（kbase\_pm\_register\_access\_enable）的函数开启了GPU寄存器的访问，别看他说的这么高大上，他就是给GPU通了个电，这样我们才能从寄存器里读取到信息。放下这个不表，接下来early\_init函数就把从GPU寄存器读取出来的信息存到了kbdev-&gt;gpu\_props里了（**kbase\_gpu\_props**的instance）。既然读取完了信息，early\_init函数就把电关了，当然我相信这不是为了省电，肯定是要断电然后进行一些后续操作，并且这些操作是不能在GPU通电的时候干的。最后，early\_init先是注册了interrupts，紧接着初始化了电源管理框架（hwaccess\_pm）。

可以看出来，既然叫做early\_init，那么这个函数主要还是初始化了电源相关的东西，并且把GPU的基本信息从寄存器里读了出来，顺手设置了中断。



### kbase\_device\_init

这个函数初始化了kernel视角的device。首先device\_init根据GPU ID设置了硬件issues mask（kbdev-&gt;hw\_issues\_mask），然后根据GPU ID设置了特性mask（kbdev-&gt;hw\_features\_mask）。随后device\_init设置了DMA。好吧，我也知道这句话太敷衍了，但是我真的没弄明白他在干嘛，一大堆enum，等我弄明白了一定回来改这一段。但我妄自推测，这个函数就是设置了东西，没啥研究价值。



### kbase\_ctx\_sched\_init

这个函数初始化了Context Scheduler（CS）。对于我个人来讲，CS并不是很顾名思义，当然不是说他是反恐精英，而是这个所谓的scheduler管理地址空间分配和kbase\_context的reference计数。CS的内部实现并没有调度context的部分。它依赖于Job Scheduler去决定什么时候去调度/驱逐一个context。在将来（当然我相信是。。。遥远的将来。。。），一旦这个接口被设计成拥有足够关于context消耗了多少GPU资源的信息，那么CS就真的是scheduler了，就不需要重复的code了。（这一段是翻译过来的，我尽力了）。

其实讲道理CS还是挺关键的，毕竟管理内存分配，但是这个函数本身就是把一些变量初始化成0。



### kbase\_mem\_init & kbase\_device\_coherency\_init

初始化了内存啥的，就跟Linux一样，繁文缛节而已。我决定省略一部分，感觉都是硬件相关的了。



### kbase\_backend\_late\_init

又是一个关键函数，先是通过一个hwaccess\_pm\_powerup方式一顿操作把GPU通上电启动起来了，然后初始化了一个timer，这个timer是用来给Job Scheduler来控制时间的。最后，late\_init初始化了等待列表头。紧接着这个late\_init，kbdev-&gt;kctx\_list被初始化了，这个链表应该是用来存储kernel context信息的。



### misc\_register

在[Linux](http://www.2cto.com/os/linux/)系统中，存在一类字符设备，他们共享一个主设备号（10），但此设备号不同，我们称这类设备为混杂设备（miscdeivce）。 我至今还没有理解这个东西，既然GPU已经被注册了，为什么还要注册一个杂项设备？



好啦，~~小朋友们~~各位，至此为止设备已经被注册到系统里了，GPU被通上了电，内存也被初始化好了，就等着被分配任务了。并且对设备的抽象kbdev也被填充的差不多了，一些要从GPU上读取的信息也读取出来了。~~在爆竹声中~~我们的这一章也结束了







