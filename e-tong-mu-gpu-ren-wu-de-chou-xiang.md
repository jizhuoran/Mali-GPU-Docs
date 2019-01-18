# 阿童木：GPU任务的抽象

如果你生于中国大陆，长于中国大陆，那么除了对祖国深深的爱，你还一定知道"鉄腕アトム"翻译过来是铁臂阿童木。其实，铁臂阿童木不光胳膊是铁的，不然他可能不能合理地站到地面上，或者要拿着大顶。。。所以阿童木和我们这章要关注的内容有什么关系，其实没啥关系。。。有毒吧，launch GPU程序的时候你又不需要拿着大顶。 当然，如果随便起个这个中二的名字作为章节名字，那太没有品味了。所以，其实，在Mali驱动里，对GPU任务的抽象class叫做atom。。。

可以猜到，应该有两种atom，一个是用户创建的要传给driver的（base_jd_atom_v2），一个是driver内部对任务的抽象（kbase_jd_atom）。在我开来这种设计最大的好处就是把和任务本身无关的信息都存在driver的抽象里，在base_jd_atom_v2里只有12个变量，然而在kbase_jd_atom里有五十多个变量，多出来的这些变量就是又硬件生成的，用户根本不想关心的玩意儿。

对于base_jd_atom_v2，最重要的一个变量就是**jc**，这个变量是个地址，指向前一章提到的那段GPU机器码。~~我没找到资料~~不难猜测，当host代码准备launch GPU代码的时候，就是造个base_jd_atom_v2，GPU代码用OpenCL啥的编好，拿这个**jc**一指，最后通过中断调用这个API。

同时，用户还可以指定一个**prio**，用来代表这个任务的priority。还有一个变量叫做**atom_number**，相当于这个atom的id，应该是程序自己决定的。当然，因为GPU允许异步执行，这就要考虑依赖的问题，所以用户可以把当前atom所依赖的atom(s)加到一个叫**pre_dep**的变量里。这个变量是个class，里面只有两个变量，一个是atom的id，一个是依赖的种类。只有当依赖都满足的时候，这个atom才会被（执行 || 提交到GPU里）。

花开两朵，各表一枝。对于kbase_jd_atom，就是driver眼中的任务了，既然是driver眼中的，那我们为什么要关心呢？所以就不讲了吧。。。好吧，这本书叫driver详解。

1. 首先katom里有一个work_struct，就是Linux的work_queue里放的那个work_struct。
2. 有一个时间戳，这个时间戳记录了这个atom是什么时候提交给GPU的
3. kctx：指向这个atom属于哪个context，这个东西不在base_jd_atom_v2里，所以提交base_jd_atom_v2的时候需要把kctx也给传进去
4. 有三个关于dependency的东西，dep_head存储了被这个atom block住的atoms，dep_item存储了和这个atom依赖相同东西的atoms，最后dep里有这个atom所依赖的两个atom
5. 指向GPU机器码的jc和其复制softjob_data
6. 还有其他许许多多的内容
