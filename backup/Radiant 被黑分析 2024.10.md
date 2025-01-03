# 引言

10 月 17 日，Radiant Capital 跨链借贷协议遭遇网络攻击，损失超过 5,000 万美元。Radiant 是一个全链通用的资金市场，用户可以在任何主流区块链上存入任何资产，并借出资产。链上数据显示，黑客将从 Arbitrum 和 BNB Chain 中盗取的资产迅速转移，其中约 12,834 ETH 和 32,112 BNB 分别存入两个地址。

# 过程分析

此次攻击的核心是攻击者控制了多位签名者的私钥，从而接管了多个智能合约。接下来，我们将深入分析这起攻击的具体过程以及背后的技术手段。
1. 攻击者通过恶意合约（即 0x57ba8957ed2ff2e7ae38f4935451e81ce1eefbf5）调用了 `multicall()` 功能。`multicall() `允许一次调用中执行多个不同的操作。在这次调用中，攻击者将目标设定为 Radiant 系统中的 2 个组件，包括借贷池地址提供者（Radiant: Pool Address Provider）和借贷池（Radiant: Lending Pool）。

![image](https://github.com/user-attachments/assets/27aaa687-82ba-47dc-9243-6d0a9f2852ba)

2. 交易 1 中，攻击者操控了一个 Gnosis 多重签名钱包 (GnosisSafeProxy_e471_1416)。通过恶意调用，攻击者成功执行了一次 `execTransaction()`，其中包括使用 `transferOwnership` 修改借贷池地址提供者的合约地址。这样，攻击者便能够控制借贷池合约并进一步进行恶意操作。
3. 攻击者利用合约升级机制，通过调用 `setLendingPoolImpl` 函数，将 Radiant 的借贷池实现合约替换为其自己的恶意合约 `0xf0c0a1a19886791c2dd6af71307496b1e16aa232`。这个恶意合约包含了后门功能，允许攻击者进一步操控系统中的资金流向。

> 后门功能（Backdoor Function），是指恶意合约中的某种隐藏功能，通常被设计为看似正常，但实际上允许攻击者绕过正常的安全措施，直接获取或转移资产。

![image](https://github.com/user-attachments/assets/c8c7ab45-ca4e-492d-8a4f-9d3a3210bbd8)

4. 在借贷池实现合约被替换后，攻击者通过调用 `upgradeToAndCall` 函数，执行了该恶意合约中的后门逻辑，进一步从借贷市场中转移资产至攻击者控制的合约中，从而获利。

# 结语

总结下来，攻击者先是控制了多位签名者的私钥，接管了多个智能合约后利用合约的升级机制，将 Radiant 的借贷池合约替换为一个恶意版本，随后通过后门功能将市场中的资金转移至自己的控制下。
10 月 18 日，Radiant Capital 官方复盘称，攻击者通过恶意软件注入，入侵多个开发人员的硬件钱包。Safe Wallet 前端显示的交易数据看似合法，但实际在后台被恶意签名并执行。事件发生在例行的多重签名调整过程中，目前官方已加强多重签名控制，并与多家机构合作争取追回资金。
此次事件警示各项目方需强化硬件钱包的防护，改进多重签名的安全管理，同时在合约升级机制中加入更多的安全审查和限制措施。


参考：
1. AnciliaInc，https://x.com/AnciliaInc/status/1846605867753591002
2. Radiant Capital，https://x.com/RDNTCapital
3. Slowmist，https://x.com/SlowMist_Team/status/1846762596365750466
4. Transaction，https://bscscan.com/tx/0xd97b93f633aee356d992b49193e60a571b8c466bf46aaf072368f975dc11841c
5. Blocksec，https://app.blocksec.com/explorer/tx/bsc/0xd97b93f633aee356d992b49193e60a571b8c466bf46aaf072368f975dc11841c
