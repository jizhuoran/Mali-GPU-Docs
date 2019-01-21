# 我发誓，我要踢爆你的jm

老伙计，我发誓，我要踢爆你的。。。jm。记得以前不怎么识得洋文得时候，经常看一些中文配音得动画片或者电影，那股浓浓的翻译腔真的很有意思： “ 噢～我的宝贝们，都来看一看吧，这是一个多么欢乐的书啊，简直太让人兴奋了”； “ 嘿，你这只愚蠢的土拨鼠，连英语翻译腔你都不知道，信不信我踢爆你的屁股”。但是由于这本书是由中文写成的，我当然不会胡乱来一通翻译腔。

这一章，我们来讲一讲scheduler三兄弟~~（那个悟空，八戒，沙僧在大观园结拜为兄弟，一起保着松江西天取经的故事）~~：Job Dispatcher，Job Scheduler， Job Manager 中的 Job Manager（JM）。让JM去干活的两个函数分别是**kbase\_jm\_kick**和**kbase\_jm\_kick\_all**，这其中**kbase\_jm\_kick\_all**就是对job slots进行遍历，然后每个job slot都执行一边**kbase\_jm\_kick**。这个kick用的确实形象，当然像我这种暴力的人肯定要加个爆字。在我的家乡，有两个更生动的词来形容这个动作，分别是“juan”和“pai”。这其中“juan”特指有身份的人用脚弓又下向上戏谑的踢身份较低的人，而“pai”更像是一个人（甲）已经把另一个人（乙）打到在地上，然后用偏脚后跟的位置从侧方由上向下的踢。

诶呀，又说闲白，其实我可愿意讲干货了。。。

kbase\_jm\_kick首先会看看当前slot有多少个可以提交新atom的位置，比如说本来有16个位置，现在这个job slot上有7个atom了，那么kbase\_jm\_kick就会query到现在还有9个空位。获得的空位数量之后，kbase\_jm\_kick会调用kbase\_jm\_next\_job。next\_job 会看当前slot上活跃的context是哪个，然后从这个context的相应slot上pull katom，直到pull出之前query的空位个或者context的这个slot里没有katom了。

对于每一个被pull出来的katom，会被传进一个叫做**kbase\_backend\_run\_atom**的函数里，这个run\_atom其实是个中间人。其实我很喜欢中间人这个名字，我每次都会想到比如在墙中间，或者辽阔的草原的正中间，站着一个一脸茫然的人。。。run\_atom会首先把这个katom放到一个红黑树里面，这是driver本身层面的事情。接着，他便念一个口诀，这个口诀我可以教给大家，以后你也能当这个硬件软件的中间人了，听好了，口诀就是“ 叮当当咚咚当当”。好吧，其实run\_atom调用了一个叫做kbase\_backend\_slot\_update的函数，这个函数唯一的参数就是kdev。也就是说，这个函数已经不在乎你之前enqueue进红黑树的katom，它心怀整个设备。

### kbase\_backend\_slot\_update



