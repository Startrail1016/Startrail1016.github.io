# 漏洞背景

随着Web3.0的兴起，我们迎来了一场创新浪潮，但也面临着前所未有的安全挑战。区块链技术、智能合约、DeFi等新概念为我们带来了新的技术前景，但也引发了各种安全问题。在Web3世界中，大多数人会想到加密货币、元宇宙或NFT等概念，这些都是基于区块链和智能合约的技术。然而，大部分Web3应用仍然需要通过传统的Web2入口来访问。
本文通过分析Netlify Next.js库中的一个存储型XSS漏洞，探讨Web3中的Web安全风险。

# 什么是存储型 XSS

本文将介绍的 Netlify Next.js 的漏洞属于 XSS 中的存储型（Stored XSS），目前 Web 端漏洞在 web3 的世界里并不凸显，但如果涉及资金问题，可能会出现大量攻击，其中较为常见的便是与 XSS 相关的攻击。
存储型XSS（Stored XSS）是一种常见的Web漏洞，它发生在攻击者将恶意代码注入到服务器存储中（如数据库、文件系统等），当其他用户请求该页面时，恶意代码被执行。例如，某个应用程序的评论系统未对用户输入进行充分过滤，攻击者可以提交包含恶意JavaScript代码的评论，当其他用户查看该评论时，恶意代码便会执行。
例如，Google 图书曾经就有这么一个漏洞，当用户在搜索栏输入以下代码：

`"><img src=x onerror=alert(1)>`

搜索结果反馈为一本书，将其上传到 https: //play.google.com ，点击网页按钮后 XSS 立即被触发。
在这种情况下，攻击者成功地将包含恶意代码的数据提交到应用程序的数据库中。随后，当目标用户在应用程序中执行查询时，服务器从数据库中检索并返回包含恶意代码的数据内容，浏览器解析和加载这些数据时执行了恶意代码，从而发生XSS攻击。

# Netlify Next.js 库核心问题分析

Netlify Next.js库是专为支持 web3 应用构建而设计的，加密货币网站都广泛采用该库。尽管这些为 web3 业务服务的网站通常不存储敏感信息，某些也不包含传统的登录、注册或配置文件等交互式功能，可能让人误以为这些网站是安全的。然而，我们也应该意识到，一旦该库存在漏洞，可能带来的风险是巨大的。

![image](https://github.com/user-attachments/assets/d7c57ebb-f5c4-4f0b-bd7f-40f62a7b3b72)

Next.js 的一个独特之处在于其出色的图像优化功能，这一功能可以显著提升网站的性能和用户体验。通过公开的路径，该功能能够代理并尝试优化所有的图像。服务提供商 Netlify 在服务器端会对图像进行一系列的处理和修改，以确保用户能够以更快的速度加载图像，从而改善整体网站性能。

用户与用Netlify Next.js 库构建的网站交互的过程如下：

1. Next.js 返回渲染后的 DOM，其中所有 <img src=”...”/> 元素都通过“/_next/image”路由重定向

2. 用户通过以下 HTTP 请求来加载图像：

`GET /_next/image?url = /example.png&w=128&h=128&q=100`

3. 在服务器端进行优化处理后返回到用户

很明显，在用户提交的 URL 中，包含了参数 URL，但是未对其进行足够的验证和过滤，攻击者可以通过修改 URL 参数注入恶意代码或者请求非法的资源。而且，当系统加载资源时，它将遵循任何 HTTP 重定向，甚至重定向到外部站点，如果资源返回错误，还会跳过所有的检查直接返回完整的页面内容。

下面我们来分析交互过程的存在的问题 ：

**“_next/image”重定向安全风险**

该_next/image处理程序用于加载本地资源，会通过 URL 从用户处获一个参数，这个参数用于发送模拟 HTTP 请求。这个安全风险由Next.js 中的路径解析漏洞导致。用户可以在 URL 参数中传递未编码的反斜杠，随后该参数会由 Next.js 处理。这里存在明显的安全问题，Next.js 服务器有一个默认行为，如果用户尝试访问无法访问的目录（例如“\”），则用户会被重定向。通过传递“? URL =/\/\example.com/…”，他们能够触发此行为并将用户重定向到任意主机。

漏洞重现：

![image](https://github.com/user-attachments/assets/0d77da3a-3c09-468c-a244-2ef3b4233f21)

**ipx 模块存在 XSS 和 SSRF 漏洞**

由于依赖易受攻击的“unjs/ufo”库，Netlify 的 ipx 模块存在 XSS 和 SSRF 漏洞。Netlify 的 ipx 模块允许加载外部资源，但仅限于白名单主机。然而，由于解析 unjs/ufo 库的错误，攻击者可以通过传递白名单主机、编码反斜杠和恶意主机来欺骗 Netlify，使其遵循攻击者提供的 URL，即使它不在白名单中。由此，攻击者可以通过提供恶意的SVG文件实现XSS攻击。


**利用 x-forwarded-proto 漏洞的 XSS 和 SSRF 攻击**

通过对“x-forwarded-proto”标头的错误处理和滥用缓存机制，攻击者可以在“netlify-ipx”上实现通用的XSS和SSRF攻击。在路由“/_ipx/w_200”中，发现了IPX代码可以读取“x-forwarded-proto”标头，并允许其他协议的使用。攻击者只需提供该值，IPX代码将按原样获取并用于构建请求URL，从而实现攻击，如：

`${protocol}://${host}${id.startsWith('/') ? '' : '/'}${id}`

此处，攻击者可以通过简单地提供带有恶意 URL 的 x-forwarded-proto 标头，绕过白名单限制，从攻击者控制的 URL 加载资源。这导致与第二个问题相同的影响，但不需要有效负载中的白名单域。

通过以上内容，我们可以看到对于 Netlify Next.js 库的安全性问题的严重性。尽管这些 web3 网站通常不存储敏感信息，也没有传统的交互式功能，但存在的漏洞也可能会导致资金被盗等严重的问题。

# Web3中的Web漏洞风险和影响

在 Web3 世界中，数据的完整性对于用户来说至关重要，尤其是来自受信任站点的数据必须保证始终可用。这是因为大多数用户在浏览当前页面之外，很少深入区块链来去验证特定的地址或交易。因此，任何影响数据可用性或完整性的漏洞都可能对用户产生不利影响。

通过Netlify Next.js库中的存储型XSS漏洞案例，我们可以看到，即使在看似不存储敏感信息的Web3应用中，安全漏洞依然可能导致严重的资金损失和数据泄露。因此，项目方和开发者在开发中必须实施严格的输入验证机制，确保所有输入均经过严格的过滤和验证，并对应用的安全性进行全面审计，避免类似漏洞的产生和利用。

# 结语

从 Netlify Next.js 库上的通用 XSS 漏洞来看，核心在于服务器未能充分验证用户输入的 URL 参数，导致攻击者可以构造恶意的请求并利用漏洞执行 XSS 和 SSRF 攻击。慢雾安全团队建议项目方以及开发者在开发中实施严格的输入验证机制，尤其是在涉及用户数据和外部资源加载时，要确保所有输入均经过严格的过滤和验证，并对应用的安全性进行全面的审计，避免类似漏洞的产生和利用。

