# 一杯敬明天，一杯敬过往

说好的一周三更可能要实现不了了，因为还有66分钟就要到第二个周了，而我才刚刚开始写写一段。现在唯一的方法就是我边写边往西方跑以此来增长我这周的时间。其实也不用跑太久，随便到一个斯坦时间就够用了。

\#这里我要给我的朋友插播一条广告，但是把广告写到书里有点太不优雅了，所以我决定把它注释掉。。。很多人说我这本书其实更像~~嫖客~~博客。当我编辑的时候，屏幕会分成三份，左边是驱动的代码，以此来假装我这些都不是瞎编的。中间就是博客的内容，右边摆着《怎样写博客》，而这个《怎么写博客》就来自我朋友的博客：https://hw311.me （其实我一直好奇一点，在他写完《怎么写博客》之前，他是怎么学会写博客的）。他的小站的名字hw311来自香港大学Haking Wong Building的一个机房，一杯敬过往。。。据说他之后会更新一些干活，我猜测可能是操作系统或者计算机语言方面的。

聊闲白真方便，根本不想谈正文，就应该一周三更，三周一个正文，其他的都是闲白，这样读者应该会更多。闲话少说，书归正传，前几天讲了怎么提交katom，做事要有始有终，这一章介绍介绍katom寿终正寝（正常返回）之后会怎么处理。

根据我的观察，Mali的驱动是用一个叫做**kbase\_job\_irq\_handler**的函数来处理返回的job的（其实顾名就能思义）。这个函数会读取JOB\_IRQ\_STATUS这个寄存器（应该是32位的）的内容然后把它传给**kbase\_job\_done**。

### kbase\_job\_done

首先我们会把这个done的高16位和低16位做一个or操作，这样如果某一位是1，就代表相应的job slot不是failure就是finish了。接着，我们会找到最高的是1的位，来进行一系列操作。

我们首先进行failure的处理，我们判断高16位相应的job slot是不是1，如果是的话了，我们就需要处理failure。为我们会先通过JS\_STATUS寄存器检查这个slot的状态。如果状态是JD\_EVENT\_STOPPED，就证明这个job是被soft stop了，就要读取出这个job的JS&lt;n&gt;\_TAIL，之后job chain可以凭此来重启这个job。如果状态是NOT\_STARTED，就知道这个是因为PRLAM-10673导致到停止，（当然我也不知道这个诡异的代码是啥意思）我们就把状态设置成TERMINATED。

最后，我们会根据这个状态来调用kbase\_gpu\_irq\_evict来evict这个job slot的NEXT，至此为止failure就算是处理完了。

在支出去处理完failure之后，函数就会回到主体。首先我们会先给JOB\_IRQ\_CLEAR相应的高16位的和低16位的位置写上1，这样的话了就会把JOB\_IRQ\_STATUS还是JOB\_IRQ\_RAWSTAT相应的位置变成0。

然后，我们会读取JOB\_IRQ\_JS\_STATE这个寄存器的值，记作active。接着，我们会读取这个job slot上有多少个已经提交的job，就是有多少个job处于SUBMITTED的状态，记作nr\_done。如果active的第i位是1，那么就把nr\_done减去1，如果active的第i+16位也是1，那么就再把nr\_done减1。很明显，在这种情况下，nr\_done不应该小于等于0。因为如前面介绍，如果active的第i位是1，就证明GPU上第i个job slot上有活跃的job，如果第i+16位是1，则代表_next上有job。如果两个都有job，这就是很奇怪的事情了，毕竟之所以产生中断就是因为有job完成或者失败了，这时候你告诉我，我就叫着中断玩儿，显然是不合适的。

当然在这里我们要注意一个race condition。如果这个中断是由于current完成而产生的，那么_next上的job就会被送到current上，如果这个时候这个新的current job恰好失败了，那么我们现在并不知道这个failure。所以如果active的第i位是0，并且failure的第i位也是0，我们就需要重新读取JOB\_IRQ\_RAWSTAT，看在调用这个处理函数的时候有没有新的failure产生。如果有的话了，我们就把active的相应位设成1以示记录。


如果nr\_done是1，就证明只有一个job完成了，我们就执行kbase\_gpu\_complete\_hw 并且 执行kbase\_jm\_try\_kick\_all试图再提交一个job到GPU上。

如果nr\_done是2，就说明有超过一个job被完成了，我们就只调用kbase_gpu_complete_hw。因为这不是最后一个被reported，这一次必须passed。这是因为硬件不允许完成后续的job直到失败的job被从IRQ status上清理掉。

最后我们再次从JOB\_IRQ\_RAWSTAT寄存器里读取值到done里，如果读取到的值不为0，就重复执行这个函数。


