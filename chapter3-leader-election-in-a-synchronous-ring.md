## 第三章 同步环中的领导者选择

同步模型需解决从一个网络的各个进程中**选择一个唯一的领导者**，这个问题是对局域令牌环网的研究。假如以网络图为环形的case（进程环），当一个“令牌”在网络中循环，令牌的当前持有者拥有发起通信的唯一特权。然而，有时候这个令牌会丢失，所以进程有必要执行一个算法来重新产生丢失的令牌。这个重新产生的过程就相当于选择一个领导者。

### 3.1 问题

* 可能要求所有非领导者的进程把其状态中的特定部分改为“非领导”值，最终输出自己不是领导者。
* 环有可能是单向也有可能是双向的。如果是单向的，每条边都从一个进程指向其顺时针的领接节点，也就是说，消息只能依照顺时针的方向发送。
* 在环中，对于进程来说，节点的个数可能是已知的，也有可能是未知的。如果是已知的，则意味着进程只需要在规模为n的环中正确工作，因此，它们可以在程序中利用值n。如果是未知的，就意味这个进程必须能在节点数不同的环中工作，此时不能使用有关环规模的信息。
* 进程可以是相同的，也可以用进程开始时的唯一标识符\(UID\)来区分，UID取值于某个大的全序标识符空间，如正整数N+。我们假设在环中每个进程的UID都是不一样的，但是并不限定哪些UID实际出现在环中。这些标识符可以只限于接受某些特定操作，如比较操作，或者它们允许不受限制的操作。

### 3.2 相同进程的不可能性结果

* 令A是一个拥有n个进程的双向环系统，其中n&gt;1。如果在A中的所有进程都是相同的，那么A不可能解决领导者选举问题。当所有的进程在r轮后都会处于相同状态，如果A中所有的进程都在同一时刻达到了领导者状态，就违反了唯一性的要求。

### 3.3 基本算法

* LCR算法：每个进程都绕环发送自己的标识符。当一个进程收到一个输入标识符时，就把它和自己的标识符做比较。如果输入的标识符比自己的标识符大，就继续传递这个标识符；如果这个标识符比自己的标识符小，就丢弃输入标识符； 如果和自己的标识符相同，就宣称自己是leader。**这个算法最大UID是leader**
* LCR解决了领导者选举问题。
* 停止和“非领导者”输出，从上面可知，所有进程都达到一个停止的状态来讲，LCR算法永远都无法完成其工作。可以像2.1节中描述的那样扩充每个进程，使它们都包括停止状态。然后就可以修改算法，让选出的领导者在网络中发送一条特殊的报告消息。任何手段这个报告消息的进程可以在传递该消息之后停止。这样的策略不仅可以允许进程停止，还可以让不少领导者的进程输出“非领导者”。更进一步，在消息上如果附上领导者进程的下标，这个策略可以让所有参与的进程都输出领导者的身份。注意，让每个不是领导者的节点看到一个UID比自己大时立即输出“**非领导者**”是可能的；但是这并不能告知非领导者节点停止。
* 复杂度分析，直到宣布一个领导者为止，基本LCR算法的时间复杂度是n轮，而通信复杂度在最坏的情况下是O（n^2^）。在带停止的算法版本中，时间复杂度是2n，但是通信复杂度还是O\(n^2^\)。停止和“非领导者”声明所需的额外时间是n轮，通信所需要的额外消息是n个。
* 变形，以上两部分描述和分析了一个基本是算法变形，即从只有领导者进程产生输出且没有进程停止的任意一种领导者选举算法，到一个领导者进程和非领导在进程都有输出并且所有进程都停止的算法。获取额外输出和停止的额外开销就是n轮和n个消息。对于其他任意的假设组合，这个变形都成立。
* 可变的开始时间，注意在具有可变的开始时间的同步模型中，LCR算法无需改变。
* 打破对称性，在一个环中选举领导者这个问题中，关键的难点是打破对称性。在分布式系统中需解决包括**资源分配**，**一致性**等问题，打破对称性也是其中一个重要的部分。

### 3.4 通信复杂度为O（nlog^n^）的算法

虽然LCR算法的时间复杂度比较低，但是用到的消息的数据比较大，共有O（n^2^）。这看起来好像不是很重要，因为在任一时刻任一链路中的消息不会多于一个。但是，在第二章中，我们已经讨论了为什么消息条数是一个值得注意的应该是越少越好的衡量标准；许多并发运行的分布式算法，其总的通信负载可能导致网络的拥塞。本节，我们将要提出一个算法，它把通信复杂度降为O（nlog^n^）。

* HS算法，我们假定只有领导者需要执行输出，尽管在3.3节结尾中提到的变形暗示这个限制并不重要。另外，我们假设**环的规模是未知的**，但是现在允许**双向通信**。HS算法也选出具有最大的UID的进程。每个进程，不是像LCR算法那样把自己的UID在网络中一直传递下去，而是把它传到某个距离之外，然后反转回来，再回到原来的进程。它如此反复逐步增大距离。
* 非形式化的运行方式，每个进程i在阶段0，1，2.......中操作。每个阶段L中，进程i向两个方向发出包含自己的UID\(u~i~\)的“令牌”。这些令牌会前进2的距离，然后再回到自己原来的进程i。如果两个令牌都能够安全回来，那么进程i就会继续下个阶段。但是，令牌可能不会安全返回。当一个u~i~的令牌在外出方向前进时，任何在u~i~的路径上的进程j都会把自己的UID\(u~j~\)和u~i~进行比较。如果u~i~ &lt; u~j~，那么进程j就会丢弃这个令牌；如果u~i~ &gt; u~j~，它就会继续传递u~i~。如果u~i~=u~j~，则说明进程j在令牌反转之前就以及接受到自己的UID，所以进程j就会把自己选为领导者。所有进程总是继续传递所有进入方向的令牌。

### 3.5 非基于比较的算法

下面考虑是否可能以少于O（nlog^n^）的消息数来选出领导者。这个问题的答案是否定的（基于比较算法），正如我们将会简短证明一个不可能性结果    O（nlog^n^）的下界。本节中，我们允许UID为正整数，且允许对它们进行一般的数学操作。在这种情况下，我们给出两个算法，时间片（TimeSlice）算法和变速（VariableSpeeds）算法，它们都是消息数为O（n）的算法。这两个算法的存在说明在一般情况下，有可能低于O（nlog^n^）是可能的。

#### 3.5.1 时间片算法

假设：环的大小n对所有进程来说是已知的，但只假定是单向通信。它选择具有最小UID的进程。注意，这个算法比LCR和HS算法更深入地利用同步性。它在某些轮处利用了非到达消息（到来的消息为null）来传递信息。

在阶段1，2.........中进行计算，**每个阶段都由n个连线的轮组成**。每个阶段都由一个携带着特殊UID的令牌一直在环中流转。更特别地，在包含轮（v-1）n+1，.......，v~n~中，只有带着UID~v~的令牌才被允许流转。如果存在UID为v的进程i，且在（v-1）n+1轮到达之前进程i没有收到任何非空消息（**表示投票成功**），那么进程i就会把自己选举为领导者，并沿着环发送一个包含其UID的令牌。当这个令牌在传递的时候，所有其他进程都接收到它，**这样它们就不会选举自己为领导者或在以后的阶段中创建令牌**。

时间片算法的好处就是消息的总数是n个。但不幸的是，时间复杂度大约是n\*u~min~，这在固定规模的环中也是一个无界数。这个时间复杂度限制了这个算法的应用。它只能用于那些小的环网络中，其中的UID都是很小的正整数。

#### 3.5.2 变速算法

时间片算法表明，在进程知道环的规模为n的那些环中，消息数为O（n）就足够了。但是如果n是未知的呢？在这种情况下，同样存在消息为O（n）的算法，我们称为**变速算法**。这个算法只使用单向通信。

但是，变速算法的时间复杂度甚至比时间片算法还要差，为O（n_2^u^ ~min~）。当然，没有人会在实际中运用这样的算法。我们称为变速算法为一个\*反例算法_。一个反例算法的主要目的是说明某个推想的不可能性结果是错误的。这种算法本身并无意义。

每个进程i创建一个令牌，令牌带着发起进程i的UID U~I~在网络中流动。不同令牌的流动速度是不一样的。一个带着UID v的令牌的速度为每2^v^轮传递一条消息，也就是说每个进程在接受这个令牌后需要等待2^v^轮之后才能将它发出。同时，每个进程都会保存自己见到的最小UID，并丢弃任何包含比最小UID更大的UID的令牌。如果令牌返回到创建进程，那么这个进程就被选为领导者。**变速算法**保证了拥有最小UID的进程会被选出。

#### 3.6 基于比较的算法的下界

基于比较算法，最小通信复杂度为O（nlogn\)，时间复杂度O（n），消息数的下界Ω（nlogn）。

#### 3.7 非基于比较的算法的下界

非基于比较算法，更大的时间复杂度代价，消息数的下界O（n）。

