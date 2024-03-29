---
title: VPN
# tags:
#   - nodejs
date: '2023-11-04'
summary: VPN 协议
---

## OpenVPN
OpenVPN 是一个用于创建虚拟私人网络加密通道的软件包，是一种开源、跨平台、也是当今领先的 VPN 提供商最推荐的选择。它是较新的 VPN 协议之一，但它的灵活性和安全性使其成为最常用的协议之一。

OpenVPN 依赖于 OpenSSL 加密库和 SSL V3/TLS V1 协议等开源技术。OpenVPN 的开源性质意味着该技术由支持者社区维护、更新和检查。

当流量通过 OpenVPN 连接时，很难区分 HTTPS over SSL 连接。隐藏在众目睽睽之下的能力使其不易受到黑客攻击，并且更难被阻止。

此外，OpenVPN可以在任何端口上运行，同时使用 UDP 和 TCP 协议，因此绕过防火墙不会成为问题。但是，如果您正在寻找速度，那么使用 UDP 端口将是最有效的。

在安全方面，它有多种方法和协议，如 OpenSSL 和 HMAC 认证和共享密钥。为了进一步提高安全标准，它通常与 AES 加密相结合。其他 VPN 协议已经受到 NSA 和其他黑客攻击，但到目前为止，OpenVPN 仍然是可靠的选择。

OpenVPN支持的其他加密算法有：3DES, AES, Camellia, Blowfish, CAST-128
如果您主要关心安全性，建议使用 AES 加密，它的 128 位块大小还使其能够处理更大的文件，而不会降低性能。

OpenVPN优点
它协议可以绕过大多数防火墙,它是开源的并经过第三方审查,它具有非常高的安全性,它适用于多种加密方法,它可以根据您的喜好进行配置和定制,
它可以绕过防火墙
它支持多种加密算法
OpenVPN缺点
设置过程比较复杂
它需要依赖于第三方软件来运行
桌面端支持和功能强大，但在移动端设备上缺乏很好的支持
WireGuard
WireGuard 是一种比较新的开源 VPN 协议。由于它的开发目的就是在性能、易用性和省电等方面全面超越当下流行的 IKEv2/IPsec 和 OpenVPN，因此许多人将 WireGuard 称为 ”VPN 协议的未来“。而已有实际测试已经证明 WireGuard（使用 UDP）确实比 OpenVPN TCP 和 UDP 都快，而且 ping 值和延迟更低。并且，与其它具有复杂加密算法的通用 VPN 协议不同，WireGuard 只是重新组装了现成的算法，以实现更简单但仍然安全的加密目标。具体来说，其前沿的密码学用法包括Noise协议框架、ChaCha20、Curve25519等。

WireGuard最初只支持 Linux，现在已经变成了可在 Windows、macOS、iOS、Android 等多平台使用。 尽管如此，它仍在开发更新中，这意味着当下版本还是有一定程度的安全风险。

WireGuard优点
它简单轻巧
它快速且安全
它对 VPN 协议采用了极简主义的方法
它有可能成为未来的VPN
WireGuard缺点
目前主流VPN厂商的支持度不高
它不像其他 VPN 协议那样灵活

SoftEther优点
它支持多种桌面和移动操作系统
它是完全开源的
它可以绕过大多数防火墙
它速度很快，但不会影响安全性
SoftEther缺点
这是比较新的
它没有本机操作系统支持
许多现有的 VPN 还没有提供它
Lightway
Lightway 比较特别，因为它是唯一一个由行业领先的 VPN 提供商 ExpressVPN 建立的只属于自己的 VPN 协议，以实现更便捷、更轻松、更快、更安全且更可靠的 VPN 连接。它真的很“小巧”，只有大约 1,000 行代码（OpenVPN 有 70,000 行代码，尽管小巧的 WireGuard 也有 4,000 行），Lightway 利用 wolfSSL（一个嵌入式 SSL/TLS 库）来提供安全通信。无论网络发生任何变化、流失或故障，保护始终处于开启状态。根据官方声明，他们将在将来开放这个 VPN 协议的核心库，使其更加透明，并进行进一步的安全审计。

Lightway优点
安全快速
省电
Lightway缺点
暂时不支持iOS系统
如何选择VPN协议？
那么多VPN协议该如何选择，主要取决于需求及使用场景，如上文详细讲解的各个VPN协议的优缺点，但是，为了保证简单易用，使用 OpenVPN 通常是一个不错的选择，了解了协议之后，看下如何自己搭建VPN吧。

常见问题
最好的协议是什么？
其实没有最好的协议存在，只有最适合的协议，这个问题的答案取决于你的需求和你在互联网上做什么。

最快的协议是什么？
最快的协议是WireGuard，目前被认为是最快的VPN协议之一，为移动设备提供更快的连接及重新连接时间，并能很好的延长电池寿命。

最安全的协议是什么？
最安全的协议是OpenVPN，它被许多VPN专家建议，使用256位加密作为默认值。

最稳定的协议是什么？
最稳定的协议是IKEv2/IPSec是目前认为的最稳定的VPN协议，它提供了强大的连接，并允许用户不会因为安全问题自由的在网络之间切换。

最简单的协议是什么？
最简单的协议是PPTP协议，由于PPTP协议内置于许多设备中，被认为是最容易设置的协议之一，但是，由于它已经过时且以安全问题为名，因此我们不建议使用它。
