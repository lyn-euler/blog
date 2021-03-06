---
title: "WWDC20-OC runtime改进"
subtitle: "Objective-C runtime 改进"
description: "Objective-C runtime 改进"
icon: "flag"
---
## 类结构体变化
在你的应用程序的磁盘上，二进制类是这样的.
![截屏2020-06-28 18.11.38](/15933251515193/%E6%88%AA%E5%B1%8F2020-06-28%2018.11.38.png)

首先是类对象本身，它包含了最常被访问的信息：指向元类、超类和方法缓存的指针.它还有一个指向更多数据的指针.存储额外信息的地方叫做 `class_ro_t`.RO代表只读，该结构体包括像类名，方法、协议和实例变量的信息.
![](/15933251515193/15933391421728.png)

Swift类和Objective-C类共享这个基础架构，所以每个Swift类也有这些数据结构.当类第一次从磁盘加载到内存中时，它们一开始也是这样的.但是一旦使用了它们，它们就会发生变化.
在阐述具体发生什么之前先了解两个概念:
- **Clean Memory**: 是指一旦加载后就不会改变的内存
  比如`class_ro_t`就是clean的, 因为它是只读的.
- **Dirty Memory**: 脏内存是指在进程运行时被改变的内存
  类结构一旦被使用，就会被弄脏，因为运行时会向它写入新的数据(例如它创建了一个新的方法缓存，并从类中指向它). 
  Dirty Memory 要比 Clean Memory 昂贵的多, 因为只有进程在运行，它就必须一直存在; 而Clean Memory是不变的, 如果需要系统能从磁盘中重新加载它, 所以可以从内存中移除.
  MacOS 中有swap脏内存的可选操作, 但是因为iOS没有使用swap,脏内存在iOS中的代价会特别大.出于这个原因,能保持干净的数据越多越好, 原来的类结构体被划分为两块.通过分离出那些永远不会改变的数据，那就可以把大部分的类数据作为干净的内存来保存.
  这些数据足以让我们开始使用，但运行时需要跟踪每个类的更多信息, 所以，当一个类第一次被使用时，运行时会为它分配额外的存储空间. 这个运行时分配的存储是可读可写的数据`class_rw_t`.在这个数据结构中，我们存储了只有在运行时产生的新信息.例如，所有的类都会使用这些First Subclass和Next Sibling Class指针链接成一个树形结构.
  ![截屏2020-06-28 18.15.03](/15933251515193/%E6%88%AA%E5%B1%8F2020-06-28%2018.15.03.png)
而且这允许运行时遍历当前使用的所有类，这对于无效的方法缓存很有用.但是，既然方法和属性也在只读数据中，为什么我们要在这里有方法和属性呢？嗯，因为它们可以在运行时改变.当一个Category被加载时，它可以向类中添加新的方法.而且程序员可以使用运行时API动态添加它们.由于class_ro_t是只读的，所以我们需要在`class_rw_t`中跟踪这些东西.
![截屏2020-06-28 18.15.32](/15933251515193/%E6%88%AA%E5%B1%8F2020-06-28%2018.15.32.png)

  现在发现，这样做会占用不少的内存.在任何一个给定的设备中，都有很多类在使用.我们在一台iPhone上测得整个系统中大约有30MB的这些`class_rw_t`结构.
  那么我们如何才能缩小这些呢？请记住，我们在读/写部分都需要这部分数据，因为它们可以在运行时改变.但是......检查实际设备上的使用情况，苹果发现只有10%左右的类真正改变过它们的方法.而且这个Demangled Name字段只有Swift类才会使用，除非有东西询问他们的Objective-C名称，否则Swift类也根本不需要它.所以，我们可以把那些平时不用的部分拆掉.
![截屏2020-06-28 18.18.28](/15933251515193/%E6%88%AA%E5%B1%8F2020-06-28%2018.18.28.png)
![](/15933251515193/15933395479251.png)


  这样一来，class_rw_t的大小就减少了一半.对于那些确实需要额外信息的类，我们可以分配一个这样的扩展记录，然后把它滑到类中供其使用.大约90%的类从来不需要这些扩展数据，整个系统节省了大约14MB.这些内存现在可以用于更有成效的用途，比如存储你的应用程序的数据.
  因此，你可以通过在终端中运行一些简单的命令，亲自在 Mac 上看到这一变化的影响.
```shell
# note:确认对应的App在运行
heap Mail | egrep 'class_rw|COUNT'
```
![](/15933251515193/15933396611691.png)

而从返回的结果中，我们可以看到，我们在邮件应用中使用了大约9000个这样的class_rw_t类型，但实际上其中只有大约十分之一，900多一点，需要使用这个扩展信息. 单邮件这一个应用我们就节省了大约25万兆的数据. 如果我们在系统范围内进行扩展，那就真正能节省了很多脏内存带来的开销.
修改之后很多从类中获取数据的代码必须同时处理那些有和没有扩展数据的类.当然，因为读取和更新这些结构的代码都在runtime内, runtime内部会为你处理所有这些操作，从外部看一切都像以前一样，只是使用更少的内存.
所以尽量使用runtime提供的API，因为这部分数据结构的更改, 任何第三方试图直接访问这些数据结构的代码在今年的操作系统版本中都会停止工作.

## 方法列表变化
接下来，让我们再深入了解一下这些类的数据结构，看看另一个变化：相对方法列表(relative method lists).每个类都有一个附加的方法列表.当你在一个类上写一个新方法时，它就会被添加到列表中.runtime使用这些列表来解析消息发送.
![截屏2020-06-28 18.23.32](/15933251515193/%E6%88%AA%E5%B1%8F2020-06-28%2018.23.32.png)

每个方法都包含三条信息:
首先是方法的名称，或者说`Selector`;`Selector`是字符串，但它们是唯一的，所以它们可以使用指针平等来比较.
其次是方法的类型编码(type encoding).这是一个表示参数和返回类型的字符串，它不是用来发送消息的，但它是运行时反省(introspection)和消息转发(forwarding)等必需的.
最后，还有一个指向方法的实现的指针--方法的实际代码.当你写一个方法时，它会被编译成一个C函数，里面有你的实现，然后条目(entry)和方法列表都指向这个函数.让我们看看一个具体的方法.
![](/15933251515193/15933399137465.png)

我选择了`init`方法.它包含了方法名、类型和实现.方法列表中的每一条数据都是一个指针.
在我们的64位系统中，这意味着每个方法表条目占用24个字节.现在这是干净的内存，但干净的内存并不是免费的.它仍然必须从磁盘加载，并且在使用时占用内存.现在这里是一个进程内内存的放大视图(NOTE:这不是按比例放大的).
![](/15933251515193/15933399608276.png)

有一个很大的地址空间，需要64位来寻址.在这个地址空间内，为栈、堆以及加载到进程中的可执行文件和库或二进制映像划分出了不同的部分，这里用蓝色显示.让我们放大来看其中的一个二进制镜像.
![截屏2020-06-28 18.26.51](/15933251515193/%E6%88%AA%E5%B1%8F2020-06-28%2018.26.51.png)
这里我们显示了三个方法表项指向其二进制中的位置.这向我们展示了另一个成本.二进制图像可以加载到内存中的任何地方，这取决于动态链接器决定把它放在哪里.这意味着链接器需要在加载时将指针解析到镜像中，并将其fix up为指向其在内存中的实际位置.而这也是有代价的.
现在，请注意，一个来自二进制的类方法条目永远只指向该二进制内的方法实现.没有办法让一个方法的元数据在一个二进制中，而实现它的代码在另一个二进制中.这意味着方法列表条目实际上不需要能够引用整个64位地址空间.它们只需要能够引用自己二进制中的函数，而且这些函数总是在附近.因此，他们可以在二进制中使用一个32位的相对偏移，而不是一个绝对的64位地址.
这也是苹果今年做的一个改变.

![截屏2020-06-28 18.28.01](/15933251515193/%E6%88%AA%E5%B1%8F2020-06-28%2018.28.01.png)

这有几个好处.首先，无论image在哪里加载到内存中，偏移量始终是相同的，所以它们不必在从磁盘加载后进行修正. 而且因为它们不需要被修正起来, 它们可以被保存在真正的只读内存中，这样更安全.当然，32位的偏移意味着我们已经将64位平台上所需的内存量减少了一半.
![截屏2020-06-28 18.28.53](/15933251515193/%E6%88%AA%E5%B1%8F2020-06-28%2018.28.53.png)

苹果工程师在一个典型的iPhone上测得关于BOMB的这些方法的系统范围, 节省了40MB.

**但是swizzling呢**？二进制中的方法列表现在不能引用整个地址空间.但是，如果你swizzle一个可以在任何地方实现的方法，而且，我们刚刚说过，我们希望保持这些方法列表只读.为了处理这个问题，苹果也提供了一个全局表，将方法映射到它们被swizzle的实现上.Swizzling是很少见的.绝大多数方法实际上从未被swizzle过，所以这个表最终不会变得很大.更好的是这个表很紧凑.
我们知道内存是按页分配的, 在旧的方法列表实现中, swizzle一个方法会弄脏它所在的整个页面，导致swizzle一次就会弄脏很多千字节的内存.有了表，我们只需为一个额外的表条目付出代价.一如既往，这些变化对你来说是看不见的，一切都会像以前一样继续工作.这些相对方法列表在今年晚些时候推出的新的操作系统版本上得到了支持.

当你使用相应的最小部署目标进行构建时，工具会自动在你的二进制文件中生成相对的方法列表，如果你需要针对老版本的OS，不用担心, Xcode也会生成老式的方法列表格式.你仍然可以从操作系统本身的构建中获得新的相对方法列表的好处，而且系统在同一应用程序中同时使用两种格式也没有问题.不过如果你能针对今年的OS版本构建，你会得到更小的二进制文件和更少的内存使用.
这在Objective-C或Swift中是一个普遍不错的提示.最小部署目标并不只是关于哪些SDK API可以给你使用.当Xcode知道它不需要支持旧的操作系统版本时，它通常可以发出更好的优化代码或数据.我们理解你们中的许多人需要支持旧的OS版本，但这也是为什么无论何时增加部署目标都是一个好主意的原因.现在，有一件事需要注意，那就是使用比你打算使用的部署目标更新的部署目标进行构建.Xcode通常会防止这种情况发生，但也有可能漏掉，特别是当你在其他地方构建自己的库或框架，然后将它们带进来的时候.当运行在旧的操作系统上时，旧的运行时会看到这些相对方法，但它对它们一无所知，所以它会尝试像解释旧式的基于指针的方法一样解释它们.这意味着它将尝试把一对32位的字段作为64位的指针来读取.结果是两个整数被粘在一起作为一个指针，这是一个无意义的值，如果真的使用它，肯定会崩溃.你可以通过运行时读取方法信息时的崩溃来识别这种情况的发生，一个坏的指针看起来就像两个32bit的值被平滑在一起，就像这个例子.
![](/15933251515193/15933404013134.png)

如果你运行的代码挖掘这些结构来读出值，那这段代码就会出现和这些旧运行时一样的问题，当用户升级设备时，App就会崩溃.所以，还是不要这样做--使用API.不管底层的东西怎么变，那些API都能继续工作. 例如，有一些函数，给定一个方法指针就会返回其字段的值.


## Tagged Pointer变化
我们再来探讨一下今年即将到来的一个变化：ARM64上`Tagged Pointer`格式的变化.首先，我们需要知道什么是`Tagged Pointer`.
我们将在这里探索底层真正的实现，但不要担心--就像我们谈过的其他事情一样，你不需要知道这些.它只是很有趣--也许能帮助你更好地理解你的内存使用情况.让我们从普通对象指针的结构开始.
通常，当我们看到这些指针时，它们被打印成这些大的十六进制数字.我们在前面看到了一些这样的东西.让我们把它分解成二进制表示.
![](/15933251515193/15933404859945.png)
我们有64位的空间.然而，我们并没有真正使用所有这些位.只有中间的几位在真正的对象指针中被设置.低位总是0，因为对齐要求：对象必须总是位于一个指针大小的倍数的地址中.高位总是0，因为地址空间是有限的：我们实际上不会用到2^64.这些高位和低位总是0.
所以让我们从这些始终为0的位子中挑出一个位子，让它变成1.这可以立即告诉我们这不是一个真正的对象指针，然后我们可以给所有其他位子赋予一些其他的意义.我们称之为`Tagged Pointer`.
![截屏2020-06-28 18.35.49](/15933251515193/%E6%88%AA%E5%B1%8F2020-06-28%2018.35.49.png)

例如，我们可以在其他位中塞入一个数值.只要我们想教NSNumber如何读取这些位，并让运行时适当地处理标签指针，系统的其他部分就可以把这些东西当做对象指针来处理，永远不会知道其中的区别.而这也为我们节省了为每一种这样的情况分配一个微小的数字对象的开销，这可能是一个重大的改进.这些值实际上是通过将它们与一个进程启动时初始化的随机值相结合来混淆的, 这是一种安全措施，它使伪造标记指针值变得困难.
![截屏2020-06-28 18.36.11](/15933251515193/%E6%88%AA%E5%B1%8F2020-06-28%2018.36.11.png)

在接下来的讨论中，我们将忽略这一点，因为它只是在上面增加了一层.只是要注意，如果你真的试图在内存中查看这些值，它们会被扰乱.所以这就是Intel上标记指针的完整格式.低位被设置为1，表示这是一个`Tagged Pointer`.正如我们讨论过的，对于一个真实的指针来说，这个位必须始终为0，所以这让我们可以区分它们.接下来的3位是标签号.这表示标记指针的类型.例如，3表示它是一个NSNumber，6表示它是一个NSDate.由于我们有3个标签位，所以有8种可能的标签类型.其余的位是有效载荷(payload), 这是特定类型可以随意使用的数据.对于标记的NSNumber，这是实际的数字.
![](/15933251515193/15933407448038.png)
现在标签7有一个特殊情况.这表示一个扩展标签.扩展标签使用下一个八位来对类型进行编码，允许多出256个标签类型，但代价是减少有效载荷.这使得我们可以将标签指针用于更多的类型，只要它们能够将其数据装入更小的空间.这被用于像标记`UlColor`或`NSIndexSet`这样的东西.如果这对你来说非常方便，你可能会失望地听到只有运行时维护者--也就是苹果--可以添加`Tagged Pointer`类型.

但如果你是一个Swift程序员，你会很高兴地知道你可以创建自己的标签指针类型. 如果你曾经使用过一个具有关联值的枚举，那就是一个类似于`Tagged Pointer`的类.Swift运行时将枚举判别器存储在关联值有效载荷的备用位中.更重要的是，Swift对值类型的使用实际上使`Tagged Pointer`变得不那么重要了，因为值不再需要完全是指针大小.例如，Swift的UUID类型可以是两个字，并保持在内联，而不是分配一个单独的对象，因为它不适合在一个指针里面.
这就是英特尔上的标记指针.我们来看看ARM.在ARM64上，苹果把事情翻转过来了.

![](/15933251515193/15933408987373.png)

最高位(而不是最低位)被设置为1，表示一个`Tagged Pointer`.然后标签号在接下来的三个位中出现.然后，有效载荷使用剩余的位.为什么苹果在ARM上使用顶部位来指示标记指针，而不是像在英特尔上那样使用底部位？嗯，这实际上是对objc_msgSend的一个小小的优化.苹果希望msgSend中最常见的路径尽可能快.而最常见的路径是一个普通的指针.我们有两种不太常见的情况：`Tagged Pointer`和nil.事实证明，当我们使用最高位时，我们可以通过一次比较来检查这两种情况.而且在msgSend中，这样就为常见的情况节省了一个条件分支，而不是分别检查nil和`Tagged Pointer`.就像在英特尔上，对标签7表示一个特殊的情况，接下来的8位被用作扩展标签，然后剩下的位被用于有效载荷.或者说这其实是旧的格式，在iOSl3中使用.在今年的版本中.我们把东西移动了一下! 标签位保持在最高位，因为那个msgSend的优化还是非常有用的.标签号现在移到了最下面的三个位.
扩展标签如果使用，则占据标签位后的高八位.
![截屏2020-06-28 18.42.28](/15933251515193/%E6%88%AA%E5%B1%8F2020-06-28%2018.42.28.png)

**为什么要这样做呢**？好吧，我们再考虑一个普通的指针.我们现有的工具，比如动态链接器，由于ARM的一个名为Top Byte Ignore的特性，忽略了指针的前8位.而我们会把扩展标签放在Top Byte Ignore位.对于一个对齐的指针来说，底部三个位总是0.但我们可以通过在指针中添加一个小数字来改变这一点.
![](/15933251515193/15933410970189.png)

我们将添加7来将低位设置为1.请记住，7 表示这是一个扩展标记.这意味着我们实际上可以将上面的这个指针放入一个扩展标签指针有效载荷中.结果就是一个标签指针，其有效载荷中包含一个普通指针.为什么这很有用呢？嗯，它开启了标记指针引用二进制中的常量数据的能力，例如字符串或其他数据结构，否则它们将不得不占用肮脏的内存.当然，现在这些变化意味着，当iOSl4今年晚些时候发布时，直接访问这些位的代码将不再工作.像这样的位检查在过去是可以工作的，但在未来的操作系统上会给你错误的答案，你的App会开始神秘地破坏用户数据.所以不要使用依赖于我们刚才谈到的任何代码.相反，你大概可以猜到我要说什么：也就是使用API.像isKindOfClass：这样的类型检查在旧的标记指针格式上工作，它们将继续在新的标记指针格式上工作. 所有的NSString或NSNumber方法都能继续工作.这些标记指针中的所有信息都可以通过标准的API来检索.
值得注意的是，这也适用于CF类型.苹果表示他们不想隐藏任何东西，也绝对不想破坏任何人的Apps. 当这些细节没有暴露出来的时候，只是因为他们需要保持灵活性来进行这样的改变，只要你的App不依赖这些内部细节，你的App就会继续正常工作.

那么，我们来总结一下.在这次Session中，我们已经看到了一些幕后的改进，这些改进缩小了我们运行时的开销，将更多的内存留给你和你的用户.你不需要做任何事情就能获得这些改进--除了可能考虑提高你的deployment target.

