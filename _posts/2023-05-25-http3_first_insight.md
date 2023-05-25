---
layout: post
title: "初探HTTP/3"
# subtitle: "This is the post subtitle."
comments: true
date:   2023-05-25 15:20:11 +0800
categories: 'cn'
---

## 从HTTP/1.1到HTTP/2

HTTP是Web的基础协议，从上个世纪90年代问世以来经历了数个版本的演化，其中使用时间最长也最广为人知的便是HTTP/1.1协议，它通过支持TCP连接重用、提供更为合理的缓存控制等创造性的解决了HTTP/1.0中存在的诸多问题，并在1997年以RFC 2068[^fn1]发布，在接下来的十多年间一直是最为主流的HTTP协议版本，但随着现代Web应用复杂性的提升，HTTP/1.1的性能越来越难以满足要求，其中最受诟病的莫过于HOLB（Head-of-Line Blocking，队头阻塞）。由于HTTP/1.1不支持多路复用，且Pipeline技术由于实现困难，一般浏览器默认采用串行发送请求，意味着后续请求需要等待前面的请求完成，虽然HTTP/1.1通过支持建立更多的TCP连接来缓解这一问题，但由于TCP连接数有限制，因此实际表现并不理想。

为此Google开发了SPDY协议，其采用多路复用技术并支持对Header进行压缩，很好的解决了HTTP/1.1的性能问题，从而成为HTTP/2协议的基础，2015年RFC 7540[^fn2]的发表标志着HTTP/2正式问世。HTTP/2通过Stream实现多路复用，不仅从应用层解决了HOLB问题，同时也避免了创建额外的TCP连接的开销，大大提高了网页加载速度，除此之外，HTTP/2表现出良好的兼容性及易升级性也帮助它迅速占领了主流站点，取得了极大的成功。

## 从 HTTP/2 到 HTTP/3
然而，HTTP/2也有一些致命缺陷，它底层所依赖的TCP协议存在众所周知的慢启动问题，以及TCP协议的重传机制也会导致发送窗口队头阻塞（从HTTP层转移到了TCP层）。此外，TCP拥塞控制算法通常采用“加性增、乘性减”的方式，当遇到丢包时，会减半发送窗口从而导致流量下降。因此，在网络状况不佳的情况下，HTTP/2的表现甚至不如HTTP/1.1[^fn5]。
为了解决TCP协议的固有问题，Google又开发了实验性的QUIC（Quick UDP Internet Connections，快速UDP互联网连接）协议[^fn6]以替换TCP协议，QUIC协议相比较于TCP协议有以下主要特性：

1. 基于UDP协议：TCP协议是面向连接的可靠传输协议，它通过序号、确认和重传来保障可靠性，而UDP是一种不可靠传输协议。QUIC使用前向纠错、重传和拥塞控制等技术来处理错误，利用Stream实现数据传输，每个Stream都是单独进行流控制，当出现丢包时只会影响该数据报内部相关的Stream而不会影响其他Stream的数据传输，从而避免了TCP协议的队头阻塞问题。
2. 极低的建立连接开销：传统的HTTPS协议栈由TCP+TLS组成，建立一次HTTPS连接需要进行多次握手，而QUIC协议内置对数据加密的支持，从而减少了初次建立连接的握手次数。

![](/assets/images/posts/2023-05-25/1.png "建立HTTPS连接的握手过程，来源于：
https://upload.wikimedia.org/wikipedia/commons/4/41/Tcp-vs-quic-handshake.svg")

3. 用户态协议：与工作在内核态的TCP协议不同，QUIC协议工作在用户态，从而将传输协议与操作系统解耦，避免了协议僵化问题，可以更快的进行技术迭代以更好的满足快速变化的用户需求。

QUIC协议经过多年的实践表现出良好的性能，随后被提议作为HTTP/2的传输层协议，即“HTTP over QUIC”，2018年被更名为HTTP/3并在2022年以RFC 9114[^fn3]发布成为新的标准。截止到2023年5月，有25.9%[^fn4]的网站正在使用HTTP/3。

## HTTP/3 性能初探
QUIC协议诞生已有十年之久，研究人员针对QUIC或HTTP/3的性能做了不少研究。最早的研究可能来自于R.Das[^fn14]在2014年对gQUIC（Google QUIC）的性能评估，他通过比较HTTP/1.1、SPDY和QUIC的PLT（Page Load Time，页面加载时间）发现QUIC协议在低带宽高延迟的场景下性能可以提升10-60%。

Konrad Wolsing[^fn7]及其研究团队认为在对比QUIC和TCP性能时忽略TCP协议本身存在的优化空间是不严谨的，而且过去的研究显示PLT（Page Load Time，页面加载时间）和用户可感知的速度没有太大关联，为此他们对TCP参数进行了针对性调优，并使用BBR拥塞控制算法以替换默认的CUIBIC算法，同时引入FVC（First Visual Change）、VC85（Visual Complete 85%）和SI（Speed Index）等指标比对QUIC和TCP性能差异，通过Mahimahi框架进行流量重放，结果显示经过调优后的TCP性能大幅提升，但总体而言QUIC因0-RTT（0 Round Trip Time，0往返时间）设计仍然占优。

Alexander Yu[^fn8]及其研究团队指出过去的QUIC协议性能研究都是在非生产环境进行，数据不具有说服力，因此他们使用Google、Facebook、Cloudflare公开的端点，分别对单对象资源和多对象网页进行测试，结果显示单对象资源在低丢包率或低延迟场景下当负荷较低（100KB）时HTTP/3表现均优于HTTP/2，而负荷大于1MB的情况下两者表现接近，但有趣的是在负荷大于5MB的情况下Cloudflare的HTTP/3端点表现远逊于HTTP/2，进一步研究显示该性能差距主要是由于Cloudflare的QUIC协议采用CUBIC拥塞控制算法而TCP协议采用BBR拥塞控制算法，多对象网页的性能测试结果显示QUIC协议针对HOLB的设计并没有带来明显的网页加载性能提升，考虑到QUIC协议本身的实现难度及不同的实现对性能的提升效果也有较大差别，因此他们认为可能在很多场景下HTTP/3并不能让人感到惊艳。

Martino Trevisan[^fn9]团队也对Google、Facebook、Cloudflare等早期支持HTTP/3的站点进行了性能测试，从网络延迟、丢包率、带宽等多个维度对比了HTTP/1.1、HTTP/2和HTTP/3的PLT和Speed Index指标，研究显示高延迟、高丢包率和低带宽场景下HTTP/3表现优于其他两者，例如对于增加200ms延迟的情况下，HTTP/3的PLT中位数是5.4s，而HTTP/2和HTTP/1.1分别是5.8s和6.4s。同时研究人员也注意到不同的服务提供商的HTTP/3表现也有差别，这可能跟底层协议的具体实现有关，例如在1Mb/s带宽场景下Facebook < Cloudflare < Google，而在增加200ms延迟场景下Cloudflare < Google < Facebook。

Saif D[^fn11]团队认为过去的研究往往仅关注底层QUIC协议（一般为gQUIC）与TCP协议的性能对比，并且指标一般都以PLT为主不够丰富，因此他们使用开源的LightHouse对HTTP/2+TLS1.3和HTTP/3（Cloudflare的QUICHE@98757ca）的QoE（Quality of Experience，用户体验质量）进行比较，具体性能指标权重分布见表1。研究结果显示，虽然总体吞吐量上HTTP/3比HTTP/2表现更好，但不管是在网络可靠的场景下还是在引入网络延迟和丢包之后，HTTP/3的 QoE总分均不如HTTP/2+TLS1.3，特别是LCP指标大幅落后，这也佐证了Cloudflare官方博客的性能数据：HTTP/3比HTTP/2要慢1-4%。

|          Item             |Percentage|    Description      |
|---------------------------|------|-------------------------|
|First Contentful Paint (FCP)| 15% | The time delta between first navigating to theweb page and the browser rendering the very first DOM content.|
|Time to Interactive (TTI)| 15% | 1. The FCP has completed <br> 2. Handlers are loaded for page elements <br> 3. The page responds to input within 50ms.|
|Speed Index(SI)| 15% | The time it takes for objects to be visibly displayed during page load.|
|Largest Contentful Paint (LCP)| 25% | The time it takes for the element on the page with the largest payload to have been completely rendered.|
|Total Blocking Time (TBT)| 25% | In the time between FCP and TTI, tasks taking longer than 50ms are summed into TBT. <br> Timing starts after 50ms of task execution.|
|Cumulative Layout Shift (CLS)| 5% | Quantifies the page’s stability as resources are loaded or DOMs are added. A higher score means more frequent layout shifts.|

A. Gupta[^fn12]及其研究团队同样使用了LightHouse分别对本地服务器、同区域（印度）服务器、全球（北美）服务器进行了性能测试，在吞吐量指标上HTTP/2全面占优，仅当使用无线连接且服务器在北美时QUICHE-H3表现亮眼，研究人员认为这证实了H3在网络不稳定的场景下有更好的性能。除了吞吐量之外，研究人员也对FCP、SI、LCP、TTI进行了对比，同样仅当使用无线连接且服务器在北美时H3性能略优于H2，其他均逊色于H2。不过，虽然整体性能测试结果显示H3不尽如人意，但研究人员认为H3仍在实验阶段，还可以不断优化，有很大的进步空间。

## 总结
本文简单介绍了HTTP发展历程和HTTP/3的几个主要特性，并研究了一些针对HTTP/3与HTTP/2性能评估的论文。尽管HTTP/3声称可以解决HOLB，且具有更低的连接建立开销，但许多研究表明，HTTP/3的性能优势并不一定明显，还依赖具体的实现和网络条件。一般来说，在高延迟、高丢包率和低带宽等不稳定的网络环境下，HTTP/3表现更好，但是不同厂商的QUIC协议具体实现不同，例如Cloudflare采用CUBIC拥塞控制算法而Google采用的是BBR拥塞控制算法，这也是导致性能差异明显的一大原因。

不过，本文未对不同使用场景如视频播放、物联网等进行研究，也没有涉及到移动网络。虽然HOLB在加载网页场景下并没有亮眼的表现，但对于视频播放也许有不错的提升。而不同代际的移动网络如3G，4G，5G其网络带宽和延迟差距较大，HTTP/3或许会有出色的表现，此外，HTTP/3的0 RTT机制在高速移动中的设备切换基站连接时应具有良好的用户体验。当然，由于个人水平有限，以上观点可能存在不准确之处，敬请谅解。

## 参考文献
[^fn1]: RFC 2068. <https://datatracker.ietf.org/doc/html/rfc2068>

[^fn2]: RFC 7540. <https://datatracker.ietf.org/doc/html/rfc7540>

[^fn3]: RFC 9114. <https://datatracker.ietf.org/doc/html/rfc9114>

[^fn4]: Usage statistics of HTTP/3 for websites. <https://w3techs.com/technologies/details/ce-http3>

[^fn5]: Saxce H D ,  Oprescu I ,  Chen Y . Is HTTP/2 really faster than HTTP/1.1?[C]// IEEE INFOCOM 2015 - IEEE Conference on Computer Communications Workshops (INFOCOM WKSHPS). IEEE, 2015.

[^fn6]: Langley A ,  Iyengar J ,  Bailey J , et al. The QUIC Transport Protocol: Design and Internet-Scale Deployment[C]// the Conference of the ACM Special Interest Group. ACM, 2017.

[^fn7]: Wolsing K , J Rüth,  Wehrle K , et al. A Performance Perspective on Web Optimized Protocol Stacks: TCP+TLS+HTTP/2 vs. QUIC[J].  2019.

[^fn8]: Yu A ,  Benson T A . Dissecting Performance of Production QUIC[C]// WWW '21: The Web Conference 2021. 2021.

[^fn9]: Trevisan M ,  Giordano D ,  Drago I , et al. Measuring HTTP/3: Adoption and Performance[C]// 2021.

[^fn10]: Saif D ,  Lung C H ,  Matrawy A . An Early Benchmark of Quality of Experience Between HTTP/2 and HTTP/3 using Lighthouse[C]// International Conference on Communications. IEEE, 2021.

[^fn11]: A. Gupta and R. Bartos, "User Experience Evaluation of HTTP/3 in Real-World Deployment Scenarios," 2022 25th Conference on Innovation in Clouds, Internet and Networks (ICIN), Paris, France, 2022, pp. 17-23, doi: 10.1109/ICIN53892.2022.9758130.

[^fn12]: Das S R . Evaluation of QUIC on web page performance[J]. massachusetts institute of technology, 2014.
