---
layout: post
title: Bitcoin Mining
date: 2017-05-21 17:00:00 +0800
tags:
- bitcoin
---

[比特币](https://github.com/bitcoin/bitcoin)最近的涨势异常凶猛 ，记得上大学的时候有人拿 💻 就能挖到比特币，所以这周末抱着好奇的心理，决定尝试一把。本文意在猎奇，无借此发家的想法。

比特币挖矿分 Solo 和 Pool 两种。

Solo 需要自己安装 [Bitcoin Core](https://bitcoin.org/en/download)，启动之后会同步互联网上的区块链（block chain, 超过100GB的数据量）并暴露 8333 端口，然后将矿机指向这个地址，如此挖到的比特币全部记录在 Bitcoin Core 中。由于现在矿机数量巨大，算力不够很难挖到比特币，因此一般人很少使用 Solo 方式。

于是人们将矿机连接到一个矿池，贡献自己的算力，矿池的算力很强，一般都是 PH/s 的量级，因此挖到比特币的几率会更大，矿池会将挖到的比特币按用户贡献的算力进行比例划分并收取一定的手续费，即所谓的 Pool 挖矿模式。

全球矿池对比：[Comparison of mining pools](https://en.bitcoin.it/wiki/Comparison_of_mining_pools)

本文选择了 [AntPool](https://www.antpool.com/home.htm) 进行测试，注册账户并设置收款地址：

![antpool sub-account](/assets/201705/antpool-sub-account.png)

设置 worker:

![antpool worker](/assets/201705/antpool-worker.png)

使用 [cpuminer](https://github.com/pooler/cpuminer) 连接矿池并贡献算力：

```shell
[root@carol zhaojw]# ./minerd --algo=sha256d --url=stratum+tcp://stratum.antpool.com:443 -u zhjwpku.carol
```

![bitcoin sha256d](/assets/201705/bitcoin-sha256d.png)

本文使用的 CPU 虽然很强，但是每线程只能贡献 3MH/s 算力，因此总共的算力大概有 (3MH/s * 160 threads) = 480MH/s 的算力，挖了 18 个小时，获得的比特币为：

![antpool dashboard](/assets/201705/antpool-dashboard.png)

使用 [Mining Calculator](https://www.antpool.com/support.htm?m=calculator) 进行计算 500MH/s 算力可获取的比特币：

![bitcoin calculator](/assets/201705/bitcoin-calculator.png)

然而，根据 Antpool 的支付规则：

![antpool payout](/assets/201705/antpool-payout.png)

如果想要把在矿池的比特币支付到自己的 bitcoin Wallet，则需要机器满负荷运行 0.001 / 0.0000803 = 12.45 年才能收获 0.001 BTC。哇！好难！所以现在使用 CPU 甚至 GPU 来挖比特币或者莱特币（litecoin），都是在浪费时间。

因此，现在有些矿池已经不支持 CPU/GPU 挖矿软件，而只支持 ASCI（Application Specific Integrated Circuit）矿机：

![slushpool](/assets/201705/slushpool.png)

**关于LiteCoin**

[LiteCoin](https://github.com/litecoin-project/litecoin) fork 自 bitcoin，所以基本理念是一样的，使用的 Hash 算法为 scrypt，计算难度更大，本文使用的 CPU 每线程的算力只有不到 6KH/s，因此没有专用硬件就别去挖了。

![litecoinpool](/assets/201705/litecoinpool.png)

**总结**

通过两天的研究及测试，发现挖比特币必须使用专用硬件才可能获得收益，专用硬件的芯片由于只做一件事情——计算特定Hash算法——因而速度较之CPU/GPU的通用计算有巨大提升，类似加密使用的专用加密硬件或特殊的加密指令（如 Intel AES-NI 指令集）来获取更快的速度。这让我不禁联想到现在有些公司在研究的人工智能专用芯片，即把人工智能算法做到专有芯片里，类比这两天研究比特币的经历，深感人工智能专有芯片在未来必将大有作为。
