# 引言

9 月 4 日，建立在 Pendle 上的 DeFi 协议 Penpie 遭到黑客攻击，被盗取约 2,700 万美元的加密资产，包括各种类型的质押以太坊、Ethena 的 sUSDE 和封装 USDC 稳定币。【1】

# 攻击分析

攻击者在需要在正式攻击之前做一些准备，在这个交易里首先调用了创建伪造的收益合约和市场，随后利用这些合约在 Pendle 上注册虚假的流动性池，铸造大量 PT（Principal Token，本金部分） 等代币。紧接着，将这些代币存入市场并获取 LPT（Liquidity Provider Token，流动性提供者代币），最终将 LPT 存入 Penpie 相应的池子以换取存款凭证 PRT（Pendle Reward Token，生态奖励代币）。【2】

![image](https://github.com/user-attachments/assets/1908be26-4b82-4448-8b23-4fe08632143f)

在攻击交易里，攻击者首先调用了闪电贷函数，这一步为之后的步骤提供了大量的 agETH 和 rswETH 代币，作为操作流动性的基础资金。【3】
![image](https://github.com/user-attachments/assets/f35be456-6a4a-4693-ad63-20baefe54442)

随后攻击者调用了 batchHarvestMarketRewards() 来获得相应的市场奖励，并调用 redeemRewards() 赎回奖励，其中 redeemRewards() 的操作里包含一步 claimRewards() 的操作。claimRewards() 操作之后，攻击者还调用了 addLiquiditySingleTokenKeepYt() 来完成添加流动性，这是为了获得更多的 LPT。

![image](https://github.com/user-attachments/assets/7b4627a2-1638-4b69-a0cb-8cab19b91847)

![image](https://github.com/user-attachments/assets/ee9b14ca-4247-44d0-9b27-ddafa97dd0e6)

由于攻击者创建的 0x5b6c_PENDLE-LPT 市场没有其他人参与，攻击者目前已经可以提取奖励。首先是调用了 multiclaim()函数提取先前操作获得的 LPT 代币，最后直接在 depositMarket() 中铸造的相应股份，并领取奖励以实现获利。

![image](https://github.com/user-attachments/assets/ee1dba5f-dcd1-45d7-aa6e-52bd524ee489)

# 结论

该攻击的根本原因在于 Penpie 项目代码缺乏对市场和池子注册过程的严格验证，同时未能对关键函数有效实现重入保护。攻击者利用了该漏洞，通过单一交易多次重入合约，反复获取市场代币，最终导致资金被盗。
截止 2024 年 9 月 6 日 4 时（UTC+0），Penpie 攻击者地址已将 1,000 ETH （价值约 244 万美元）转移到混币平台 Tornado Cash。

参考：
1. Penpie，https://x.com/Penpiexyz_io/status/1831058385330118831
2. Blocksec Flow, https://app.blocksec.com/explorer/tx/eth/0x7e7f9548f301d3dd863eac94e6190cb742ab6aa9d7730549ff743bf84cbd21d1
3. Blocksec Flow，https://app.blocksec.com/explorer/tx/eth/0x42b2ec27c732100dd9037c76da415e10329ea41598de453bb0c0c9ea7ce0d8e5




