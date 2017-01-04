---
layout: post
title: RFC2544 网络互联设备的基准测试方法论
date: 2016-12-30 10:00:00 +0800
categories: rfc
tags:
- rfc2544
- translation
---

本文对因特网 IETF 发布的 [RFC2544][rfc2544] 文档进行了翻译，并删除了一些不影响理解文档的内容。译者认为本文档的重点内容为 [Section 9][9] 和 [Section 26][26]。

<h6><center>------以下是翻译正文------</center></h6>

网络工作组<span style="float:right">S. Bradner</span>

请求评议: 2544<span style="float:right">Harvard University</span>

原始文档: 1944<span style="float:right">J. McQuaid</span>

分类: 信息化<span style="float:right"> NetScout Systems</span>

<span style="float:right">March 1999</span><br>

<center style="font-size:22px">网络互联设备的基准测试方法论</center><br>

<h4>因特网工程指导小组声明</h4>

本文档是 [RFC1944][rfc1944] 的再版，修正了IP地址中被分配用于网络测试设备的默认值（见 C.2.2 节），本文档将取代并淘汰RFC1944文档。

<h4>摘要</h4>

本文档讨论并定义了一组被用于描述网络互连设备表现特征的测试。除此之外，本文档还描述了用于报告测试结果的特定格式。[附录 A][appendixA] 列出了我们认为应该包含的某些特别个案的测试和相关条件，并给出了测试实践的补充信息。[附录 B][appendixB] 是一份在不同媒介上特定帧长的最大帧频的参考列表。[附录 C][appendixC] 给出了一些用于测试的帧格式实例。

<h4>1. 简介</h4>

供应商经常忙于制定“规范”，以试图让他们的产品在市场上一个更高的地位。这通常涉及到使用“烟雾和镜子”（译者注：成语，含义为欺诈）的手段来迷惑潜在的产品使用者。

本文档定义了一组特定的测试集合使卖方能评估报告网络设备的表现特征。这些测试结果将给用户提供来自不同供应商的可比性数据，从而使他们能够评估这些设备。

“网络互连设备的基准术语”（RFC1242）定义的很多术语都被用于此文档，在使用本文档前应该先阅读 [RFC1242][rfc1242]。

<h4>2. Real world</h4>

在撰写本文档过程中，作者试图牢记那些用来执行所描述测试的设备必须确实被建立起来的要求。我们不确定市面上的的成品设备是否足以完成所有测试，但我们认为它应该能够胜任。

<h4>3. 需要完成的测试</h4>

本文档描述了许多需要被完成的测试。并非所有的测试都适用于所有型号的测试设备。供应商应该完成所有的能被一种特定设备支持的测试。在推荐条件下完成所有被建议的测试过程要花费相当长的一段时间。我们认为这些投入是值得的。附录A列出了一些我们认为有必要包含进来的某些个案的相关条件和测试。

<h4>4. 评估结果</h4>

完成所有推荐的测试将产生大量的数据。大部分数据并不适用于其他环境下对设备的评价。例如，一个路由器转发IPX帧格式的速率对挑选一个用于不支持这种协议环境的路由器没有什么作用。甚至评价与某个特定网络设置相关的数据也需要并不是很容易获取的经验。另外，选择需要完成的测试和对测试数据的评价也必须在一个大众接受的理解基础上完成，包括可重复性、差异性、以及少量试验的统计意义。

<h4>5. 要求</h4>

略...

<h4>6. 测试设置</h4>

实现这一系列测试理想的方式是使用一个同时带发送端口和接收端口的测试仪。测试仪的发送端口与被测设备的接收端口连接，测试设备的发送端口连接测试仪的接收端口（如图1）。测试仪同时发送并接收数据流，当数据流被测试仪转发但没有被测试设备转发时，测试仪能够很容易地判断出是否所有的已发送数据包都被接收，并验证正确的数据包被接收。

                             +------------+
                             |            |
                +------------|  tester    |<-------------+
                |            |            |              |
                |            +------------+              |
                |                                        |
                |            +------------+              |
                |            |            |              |
                +----------->|    DUT     |--------------+
                             |            |
                             +------------+
                                Figure 1

同样的功能也能通过分离的发送、接收设备获得（如图2），但除非它们被一些计算机远程控制并用一种方法将其模拟成单个设备，要求操作人员精确地完成一些测试（特别是吞吐量测试）可以被禁止。

          +--------+         +------------+          +----------+
          |        |         |            |          |          |
          | sender |-------->|    DUT     |--------->| receiver |
          |        |         |            |          |          |
          +--------+         +------------+          +----------+
                                Figure 2

6.1 多媒介测试设置

可以使用两种不同的设置来检测一个被用于连接现实网络和不同媒介类型网络的测试设备，例如，连接一个局部网到一个骨干FDDI线路。测试仪必须能支持不同的媒介类型，在这种情形下，必须使用图1中的设置。

两个完全相同的测试设备被用于其他的测试设置（如图3），在很多情形下，这种设置能更精确地模拟现实。例如，使用一个WAN链路或高速骨干网将两个LAN连接起来的设置不如模拟一个处于局部网的客户机与处于FDDI骨干网的服务器互连情形的系统。

                             +-----------+
                             |           |
       +---------------------|  tester   |<---------------------+
       |                     |           |                      |
       |                     +-----------+                      |
       |                                                        |
       |        +----------+               +----------+         |
       |        |          |               |          |         |
       +------->|  DUT 1   |-------------->|   DUT 2  |---------+
                |          |               |          |
                +----------+               +----------+
   
                               Figure 3

<h4>7. 测试设备（DUT）设置</h4>

在测试前，被测试的设备必须按照提供给用户的指示进行配置。特别地，最好能在设置过程中配置并使能所有支持的协议（见附录A）。除了特定的测试之外，所有的测试最好都在不修改被测试设备的配置的情况下进行。例如，在帧处理频率测试中改变帧处理缓冲区的大小或者在测试某协议吞吐量时关闭除传输层协议外的所有协议，这些都是不允许的。在开始一项测试时修改配置以决定过滤器对吞吐量的影响是有必要的，但这种改变必须能开启某种特定的过滤器。被测设备的设置应该包含正常推荐的路由更新间隔时间和保持有效频率。软件的特殊版本与测试设备的正确配置，包括在测试期间哪些功能已被关闭，哪些功能正在被使用都必须作为结果报告的一部分被包含进来。

<h4>8. 帧格式</h4>

以太网协议之上TCP/IP协议测试帧格式附录C：测试帧格式。这些特定的帧格式应该会在本文件中描述的专为某种协议媒介组合设计的测试过程中被使用，然后这些帧将作为其他协议媒介组合的模版使用。被用来定义某种特别测试序列的测试帧特定格式必须被包含在结果报告中。

<h4>9. 帧大小</h4>

所有被描述的测试都应该使用多种不同大小的帧完成。特别的，帧大小应该包含测试协议、测试媒介允许的最大与最小合法限制值，之间应该有足够多的帧大小以使能获得测试设备的全方面性能数据。除非特别说明，至少五种帧大小应该在每种测试条件下全部使用。

理论上最小的UDP回复请求帧长会包含一个IP头（最小20字节），一个UDP头（8字节）和其他在媒介中需要的MAC层帧头。理论最大帧长由IP头部的length字段决定。几乎在所有情形中，实际的最大与最小帧长都受到媒介的限制。


理论上，理想的状态应该按照某种方式分配帧长以使之最终同时分布在某种理论帧频率上。下面的建议包含了这种理论，但指定了一些容易理解记忆的帧长。另外，许多同样的帧长都被指定用于各种媒介类型中以使能够形成简单的性能对比。

注意：文件中包含某些媒介上使用的不现实的小帧长只是为了帮助描述被测设备的单帧处理开销。

9.1 以太网中使用的帧长

    64, 128, 256, 512, 1024, 1280, 1518

这些帧长包含以太网标准允许的最大与最小帧长和处于这两者之间为了达到更小的帧长和更高的帧频的一些选择值。

9.2 4Mb/6Mb口令环中使用的帧长

    54, 64, 128, 256, 1024, 1518, 2048, 4472

口令环推荐的帧长假设在路由协议帧中不存在RIF字段。RIF字段将会在所有源路由桥接性能测试试验中出现。口令环上最小的UDP帧长是54字节，由于很多口令环接口的限制，16Mb口令环中推荐使用的最大帧长是4472字节而不是理论上帧长17.9Kb。余下的帧长值被用于与其他媒介类型形成对比。另外，一个IP帧（非UDP）也可能被使用，如果想要得到更高的数据率，在这种情况下，最小帧长将是46字节。

9.3 FDDI中使用的帧长

    54, 64, 128, 256, 1024, 1518, 2048, 4472

FDDI中使用的UDP最小帧长是53字节，使用54字节的最小帧长是为了与口令环性能情况形成字节对比。使用4472字节的最大帧长而不是4500字节也是由于上述原因。如果想要获得更高的数据率，可能要使用IP帧格式（而不是UDP），在这种情况下，最小帧长将为45字节。

9.4 不同的MTU中使用的帧长

当测试设备支持连接到不同的MTU时，连接MTU的帧长应该使用两者间较大者，直至达到测试协议的限制值。如果MTU不匹配时互联的测试设备不支持帧分片，则该帧长的转发率将被报告为0. 

例如，如果是FDDI连接到以太网，则IP包在通过连接了FDDI和以太网的网桥或路由器的转发测试时应该使用FDDI的帧长。如果网桥不支持IP分片，则那些对以太网来说太长的帧将会被报告为0.

<h4>10. 验证接收到的帧</h4>

测试设备应该丢弃所有接收到非测试的帧。例如，保持活动帧与路由更新帧就不应该被包含在接受的帧总量中。在任何情形下，测试设备都应该检查接收到的帧长，并验证其是否符合期望的长度。

更好的情形是，测试设备应该能包括发送帧的序列号码，并在接受帧中检查这些号码。如果这样做了，报告的结果应该包括丢弃的帧数量，接受的过期帧数量，重复帧数量以及在接受帧序列间的空白间隙数量。这些功能特性在某些测试中是需要的。

<h4>11. 修改器</h4>

知道测试设备在大量条件下的表现性能可能是有用的，其中的一些条件在下面列出。报告的结果应该包含测试设备能产生的尽可能多的这些条件。测试套件应该首先在不修改任何测试条件的情况下运行，然后分别在每种不同条件下重复。为了保存对比这些测试结果的能力，要求产生修改条件下的任何帧将会被包含在相同的数据流中作为正常的测试帧以代替其中的一个测试帧，这些没有被包含在测试设备在一个独立的网络端口上。

11.1 广播帧

在大多数路由器中，当指向硬件广播地址的帧被接受时，需要设计特殊的处理过程。在网桥中（或路由器桥接模式下），这些广播帧必须广播到大量的端口。测试帧序列应该使用1%的帧指向相应的硬件广播地址。发送到广播地址的帧应该是一种路由器不必处理的类型。这种测试的目的是检测数据流中的其他数据的转发率是否受到影响。使用的特定帧应该被包含在测试帧格式文件中。广播帧应该通过数据流被平均分配，例如，每第一百帧。

同样的测试应该也在类网桥测试设备上实施，但这种情形下，广播包会被处理并转发到所有的输出端。

1%的广播帧水平比许多实际网络中更高，这是可以理解的，就像在药物的毒性评估中，需要更高的水平来防止正常水平下可能失效的影响。由于设计的因素，有些测试设备不能产生如此低的可替换水平的帧。在这些情形下，广播帧的比例应该达到设备能提供的、在测试结果报告中描述的实际水平尽可能小的水平。

11.2 管理帧

现在大多通信网络使用如SNMP(译者注：[Simple Network Management Protocol][snmp])的管理协议。在许多环境中，将有大量的管理站在同一时间向相同测试设备发送请求。

测试帧序列应该附加使用一个管理询问请求作为第一帧在测试过程没一秒发送一次。询问结果必须组合到一个应答帧中。应答帧应该使用测试设备验证。一个特定询问请求帧例子在附录C中给出。

11.3 路由更新帧

动态路由协议更新过程会对一个路由器转发数据帧的能力产生重大的影响。测试帧序列应该使用一个路由更新信息帧作为第一帧在测试序列中发送。

路由更新帧应该以附录C中指定的测试过程中使用的特定路由协议相对应的频率发送。附录C给出了两个以太网TCP/IP协议路由更新帧的例子。这种路由帧被设计用于改变到许多在转发的测试数据中没有包含的网络的路由信息，第一帧将路由标签设置为A，第二个将状态改变到B，这些帧必须在发送序列中轮流替换。

测试过程应该能验证路由更新信息已被测试设备处理。

11.4 过滤器

过滤器被添加到路由器及网桥中，使之有选择的阻止帧的转发，帧在正常情况下将会被转发出去。这个过程通常被用来实现区域之间数据的安全控制。不同的产品实现过滤器的能力不同。

测试设备开始应该被设置为增加一种测试所需的过滤条件。过滤器应该允许测试数据流的转发。在路由器中，这种过滤器应该是如下形式中的一种：

- 转发输入协议地址到输出协议地址

在网桥中过滤器应该是如下形式中的一种：

- 转发目标硬件地址。

测试设备之后应该被配置实现总共25种过滤器。这些过滤器中的前24种应该是如下形式中的一种：

- 阻塞输入协议地址到输出协议地址。 

这24种输入、输出协议地址应该不是在测试数据流中出现的任何值。最后一个过滤器应该允许测试数据流的转发。这里所说的第一个、最后一个，我们是用于确保在第二种情形下，25中情形都必须能被检查，这种检查在数据帧将要符合允许转发帧的条件之前完成。当然，如果测试设备对过滤器重新排序，或者不使用对过滤器的线性扫描方式进行，这些过滤器决定了那些以某种序列排列以使所有过滤器都有输入的影响可能消失。

使用的过滤器配置命令行应该被包含在结果报告中。

11.4.1 过滤器地址

两组过滤器地址集合是需要的，一种是单过滤器情形，另一种是25个过滤器的情形。

单过滤器情形应该允许从IP地址为192.168.1.2到IP地址为192.168.65.2的所有数据流，并阻止所有其他的数据流。

25个过滤器的情形中应该按照如下的序列进行设置：

    deny aa.ba.1.1 to aa.ba.100.1
    deny aa.ba.2.2 to aa.ba.101.2
    deny aa.ba.3.3 to aa.ba.103.3
    ...
    deny aa.ba.12.12 to aa.ba.112.12
    allow aa.bc.1.2 to aa.bc.65.1
    deny aa.ba.13.13 to aa.ba.113.13
    deny aa.ba.14.14 to aa.ba.114.14
    ...
    deny aa.ba.24.24 to aa.ba.124.24
    deny all else

所有先前的过滤条件在该序列进入前从路由器中清除。测试过程中选择该序列以检查路由器是否能整理过滤条件，或者能否按数据进入的顺序接受他们。这两种过程都将导致比一些哈希编码更大的影响效果。

<h4>12. 协议地址</h4>

使用一种单一的逻辑数据流，使用一种源协议地址，使用一种目的协议地址，或者一种实际要求中的使用的像上面描述的某些过滤条件来完成测试将会更加简单。但现实世界中使用的网络并没有被限制在一种单一的数据流类型。测试集合应该首先使用单一协议源（或者对网桥测试过程中的单一硬件）和目的地址对来运行。测试过程然后应该使用随机目的地址重复。而在测试路由器过程中，地址应该被随机分配，并在256种网络范围中唯一分发，同时在网桥中的全部MAC范围内随机且唯一分发。IP测试中使用的特定的地址范围在附录C中给出。

<h4>13. 路由设置</h4>

所有路由信息都需要实现测试数据流的转发是不合理的，特别是在多地址情形时，需要进行手动设置。在每个测试的开始时，一个路由更新信息必须被发送到测试设备。路由更新必须包含测试过程中要求的所有的网络地址。所有的地址应该重新解析同一“下一跳”。正常情况下，这将是测试设备接收端的地址。这种路由更新将必须按一定的时间间隔重发，这些时间间隔是使用的路由协议要求的。一种格式及其相关的数据更新帧重复间隔在附录C中给出。

<h4>14. 双向通信</h4>

正常网络活动并不是只在一个方向上进行。为了检查测试设备的双向通信性能，测试序列应该按照提供的每个方向同样的数据率运行。数据通信的总量应该不能超过传输媒介的理论限制值。

<h4>15. 单流量路径</h4>

全部测试套件应该在所有修改器条件下运行，这些条件与测试设备上的单输入输出网络端口相关。如果测试设备的内部设计使用多重独特路径，例如，配置多重网络端口的多重接口卡，那么所有可能路径类型应该被分开测试。

<h4>16. 多端口</h4>

当前很多路由器和网桥产品在相同的模式下提供很多网络端口。完成这些测试时，端口中的一半被设计为输入端口，另一半则被设计为输出端口。这些端口应该被平均分配到测设备结构中。例如，如果一个测试设备有两个各有四端口的网卡，其中每个网卡卡上的两个端口被设计成输入，两个被设计为输出。某些特定的测试要使用被提供到每个输入端口的相同数据率完成。输入数据流中的地址应该被设置，帧会被重定向到序列中的每个输出端口，因此所有的输出端口都将得到从这个输入的平均流量分布。相同的配置可能被使用来完成一个双向多流量测试过程。在这种情形下，所有的端口都被认为同时是输入输出端口，每个数据流必须包含有地址指向所有其他端口的帧。
    
考虑如下的6端口测试设备：


             --------------
    ---------| in A  out X|--------
    ---------| in B  out Y|--------
    ---------| in C  out Z|--------
             --------------

每个输入数据流的地址应该是： 

    数据流向输入端口A：
    分组输出到X，分组输出到Y，分组输出到Z。
    数据流向输入端口B：
    分组输出到X，分组输出到Y，分组输出到Z。
    数据流向输入端口C：
    分组输出到X，分组输出到Y，分组输出到Z。

注意这些数据流按照相同的序列，以致3个分组包将在同一时间到达输出端X，然后3个数据包同时到达Y，然后3个数据包同时到达Z。这个过程将确保在现实世界中，测试设备将必须在同一时间处理地址标记到相同输出端的多分组包。

<h4>17. 多协议</h4>

本文档并不涉及测试一个混合协议环境的影响问题，除了建议如果需要这类测试，那么帧应该在所有测试协议间分发。这种分发过程可能近似模拟测试设备将要使用的网络条件。

<h4>18. 多帧长</h4>

本文档并不涉及测试一个混合帧长环境的影响问题，除了建议如果需要这类测试，那么所有帧都应该在所有测试中列出的协议长度间分发。这种分发过程可能近似模拟测试设备将要使用的网络条件。作者们也不知道这样一个测试的结果将会被展现出来，除了在一些非常特殊的仿真网络环境中直接比较多测试设备。

<h4>19. 在单测试设备的另一端的测试表现</h4>

在单测试设备的另一端的测试表现中，范例能够被描述成在测试设备上应用一些输入，同时监控输出情况。测试结果能够用于形成一套在这些测试条件下的测试设备的基本特性。

当测试的输入输出端是同步时，这种模型是有用的（例如，测试设备上的64字节IP，802.3帧输入；64字节IP，802.3帧输出），或者测试模型能够在不相似的输入输出端各部相同（例如，1518字节IP，802.3体系帧输入；576字节分片IP，X.25帧输出）。

通过扩展但测试设备测试模型，关于多测试设备或者混杂环境的合理标准可能被找到。在这种扩展中，单测试设备被一个互联的网络测试设备系统替代。这种测试方法将支持大量设备、媒介、服务、协议的标准。例如，一个LAN-to-WAN-to-LAN配置的测试过程就可能如下：


    (1) 802.3-> DUT 1 -> X.25 @ 64kbps -> DUT 2 -> 802.3

    或者一种混合的LAN配置方案可能为:

    (2) 802.3 -> DUT 1 -> FDDI -> DUT 2 -> FDDI -> DUT 3 -> 802.3

在示例1，2中，每个系统的端到端标准都能被试验性确定。其他行为可能通过中间设备的使用刻画出来。在示例2中，配置可能被用于给出测试设备2的FDDI到FDDI能力暗示。

因为多测试设备被当做一个单系统，这种方法仍然存在限制。例如，这种方法可能会给测试系统产生一个聚合测试标准。然而，仅仅是那种标准的话，可能不能反映出测试设备间的行为非对称性，以及其他属性（例如CSU，DSU，交换机）带来的时延等等。

而且，当对比不同系统的标准时必须要小心，以确保测试系统测试设备的特性、配置有合适的共同公共点来允许对比。

<h4>20. 最大帧频</h4>

当测试LAN连接时需要使用的最大帧频应该就是被列出的媒介上帧大小的理论最大频率值。

当测试WAN连接时需要使用的最大帧频应该比该速度连接上帧大小的理论最大频率值更大。WAN测试的更高的频率是为了补偿一些经销商使用各种各样形式的头部压缩的现实。

一份LAN连接的最大帧频列表被包括在附录B中。

<h4>21. 突发性流量</h4>

测量测试设备在稳定状态负载下的性能是方便的，但这是一种不现实的方法来衡量测试设备的功能，因为实际的网络流量正常情况下包含突发性帧流量。下面描述的有些测试需要在稳定状态负载和突发性帧流量状态下分别进行测试。一次突发性传输中的帧使用最小的合法内部帧间隔来被传输。

测试的目的是为了决定突发性流量间的最小的间隔，测试设备能不带任何帧损失的处理这些流量。在每个测试过程中，每次突发性流量中帧的数量保持平稳，内部突发间隔也可以各不相同。测试过程应该使用突发流量大小为16，64，256，1024帧运行。

<h4>22. 每次口令的帧</h4>

尽管配置一些口令环和FDDI接口来一次转发多于各口令接受到的一帧是可能的，大多数网络设备目前仅仅能够一次口令下转发一帧。当一次口令下仅仅转发一帧时这种测试应该首先被完成。

一些目前的高性能工作站服务器的确可以在FDDI上一次口令下转发多于一帧来最大化流量。因为这可能是未来工作站和服务器上的一种平常特性，与FDDI接口互连的设备应该分别使用每口令1，4，8，16帧来进行测试。报告的帧频应该是在整个通信测试期间的平均帧转发频率。

<h4>23. 测试描述</h4>

一个精确的测试包含多个实验过程。每个实验返回一份信息，例如在一个特定输入帧频上的丢失率。每个实验包含大量的阶段：

    a）如果测试设备是一个路由器，发送路由更新信息到输入端口并暂停两秒来确保路由信息已经被设置。
    b)发送“学习帧”到输出端口并等待2秒来确保学习到的信息已经被设置。网桥学习帧是测试帧使用的源地址与目标地址相同的帧。其他协议的学习帧被用于为测试设备的地址解析表做好准备。需要使用的学习帧格式在测试帧格式文件中给出。
    c）运行测试实验过程。
    d）等待2秒以使所有剩余的帧都能被接受。
    e）等待至少5秒以使测试设备重新达到稳定。

<h4>24. 实验持续时间</h4>

测试的目标是为了决定测试设备能够连续支持的频率。实际的测试实验过程必须是在这种目标与测试集标准持续时间之间的一种平衡。每个实验的测试部分持续时间应该最少达到60秒。涉及到一些二分查找形式的测试，例如流量测试来决定准确的结果可能要使用一个更短的实验时间来最小化搜寻过程，但最终的决定应该由全长度实验做出。

<h4>25. 地址解析</h4>

测试设备应该能够回答测试设备发出的地址解析请求，不管该协议过程是否需要这样的过程。

<h4>26. 标准测试</h4>

注意：“数据流类型”的概念是指以上使用一个稳定的内部帧间隔对于一个帧流的修改，例如，在测试设备设置中添加流量过滤。

26.1 吞吐量

目标：决定[RFC1242][rfc1242]中定义的测试设备吞吐量。

过程：以特定的频率向测试设备发送特定数量的帧，然后对测试设备传送的帧进行技术。如果提供的帧数量与接收到的帧计数相等，或者接收到的帧少于发送的，则减少提供的数据流频率并重新运行测试。

吞吐量是测试设备转发的测试帧计数量与测试仪器发出的测试帧数量相等的最高频率。

报告格式：吞吐量测试的结果应该以图表的形式报告。在这种方式中，X轴应该是帧大小，Y轴应该是帧频率。应该在图表中至少存在两条线。一条显示在各种帧大小下媒介的理论帧频，另一条则由测试结果绘出。附加的线条可能被用于显示每种测试的数据流类型的结果。紧挨着图表的文字应该指出协议、数据流格式以及测试过程中使用的媒介类型。

我们必须确保如果为了宣传目的一个单值是需要的时候，经销商将会选择相应媒介上的最小帧长度的频率。如果这样做的话，图表就必须通过帧/每秒的形式展示出来。如果经销商要求的话，频率也可能需要用比特（字节）每秒表示出。性能描述必须包含a)一个测量的最大帧频，b)使用的帧大小，c)这种帧大小媒介上的理论限制值，d)测试中使用的协议类型。即使某单个数据值被用作了宣传副本的一部分，结果的全部数据表格都必须被包含在产品数据清单中。

26.2 时延

目标：决定[RFC1242][rfc1242]文档中定义的时延。

过程：先测定在每个列出的帧大小下测试设备的吞吐量。同一个确定了的吞吐量频率下通过测试设备发送某一特定帧长的一串帧流量。这串流量应该必须至少达到120秒的周期。一个有被实现独立的标志类型的标志符号应该被包含在60秒后的一个帧中。这个帧被完全转发的时间被记录（时间戳A）。测试仪器中的接受逻辑必须认出帧序列中的标记信息，并记录标记的帧被接受的时间（时间戳为B）。

时延就是在RFC1242中定义的相关定义，时戳B减去时戳A的结果，也就是为存储和转发设备定义的时延，或者为比特转发设备定义的延迟时间。

测试过程必须要被重复至少20次，使用被记录的平均值作为报告的值。

这个测试应该使用地址指向与剩余的数据流中相同目的地址的测试帧完成，每个测试帧的地址也指向一个新的目标网络。

报告格式：报告必须表明从RFC1242中定义的哪种延迟时间在这个测试过程中被使用。时延结果应该以一个表格的形式报告出来，这个表格的一行代表每个测试的帧长。也应有列表示出帧长，以及该帧长下测试适用的等待时间，测试的媒介类型，测试的每个数据流类型的合成等待时间值。

26.3 帧损失率

目标：决定[RFC1242][rfc1242]文档中定义的帧损失率，这个过程使用一个测试设备的范围包括输入数据率和帧长的整个范围。

过程：以特定频率通过被测试的测试设备发送特定数量的帧，并记录被测试设备发送的帧总量。在每个点的帧损失率使用下面的公式计算：

    ( ( input_count - output_count ) * 100 ) / input_count

第一次实验应该使用与100%的最大频率一致的帧频运行，最大帧频是在输入媒介上相应帧长的对应的。用于90%的使用的最大频率一致的频率重复这个过程，然后该频率下的80%。这个序列应该按每次减少10%间隔继续进行，直到出现两个没有帧丢失的连续时间间隔。实验的最大间隔尺寸必须为最大频率的10%，最好能找到一个更合适的间隔尺寸。

报告格式：帧丢失率测试结果应该用图形标出。如果这样做，那么X轴必须是输入帧频作为在特定帧长上媒介理论频率的百分比。Y轴必须是在特定输入频率下的丢失百分比。X轴的左端点和Y轴的底部必须是零，X轴的右端点和Y轴的顶部必须是100%。图形上的多线条可能用于报告不同帧长、协议、数据流类型的帧丢失率。

注意：查看需要使用的第18部分中的最大帧频描述。

26.4 背靠背帧

目标：用于描述测试设备处理背靠背帧能力，定义在[RFC1242][rfc1242]中给出。

过程：使用最小的内部帧间隔发送一串帧序列到测试设备，记录被测试设备转发的帧数量。如果记录的发送帧数量与转发的帧数量相同，帧流量串的长度被增加，重新运行测试。如果转发的帧数量少于发送的帧数量，帧流量串的长度被减少，重新运行测试。

背靠背值是测试设备可以不带任何帧损失处理的最长的帧流量串中帧的数量值。实验长度必须至少2秒，应该使用被报告的记录值的平均值被重复至少50次。

报告格式：背靠背结果应该使用表格的形式报告出来，表格中的行表示每个测试的帧长，列表示帧长与每种测试的数据流类型的合成平均帧计数值。每次测试的标准方差也要被报告出来。

26.5 系统恢复

目标：描述测试设备从一次超负载条件下恢复的速度。 

过程：首先测试设备在每个列出的帧长下的吞吐量。

按记录的吞吐量频率或者媒介上更低的最大频率的110%的频率发送一串数据帧，持续至少60秒。在时间戳A时刻，减少帧频到上面频率的50%，记录最后的帧丢失时间（时间戳B）。系统恢复时间由时间戳B减去时间戳A的差值决定。测试应该按照被报告的记录平均值被重复进行许多次。

报告格式：系统恢复结果应该按照表格的形式报告，表格的行表示每个测试的帧长，列表示帧长，被测试的各个数据流类型的作为吞吐量频率的帧频，测量的每种被测试的数据流类型的恢复时间。

26.6 重置

目标：描述测试设备从一次硬件或者软件重置后的恢复速度。

过程：首先测试测试过程中所用媒介上的最小帧长下的吞吐量。

按照决定了的吞吐量频率为最小长度帧发送一系列连续的数据帧流。在测试设备中制造一次重置。监控输出端直到数据帧开始被转发，并记录开始数据流的最后帧时间（时间戳A）和被接受到的新数据流的第一帧时间（时间戳B）。一种电源中断重置测试可以按照如上过程完成，除了测试设备的电源应该每10秒被中断而不是产生一次重置。

这个测试只能使用地址指向直接连接到测试设备的网络的帧运行，以使直到一次路由更新信息被接受前都不需要延迟。

重置值可以通过时间戳B减去时间戳A获得。

硬件和软件重置以及电源中断都应该被测试。

报告格式：重置值应该使用每种重置类型对应的单一描述集报告。

<h4>27. 安全考虑</h4>

本文档不讨论安全问题

<h4>28. Editors' Addresses</h4>

略...

<h4>Appendix A: Testing Considerations</h4>

A.1 Scope Of This Appendix

This appendix discusses certain issues in the benchmarking methodology where experience or judgment may play a role in the tests selected to be run or in the approach to constructing the test with a particular DUT.  As such, this appendix MUST not be read as an amendment to the methodology described in the body of this document but as a guide to testing practice.

1. Typical testing practice has been to enable all protocols to be tested and conduct all testing with no further configuration of protocols, even though a given set of trials may exercise only one protocol at a time. This minimizes the opportunities to "tune" a DUT for a single protocol.

2. The least common denominator of the available filter functions should be used to ensure that there is a basis for comparison between vendors. Because of product differences, those conducting and evaluating tests must make a judgment about this issue.

3. Architectural considerations may need to be considered. For example, first perform the tests with the stream going between ports on the same interface card and the repeat the tests with the stream going into a port on one interface card and out of a port on a second interface card. There will almost always be a best case and worst case configuration for a given DUT architecture.

4. Testing done using traffic streams consisting of mixed protocols has not shown much difference between testing with individual protocols. That is, if protocol A testing and protocol B testing give two different performance results, mixed protocol testing appears to give a result which is the average of the two.

5. Wide Area Network (WAN) performance may be tested by setting up two identical devices connected by the appropriate short- haul versions of the WAN modems.  Performance is then measured between a LAN interface on one DUT to a LAN interface on the other DUT.

The maximum frame rate to be used for LAN-WAN-LAN configurations is a judgment that can be based on known characteristics of the overall system including compression effects, fragmentation, and gross link speeds. Practice suggests that the rate should be at least 110% of the slowest link speed. Substantive issues of testing compression itself are beyond the scope of this document.

<h4>Appendix B: Maximum frame rates reference</h4>

    (Provided by Roger Beeman, Cisco Systems)

    Size       Ethernet    16Mb Token Ring      FDDI
    (bytes)       (pps)           (pps)         (pps)
    
    64           14880          24691         152439
    128           8445          13793          85616
    256           4528           7326          45620
    512           2349           3780          23585
    768           1586           2547          15903
    1024          1197           1921          11996
    1280           961           1542           9630
    1518           812           1302           8138
    
    Ethernet size
    Preamble 64 bits
    Frame 8 x N bits
    Gap  96 bits
    
    16Mb Token Ring  size
    SD               8 bits
    AC               8 bits
    FC               8 bits
    DA              48 bits
    SA              48 bits
    RI              48 bits ( 06 30 00 12 00 30 )
    SNAP
    DSAP             8 bits
    SSAP             8 bits
    Control          8 bits
    Vendor          24 bits
    Type            16 bits
    Data      8x(N-18) bits
    FCS             32 bits
    ED               8 bits
    FS               8 bits
    
    Tokens or idles between packets are not included
    
    FDDI            size
    Preamble        64 bits
    SD               8 bits
    FC               8 bits
    DA              48 bits
    SA              48 bits
    SNAP
    DSAP             8 bits
    SSAP             8 bits
    Control          8 bits
    Vendor          24 bits
    Type            16 bits
    Data      8x(N-18) bits
    FCS             32 bits
    ED               4 bits
    FS              12 bits

<h4>Appendix C: Test Frame Formats</h4>

This appendix defines the frame formats that may be used with these tests. It also includes protocol specific parameters for TCP/IP over Ethernet to be used with the tests as an example.

C.1. Introduction

The general logic used in the selection of the parameters and the design of the frame formats is explained for each case within the TCP/IP section. The same logic has been used in the other sections. Comments are used in these sections only if there is a protocol specific feature to be explained. Parameters and frame formats for additional protocols can be defined by the reader by using the same logic.

C.2. TCP/IP Information

The following section deals with the TCP/IP protocol suite.

C.2.1 Frame Type.

An application level datagram echo request is used for the test data frame in the protocols that support such a function. A datagram protocol is used to minimize the chance that a router might expect a specific session initialization sequence, as might be the case for a reliable stream protocol. A specific defined protocol is used because some routers verify the protocol field and refuse to forward unknown protocols.

For TCP/IP a UDP Echo Request is used.

C.2.2 Protocol Addresses

Two sets of addresses must be defined: first the addresses assigned to the router ports, and second the address that are to be used in the frames themselves and in the routing updates.

The network addresses 192.18.0.0 through 198.19.255.255 are have been assigned to the BMWG by the IANA for this purpose. This assignment was made to minimize the chance of conflict in case a testing device were to be accidentally connected to part of the Internet. The specific use of the addresses is detailed below.

C.2.2.1 Router port protocol addresses

Half of the ports on a multi-port router are referred to as "input" ports and the other half as "output" ports even though some of the tests use all ports both as input and output. A contiguous series of IP Class C network addresses from 198.18.1.0 to 198.18.64.0 have been assigned for use on the "input" ports. A second series from 198.19.1.0 to 198.19.64.0 have been assigned for use on the "output" ports. In all cases the router port is node 1 on the appropriate network. For example, a two port DUT would have an IP address of 198.18.1.1 on one port and 198.19.1.1 on the other port.

Some of the tests described in the methodology memo make use of an SNMP management connection to the DUT. The management access address for the DUT is assumed to be the first of the "input" ports (198.18.1.1).

C.2.2.2 Frame addresses

Some of the described tests assume adjacent network routing (the reboot time test for example). The IP address used in the test frame is that of node 2 on the appropriate Class C network. (198.19.1.2 for example)

If the test involves non-adjacent network routing the phantom routers are located at node 10 of each of the appropriate Class C networks. A series of Class C network addresses from 198.18.65.0 to 198.18.254.0 has been assigned for use as the networks accessible through the phantom routers on the "input" side of DUT. The series of Class C networks from 198.19.65.0 to 198.19.254.0 have been assigned to be used as the networks visible through the phantom routers on the "output" side of the DUT.

C.2.3 Routing Update Frequency

The update interval for each routing protocol is may have to be determined by the specifications of the individual protocol. For IP RIP, Cisco IGRP and for OSPF a routing update frame or frames should precede each stream of test frames by 5 seconds. This frequency is sufficient for trial durations of up to 60 seconds. Routing updates must be mixed with the stream of test frames if longer trial periods are selected. The frequency of updates should be taken from the following table.

    IP-RIP  30 sec
    IGRP    90 sec
    OSPF    90 sec

C.2.4 Frame Formats - detailed discussion

C.2.4.1 Learning Frame

In most protocols a procedure is used to determine the mapping between the protocol node address and the MAC address. The Address Resolution Protocol (ARP) is used to perform this function in TCP/IP. No such procedure is required in XNS or IPX because the MAC address is used as the protocol node address.

In the ideal case the tester would be able to respond to ARP requests from the DUT. In cases where this is not possible an ARP request should be sent to the router's "output" port. This request should be seen as coming from the immediate destination of the test frame stream. (i.e. the phantom router (Figure 2) or the end node if adjacent network routing is being used.) It is assumed that the router will cache the MAC address of the requesting device.  The ARP request should be sent 5 seconds before the test frame stream starts in each trial. Trial lengths of longer than 50 seconds may require that the router be configured for an extended ARP timeout.

                  +--------+            +------------+
                  |        |            |  phantom   |------ P LAN
    A
        IN A------|   DUT  |------------|            |------ P LAN
    B
                  |        |   OUT A    |  router    |------ P LAN
    C
                  +--------+            +------------+

                              Figure 2

In the case where full routing is being used

C.2.4.2 Routing Update Frame

If the test does not involve adjacent net routing the tester must supply proper routing information using a routing update. A single routing update is used before each trial on each "destination" port(see section C.24). This update includes the network addresses that are reachable through a phantom router on the network attached to the port. For a full mesh test, one destination network address is present in the routing update for each of the "input" ports. The test stream on each "input" port consists of a repeating sequence of frames, one to each of the "output" ports.

C.2.4.3 Management Query Frame

The management overhead test uses SNMP to query a set of variables that should be present in all DUTs that support SNMP. The variables for a single interface only are read by an NMS at the appropriate intervals. The list of variables to retrieve follow:

    sysUpTime
    ifInOctets
    ifOutOctets
    ifInUcastPkts
    ifOutUcastPkts

C.2.4.4 Test Frames

The test frame is an UDP Echo Request with enough data to fill out the required frame size. The data should not be all bits off or all bits on since these patters can cause a "bit stuffing" process to be used to maintain clock synchronization on WAN links. This process will result in a longer frame than was intended.

C.2.4.5 Frame Formats - TCP/IP on Ethernet

Each of the frames below are described for the 1st pair of DUT ports, i.e. "input" port #1 and "output" port #1. Addresses must be changed if the frame is to be used for other ports.

C.2.6.1 Learning Frame

ARP Request on Ethernet

    -- DATAGRAM HEADER
    offset data (hex)            description
    00     FF FF FF FF FF FF     dest MAC address send to broadcast address
    06     xx xx xx xx xx xx     set to source MAC address
    12     08 06                 ARP type
    14     00 01                 hardware type Ethernet = 1
    16     08 00                 protocol type IP = 800
    18     06                    hardware address length 48 bits on Ethernet
    19     04                    protocol address length 4 octets for IP
    20     00 01                 opcode request = 1
    22     xx xx xx xx xx xx     source MAC address
    28     xx xx xx xx           source IP address
    32     FF FF FF FF FF FF     requesting DUT's MAC address
    38     xx xx xx xx           DUT's IP address

C.2.6.2 Routing Update Frame

    -- DATAGRAM HEADER
    offset data (hex)            description
    00     FF FF FF FF FF FF     dest MAC address is broadcast
    06     xx xx xx xx xx xx     source hardware address
    12     08 00                 type

    -- IP HEADER
    14     45                    IP version - 4, header length (4 byte units) - 5
    15     00                    service field
    16     00 EE                 total length
    18     00 00                 ID
    20     40 00                 flags (3 bits) 4 (do not fragment), fragment offset-0
    22     0A                    TTL
    23     11                    protocol - 17 (UDP)
    24     C4 8D                 header checksum
    26     xx xx xx xx           source IP address
    30     xx xx xx              destination IP address
    33     FF                    host part = FF for broadcast

    -- UDP HEADER
    34     02 08                 source port 208 = RIP
    36     02 08                 destination port 208 = RIP
    38     00 DA                 UDP message length
    40     00 00                 UDP checksum

    -- RIP packet
    42     02                    command = response
    43     01                    version = 1
    44     00 00                 0

    -- net 1
    46     00 02                 family = IP
    48     00 00                 0
    50     xx xx xx              net 1 IP address
    53     00                    net not node
    54     00 00 00 00           0
    58     00 00 00 00           0
    62     00 00 00 07           metric 7

    -- net 2
    66     00 02                 family = IP
    68     00 00                 0
    70     xx xx xx              net 2 IP address
    73     00                    net not node
    74     00 00 00 00           0
    78     00 00 00 00           0
    82     00 00 00 07           metric 7

    -- net 3
    86     00 02                 family = IP
    88     00 00                 0
    90     xx xx xx              net 3 IP address
    93     00                    net not node
    94     00 00 00 00           0
    98     00 00 00 00           0
    102    00 00 00 07           metric 7

    -- net 4
    106    00 02                 family = IP
    108    00 00                 0
    110    xx xx xx              net 4 IP address
    113    00                    net not node
    114    00 00 00 00           0
    118    00 00 00 00           0
    122    00 00 00 07           metric 7

    -- net 5
    126    00 02                 family = IP
    128    00 00                 0
    130    00                    net 5 IP address
    133    00                    net not node
    134    00 00 00 00           0
    138    00 00 00 00           0
    142    00 00 00 07           metric 7

    -- net 6
    146    00 02                 family = IP
    148    00 00                 0
    150    xx xx xx              net 6 IP address
    153    00                    net not node
    154    00 00 00 00           0
    158    00 00 00 00           0
    162    00 00 00 07           metric 7

C.2.4.6 Management Query Frame

To be defined.

C.2.6.4 Test Frames

UDP echo request on Ethernet

    -- DATAGRAM HEADER
    offset data (hex)            description
    00     xx xx xx xx xx xx     set to dest MAC address
    06     xx xx xx xx xx xx     set to source MAC address
    12     08 00                 type

    -- IP HEADER
    14     45                    IP version - 4 header length 5 4 byte units
    15     00                    TOS
    16     00 2E                 total length*
    18     00 00                 ID
    20     00 00                 flags (3 bits) - 0 fragment offset-0
    22     0A                    TTL
    23     11                    protocol - 17 (UDP)
    24     C4 8D                 header checksum*
    26     xx xx xx xx           set to source IP address**
    30     xx xx xx xx           set to destination IP address**

    -- UDP HEADER
    34     C0 20                 source port
    36     00 07                 destination port 07 = Echo
    38     00 1A                 UDP message length*
    40     00 00                 UDP checksum

    -- UDP DATA
    42     00 01 02 03 04 05 06 07    some data***
    50     08 09 0A 0B 0C 0D 0E 0F

    * - change for different length frames

    ** - change for different logical streams

    *** - fill remainder of frame with incrementing octets,
    repeated if required by frame length

    Values to be used in Total Length and UDP message length fields:

    frame size   total length  UDP message length
    64            00 2E          00 1A
    128           00 6E          00 5A
    256           00 EE          00 9A
    512           01 EE          01 9A
    768           02 EE          02 9A
    1024          03 EE          03 9A
    1280          04 EE          04 9A
    1518          05 DC          05 C8

[rfc1944]: https://www.ietf.org/rfc/rfc1944.txt
[rfc2544]: https://www.ietf.org/rfc/rfc2544.txt
[rfc1242]: https://www.ietf.org/rfc/rfc1242.txt
[appendixA]: http://zhjwpku.com/rfc/2016/12/30/RFC2544-benchmarking-methodology-for-network-interconnect-devices.html#appendix-a-testing-considerations
[appendixB]: http://zhjwpku.com/rfc/2016/12/30/RFC2544-benchmarking-methodology-for-network-interconnect-devices.html#appendix-b-maximum-frame-rates-reference
[appendixC]: http://zhjwpku.com/rfc/2016/12/30/RFC2544-benchmarking-methodology-for-network-interconnect-devices.html#appendix-c-test-frame-formats
[snmp]: https://en.wikipedia.org/wiki/Simple_Network_Management_Protocol
[9]: http://zhjwpku.com/rfc/2016/12/30/RFC2544-benchmarking-methodology-for-network-interconnect-devices.html#9-帧大小
[26]: http://zhjwpku.com/rfc/2016/12/30/RFC2544-benchmarking-methodology-for-network-interconnect-devices.html#26-标准测试
