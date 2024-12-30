# 背景

2024 年 6 月 10 日，据慢雾 MistEye 安全监控系统监测，EVM 链上提供数字资产借贷服务的平台 UwU Lend 遭攻击，损失约 1,930 万美元。慢雾安全团队对该事件展开分析并将结果分享如下：

![image](https://github.com/user-attachments/assets/38265094-d856-4912-b598-b9edb8f39f5b)

# 相关信息
攻击者地址： 
0x841ddf093f5188989fa1524e7b893de64b421f47
存在漏洞的合约地址： 
0x9bc6333081266e55d88942e277fc809b485698b9

攻击交易：
0xca1bbf3b320662c89232006f1ec6624b56242850f07e0f1dadbe4f69ba0d6ac3
0xca1bbf3b320662c89232006f1ec6624b56242850f07e0f1dadbe4f69ba0d6ac3
0xb3f067618ce54bc26a960b660cfc28f9ea0315e2e9a1a855ede1508eb4017376
0xb3f067618ce54bc26a960b660cfc28f9ea0315e2e9a1a855ede1508eb4017376
0x242a0fb4fde9de0dc2fd42e8db743cbc197ffa2bf6a036ba0bba303df296408
0x242a0fb4fde9de0dc2fd42e8db743cbc197ffa2bf6a036ba0bba303df296408b

# 攻击核心

本次攻击核心点在于攻击者可以通过在 CurveFinance 的池子中进行大额兑换直接操纵价格预言机，影响 sUSDE 代币的价格，并利用被操控后的价格套出池子中的其他资产。

# 交易分析

攻击流程主要包括以下几个步骤：

1. 闪电贷借入资产并砸低 USDE 的价格：攻击者首先通过闪电贷借入大量资产，并在可以影响 sUSDE 价格的 Curve 池子中将借来的部分 USDE 代币兑换成其他代币。
![image](https://github.com/user-attachments/assets/1e0a27a2-8420-482c-9d62-e8c903dc10df)

2. 大量创建借贷头寸：在当前 sUSDE 价格大跌的情况下，通过存入其他的底层代币来大量借出 sUSDE 代币。
![image](https://github.com/user-attachments/assets/1d17951c-2e91-4ca8-9ab8-5a0d590fd8d9)

3. 再次操纵预言机拉高 sUSDE 的价格：通过在之前的 Curve 池子中进行反向兑换操作将 sUSDE 的价格迅速拉高。
![image](https://github.com/user-attachments/assets/3498f691-e9be-4377-8c69-d74a11168857)

4. 大量清算负债头寸：由于 sUSDE 的价格被迅速拉高，使得攻击者可以大量清算之前借款的头寸来获得 uWETH。![image](https://github.com/user-attachments/assets/9980738f-2707-488c-b072-b32031fd47f0)
5. 存入剩余的 sUSDE 并借出合约中的其他底层代币：攻击者再次存入当前处于高价的 sUSDE 来借出更多的底层资产代币获利。
![image](https://github.com/user-attachments/assets/c78d3130-32f7-4490-a46d-7d7828141eae)

不难看出，攻击者主要是通过反复操控 sUSDE 的价格，在低价时进行大量的借款，而在高价时进行清算和再抵押获利。我们跟进到计算 sUSDE 价格的预言机合约 sUSDePriceProviderBUniCatch 中：

![image](https://github.com/user-attachments/assets/1881c8f9-e4cf-41b0-95ac-6fad25915f34)

可以看出 sUSDE 的价格是先从 CurveFinance 上的 USDE 池子和 UNI V3 池子获取 11 个 USDE 代币的不同价格，再根据这些价格进行排序和计算中位数来确定的。
而在这里的计算逻辑中，其中 5 个 USDE 的价格是直接使用 get_p 函数获取 Curve 池子的即时现货价格，这才导致了攻击者可以在一笔交易内以大额兑换的方式直接影响价格中位数的计算结果。
![image](https://github.com/user-attachments/assets/55ec8456-52f1-4634-bcb7-20dc00b6bac6)

# MistTrack 分析

据慢雾 MistTrack 分析，黑客 0x841ddf093f5188989fa1524e7b893de64b421f47 在此次攻击中获利约 1,930 万美元，包括币种 ETH, crvUSD, bLUSD, USDC，随后 ERC-20 代币均被换为 ETH。
![image](https://github.com/user-attachments/assets/aea68268-3cf6-4a16-9a10-12ae7b1b7557)

通过对黑客地址手续费溯源，查询到该地址上的初始资金来自混币平台 Tornado Cash 转入的 0.98 ETH，随后该地址还接收到 5 笔来自 Tornado Cash 的资金。

![image](https://github.com/user-attachments/assets/622dbfd1-2199-486c-bbac-acbcc5bc45c7)

拓展交易图谱发现，黑客将 1,292.98 ETH 转移至地址 0x48d7c1dd4214b41eda3301bca434348f8d1c5eb6，目前该地址的余额为 1,282.9 ETH；黑客将剩下的 4,000 ETH 转移至地址 0x050c7e9c62bf991841827f37745ddadb563feb70，目前该地址的余额为 4,010 ETH。
![image](https://github.com/user-attachments/assets/084b6d7b-c0aa-4654-8066-66f3324b83fd)


MistTrack 已将相关黑客地址拉黑，并将持续关注被盗资金的转移动态。

# 总结
本次攻击的核心在于攻击者利用价格预言机的直接获取现货即时价格和中位数计算价格的兼容缺陷来操纵 sUSDE 的价格，从而在严重价差的影响下进行借贷和清算来获取非预期的利润。慢雾安全团队建议项目方增强价格预言机的抗操纵能力，设计更为安全的预言机喂价机制，避免类似事件再次发生。




