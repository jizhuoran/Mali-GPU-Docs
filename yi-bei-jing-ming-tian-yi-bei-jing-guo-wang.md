# 一杯敬明天，一杯敬过往

说好的一周三更可能要实现不了了，因为还有66分钟就要到第二个周了，而我才刚刚开始写写一段。现在唯一的方法就是我边写边往西方跑以此来增长我这周的时间。其实也不用跑太久，随便到一个斯坦时间就够用了。

\#这里我要给我的朋友插播一条广告，但是把广告写到书里有点太不优雅了，所以我决定把它注释掉。。。很多人说我这本书其实更像~~嫖客~~博客。当我编辑的时候，屏幕会分成三份，左边是驱动的代码，以此来假装我这些都不是瞎编的。中间就是博客的内容，右边摆着《怎样写博客》，而这个《怎么写博客》就来自我朋友的博客：https://hw311.me （其实我一直好奇一点，在他写完《怎么写博客》之前，他是怎么学会写博客的）。他的小站的名字hw311来自香港大学Haking Wong Building的一个机房，一杯敬过往。。。据说他之后会更新一些干活，我猜测可能是操作系统或者计算机语言方面的。

聊闲白真方便，根本不想谈正文，就应该一周三更，三周一个正文，其他的都是闲白，这样读者应该会更多。闲话少说，书归正传，前几天讲了怎么提交katom，做事要有始有终，这一章介绍介绍katom寿终正寝（正常返回）之后会怎么处理。

根据我的观察，Mali的驱动是用一个叫做**kbase\_job\_irq\_handler**的函数来处理返回的job的（其实顾名就能思义）。这个函数会读取JOB\_IRQ\_STATUS这个寄存器（应该是32位的）的内容然后把它传给**kbase\_job\_done**。

### kbase\_job\_done

如果job slot报告了一个failure，那么就会先通过相应的寄存器检查这个slot的状态。如果状态是JD\_EVENT\_STOPPED，就证明这个job是被soft stop了，就要读取出这个job的JS&lt;n&gt;\_TAIL，之后job chain可以凭此来重启这个job。如果状态是NOT\_STARTED，就知道这个是因为PRLAM-10673导致到停止，（当然我也不知道这个诡异的代码是啥意思）我们就把状态设置成TERMINATED。最后调用kbase\_gpu\_irq\_evict来evict这个job slot的NEXT。

在支出去处理完failure之后，函数就会回到主体。我们先给JOB\_IRQ\_CLEAR写上相应的值，然后读取JOB\_IRQ\_JS\_STATE这个寄存器的值，记作active。

接着，我们会读取这个job slot上有多少个已经提交的job，就是有多少个job处于SUBMITTED的状态，记作nr\_done。如果active的第i位是1，那么就把nr\_done减去1，如果active的第i+1位也是1，那么就再把nr\_done减1。很明显，在这种情况下，nr\_done不应该小于0。

如果nr\_done是1，就证明只有一个job完成了，我们就执行kbase\_gpu\_complete\_hw 并且 执行kbase\_jm\_try\_kick\_all试图再提交一个job到GPU上。

如果nr\_done是2，就说明有超过一个job被完成了。因为这不是最后一个被reported，这一次必须passed。这是因为硬件不允许完成后续的job直到失败的job被从IRQ status上清理掉。

之后再次从JOB\_IRQ\_RAWSTAT寄存器里读取值到done里，如果读取到的值不为0，就重复执行这个函数。



感觉这个处理完成的job的逻辑远远比我想象的要复杂，下个周会再详细解释这一部分。感觉很对不起大家，草草写完这一章。我发现强行一周三更会损害质量，所以我决定随心所欲，不逾矩。







请无视这一段（如果active的第i位和done的第i+1位都是0，那么会有潜在的race。在这种情况下，一个job slot在current和next寄存器上都有job，在current上的job成功的完成了，IRQ handler读取了RAWSTAT并且在调用这个function的时候把相应的bit设置成了“done”。在next寄存器上的job变成了在current寄存器上的job，）

