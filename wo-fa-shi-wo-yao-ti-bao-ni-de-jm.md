# 我发誓，我要踢爆你的jm

老伙计，我发誓，我要踢爆你的。。。jm。记得以前不怎么识得洋文得时候，经常看一些中文配音得动画片或者电影，那股浓浓的翻译腔真的很有意思： “ 噢～我的宝贝们，都来看一看吧，这是一个多么欢乐的书啊，简直太让人兴奋了”； “ 嘿，你这只愚蠢的土拨鼠，连英语翻译腔你都不知道，信不信我踢爆你的屁股”。但是由于这本书是由中文写成的，我当然不会胡乱来一通翻译腔。

这一章，我们来讲一讲scheduler三兄弟~~（那个悟空，八戒，沙僧在大观园结拜为兄弟，一起保着松江西天取经的故事）~~：Job Dispatcher，Job Scheduler， Job Manager 中的 Job Manager（JM）。让JM去干活的两个函数分别是**kbase\_jm\_kick**和**kbase\_jm\_kick\_all**，这其中**kbase\_jm\_kick\_all**就是对job slots进行遍历，然后每个job slot都执行一边**kbase\_jm\_kick**。这个kick用的确实形象，当然像我这种暴力的人肯定要加个爆字。在我的家乡，有两个更生动的词来形容这个动作，分别是“juan”和“pai”。这其中“juan”特指有身份的人用脚弓又下向上戏谑的踢身份较低的人，而“pai”更像是一个人（甲）已经把另一个人（乙）打到在地上，然后用偏脚后跟的位置从侧方由上向下的踢。

诶呀，又说闲白，其实我可愿意讲干货了。。。

kbase\_jm\_kick首先会看看当前slot有多少个可以提交新atom的位置，比如说本来有16个位置，现在这个job slot上有7个atom了，那么kbase\_jm\_kick就会query到现在还有9个空位。获得的空位数量之后，kbase\_jm\_kick会调用kbase\_jm\_next\_job。next\_job 会看当前slot上活跃的context是哪个，然后从这个context的相应slot上pull katom，直到pull出之前query的空位个或者context的这个slot里没有katom了。

对于每一个被pull出来的katom，会被传进一个叫做**kbase\_backend\_run\_atom**的函数里，这个run\_atom其实是个中间人。其实我很喜欢中间人这个名字，我每次都会想到比如在墙中间，或者辽阔的草原的正中间，站着一个一脸茫然的人。。。run\_atom会首先把这个katom放到一个红黑树里面，这是driver本身层面的事情。接着，他便念一个口诀，这个口诀我可以教给大家，以后你也能当这个硬件软件的中间人了，听好了，口诀就是“ 叮当当咚咚当当，葫芦娃”。好吧，其实run\_atom调用了一个叫做kbase\_backend\_slot\_update的函数，这个函数唯一的参数就是kdev。也就是说，这个函数已经不在乎你之前enqueue进红黑树的katom，它心怀整个设备。

### kbase\_backend\_slot\_update

在这个kbase\_backend\_slot\_update函数里有个两层的for loop，第一层是遍历job\_slot（说实话，装啥啊，一共就三个job\_slots，跟我搁这儿装），然后是对于每个job\_slot的大小遍历（其实每个slot最多放俩。。。）。这里注意到，在第二层for loop里，最终的单元其实还是katom。

1. **WAITING\_BLOCKED：** 首先检查这个katom的dependency满足的没有，要是满足了就把状态更新到下一步，并通过switch的fall through进入下一步的操作，要是没有满足就break。这样的话了，在下次执行kbase\_backend\_slot\_update的时候，再遍历到这个katom的时候，就能从上次不满足的地方继续跑。
2. **WAITING\_PROTECTED\_MODE\_PREV：**这里是要检查一个叫做protect\_mode的东西，虽然我不知道这个protect\_mode具体是干嘛的，但是我知道这个mode有开和关两个状态，在开的时候不能执行需要protect\_mode为关的katom。所以，这里要检查有没有已经进入 4. WAITING\_FOR\_CORE\_AVAILABLE状态的katom和当前的katom需要的protect\_mode冲突，如果冲突就break。
3. **WAITING\_PROTECTED\_MODE\_TRANSITION：**因为之前没有protect\_mode的冲突，所有在这个状态下，我们会按需改变GPU的protect\_mode。
4. **WAITING\_FOR\_CORE\_AVAILABLE：** 在这个状态下，故名而思意，就是在等core准备好。这里有个东西就是BASE\_HW\_ISSUE\_8987，如果当前的设备有这个issue，那么第0个和第1个slots作为一个整体不能同时和第2个slot同时有任务。就是说，如果第0个slot或者第1个slot有任务，那么第2个slot就不能有，反之亦然。这里还有一点就是，如果even\_code是BASE\_JD\_EVENT\_PM\_EVENT，那么就会进入一个叫RETURN\_TO\_JS的状态，这个状态最终会导致这个katom还给Job Scheduler。你踢我一脚，我就把你的鞋抢下来，然后还给你。。。
5. **WAITING\_AFFINITY：** 检查rmu\_workaround，用一句洋文讲就是nothing special。
6. **READY：**到了这一步就是准备好提交给GPU了，但是身为世界上最事儿的driver，肯定还是要吃拿卡要啊。如果这是这个slot里的第1个katom，那么第0个katom必须处于SUBMITTED或者NOT\_IN\_SLOT\_RB的状态。另外，如果KBASE\_SERIALIZE\_INTER\_SLOT这面大旗被竖了起来，那么这第1个katom就不能提交（只能提交第0个）。其次，如果KBASE\_SERIALIZE\_INTER\_SLOT迎风而飘扬，那么同时只能有一个slot上面有任务。就是说如果slot 0有任务了，其他的都只能停在READY这个阶段然后break出去。最后，kbase\_job\_hw\_submit被调用，我们的katom终于要被submit到GPU。**（参考：代码是怎么到GPU的）**
7. **SUBMITTED：**已经被提交了。。。

说实话，我觉得这一章的干货给多了，兑水没兑够。竟然跟那些不熟练的代码辅导员一样心急了。。。



