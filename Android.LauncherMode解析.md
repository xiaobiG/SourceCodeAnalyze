# 一些结论

- standard 标准模式

- singleTop 栈顶复用模式， 推送点击消息界面

- singleTask 栈内复用模式， 首页

- singleInstance 单例模式，单独位于一个任务栈中 拨打电话界面

  ```
  细节： taskAffinity：任务相关性，用于指定任务栈名称，默认为应用包名 allowTaskReparenting：允许转移任务栈
  ```



# Task 和回退栈

当我们的 App 图标在桌面上被点击的时候，App 的默认 Activity——也就是那个配置了 MAIN + LAUNCHER 的 intent-filter 的 Activity——会被启动，并且这个 Activity 会被放进系统刚创建的一个 Task 里。我们通过最近任务键可以在多个 App 之间进行切换，但其实更精确地说，我们是在多个 Task 之间切换。

每个 Task 都有一个自己的回退栈，它按顺序记录了用户打开的每个 Activity，这样就可以在用户按返回键的时候，按照倒序来依次关闭这些 Activity。当回退栈里最后一个 Activity 被关闭，这个 Task 的生命也就结束了。

但它并不会在最近任务列表里消失。系统依然会保留这个 Task 的一个残影给用户，目的是让用户可以方便地「切回去」；只是这种时候的所谓「切回去」，其实是对 App 的重新启动，因为原先的那个 Task 已经不存在了。

所以，在最近任务里看见的 Task，未必是还活着的。



# singleTask

Activity 是一个可以跨进程、跨应用的组件。当你在 A App 里打开 B App 的 Activity 的时候，这个 Activity 会直接被放进 A 的 Task 里，而对于 B 的 Task，是没有任何影响的。

当你在不同的 Task 里打开相同的 Activity 的时候，这个 Activity 会被创建出不同的实例，分别放在每一个 Task 里，互不干扰。这是符合产品逻辑，也是符合用户心理的。

singleTask 可以让 Activity 被别的 App 启动的时候不会进入启动它的 Task 里，而是会在属于它自己的 Task 里创建，放在自己的栈顶，然后把这整个 Task 一起拿过来压在启动它的 Task 上面。

**singleTask 除了保证 Activity 在固定的 Task 里创建，还有一个行为规则：如果启动的时候这个 Task 的栈里已经有了这个 Activity，那么就不再创建新的对象，而是直接复用这个已有的对象；同时，因为 Activity 没有被重建，系统也就不会调用它的 onCreate() 方法，而是调用它的 onNewIntent() 方法，让它可以从 Intent 里解析数据来刷新界面（如果需要的话）；另外在调用 onNewIntent() 之前，如果这个 Activity 的上面压着的有其他 Activity，系统也会把这些 Activity 全部清掉，来确保我们要的 Activity 出现在栈顶。**

那么这样 singleTask 其实是既保证了「只有一个 Task 里有这个 Activity」，又保证了「这个 Task 里最多只有一个这个 Activity」，所以虽然它名字叫 singleTask，但它在实质上限制了它所修饰的 Activity 在全局只有一个对象。

## singleInstance

刚才我说，singleTask 其实是个事实上的全局单例，是吧？那这个 singleInstance 单一实例又是什么意思呢？它的行为逻辑和 singleTask 基本是一致的，只是它多了个更严格的限制：它要求这个 Activity 所在的 Task 里只有这么一个 Activity——下面没有旧的，上面也不许有新的。

具体来说，比如我把编写邮件的 Activity 设置成了 singleInstance，那么当用户在短信 App 里点击了邮件地址之后，邮件 App 不仅会创建这个 Activity 的对象，而且会创建一个单独的 Task 来这个 Activity 放进去，或者如果之前已经创建过这个 Task 和 Activity 了，那就像 singleTask 一样，直接复用这个 Activity，调用它的 onNewIntent()；另外，这个 Task 也会被拿过来压在短信 Task 的上面，入场动画是切换 Task 的动画。这时候如果用户点击返回，上面的 Task 里因为只有一个 Activity，所以手机会直接回到短信 App，出场动画也是切换 Task 的动画；而如果用户没有直接点击返回，而是先看了一下最近任务又返回来，这时候因为下面的短信的 Task 已经被推到后台，所以用户再点返回的话，就会回到桌面，而不是回到短信 App；而如果用户既没有点击返回也没有切后台，而是在编写邮件的 Activity 里又启动了新的 Activity，那么由于 singleInstance 的限制，这个新打开的 Activity 并不会进入当前的 Task，而是会被装进另一个 Task 里，然后随着这个 Task 一起被拿过来压在最上面。

这就是 singleInstance 和 singleTask 的区别：**singleTask 强调的只是唯一性：我只会在一个 Task 里出现；而且这个 Task 里也只会有一个我的实例。而 singleInstance 除了唯一性，还要求独占性：我要独自霸占一个完整的 Task。**

那么在实际的操作中，它们的区别就是：在被启动之后，用户按返回键时，singleTask 会在自己的 App 里进行回退，而 singleInstance 会直接回到原先的 App；以及用户稍后从桌面点开 Activity 所在的 App 的时候，singleTask 的会看到这个 Activity 依然在栈顶，而 singleInstance 的会看到这个 Activity 已经不见了——它在哪？它并没有被杀死，而是在后台的某个地方默默蹲着，当你再次启动它，它就会再次跑到前台来，并被再得到一次 onNewIntent() 的回调。

刚才我说，在最近任务里看见的 Task 未必还活着；那么这里就可以再加一句：在最近任务里看不见的 Task，也未必就死了，比如 singleInstance。

## taskAffinity

那既然它还活着，为什么会被藏起来呢？因为它们的 taskAffinity 冲突了。

在 Android 里，一个 App 默认只能有一个 Task 显示在最近任务列表里。但其实用来甄别这份唯一性的并不是 App，而是一个叫做 taskAffinity 的东西。Affinity 就是相似、有关联的的意思，在 Android 里，每个 Activity 都有一个 taskAffinity，它就相当于是对每个 Activity 预先进行的分组。它的值默认取自它所在的 Application 的 taskAffinity，而 Application 的 taskAffinity 默认是 App 的包名。

另外，每个 Task 也都有它的 taskAffinity，它的值取自栈底 Activity 的 taskAffinity；我们可以通过 AndroidManifest.xml 来定制 taskAffinity，但在默认情况下，一个 App 里所有的 Task 的 taskAffinity 都是一样的，就是这个 App 的包名。当我们启动一个新的 Task 的时候——比如开机后初次点开一个 App——这个 Task 也会得到一个 taskAffinity，它的值就是它所启动的第一个 Activity 的 taskAffinity。当我们继续从已经打开的 Activity 再打开新的 Activity 的时候，taskAffinity 就会被忽略了，新的 Activity 会直接入栈，不管它来自哪；但如果新的 Activity 被配置了 singleTask，Android 就会去检查新的 Activity 和当前 Task 的 taskAffinity 是不是相同，如果相同就继续入栈，而如果不同，新 Activity 就会进入和它自己的 taskAffinity 相同的 Task，或者创建一个新的 Task。

所以当你在 App 里启动一个配置了 singleTask 的 Activity，如果这个 Activity 来自别的 App，就会发生 Task 的切换；而如果这个 Activity 是你自己 App 里的，你会发现它直接进入了当前 Task 的栈顶，因为这种情况下新 Activity 和当前的 Task 的 taskActivity 是相同的。而你如果再给这个 Activity 设置一个独立的 taskAffinity，你又会发现，哪怕是同一个 App，这个 Activity 也会被分拆到另一个 Task 里。而且如果这个独立设置的 taskAffinity 恰好和另一个 App 的 taskAffinity 一样，这个 Activity 还会直接进入别人的 Task 去。

当我们查看最近任务的时候，不同的 Task 会并列展示出来，但有一个前提：它们的 taskAffinity 需要不一样。**在 Android 里，同一个 taskAffinity 可以被创建出多个 Task，但它们最多只能有一个显示在最近任务列表。这也就是为什么刚才例子里 singleInstance 的那个 Activity 会从最近任务里消失了：因为它被另一个相同 taskAffinity 的 Task 抢了排面。**

说到这儿，有一点需要注意，Android 的官方文档在 launchMode 方面的描述有很多的错误和自相矛盾。比如官方文档里说 singleTask 「只会出现在栈底」，但其实完全没有这回事。我们在官方文档里看到的错误一般是什么呢：错别字，或者有歧义、有误导性的表达。但是这个错误说实话让我有点莫名其妙，就是你根本没法猜出来写文档的人的原本想表达的是什么意思，给我的感觉就跟造谣似的。总之你如果在官方文档里看到一些和你的测试结果不符的描述，以你的测试为准；或者如果你发现它有一些话自相矛盾，你就当它没说。

## singleTop

launchMode 除了刚才讲的默认的——也就是 standard——和 singleTask 以及 singleInstance 之外，还有一种叫做 singleTop。singleTop 虽然名字上也带有一个 single，但它的关系和默认的 standard 其实更近一些。它和默认一样，也是会直接把 Activity 创建之后加入到当前 Task 的栈顶，唯一的区别是：如果栈顶的这个 Activity 恰好就是要启动的 Activity，那就不新建了，而是调用这个栈顶的 Activity 的 onNewIntent()。

简单说来就是，默认的 standard 和 singleTop 是直接摞在当前的 Task 上；而 singleTask 和 singleInstance 则是两个「跨 Task 打开 Activity」的规则，虽然也不是一定会跨 Task，但它们的行为规则展现出了很强的跨 App 交互的意图。

**在实战上，我们会比较多地在 App 内部使用默认和 singleTop；singleInstance 会比较多用于那些开放出来给其他 App 一起用的共享 Activity；而 singleTask 则是个兼容派，内部交互和外部共享都用得着。**至于具体用谁，就要根据需求具体分析了。

## 总结

讲了这么多，其实一直都在围绕任务启动和任务切换的问题，瞄准的就是更精准可控的界面导航。如果记不全，Task 的工作模型一定要记住，这是最核心最重要的。别的你都可以忘，这个模型一定记清楚了，这能让你站在一个更高的高度去理解 Android 的 Activity 启动和任务切换，对工作会非常有帮助，而且这些内容是你无论在网上现有的博客还是官方文档里都很难看到的。
