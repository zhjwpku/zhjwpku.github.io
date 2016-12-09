---
layout: post
title: An Introdution to LDAP
date: 2016-11-20 20:00:00 +0800
categories: rfc
tags:
- ldap
- translation
---

[LDAP][ldap-wiki]（Lightweight Directory Access Protocol，轻量目录访问协议）是一个开放的，供应商中立的行业应用协议，用于通过Internet协议访问并维护分布式目录信息服务。目录服务对于内部网络应用及因特网应用有重要的作用，它允许在整个网络中共享关于用户、系统、网络、服务和应用的信息。本文对 Michael Donnelly 的 [An Introdution to LDAP][introdution] 一文进行了翻译，希望对大家有所帮助。

**2016-12-09 更新**

推荐一个Windows平台LDAP管理工具：[Softerra LDAP Administrator][ldaptool]。

下载链接：[Download][ldapdownload]

<h6>------ 下面是翻译正文 ------</h6>

如果你在计算机行业工作，现在听说过[LDAP][ldap-wiki]是一个大好机遇。想知道令人兴奋的是什么？想了解关于LDAP更多的底层技术？你来对地方了。这篇介绍——也是系列文章的第一篇，描述了如何在贵公司设计、实施并集成LDAP环境——会让你熟悉LDAP背后的概念，而忽略核心细节供以后讨论。本文探讨的主题包括：

- [What is LDAP, anyway?][anchor0]
- [When should you use LDAP to store your data?][anchor1]
- [The structure of an LDAP directory tree][anchor2]
- [Individual LDAP records][anchor3]
- [Customizing your directory's object classes][anchor4]
- [An example of an individual LDAP entry][anchor5]
- [LDAP replication][anchor6]
- [Security and access control][anchor7]

首先，LDAP的发展让人兴奋。一个在公司范围实施的LDAP可以使几乎在任何计算机平台上运行的任何应用程序都能从LDAP目录中获取信息，目录可用于存储各种类型的数据：电子邮件地址及邮件路由信息、人力资源数据、公共安全密钥、联系人列表等。通过让LDAP作为系统集成的焦点，当人们在公司内寻找信息时，你可以为其提供一站式的登录，即使数据的主要来源位于其它地方。

"但，请等一下"，你可能会说。你已经在使用Oracle，Sybase，Informix，或Microsoft SQL等数据库来存储大部分这样的数据了。LDAP有什么不同吗？What makes it better？请继续往下读。

<h4>什么是LDAP</h4>

轻量级目录访问协议（即LDAP）基于[X.500][x500]标准，但是显然更简单、更容易适应满足定制需求。与X.500不同，LDAP支持TCP/IP协议，这对于因特网访问很重要（译者注：X.500现在也支持TCP/IP网络栈）。LDAP的核心规范都定义在RFC中，一份完整的LDAP相关的RFCs可在[LDAPman RFC page][ldap_rfcs]中找到。

**在语句中使用"LDAP"**

在日常对话中，你会听到人们说：“我们应该把它存储在LDAP中吗？”，或“只需要从LDAP数据库获取数据”，或“我们如何从LDAP切换到关系型数据库？”。严格地说，LDAP不是数据库，而是用于访问存储在信息目录（也称LDAP目录）中的信息的协议。一个更精确的陈述可能更应该像这样：“使用LDAP，数据将（从/向）我们的信息目录中的正确位置（检索/存储）”。但我并不会在这一点上纠正任何人，重要的是你现在知道了。

**LDAP信息目录是数据库吗？**

正如来自Sybase、Oracle、Infomix或Microsoft的数据管理系统用于处理关系数据的查询和更新一样，LDAP服务器用于处理LDAP信息目录的查询和更新。换句话说，LDAP信息目录的确是一种数据库，但它不是关系型数据库。与为每分钟处理数据乃至数千更改而设计的数据库——如在电子商务中经常使用的在线事务处理（OLTP）系统——不同，LDAP目录针对读取性能进行了大量优化。

**LDAP目录的优点**

现在我们已经弄清楚了，LDAP目录的优点是什么？LDAP的流行是许多因素的结果。我会给你几个基本的原因，但你要知道，这只是故事的一部分。

也许LDAP最大的优点是，你的公司可以从几乎任何计算平台，以及任何支持LDAP的应用程序访问LDAP目录，还可以轻松将LDAP集成到公司的内部应用程序。

LDAP协议是跨平台和基于标准的，因此应用程序不必担心托管目录的服务器类型。事实上，LDAP作为互联网的标准有着更广泛的行业认同度，供应商们更愿意将LDAP集成到他们的产品中（译者注：这里应该指LDAP客户端），因为他们不必担心另一端是什么。你可以使用任何一个开源或商业LDAP目录服务器（甚至是具有LDAP接口的DBMS服务器），因为与任何真正的LDAP服务器交互涉及相同的协议、客户端连接包和查询命令。相比之下，寻求直接与DBMS集成的供应商通常必须调整其产品，以单独与每个数据服务器供应商合作。

与许多关系数据库不同，你不必为客户端连接软件或许可付费。

大多数LDAP服务器易于安装、维护和优化。

LDAP服务器可以通过push或pull的方式复制部分或全部数据，从而允许你将数推送到远程办公室，以提高安全性等。

复制技术是内置的，易于配置。相比之下，许多大型DBMS供应商为此功能收取额外费用，而且管理起来要困难得多。

LDAP允许你使用ACIs（统称为ACL，访问控制列表）根据你的特定需求安全的委派读取和修改权限。例如，你的设施组可能有权更改员工的位置，多维数据集（原文为cube）或办公室电话号码，但不允许修改任何其他字段的条目。ALC可以根据谁在请求数据，要求什么数据，数据存储的位置以及正在修改的记录的其它方面来控制访问。这都是通过LDAP目录直接完成的，因此你不必担心在用户应用程序级别进行安全检查。

LDAP对于存储你希望从多个位置读取但不频繁更新的信息特别有用。例如，你的公司可以在LDAP目录中非常有效地存储以下内容：

- 公司员工电话薄和组织图
- 外部客户联系信息
- 基础设施服务信息，包括NIS（译者注：Network Information Service，最初称为黄页-Yellow Page）映射、电子邮件别名等
- 分布式软件包的配置信息
- 公共证书和安全密钥

<h4>什么时候应该用LDAP存储数据</h4>

大多数LDAP服务器针对读取密集型操作进行了大量优化。因此，从LDAP目录读取数据时，对比从为OLTP优化的关系型数据库获取相同数据，通常可以看到一个数量级的差异。然而，由于这种优化，大多数LDAP目录不适合存储频繁更改的数据。例如，LDAP目录服务器非常适合存储你公司内部的电话号码簿，但千万别想将其用作大容量电子商务站点的数据库数据库后端。

如果以下每个问题的答案为`Yes`，则使用LDAP存储你的数据是一个好主意。

- 你希望你的数据可跨平台吗？
- 你需要从多个计算机或应用程序访问此数据吗？
- 你存储的记录平均每天更改的次数很少？
- 将这种类型的数据存储在平面数据库而不是关系数据库中是有意义的吗？也就是说，你是否可以有效地将给定项目的所有数据存储在单个记录中？

最后这个问题通常会让人们暂停，因为访问平面记录以获取关系型数据是很常见的。例如，公司雇员的记录可能包含该雇员经历的登录名。可以使用LDAP来存储这种信息。经验法则：如果你可以想象将数据存储在大型电子名片簿中，那就可以轻松地将其存储在LDAP目录中。

<h4>LDAP目录树结构</h4>

LDAP目录服务器按层次存储其数据。如果你看过DNS树或UNIX文件目录自上而下的表示，LDAP目录结构是类似的。

与DNS主机名一样，从单个条目中读取LDAP目录记录的可辨识名称（DN），通过树向后读取，直到最高级别。后边会详细讨论这点。

为什么药分解成层次结构？有很多原因。这里列出几个可能的情况：

- 你可能希望将所有美国客户客户联系信息推送到西雅图办公室（专门用于销售）的LDAP服务器，而不需要向那里推送公司的资产管理信息。
- 你可能希望根据目录结构向一组人员授予权限。在下面列出的示例中，公司的资产管理团队可能需要完全访问资产管理部分，但不能访问其他区域。
- 结合`复制`操作，你可以定制目录结构的布局，以最小化WAN带宽利用率。你在西雅图的销售办公室可能需要美国销售联系人的每分钟更新，但欧洲销售的信息可能只需要每小时的更新。

**Getting to the root of the matter: Your base DN and you**

LDAP目录树的顶层是基（base），称为"base DN"。

"base DN"通常采用此处列出的三种形式之一。假设我在一家名为`FooBar, Inc.`的美国电子商务公司工作，该公司在互联网上的域名为`foobar.com`。

**o="FooBar.Inc.", c=US**

（X.500格式的`base DN`）

在这个例子中，`o=FooBar, Inc.`指的是组织，在本上下文中应该被视为与公司名称同义。`c=US`表示公司总部在美国。在以前，这是指定base DN的首选方式，时过境迁，现今大多数公司都是（或计划是）在互联网上。对于互联网全球化，在base DN中使用国家代码可能使事情更混乱。随着时间的推移，X.500格式演变成下面列出的其它格式。

**o=foobar.com**

（源于公司互联网存在的`base DN`）

这种格式很简单，使用公司的互联网域名作为base。一旦你获得`o=`部分（代表organization），公司的每个人都应该知道其余的来自哪里。这可能是直到最近最为常见的格式。

**dc=foobar, dc=com**

（源于公司DNS域模块的`base DN`）

像之前的格式一样，这里使用DNS域名作为`base DN`的基础。但是与其他格式的域名保持不变（人类可读），这种格式将域名拆分为组件：`foobar.com` 变为了`dc=foobar, dc=com`。虽然它有点不便于记忆，但它可能会稍微更通用一些。拿foobar.com进一步说明，当foobar.com与gizmo.com合并时，你只需考虑`dc=com`作为`base DN`，将新记录放到`dc=gizmo, dc=com`下现有的目录就可以了（当然，如果foobar.com与wocket.edu合并，这种方法没有帮助）。对于新的安装，我推荐使用这种格式。如果你打算Active Directory，微软已经决定这是你想要的格式。

**分支时间：如何在目录树中组织数据**

在UNIX文件系统中，顶层是根。在根目录下有很多文件和目录。如上所述，LDAP目录的设置方式大致相同。

在目录基础之下，你需要创建逻辑上分割数据的容器。由于历史（X.500）原因，大多数LDAP目录将这些逻辑分隔设置为OU条目。OU代表"Organization Unit"，在X.500中用于表示公司内的职能组织：销售、财务等。当前的LDAP实现保持了`ou=`的命名约定，但是通过诸如`ou=people`，`ou=groups`，`ou=devices`等宽泛的类别来分开。较低级别的OU优势用于进一步将类别进行分离。例如，LDAP目录树（不包括单个条目）可能如下所示：

{% highlight shell %}
dc=foobar, dc=com 
    ou=customers 
        ou=asia 
        ou=europe 
        ou=usa 
    ou=employees 
    ou=rooms 
    ou=groups 
    ou=assets-mgmt 
    ou=nisgroups 
    ou=recipes
{% endhighlight %}

<h4>单个LDAP记录</h4>

**LDAP条目中的DN有什么？**

存储在LDAP目录中的所有条目都有唯一的"Distingushed Name, DN"。每个LDAP条目的DN由两部分组成：Relative Distinguished Name (RDN)和记录所在的LDAP目录中的位置。

RDN是DN与目录结构无关的部分。存储在LDAP目录中的大多数条目将会有一个名称，并且该名称通常存储在cn(Common Name)属性中。由于几乎所有记录都有一个名字，你将存储在LDAP中的大多数对象将使用它们的cn值作为其RDN基础。如果存储我最喜欢的燕麦食谱，我将使用`cn=Oatmeal Deluxe`作为我条目的RND。

- 我的目录的`base DN`为`dc=foobar, dc=com`
- 我将所有食谱的LDAP记录记录在`ou=recipes`中
- LDAP记录的RDN是`cn=Oatmeal Deluxe`

鉴于这一点，这个燕麦食谱LDAP记录的完整DN是什么？记住，它向后读取，就像DNS中的主机名一样。

**cn=Oatmeal Deluxe,ou=recipes,dc=foobar,dc=com**

**人们总是比无生命物体更麻烦**

现在是时候处理公司员工的DN了。对于用户账号，你通常会看到基于cn或uid（User ID）的DN。例如，FooBar的员工Fran Smith（登录名：fsmith）的DN可能看起来像这两种格式：

**uid=fsmith,ou=employees,dc=foobar,dc=com** // (login-based)

LDAP（及X.500）使用uid表示“用户ID”，不要与UNIX uid号相混淆。大多数公司试图给每个人一个唯一的登录名，所以这种方式对于存储员工的信息是有意义的。你不必担心当你雇用下一个Fran Smith的时候你会做什么，如果Fran改变了她的名字（结婚、离婚、宗教信仰？），你不必改变LDAP条目的DN。

**cn=Fran Smith,ou=employees,dc=foobar,dc=com** // (name-based)

这里我们看到使用CN的条目。在人员的LDAP记录的情况下，请将CN视为其全名。可以很容易看出这种方法的缺点：如果名称更改，LDAP记录必须从一个DN`移动`到另一个。如上所述，您希望尽可能避免更改条目的DN。

<h4>自定义目录对象类</h4>

你可以使用LDAP来存储几乎任何类型对象的数据，只要改对象可以根据各种属性来描述。下面列举几个你可能会存储的信息：

- 员工：员工的全名，登录名，密码，员工号，经历的登录名，邮件服务器是什么？
- 资产跟踪：计算机名称，IP地址，资产标签，品牌和型号信息，物理位置是什么？
- 客户联系人列表：客户的公司名称是什么？主要联系人的电话，传真和电子邮件信息？
- 会议室信息：房间名称，位置，座位容量，电话号码是什么？有轮椅通道吗？有投影机吗？
- 食谱信息：给出菜的名称，成分列表，菜肴类型和准备说明。

由于你的LDAP目录可以自定义来存储任何类型的文本或二进制数据，想存什么你来决定。LDAP目录使用对象类的概念来定义对于任何给定类型的对象允许哪些属性。在几乎每个LDAP实现中，你都希望通过创建新的对象类或扩展现有的对象类来扩展LDAP目录的基本功能以满足特定需求。

LDAP目录将给定记录条目的所有信息存储为一系列属性对，每个属性对由属性类型和属性值组成。（这与关系数据库服务器以列和行的方式存储数据的方式完全不同。）考虑我的配方记录的这部分，存储在LDAP目录中：

{% highlight shell %}
dn: cn=Oatmeal Deluxe, ou=recipes, dc=foobar, dc=com 
cn: Instant Oatmeal Deluxe 
recipeCuisine: breakfast 
recipeIngredient: 1 packet instant oatmeal 
recipeIngredient: 1 cup water 
recipeIngredient: 1 pinch salt 
recipeIngredient: 1 tsp brown sugar 
recipeIngredient: 1/4 apple, any type
{% endhighlight %}

注意，在这种情况下，每个成分作为属性recipeIngredient的值列出。LDAP目录设计为以此方式存储单个类型的多个值，而不是将整个列表存储在具有某种分隔符的单个数据库字段中以区分各个值。

因为数以这种方式存储，所以数据库的形状可以任意变动（译者注：原文为completely fluid）——你不需要重新创建数据库表（及其所有索引）以开始跟踪数据。更重要的是，LDAP目录不使用内存或存储来处理“空”字段——实际上，包含未使用的可选字段不会产生任何消耗。

<h4>单个LDAP条目示例</h4>

来看个例子。我们将使用来自Foobar, Inc.的友好员工Fran Smith的LDAP记录。此条目的格式为[LDIF][ldif]，即导出和导入LDAP条目时使用的格式。

{% highlight shell %}
    dn: uid=fsmith, ou=employees, dc=foobar, dc=com
    objectclass: person
    objectclass: organizationalPerson
    objectclass: inetOrgPerson
    objectclass: foobarPerson
    uid: fsmith
    givenname: Fran
    sn: Smith
    cn: Fran Smith
    cn: Frances Smith
    telephonenumber: 510-555-1234
    roomnumber: 122G
    o: Foobar, Inc.
    mailRoutingAddress: fsmith@foobar.com
    mailhost: mail.foobar.com
    userpassword: {crypt}3x1231v76T89N
    uidnumber: 1234
    gidnumber: 1200
    homedirectory: /home/fsmith
    loginshell: /usr/local/bin/bash
{% endhighlight %}

首先，属性值以原封不动的方式存储，但对它们的搜索默认情况下不区分大小写。某些属性（如密码）在搜索时区分大小写。

让我们拆开这个条目，一块一块地看。

{% highlight shell %}
    dn: uid=fsmith, ou=employees, dc=foobar, dc=com
{% endhighlight %}
这是Fran的LDAP条目的完整DN，包括目录树中的整个路径。LDAP（和X.500）使用uid表示“用户ID”，不要与UNIX uid号码混淆。

{% highlight shell %}
    objectclass: person
    objectclass: organizationalPerson 
    objectclass: inetOrgPerson 
    objectclass: foobarPerson
{% endhighlight %}
可以分配与使用任何给定类型的对象一样多的对象类。person对象类要求cn和sn（surname）字段具有值。对象类person也允许其他可选字段，包括givename，telephonenumber等。对象类organizationalPerson向person中的值添加更多选项，inetOrgPerson向其中添加更多选项（包括电子邮件信息）。最后，foobarPerson是Foobar的自定义对象类，它添加了它们希望在它们公司追踪的所有自定义属性。

{% highlight shell %}
    uid: fsmith
    givenname: Fran
    sn: Smith
    cn: Fran Smith
    cn: Frances Smith
    telephonenumber: 510-555-1234
    roomnumber: 122G
    o: Foobar, Inc.
{% endhighlight %}
如前所述，uid代表"User ID"。无论任何时候看到"login"，使用它就对了。

注意CN有多个条目。像之前提到的，LDAP允许一些属性具有多个值，值的数量是任意的。什么时候会用到这个？假设你在公司的LDAP目录中搜索Fran的电话号码。虽然你可能知道她是Fran（多次在午餐时间她将它名字这样拼写），而人力资源的人可能会把她（更正式地）称为Frances。因为她的名字的两个版本都被存储，所以搜索将成功地查找Fran的电话号码，电子邮件，多为数据集等。

{% highlight shell %}
    mailRoutingAddress: fsmith@foobar.com 
    mailhost: mail.foobar.com
{% endhighlight %}
像互联网上的大多数公司一样，Foobar使用Sendmail进行内部邮件传递和路由。Foobar将所有用户的邮件路由信息存储在LDAP中，这是最新版本的Sendmail完全支持的。

{% highlight shell %}
    userpassword: {crypt}3x1231v76T89N
    uidnumber: 1234
    gidnumber: 1200
    gecos: Frances Smith
    homedirectory: /home/fsmith
    loginshell: /usr/local/bin/bash
{% endhighlight %}
注意，Foobar的系统管理员还在LDAP中存储所有的NIS密码映射信息。在Foobar，foobarPerson对象类添加了此功能。请注意，用户密码以UNIX crypt格式存储。UNIX uid在此存储为uidnumber。提示一下，有一个在LDAP中存储NIS信息的完整RFC，我将在此后的文章讨论NIS集成。

<h4>LDAP复制</h4>

LDAP服务器可以设置为以推或拉的方式复制部分或全部数据，使用简单身份验证或基于证书的身份验证。

例如，Foobar在ldap.foobar.com上运行着一个公开的LDAP服务器，端口为389。该服务器被Netscape Communicator的精确电子邮件特性，UNIX的"ph"命令，以及其他想要查询员工或客户联系方式的地方所使用。公司的主LDAP服务器在同一系统上运行，但在端口1389上。

你一定不希望一个员工搜索去资产管理或配方数据中去查询，也不希望看到IT账户（如root）显示在公司目录中。为了适应这些不愉快的现实，Foobar将所选择的目录子树从其主LDAP服务器复制到其“公共”服务器。复制排除了包含它们希望隐藏的数据的子树。为了始终保持当前状态，主目录服务器设置为执行基于推送的立即同步。请注意，此方法是为了方便而不是安全而设计的：这个想法是允许高级用户只要查询其他LDAP端口，如果他们想要搜索所有可用的数据。

假设Foobar通过LDAP在奥克兰和欧洲之间的低带宽连接上管理其客户联系信息。他们可能设置从ldap.foobar.com:1389到munich-ldap.foobar.com:389的复制，如下所示：

{% highlight shell %}
periodic pull: ou=asia,ou=customers,o=sendmail.com
periodic pull: ou=us,ou=customers,o=sendmail.com
immediate push: ou=europe,ou=customers,o=sendmail.com
{% endhighlight %}
拉连接将保持每15分钟进行一次同步，这种情况下可能正好合适。而推连接将保证对欧洲联系信息所做的任何更改将立即推送到慕尼黑。

这种复制方案，用户连接到哪里可以访问他们的数据？慕尼黑的用户可以简单地连接到他们的本地服务器。如果他们正在更改数据，本地LDAP服务器会将这些更改引用到主LDAP服务器，然后将所有更改推回到本地LDAP服务器以保持同步。这对本地用户是非常有利的：他们的所有LDAP查询（大多数是读取）对其本地服务器，这将快得多。当需要对其信息进行更改时，最终用户不必担心重新配置其客户端软件，因为LDAP目录服务器处理它们的数据交换。

<h4>安全和访问控制</h4>

LDAP提供了复杂的访问控制实例或ACIs。因为访问可以在服务器端进行控制，所以比通过客户端软件保护数据的安全方法更安全。

使用LDAP ACIs，您可以执行以下操作：

- 授予用户更改家庭电话号码和家庭住址的权限，同时限制他们对其他数据类型（如职位或经理的登录名）进行只读访问。
- 授予“HR管理员”组中的任何人员修改以下字段的任何用户信息的能力：经理，职称，员工ID号，部门名称和部门号。对其他字段没有写权限。
- 拒绝任何尝试查询LDAP以获取用户密码的用户的读取权限，同时仍允许用户更改自己的密码
- 授予管理员对其直接下属的家庭电话号码的只读权限，同时拒绝任何其他人的此权限。
- 授予组“host-admins”中的任何人以创建，删除和编辑存储在LDAP中的主机全部信息的权限。
- 通过网页，允许“foobar-sales”的用户有选择地授予或拒绝自己对客户联系人数据库子集的读取访问。这又将允许这些个人将客户联系信息下载到其本地笔记本电脑或PDA。 （如果销售队伍的自动化工具是LDAP感知的，这将是最有用的。）
- 通过网页，允许任何群组所有者在自己拥有的群组中添加或删除任何条目。例如，这将允许销售经理授予或删除销售人员修改网页的访问权限。这将允许邮件别名的所有者添加和删除用户，而无需与IT联系。指定为“公开”的邮件列表可以允许用户向/从这些邮件别名添加或删除自己（但只有自己）。限制也可以基于IP地址或主机名。例如，只有当用户的IP地址以`192.168.200.*`开头，或者用户的反向DNS主机名映射到`*.foobar.com`时，字段才可读。

这将让你了解使用LDAP目录的访问控制可能实现的功能，但请注意，正确的实施需要的信息要比此处提供的多得多。我将在以后的文章中更详细地讨论访问控制。

---
That's it for now. I hope you've found this article useful. If you have comments or questions, send email to donnelly@ldapman.org.

28 april 2000
<h6>------ 翻译结束 ------</h6>

本文是介绍LDAP系列文章的第一篇，另外两篇可以在[http://www.ldapman.org/articles][article]中找到。

[ldap-wiki]: https://en.wikipedia.org/wiki/Lightweight_Directory_Access_Protocol
[introdution]: http://www.ldapman.org/articles/intro_to_ldap.html
[anchor0]: {{page-url}}#什么是ldap
[anchor1]: {{page-url}}#什么时候应该用ldap存储数据
[anchor2]: {{page-url}}#ldap目录树结构
[anchor3]: {{page-url}}#单个ldap记录
[anchor4]: {{page-url}}#自定义目录对象类
[anchor5]: {{page-url}}#单个ldap条目示例
[anchor6]: {{page-url}}#ldap复制
[anchor7]: {{page-url}}#安全和访问控制
[x500]: https://en.wikipedia.org/wiki/X.500
[ldap_rfcs]: http://www.ldapman.org/ldap_rfcs.html
[ldif]: https://en.wikipedia.org/wiki/LDAP_Data_Interchange_Format
[article]: http://www.ldapman.org/articles
[ldaptool]: http://www.ldapbrowser.com/
[ldapdownload]: https://pan.baidu.com/s/1qYIr4Qg
