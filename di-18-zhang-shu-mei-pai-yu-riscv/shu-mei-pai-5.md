# 第 18.9 节 树莓派 5


经过测试，树莓派 5 8G，使用 [UEFI](https://github.com/worproject/rpi5-uefi)、FreeBSD 15.0（测试用例 `FreeBSD-15.0-CURRENT-arm64-aarch64-20240628-14fee5324a9b-270986-memstick.img.xz`）、支持从存储卡、USB 设备、m2 扩展板（微雪的 [PCIe_TO_M.2_HAT+](https://www.waveshare.net/wiki/PCIe_TO_M.2_HAT+)）的 m2 NVme SSD 启动，后者亦可兼容 PCIe 3.0 速度。但是网卡等没驱动（可使用 USB 网卡，具体型号参考第一章相关）。风扇由固件控制，所以默认会一直转，不会停。HDMI 正常。USB 2/3 均正常。


## 树莓派 5 与 HAT+

目前市面上出售的树莓派 5 扩展板，请确认其是否符合 HAT+ 标准。如果不符合，通常它只能通过 PCIe FPC 获取电源。

>**请注意，树莓派 5 所提供的并非标准的 PCIe 接口，而是自己设计的 FPC 接口，因此需使用转接器才能连接 PCIe 设备到树莓派 5。** 

根据树莓派 5 的排线规范，该接口最多只能提供 5V 1A，即最多 5W 的电力。树莓派的 HAT+ 规范要求应该从 GPIO 接口获取电源。

我们知道 M.2 只是一个物理接口，它本质上是一种迷你的 PCIe 接口，可以直接连接物理引脚而无需额外的芯片。

根据 M.2 规范和 PCIe 规范，M.2 接口最大支持 3.3V 3A，约为 10W。

因此，为了符合规范，需要额外提供 5V 1A 的电源。总计电力需求为 5V 2A，即 10W。

## 总线、接口与协议

我们需要首先区分三个概念：物理接口、总线协议和通道（或协议）。实际上，从本质上来看，这些概念可以简化为两个，因为总线本质上等同于总线协议（通道）和物理总线（例如 PCIe、USB、SATA 等物理接口或线）。


- 物理接口：指的是实际的物理连接和传输介质，例如 M.2、Type-C、SATA、以太网线等。
- 物理总线：即物理接口。
- 总线协议（通道）：指的是用于数据传输的规则和标准，例如 PCIe 协议、USB 协议、SATA 协议、TCP/IP 协议。
- 总线：总线 = 物理总线（物理介质） + 总线协议。**总线是指整个数据传输路径的概念，包括物理总线（即物理连接和传输介质）和总线协议（即数据传输的规则和标准）的组合。**


>例如，可以使用以太网线来传输 HDMI 数字信号。这展示了物理总线的可复用性。
>
>参考[绿联（UGREEN）HDMI延长器50米 HDMI转RJ45网传网口转换器 单网线网络高清1080P视频传输信号放大器 一对装 90811](https://item.jd.com/100053619301.html)。它使用以太网线来延长 HDMI 信号，其传输的物理介质是以太网线，接头是 RJ45。
>
>从设计上讲，以太网线通常使用以太网协议传输数据，物理介质是以太网线，接头是 RJ45。然而，该产品的存在表明，物理总线可以承载不同的信号类型。以太网线作为物理介质，既可以用于传输以太网协议下的 TCP/IP 数据，也可以通过不同的信号编码和转换器传输 HDMI 的数据。
>
>并且该产品在实际传输 HDMI 数据时，并不是使用以太网协议，而是通过 HDMI 协议下的信号编码传输。


物理接口在理论上决定了其承载的上限和电气性能，如能够承受的电压、电流和功率，IP 等级，以及尺寸规格如多少 X 多少。常见的物理接口包括 M.2、U.2、SATA、USB（包括 Type-C）、HDMI、DP 和 RJ45。

在大多数情况下，我们通常讨论的是物理接口，但严格来说，将物理接口和其承载的协议混为一谈是不准确的。因为任何物理接口在理论上都可以通过适配器相互转换。

>例如，Type-C 接口可以支持 USB2.0、USB3.0、DP、HDMI、Thunderbolt 和 USB PD 等多种协议。因此，M.2 接口在理论上既可以连接 NVMe 硬盘，也可以连接 SATA 硬盘，还可以连接显卡、声卡以及主板上的 PCIe 设备，而无需任何驱动，但可能需要额外的 12V 电源供应。

总线协议（我们将直接连接到 CPU 的通道称为总线，总线本质上也是通道）。现代 x86 计算机通常只有三种总线：USB、SATA、PCIe。

>例如，英特尔处理器通常只直接支持这三种总线。

SATA 协议或 SATA 硬盘之所以慢，是由于协议设计问题造成的，与 SATA 这个物理接口本身关系不大。并且 SATA 协会已经摆烂了，他们认为与其设计更多不兼容传统 SATA 设备的新协议，去改造 SATA 这个总线协议，还不如让用户去用使用 PCIe 硬盘或者直接把 SATA 硬盘接到 PCIe。SATA 目前也兼容了 PCIe 总线。

因为直接连接到 CPU 的通道我们称之为总线协议，因此其他通道则被称为普通协议或通道。对于现代处理器而言，只有 USB、SATA 和 PCIe 这三种总线直接连接到 CPU。而大多数其他设备，如网卡、显卡、声卡、摄像头通常连接到 PCIe 接口。

>需要注意的是，尽管这些设备连接到了 PCIe 总线，但并非所有设备都是直接连接到了 CPU（通常是通过主板上的 DMI 通道转接的）。

我们需要先来介绍一下 2010 以前的大部分 X86 计算机（即酷睿三代以前）的架构：

- CPU——>前端总线 FSB—>北桥—>PCI—>南桥—>PCI—>其他设备。

>以上 PCI 也可以替换为 PCIe。

但是在 PCIe 和新一代酷睿大规模使用后北桥在主板上就消失了。北桥负责高速设备（比如内存、集显），南桥负责低速设备（网卡、声卡、摄像头等）。南北桥直接通过 PCI 总线连接。

>在某种意义上，以前的 PS/2 鼠标、PS/2 键盘相当于直连 CPU（北桥）。
>
>但是现在仍然有直连 CPU 的 USB 总线，一样可以接入键盘和鼠标。所以，用是否直连 CPU 来论证 PS/2 接口的优越性是毫无道理可言的。

在现代 x86 CPU 中，已经在主板上看不见北桥了，但是北桥并未真正消失，他只是被集成到了 CPU 而已，并且由若干 CPU 核心负责处理这些，PCIe 总线也从这些核心上面引出。

我们所说的直连到 CPU 的 PCIe，本质上就是之前直连北桥的 PCI。现代 CPU 通过引脚连接到主板，CPU 和主板之间的 PCIe 被英特尔称为 DMI（Direct Media Interface，直接媒体接口），而 AMD 则称之为 PCIe。DMI 3.0x4 相当于 PCIe 3.0x4。

在主板上的设备要么直连到 CPU，要么通过主板上的 DMI 总线转接到 CPU。

由于北桥已经集成到了 CPU 中，看上去就有了两种方式连接 CPU：

- ① CPU -> PCIe（处理器上的 PCIe） -> 设备（任何设备），我们称之为直连 CPU 的 PCIe；

- ② CPU -> PCIe（DMI，芯片组/PCH 上的 PCIe） -> 主板 -> PCIe（或转换成其他接口，如 SATA、USB、M.2） -> 设备。

因此，理论上要获得最大速度，需要使用 ①，但可能会影响 CPU 和设备的稳定性。

同时需要注意，PCIe 并不是无限多的，这由 CPU 的规格决定，一般情况下，英特尔 CPU 可能只有 20 条 PCIe，例如 PCIe 3.0x20，其中的 3.0 表示 PCIe 的版本号，20 表示通道数。

>因此，DMI 的上限也不是无穷大的，如果在主板上安装了很多 PCIe x16 插槽，由于 DMI 通道的限制，这只是看上去很好，但实际上不能达到预期效果（如果插满，肯定不会符合预期）。

>**除了直连到 CPU 的设备外，所有其他设备都共享主板上的 DMI 总线。**

>其实从根本上说无论是否将北桥进行集成没有任何区别。在本质上都是相同的。这是由冯·诺依曼架构所决定的。但在性能和效率有所提升。

一般来说，任何设备要进行转换，不仅需要在物理上连接（将线连接在一起），还需要在软件层面（即驱动程序）进行适配。例如，所谓 NVMe 转 USB 的正确说法应该是 USB 3.1 Gen 2 到 PCIe Gen3 x2 桥接控制器。这种转换并不是将 NVMe 转换为 USB，而是将 PCIe 转换为 USB。

>并不是因为不需要安装驱动就意味着设备内部没有芯片，因为大多数这类驱动程序都是内置在操作系统中的。

事实上，总线、接口与协议这三者在严格意义上应该是分开的。因此，当我们谈论 M.2 转换为 PCIe 时，只是物理上将它们连接起来，不需要额外的芯片（因此也不需要驱动程序）。但是，如果要将 M.2 支持 SATA，就需要进行协议转换。

>例如，虽然 M.2 物理接口相同，但如果要连接 SATA 硬盘，就必须通过转换芯片进行连接。

>同样，严格来说并不是 SATA 转换为 M.2，而是 SATA 转换为 PCIe，即使用的是 PCIe Gen3 x1 转 2xSATA 桥接控制器。

因此，当我们提及 M.2 NVMe SSD 时，应该明确它是一种 PCIe 硬盘，NVMe 则是基于 PCIe 的应用层协议。就像英特尔连接主板与 CPU 的通道的 DMI 在本质上也是 PCIe 一样。

首先，任何 M.2 在物理上都完全等于 PCIe。无需任何芯片或电平转换。M.2 接口=mini PCIe，并且在历史上也的确如此。但是 M.2 没有 12V （U.2 接口支持，可以视为增强型的 M.2）。

标准的 PCIe 接口同时供电 3.3V、12V，最大支持 75W。而标准的 M.2 最高支持 3.3V 3A，即 10W。

总线就是协议的一种，无非总线是公用通道，大家都直接或间接兼容而已。真正存在的东西只有两种，物理的接口和软件的协议。与协议分层实际上无关。只要是硬件就可以随便转（硬件本质也是电压电流，电信号随便转）。

## 散热器

### 黄铜与紫铜

- 紫铜：我们把铜（Cu）含量不低于 99.7% 的金属视为纯铜。为了便于区分，我们一般将纯铜称为“紫铜”（纯铜会在空气中氧化成紫色，氧化前是红色）。
- 黄铜：黄铜是铜锌合金。铜（Cu）含量从 60% 到 95% 不等。

>**黄铜相比紫铜更容易加工且黄铜材料价格便宜。黄铜价格仅约为纯铜（紫铜）的三分之二。**

>**对于散热器来说，在任何情况下，为了实用，都不应该使用黄铜。** 你花了紫铜的钱得到的却只能是远不如铝的散热体验。

### 常见导热材料对比

>**首先要确认材质是纯单质金属（杂质低于某值，一般按等级划分）、还是合金（谁和谁合金）。他们的电气参数可能并不接近，甚至完全相反。例如黄铜与纯铜。**

一般来说：

- 密度 g/cm3

| 紫铜 |   黄铜   | 铝  |  金  |  银  | 石墨烯 |
| :--: | :------: | :-: | :--: | :--: | :----: |
| 8.9  | 最高 8.4 | 2.7 | 19.3 | 10.5 |  2.2   |

- 导热系数 W/(m·K)

| 紫铜 |  黄铜  | 铝  | 金  | 银  |  石墨烯  |
| :--: | :----: | :-: | :-: | :-: | :------: |
| 401  | 约 120 | 237 | 317 | 429 | 800-5300 |

- 比热容 J/（kg.k）20 ℃

| 紫铜 |   黄铜 |  铝 |  金 |  银 | 石墨烯 |
| :--: | -----: | --: | --: | --: | :----: |
| 418  | 约 120 | 902 | 126 | 233 | 约 0.7 |

- 电导率 %IACS （以纯铜为标准进行对比）

| 紫铜 | 黄铜  | 铝  | 金  | 银  |        石墨烯        |
| :--: | :---: | :-: | :-: | :-: | :------------------: |
| 100  | 约 30 | 61  | 72  | 104 | 可大于 100，最大 110 |

### 结论

#### 为什么不使用石墨烯做散热器？

综上，石墨烯只适合导热，不适合散热（比热容和密度都极低），例如适合取代导热硅胶或硅脂。但是由于石墨烯的电导率甚至比铜还大，无法绝缘，所以石墨烯无法直接接触电子元器件，会造成短路。

>也就是说，石墨烯只适用于不带电，又需要导热而且不能直接接触散热器的设备。但是我目前还没有发现这种电子设备（地暖？）。

#### 简单结论

**相同体积下，** 金的散热效果是最佳的，其次是银、紫铜。也就是说，紫铜是我们能得到的最经济实惠的散热材料。

但是对于服务器等大型设备，散热器的重量也很重要，**相同重量下，** 铝的散热效果超越于其他金属。

#### 如果只考虑体积

一般我们只需要考虑体积即可，也就是说：

- 不考虑散热器重量，但考虑价格紫铜是最优选。
- 不考虑重量及价格，黄金是最优选。（一般只有 CPU 等极少数元器件含金，但并非为了散热）
- 二者中间的是金属银。


>回收废旧电子元器件的一个重要目的就是炼金。

## 鼓风机与风扇

### 原理

- 在本质上只有电扇，鼓风机是电扇的一种，鼓风机也叫“涡轮风扇”，相对来说普通风扇叫“轴流风扇”。鼓风机和风扇都是依靠电动机驱动的空气流动装置，但它们的设计和应用有所不同。
- 鼓风机通常采用离心式设计，空气从侧面或底部进入，然后通过涡轮叶片加速，最后从侧面或顶部排出。普通风扇通常是轴流式，空气从前面吸入，直接从后面排出。
- 鼓风机适合需要高静压的场景，而普通风扇适合需要大流量的场景。

- 普通风扇（轴流风扇）通过直接吹动空气来均匀分布温度，适用于大面积散热。因此，在散热器上通常使用鼓风机，而在机箱内则使用普通风扇。

>根据不同的风道设计，功率足够的轴流风扇可以在密闭区域内实现温度的均匀分布；而鼓风机则更适合将热量从点 A 传输到点 B。

- 鼓风机（涡轮风扇）通过高压气流将热量从热源带走，更适合定向散热，如笔记本电脑和服务器等小空间高热量设备（大多数笔记本电脑通常采用鼓风机，将散热管传导的热量带出到外部环境）。
- 散热器上通常使用鼓风机是因为它们能够提供集中、定向的高压气流，快速带走热量。


>根据热力学第一定律，能量守恒定律，任何散热过程必须有热量转移到环境中。普通风扇和鼓风机通过空气对流带走热量，而制冷设备如空调则通过制冷剂循环将热量转移到更大的环境中。

>**论证：** 现假设空间足够小，风扇功率足够大，且设备温度不断提升，用风扇是无法再实现其散热功能的，因为空间的温度已经平均了。
>
>**换句话说，你在沙漠里用风扇是没有意义的。**
>
>
>尽管风扇可以带走身体表面的热量，但其效果有限。人是哺乳动物，故人体需要通过新陈代谢维持相对恒定的体温，风扇的作用是通过加速空气流动，帮助蒸发汗液和带走热量，但无法有效降低环境温度。
>
>对于高功率设备，主动制冷如空调或液冷系统是必需品，因为被动散热无法在高热密度环境中有效工作。

### 特性

- 鼓风机和电扇的标准型号是 3007、3010、4010 等，即 3007 的外形尺寸是 30mm\*30mm\*7mm，1234 的外形尺寸理论上应该是 12mm\*12mm\*34mm。

>最后是厚度。为什么？因为一般小电扇都是正方形的：他要转起来，也要节约空间，那么他最大的可能就是圆形或正方形。

- 鼓风机由于其设计特点能够产生更高的风压，因此在相同尺寸和转速下，鼓风机的噪音约为普通风扇的 2 倍。


  
### 树莓派 5 的风扇

- 树莓派 5 的风扇接头是 SH1.0-4P，并且支持 PWM（脉冲宽度调制，用于调速）和 FG（风扇转速反馈，用于转测检测）。和一般电脑风扇的要求基本相同，就是外形尺寸小了些，且线序不一样，一般 12 V 的电脑散热风扇为 PH2.0-4P。
- 树莓派 5 风扇针脚定义如下：
  - JST SH1.0-4P（1.0mm 针脚间距）
    - 引脚 1: 5V（红）
    - 引脚 2: PWM（蓝）
    - 引脚 3: GND（黑）
    - 引脚 4: FG（黄）

>**SH1.0-4P 是有防呆设计的，理论上反着是插不进去的，只能按设定的方向插入。**

将树莓派 5 平放在桌面上，从较窄的一侧看，风扇接头的线序从左到右依次为：黄（FG）、黑（GND）、蓝（PWM）、红（5V），即黄线在最左侧，最靠近边缘，红线在最右侧，最靠近网卡方向。如果颜色和线序不一致，可能是风扇制造商未按规范生产。
