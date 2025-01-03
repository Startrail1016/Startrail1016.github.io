# 背景
2024 年 3 月 14 日，据 MistEye 安全监控系统预警，Optimism 链上的去中心化借贷协议 MOBOX 遭攻击，损失约 75 万美元。现对该攻击事件展开分析并将结果分享如下：
![image](https://github.com/user-attachments/assets/5511e810-1949-45a0-abf6-500ce4f08654)

# 相关信息
攻击者地址： 
0x4e2c6096985e0b2825d06c16f1c8cdc559c1d6f8
0x96f004c81d2c7b907f92c45922d38ab870a53945
被攻击的合约地址： 
0xae7b6514af26bcb2332fea53b8dd57bc13a7838e

攻击交易：
0x4ec3061724ca9f0b8d400866dd83b92647ad8c943a1c0ae9ae6c9bd1ef789417

# 攻击核心

本次攻击的核心点主要有两个，其一是攻击者利用合约中的` borrow() `函数漏洞，每次调用该函数都会触发对推荐人地址的奖励分配。由于奖励的计算是基于转移的代币数量确定的，所以攻击者可以通过将给推荐人的奖励再次转回被攻击合约来增加下一次借款的数量。其二是每次调用` borrow()` 函数会燃烧掉池子中的一部分 MO 代币，因此 MO 代币的价格被不断拉高，最终攻击者可以通过不断借款并叠加奖励获利。

# 交易分析

我们可以发现，整个攻击的流程主要是通过循环调用存在漏洞的` borrow()` 函数，随后立即调用 `redeem() `进行赎回，再将分配给推荐人的代币转移回攻击合约。
![image](https://github.com/user-attachments/assets/83245a27-f010-4ea4-b88a-f46be4aab0aa)
跟进 `borrow() `函数进行分析，可以发现每次调用该函数都会燃烧掉池子中的部分 MO 代币。
![image](https://github.com/user-attachments/assets/1671c841-b905-4894-b267-25c4d64dfbfa)

然而借出去的 USDT 的数量是根据池子中的 MO 代币的价格进行计算的，由于 MO 代币的价格因为燃烧而不断的拉高，最终导致攻击者用少量的 MO 代币就能借出大量的 USDT 代币。
![image](https://github.com/user-attachments/assets/64a4082f-ce27-4795-97d1-7119e889a369)

此外，每次借款都会给一个推荐人地址进行分红奖励，并且该函数是根据传入的 MO 代币的数量进行计算的。

![image](https://github.com/user-attachments/assets/91d1fa43-bb8d-4fde-997d-9e1b2aa50b92)

然而由于推荐人地址也是由攻击者(0x96f004c81d2c7b907f92c45922d38ab870a53945) 控制的，所以攻击者可以在完成借贷操作后，将这部分奖励转回来，以此叠加下一次借款的数量和分红奖励。
![image](https://github.com/user-attachments/assets/a1262caf-b986-4283-a55c-b878fadef59a)

经过以上循环操作后，攻击者拉高了 MO 代币的价格，最终可以用极少数量的 MO 代币就能借出合约中的大额的 USDT，并且直接在失衡的池子中将 USDT 全部兑换出来，获利离场。

![image](https://github.com/user-attachments/assets/986eb3cb-0966-466b-8e8a-da7b25034af5)


# 总结
本次攻击的核心在于攻击者利用` borrow() `函数会燃烧池子中部分代币的机制，不断借用资产，拉高池子中代币的价格并获取推荐人奖励，随后把代币转移回来并再次借用，以此不断叠加奖励和操控价格。慢雾安全团队建议项目方在类似功能函数中添加锁仓时间的限制，并且在设计借贷的价格模型时考虑多种因素，避免类似事件再次发生。


