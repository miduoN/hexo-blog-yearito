---
title: Type A 卡存储结构与通信
copyright: true
reward: true
rating: true
related_posts: false
date: 2019-02-20 12:02:27
tags:
categories:
- 工作
---

> 本文写于2016年在武汉精伦电子实习期间。

[非接触式IC卡](http://baike.baidu.com/link?url=MPS2lJG6v5RTCtKu-x5RzsVzMetUTpAoRISY2mx71KCZZ1KmjgXH06cGDqBM5zjHQxKpZiaqldRZyXC-wtqUo_7)又称射频卡，由IC芯片、感应天线组成，封装在一个标准的PVC卡片内，芯片及天线无任何外露部分。是世界上最近几年发展起来的一项新技术，它成功的将射频识别技术和IC卡技术结合起来，结束了无源（卡中无电源）和免接触这一难题，是电子器件领域的一大突破。卡片在一定距离范围(通常为5—10cm)靠近读写器表面，通过无线电波的传递来完成数据的读写操作。

<!-- more -->

非接触式IC卡本身是无源卡，当射频读写器对卡进行读写操作时，通过13.56Mhz的射频载波向IC卡发送无线电波，读写器发出的信号由两部分叠加组成：一部分是电源信号，为内部芯片的工作提供能量。在无线电磁波激励下，卡片内的LC串联谐振电路（与发射端同频）产生共振，从而使电容内产生感应电荷，在电容另一端接有一个单向导通的电子泵，将谐振电容内的电荷送到另一个电容内存储，当所积累的电荷电压达到2V时，此电容可作为电源为提供工作电压。另一部分则是指令和数据信号，指挥芯片完成数据的读取、修改、储存等，并返回信号给读写器，完成一次读写操作。读写器则一般由单片机，专用智能模块和天线组成，并配有与PC的通讯接口，打印口，I/O口等，以便应用于不同的领域。

在射频IC卡的设计上，国际标准化组织(ISO)和国际电子技术委员会(IEC)确定了非接触式IC卡的国际标准——ISO/IEC14443。根据信号发送和接收方式的不同，ISO/IEC14443-3定义了TypeA卡（以飞利浦、西门子公司为代表）和TypeB卡（以摩托罗拉、意法半导体公司为代表）两种卡型，它们的不同主要在于载波的调制深度及二进制数的编码方式。

目前，最广泛使用的Mifare1卡即符合TypeA标准，而居民身份证则是TypeB标准的逻辑加密卡。本文重点介绍Type A标准下的M1卡与CPU卡的存储结构与读写操作。

# M1卡

M1卡，是指菲利浦下属子公司恩智浦出品的芯片卡的缩写，全称为NXP Mifare1系列，早在2008年，德国研究员亨里克·普洛茨（Henryk Plotz）和弗吉尼亚大学计算机科学在读博士卡尔斯滕·诺尔（Karsten Nohl）就成功地破解了NXP的M1卡的安全算法，虽然M1卡的加密算法存在安全漏洞，但由于其价格低廉，使用方便，现仍是最广泛使用的非接触式IC卡。常用的M1卡有S50及S70两种型号，S50与S70的唯一区别在于存储容量不同，S50内存大小为1K字节（8K bit），分为16个扇区，S70内存大小为4K字节（32K bit），分为64个扇区。精伦电子员工卡、武汉大学校园卡、好礼客会员卡均是S50卡，中国地质大学（武汉）校园卡是S70卡。后续篇幅主要基于S50展开。

## 存储器结构

M1-S50卡存储器划分为从0到15共16个扇区，每个扇区又划分为从0到3共三个段，每个段有16个字节。除了第一个扇区（0扇区）的第一段（0段）为厂商段之外，每个扇区的0段到2段为数据段，3段为区尾控制段。存储器映射如下图所示。

![存储器映射](http://qiniu.yearito.cn/storage-structure-typeA/image_1.png "存储器映射")

### 厂商段

每张M1卡都有一个全球唯一的UID号，这个UID号保存在卡的厂商段，其中前4个字节是卡的UID，第5个字节是卡UID的校验位，剩下的是厂商数据，并且这个段在出厂之前就会被设置写入保护，只能读取不能修改。当然也有例外，有种叫UID卡的特殊IC卡，该卡片完全兼容M1-S50卡，卡片的厂商段（UID所在段）是没有设置写入保护的，可以任意修改，重复修改，这种卡可以用于克隆复制M1卡。M1 卡厂商段结构如下图。

![厂商段结构](http://qiniu.yearito.cn/storage-structure-typeA/image_2.png "厂商段结构")

### 数据段

数据段的数据类型可以被区尾的控制位（Access Bits）配置为读/写段（用于譬如无线访问控制）或者值段（用于譬如电子钱包）。值段有固定的存储格式，只能在值段格式的写操作时产生，值段可以进行错误检测和纠正并备份管理，其有效命令包括读、写、加、减、恢复、传送，如下图所示。

![值段的存储格式](http://qiniu.yearito.cn/storage-structure-typeA/image_3.png "值段的存储格式")

Value表示一个带符号4字节值。这个值的最低一个字节保存在最低的地址中。取反的字节保存在字节4-字节7中。为了保证数据的正确性和保密性，值被保存了3次，两次不取反保存，一次取反保存。Adr表示一个字节的地址，当执行强大的备份管理时用于保存存储段的地址。地址字节保存了4次，取反和不取反各保存了2次。在执行加值、减值、恢复和传送等操作时，地址保持不变。它只能通过写命令改变。

### 控制段         

每个扇区都有一个区尾控制段，它包括密钥A和密钥B（可选），以及本扇区四个段的访问控制位(Access bits)。访问控制位也可用于指出数据段的类型（读/写或值段）。控制段的存储格式如下图所示。

![控制段的存储格式](http://qiniu.yearito.cn/storage-structure-typeA/image_4.png "控制段的存储格式")

如果不需要密钥B，那么区尾的最后6个字节可以作为数据字节。用户数据可以存储在区尾的第9个字节，这个字节具有和字节6、7、8一样的访问权限。

## 访问存储器

根据使用的密钥和相应区尾访问条件的不同，数据段所支持的存储器操作也不同。存储器的操作类型如下图所示。

![存储器操作类型](http://qiniu.yearito.cn/storage-structure-typeA/image_5.png "存储器操作类型")

每个数据段和区尾的访问条件由3个位来定义，它们以取反和不取反的形式保存在区尾指定字节中。访问位控制了使用密钥A和B操作存储器的权利。当知道相关的密钥和当前的访问控制条件时，可以修改访问条件。各区的访问位定义如下图所示。

![各区访问位定义](http://qiniu.yearito.cn/storage-structure-typeA/image_6.png "各区访问位定义")

访问位在区尾的存储形式如下图所示。

![访问位存储格式](http://qiniu.yearito.cn/storage-structure-typeA/image_7.png "访问位存储格式")

### 区尾的访问条件

根据区尾（段3）访问位的不同，访问条件可分为“从不”、“密钥A”、“密钥B”或“密钥A|B”（密钥A或密钥B）。区尾的访问条件如下图所示。

![区尾的访问条件](http://qiniu.yearito.cn/storage-structure-typeA/image_8.png "区尾的访问条件")


{% note %}
注：用灰色标明的行是密钥B可被读的访问条件，此时密钥B可以存放数据。

例如：当段3的访问条件C1<sub>3</sub>C2<sub>3</sub>C3<sub>3</sub>=100时，表示：密钥A不可读（隐藏），验证密钥B正确后，可写（或更改）；访问控制位在验证密钥A或密钥B正确后，可读不可写（写保护）；密钥B不可读，在验证密钥B正确后可写。

又如：当段3的访问条件C1<sub>3</sub>C2<sub>3</sub>C3<sub>3</sub>=110或者111时，除访问控制位需要在验证密钥A或密钥B正确后可读外，其他如访问控制位的改写，密钥A，密钥B的读写权限均被锁死而无法访问。
{% endnote %}


### 数据段的访问条件

根据 数据段（段 0-2 ）访问位的不同， 访问条件可分为“从不”、“密钥 A ”、“密钥 B ”或“密钥 A|B ”（密钥 A 或密钥 B ）。相关访问位的设置定义了该段的应用（或者说数据段类型）以及所支持的应用命令，不同的数据段类型可以进行不同的访问操作。 读 / 写段可以进行读操作和写操作。  值段可以进行加、减、传送和恢复的值操作。其中一种情况中（" 001"） 只能对不可再充电的卡进行读操作和减操作，另一种情况中（" 110"） 使用密钥 B 可以再充电。 厂商段无论设置任何的访问位都只是只读的。  数据段的访问条件如下图所示。

![数据段的访问条件](http://qiniu.yearito.cn/storage-structure-typeA/image_9.png "数据段的访问条件")

{% note %}
注：如果密钥B可以在相应的区尾被读出，它就不能用于确认（在前面所有表中的灰色行）。如果读卡器要用这些（带灰色标记的）访问条件的密钥B确认任何段，卡会在确认后拒绝任何存储器访问操作。
{% endnote %}

### Mifare S50初始化

Mifare S50出厂时，访问控制字节（字节6-字节9）被初始化为 `"FF 07 80 69"`，所以初始访问控制位默认值为：C1<sub>0</sub>C2<sub>0</sub>C3<sub>0</sub>=000；C1<sub>1</sub>C2<sub>1</sub>C3<sub>1</sub>=000；C1<sub>2</sub>C2<sub>2</sub>C3<sub>2</sub>=000；C1<sub>3</sub>C2<sub>3</sub>C3<sub>3</sub>=001。KEY A和KEY B的默认值为 `"FF FF FF FF FF FF"` 。控制段初始值和访问控制字节如下图所示。

![控制段初始值](http://qiniu.yearito.cn/storage-structure-typeA/image_10.png "控制段初始值")

![区尾访问条件](http://qiniu.yearito.cn/storage-structure-typeA/image_11.png "区尾访问条件")


# CPU卡

CPU卡的集成电路中带有微处理器CPU、存储单元（包括随机存储器RAM、程序存储器ROM、用户数据存储器EEPROM）以及芯片操作系统COS（Chip Operating System）。装有COS的CPU卡相当于一台微型计算机，不仅具有数据存储功能，同时具有命令处理和数据安全保护等功能。

由于经济全球化趋势发展迅猛，全球化的金融服务系统纷纷建立起来，为使CPU卡及其接口设备的制造能够统一起来，国际标准组织从1987年开始，相继制定和颁布了CPU卡的国际标准，以保证不同国家、不同行业都采用统一的CPU卡软硬件技术规范开发应用系统。有关CPU卡本身的标准有：

- ISO 10536：识别卡－非接触式的集成电路卡
- ISO 7816：识别卡－带触点的集成电路卡
- ISO 7816-1：规定卡的物理特性。卡的物理特性中描述了卡应达到的防护紫外线的能力、X光照射的剂量、卡和触点的机械强度、抗电磁干扰能力等等。
- ISO 7816-2：规定卡的尺寸和位置。
- ISO 7816-3：规定卡的电信号和传输协议 。传输协议包括两种：同步传输协议和异步传输协议。
- ISO 7816-4：规定卡的行业间交换用命令。包括：在卡与读写间传送的命令和应答信息内容；在卡中的文件、数据结构及访问方法；定义在卡中的文件和数据访问权限及安全结构。

## COS简介

COS的主要功能是控制CPU卡和外界的信息交换 ，管理CPU卡内的存储器 并在卡内部完成各种命令的处理。其中，与外界进行信息交换是COS最基本的要求。在交换过程中，COS所遵循的信息交换协议包括两类：异步字符传输的 T=0协议以及异步分传输的T=1协议。这两种信息交换协议的具体内容和实现机制在ISO/IEC 7816 - 3中作了规定；而COS所应完成的管理和控制的基中功能则是在ISO/IEC 7816 - 4标准中作出规定的。在该国际标准中，还对CPU卡的数据结构以及COS的基本命令集作出了较为详细的说明。 至于ISO/IEC7816 - 1和2，则是对CPU卡的物理参数、外形尺寸作了规定，它们与COS的关系不是很密切。

根据CPU卡的基本硬件环境可以设计出各种各样的COS，但是，所有的COS都必须能够解决三个基本的问题，即：文件操作、鉴别与核实、安全机制。事实上，鉴别与核实和安全机制都属于CPU卡的安全体系的范畴之中，所以，CPU卡的COS最重 要的两方面就是文件与安全。但具体的分析一下，可以把从读卡设备发出命令到卡给出响应的一个完整过程划分为四个阶段，也可以说是四个功能模块：传送管理、安全管理、命令管理和 文件管理 。其中，传送管理用于 按 ISO 7816-3 、 ISO 14443-4 标准监督卡与终端之间的通信，保证数据正确地传输，防止卡与终端之间通讯数据被非法窃取和篡改 ； 安全管理是 COS 的核心部分，它涉及到卡的鉴别与核实，对文件访问时的权限控制机制 ； 命令管理 根据接收到的命令检查各项参数是否正确，执行相应的操作； 文件管理 将用户数据以文件形式存储在 EEPROM 中，保证访问文件时快速性和数据安全性 。 对于一个具体的COS命令而言，这四个阶段并非都必须具备，有些阶段可以省略，或者是并入另一阶段中。但一般来说，具备这四个阶段的COS是比较常见的。


## COS文件系统

文件是COS中一个非常重要的概念，COS通过给每个应用建立一个对应文件夹的方法来实现对该应用的存储和管理。每个文件在CPU卡中都有一个唯一标识，COS通过文件标识来查找文件，文件不仅在逻辑上是连续的，而且在物理上也是连续的。

本文涉及到的术语及其缩略如下表所列

缩略语 | 含义
:--- | :---
ADF | 应用定义文件（Application Definition File）
AEF | 应用基本文件（Application Elementary File）
AFL | 应用文件定位器（Application File Locator）
AID | 应用标识符（Application Identifier）
APDU | 应用协议数据单元（Application Protocol Data Unit）
ASI | 应用选择指示器（Application Selection Indicator）
ATR | 复位应答（Answer To Reset）
DDF | 目录定义文件（Directory Definition File）
DF | 专用文件（Dedicated File）
DIR | 目录（Directory）
EF | 基本文件（Elementary File）
FCI | 文件控制信息（File Control Information）
FID | 文件标识符（File Identifier）
MF | 根目录文件（Master File）
PSE | 支付系统环境（Payment System Environment）
PTS | 协议类型选择（Protocol Type Selection）
SFI | 短文件标识符（Short File Identifier）
SW1 | 状态字1（Status Word One）
SW2 | 状态字2（Status Word Two）
TPDU | 传输协议数据单元（Transport Protocol Data Unit）

### 文件组织结构

ISO 7816支持两类文件：
- 专用文件 （DF ）
- 基本文件 （EF ）

DF类似于DOS的目录，含有文件控制信息和可分配的存储空间的信息，其下可以建立各种DF和EF文。 DF在用户 存储器中占据一块静态存储器。一旦DF建立，其存 储器的大小就不能变动，但在该DF下的EF则可 以重新分配所使用存储器大小，也可以被删除。该DF 被删除之后，其下的DF和EF也同时被删除，释放 的存储器块可由其他DF使用。

EF类似于DOS的数据 文件， 是IC卡树形文件系统的叶，其下不能再建立其它文件 。EF 用 于存储用户数据或密钥， 存放用户数据的文件称为工作基本文件（Working EF），这些数据仅供IC卡外部的各种应用来使用，如交易日期、交易金额等，在满足一定的安全条件下用户可对文件进行相应的操作。存放密钥的文件称为内部基本文件（Internal EF），这些信息由IC卡内部使用，不可由外界读出，如加密密钥、个人密码等，但当获得许可的权限时可在卡内进行相应的密码运算，在满足写的权限时可以修改密钥。

从终端的角度来看，IC卡上的结构是一种树形结构，这种树形结构能够将数据文件与应用联系起来，可以确保应用之间的独立性，还可以通过应用选择实现对其逻辑结构的访问。 作为根结点的 DF 称做 MF （ Master file ）， MF 是强制存在的， IC卡复位后，卡片自动选择MF为当前文件。虽然系统允许在根目录MF下生成各种EF作为应用文件，但最佳的文件组织方法是每一种应用均分配一个DF，在相应的DF下再具体组织各种EF应用数据。这样可以减少不同应用之间的相互干扰，便于应用设计，有利于一卡多用。卡内的逻辑文件组织结构示例如下图。

![逻辑文件组织结构](http://qiniu.yearito.cn/storage-structure-typeA/image_12.png "逻辑文件组织结构")

### 文件引用方法

要使用某个文件的数据时，必须先引用到该文件，通俗地说，就是选择该文件，对文件的显式选择有以下四种基本方式：  

- __通过文件标识符（FID）来访问（针对所有DF和EF）__

  在智能卡 文件系统中每个文件都有一个FID，它占用2个字 节。值“3FFF”和“FFFF”系统保留，不能用于具体的 文件。对于MF来说，它的FID必须为“3F00”。 因此，当我们访问IC卡的文件系统时，起点就是从FID为“3F00”的MF 开始。为了明确无误地访问一 个文件，对每一个DF文件而言，其下所有的文件（包括EF 文件和DF文件）的FID都必须不相同。

- __通过文件路径来访问（针对所有DF和EF）__

  所谓文件路径，就是无分隔符的FID的串联形 式。这有点类似DOS的文件路径，不过，不同的是DOS 是文件名（DOS的根目录也是一个特殊的文件）的串联，而IC卡 文件系统的文件路径是FID的串联；而且DOS 的路径采用反斜杠 `\` 作为分隔符，而IC卡文件系统的 文件路径没有分隔符。IC卡文件系统的文件路径起始于 MF或者当前的DF的FID，而以要访问文件 的文件标志符结束， 在这两者之间则是相关的DF的FID。 如果当前的DF的FID未知， 则“3FFF”可以作为文件路径的开始。如果文件路径以 MF的FID开始，则称该文件路径为绝对路径； 如果以当前DF的FID开始，则称为相对路径。 这类似DOS的绝对路径和相对路径。通过文件路径，我们 可以明确无误地访问任意文件。

- __通过短文件标识符（SFI）来访问（仅针对EF）__

  对于任意一个EF，可以通过一个5bit编码的短标志符来访问，其范围为1～30。0具有特殊含义，表示当前正在访问的EF文件。SFI不能用于文件路径，也不能用作FID。在一个应用中SFI必须是唯一的。

- __通过DF文件名来访问（仅针对DF）__

  每个DF可以有一个1～16字节长的文件名。为了能够准确无误地通过DF文件名采访问DF，IC卡 中的每个DF文件名必须唯一 。

### 基本文件结构

EF的结构可以分为透明结构和记录结构两种。透明结构EF在通过接口被访问时只被视为数据单元序列而好像没有结构一样，所以称之为透明结构。所谓数据单元，就是可被访问的最小的位集，如1个字节，2个字节等等。透明结构本质上也就是二进制数据结构。记录结构EF在通过结构被访问时被视为具有结构的记录序列。所谓记录，就是可被视为一个整体加以处理的具有结构的字节串。

对记录结构的EF而言，按照长度属性可以分为定长和非定长两种，按照组织结构属性可以分为线性结构和循环结构两种，所以EF可以细分为以下四类，如下图所示：

-  __透明文件（二进制文件）__

  即二进制数据，使用这种数据结构时，一般由用户寻址管理该数据。

- __线性定长记录文件__

  这种结构处理多个固定长度的记录，允许随机的对记录进行读写访问。每一个记录有一个和其位置相关的唯一的记录号，操作系统通过这个 记录号来对记录进行读写操作。根据 ISO 7816标准，这个记录号的范围为 1～254。线性定长文件创建时，操作系统根据记录的总数和记录的长度为其分配存储空间，对数据的访问是以记录为单位进行的，每条记录的长度是相同的，记录满后再不能 写入数据。

- __线性变长记录文件__

  与定长记录文件基本相同，不同之处在于记录的长度可以 不同，文件空间大小在建立文件时给定，以后写入文件的记录长度和记录个数受文件空间大小的限制。

-  __循环记录文件__

  这种结构处理固定长度的元素，并且不允许随机操作。数据元素顺序存储，但只有有限个元素保存在文件中，当数目超过 x 时，最旧的元素被覆盖。循环文件创建时，操作系统根据元素的个数和元素的长度为其分配存储空间。最新插入的元素编号永远是1，上一次插入的元素编号为2，依此类推，如果访问编号超过 x 的元素，操作将被拒绝。

![四类基本文件的存储结构图](http://qiniu.yearito.cn/storage-structure-typeA/image_13.png "四类基本文件的存储结构图")


### 基本文件数据引用

EF中的数据可以通过 数据单元、 记录或者数据 对象来访问。 对透明结构的EF而言，数据被 存储在连续的数据单元序列中； 对记录结构的EF而言，数据被存储在 连续的记录序列中 。如果试图访问不在EF 中的记录、数据单元或者数据对象，将导致错误。数据 访问方法、记录编号方法以及数据单元的大小等作为文件 系统的特征，在IC卡的复位应答（ATR ）过程中由IC卡给出，还可以由IC卡中的ATR文件 给出，以及由其他文件控制信息（FCI）给出。如果IC卡在上面 提及的三种方式中不止一处给出了数据访问方法、记录编 号方法以及数据单元的大小等信息， 则只有从MF 到该EF的文件路径上最靠近该EF的位置 给出的信息是有效的。

- __数据单元引用__

  对每一个透明结构的EF而言，其内部的数据单 元通过偏移 （ offset ） 来访问。偏移是一个无符号整数，长 度为8位或者15位 （ 由不同的访问命令来决定 ） 。当偏移为 0时，则访问该透明结构EF的第一个数据单元；偏移 为1时，访问第二个数据单元；偏移为2时，访问第三个数 据单元……在缺省情况下，也就是IC卡没有给出数据单 元大小的信息肘，默认每个数据单元大小为1字节。

- __记录引用__

  对记录结构的EF，其中的记录可以通过记录编 号来访问。 读卡器把需要读取的记录编号发给IC卡，IC卡把记录编号所对应的记录数据返回给读卡器，一次可以读出一整条记录数据。 记录编号是无符号8位整数，其取值范围为01～ FE。值00保留作特殊用途；值FF保留作将来使用。在每 个记录结构的EF中，每个记录的记录编号都是唯一 并且是有序的。

  对于线性记录结构的EF，当创建或添加记录时，记 录编号按照以一定顺序予以指定。也就是说记录的记录编 号按照记录的创建顺序指定。因此，第一个记录（记录编 号为1）就是第一个创建的记录；第二个记录（记录编号为 2）就是第二个创建的记录……以此类推。

  对于循环记录结构的EF，第一个记录（记录编 号为1）总是最后创建的记录；第二个记录（记录编号为2） 就是倒数第二个创建的记录……以此类推。很显然， 循环记录结构的EF记录编号顺序刚好和线性记 录结构的EF记录编号顺序相反。 记录编号值00总是表示当前记录，也就是记录指针（ record p ointer）当前指向的记录。

- __数据对象引用__

  所谓数据对象，就是采用抽象语法对数据按照一定的格式编码形成的一个数据结构。该数据结构通常包含三部分信息：标签（tag）、长度（length）和值（value）。标签给出了数据的数据类型，如整数、ASCII字符串、Unicode字符串、structure结构类型等等；长度则给出了数据的长度。当访问数据对象时，可以通过标签等对数据对象进行访问。

## APDU报文

为了运行一个应用，在终端上还要实现一个附加的应用协议层，它包括向卡片发送命令、卡片内处理命令和返回IC卡处理响应等步骤。本部分定义的所有命令和响应都定义在应用层应用层发出的命令报文和卡片回送到应用层的响应报文统称为应用协议数据单元（APDU）。响应是和命令相对应的，通常被称为APDU命令-响应对。在一个APDU命令-响应对中，命令报文或响应报文都可能包含数据。

### 命令APDU格式

命令APDU由一个4字节长的必备头后跟一个变长的条件体组成，详见下图。

![命令APDU结构](http://qiniu.yearito.cn/storage-structure-typeA/image_14.png "命令APDU结构")

命令 APDU中发送的数据长度用Lc（命令数据域的长度）表示，期望响应APDU返回的数据字节数用Le（期望数据长度）表示。当Le存在且值为0时，表示要求可能的最大字节数（ ≤ 256）。命令 APDU报文的内容 详 见下图。

![命令APDU内容](http://qiniu.yearito.cn/storage-structure-typeA/image_15.png "命令APDU内容")



### 响应APDU格式

响应APDU由一个变长的条件体和后随两字节长的必备尾组成，详见下图 。

![响应APDU结构](http://qiniu.yearito.cn/storage-structure-typeA/image_16.png "响应APDU结构")

APDU中接收到的数据字节数用Lr（响应数据域长度）表示。Lr不通过传输层返回，应用层在需要时可以依靠响应报文数据域对象结构计算出Lr。响应结尾的2个字节代码是命令的处理状态，它们通过传输层回送。响应APDU报文的内容详见下图 。

![响应APDU内容](http://qiniu.yearito.cn/storage-structure-typeA/image_17.png "响应APDU内容")

### APDU命令-响应结构

在一个APDU命令-响应对中，根据命令报文和响应报文是否包含数据，可以分为以下四种情形。

- 情形一：

    ![](http://qiniu.yearito.cn/storage-structure-typeA/image_18.png)

- 情形二：

    ![](http://qiniu.yearito.cn/storage-structure-typeA/image_19.png)

- 情形三：

    ![](http://qiniu.yearito.cn/storage-structure-typeA/image_20.png)

- 情形四：

    ![](http://qiniu.yearito.cn/storage-structure-typeA/image_21.png)

### 响应状态字含义

任意一条命令的应答至少由一个状态字（ 2个字节）组成。状态字说明了命令处理的情况，即命令是否被正确执行，如果未被正确执行，原因是什么，详见下表。

![](http://qiniu.yearito.cn/storage-structure-typeA/image_22.png)


{% note %}
注意：
当 SW1 的高半字节为 "9" ，且低半字节不为 "0" 时，其含义依赖于相关应用。
当 SW1 的高半字节为 "6" ，且低半字节不为 "0" 时，其含义与应用无关。
{% endnote %}