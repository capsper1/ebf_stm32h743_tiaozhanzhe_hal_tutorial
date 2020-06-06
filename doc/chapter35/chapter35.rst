SDIO—SD卡读写测试
-----------------

本章参考资料：《STM32H74xxx参考手册》、《STM32H743用户手册》WWDG章节、《STM32H743xI规格书》、库帮助文档《STM32H753xx_User_Manual.chm》，
以及SD简易规格文件《Physical Layer Simplified Specification V2.0》(版本号：2.00)。

特别说明，本书内容是以STM32H7xx系列控制器资源讲解。

阅读本章内容之前，建议先阅读SD简易规格文件。

SDIO简介
~~~~~~~~

SD卡(Secure Digital Memory Card)在我们生活中已经非常普遍了，控制器对SD卡进行读写通信操作一般有两种通信接口可选，一种是SPI接口，另外一种就是SDIO接口。SDIO全称是安全数字输入/输出接口，多媒体卡(MMC)、SD卡、SD I/O卡都有SDIO接口。STM32H743x系列控制器有两个SDIO主机接口，它可以与MMC卡、SD卡、SD I/O卡以及CE-ATA设备进行数据传输。MMC卡可以说是SD卡的前身，现阶段已经用得很少。SD I/O卡本身不是用于存储的卡，它是指利用SDIO传输协议的一种外设。比如Wi-Fi Card，它主要是提供Wi-Fi功能，有些Wi-Fi模块是使用串口或者SPI接口进行通信的，但Wi-Fi SDIO Card是使用SDIO接口进行通信的。并且一般设计SD I/O卡是可以插入到SD的插槽。CE-ATA是专为轻薄笔记本硬盘设计的硬盘高速通讯接口。

多媒体卡协会网站\ `www.mmca.org <http://www.mmca.org>`__\ 中提供了有MMCA技术委员会发布的多媒体卡系统规范。

SD卡协会网站\ `www.sdcard.org <http://www.sdcard.org>`__\ 中提供了SD存储卡和SDIO卡系统规范。

CE-ATA工作组网站\ `www.ce-ata.org <http://www.ce-ata.org>`__\ 中提供了CE_ATA系统规范。

随之科技发展，SD卡容量需求越来越大，SD卡发展到现在也是有几个版本的，关于SDIO接口的设备整体概括见
图35_1_。

.. image:: media/image1.png
   :align: center
   :alt: 图 35‑1 SDIO接口的设备
   :name: 图35_1

图 35‑1 SDIO接口的设备

关于SD卡和SD I/O部分内容可以在SD协会网站获取到详细的介绍，比如各种SD卡尺寸规则、读写速度标示方法、应用扩展等等信息。

本章内容针对SD卡使用讲解，对于其他类型卡的应用可以参考相关系统规范实现，所以对于控制器中针对其他类型卡的内容可能在本章中简单提及或者被忽略，本章内容不区分SDIO和SD卡这两个概念。即使目前SD协议提供的SD卡规范版本最新是6.0版本，但STM32H743x系列控制器只支持SD卡规范版本2.0，即只支持标准容量SD和高容量SDHC标准卡，不支持超大容量SDXC标准卡，所以可以支持的最高卡容量是32GB。

SD卡物理结构
~~~~~~~~~~~~

一张SD卡包括有存储单元、存储单元接口、电源检测、卡及接口控制器和接口驱动器5个部分，见图
35‑2。存储单元是存储数据部件，存储单元通过存储单元接口与卡控制单元进行数据传输；电源检测单元保证SD卡工作在合适的电压下，如出现掉电或上状态时，它会使控制单元和存储单元接口复位；卡及接口控制单元控制SD卡的运行状态，它包括有8个寄存器；接口驱动器控制SD卡引脚的输入输出。

.. image:: media/image2.png
   :align: center
   :alt: 图 35‑2 SD卡物理结构
   :name: 图35_2

图 35‑2 SD卡物理结构

SD卡总共有8个寄存器，用于设定或表示SD卡信息，参考表
35‑1。这些寄存器只能通过对应的命令访问，对SD卡进行控制操作并不是像操作控制器GPIO相关寄存器那样一次读写一个寄存器的，它是通过命令来控制，SDIO定义了64个命令，每个命令都有特殊意义，可以实现某一特定功能，SD卡接收到命令后，根据命令要求对SD卡内部寄存器进行修改，程序控制中只需要发送组合命令就可以实现SD卡的控制以及读写操作。

   表 35‑1 SD卡寄存器

======== =========== ======================================================================================
  名称     bit宽度     描述
  CID    128         卡识别号(Card identification number):用来识别的卡的个体号码(唯一的)
  RCA    16          相对地址(Relative card address):卡的本地系统地址，初始化时，动态地由卡建议，主机核准。
  DSR    16          驱动级寄存器(Driver Stage Register):配置卡的输出驱动
  CSD    128         卡的特定数据(Card Specific Data):卡的操作条件信息
  SCR    64          SD配置寄存器(SD Configuration Register):SD 卡特殊特性信息
  OCR    32          操作条件寄存器(Operation conditions register)
  SSR    512         SD状态(SD Status):SD卡专有特征的信息
  CSR    32          卡状态(Card Status):卡状态信息
======== =========== ======================================================================================

每个寄存器位的含义可以参考SD简易规格文件《Physical Layer Simplified
Specification V2.0》第5章内容。

SDIO总线
~~~~~~~~

总线拓扑
^^^^^^^^

SD卡一般都支持SDIO和SPI这两种接口，本章内容只介绍SDIO接口操作方式，如果需要使用SPI操作方式可以参考SPI相关章节。另外，STM32H743x系列控制器的SDIO是不支持SPI通信模式的，如果需要用到SPI通信只能使用SPI外设。

SD卡总线拓扑参考图
35‑3。虽然可以共用总线，但不推荐多卡槽共用总线信号，要求一个单独SD总线应该连接一个单独的SD卡。

.. image:: media/image3.png
   :align: center
   :alt: 图 35‑3 SD卡总线拓扑
   :name: 图35_3

图 35‑3 SD卡总线拓扑

SD卡使用9-pin接口通信，其中3根电源线、1根时钟线、1根命令线和4根数据线，具体说明如下：

-  **CLK：**\ 时钟线，由SDIO主机产生，即由STM32控制器输出；

-  **CMD：**\ 命令控制线，SDIO主机通过该线发送命令控制SD卡，如果命令要求SD卡提供应答(响应)，SD卡也是通过该线传输应答信息；

-  **D0-3：**\ 数据线，传输读写数据；SD卡可将D0拉低表示忙状态；

-  **V\ DD\ 、V\ SS1\ 、V\ SS2\ ：**\ 电源和地信号。

在之前的I2C以及SPI章节都有详细讲解了对应的通信时序，实际上，SDIO的通信时序简单许多，SDIO不管是从主机控制器
向SD卡传输，还是SD卡向主机控制器传输都只以CLK时钟线的\ **上升沿**\ 为有效。SD卡操作过程会使用两种不同频率
的时钟同步数据，一个是识别卡阶段时钟频率FOD，最高为400kHz，另外一个是数据传输模式下时钟频率FPP，默认最高
为25MHz，如果通过相关寄存器配置使SDIO工作在高速模式，此时数据传输模式最高频率为50MHz。

虽然STM32H743控制器有两个SDIO主机，但是我们的开发板只使用了一个SDIO主机，开发板上集成了一个Micro
SD卡槽和SDIO接口的WiFi模块，要求只能使用其中一个设备。SDIO接口的WiFi模块一般集成有使能线，如果需要用到SD卡需要先控制该使能线禁用WiFi模块。

总线协议
^^^^^^^^

SD总线通信是基于命令和数据传输的。通讯由一个起始位(“0”)，由一个停止位(“1”)终止。SD通信一般是主机发送一个命令(Command)，从设备在接收到命令后作出响应(Response)，如有需要会有数据(Data)传输参与。

SD总线的基本交互是命令与响应交互，见图 35‑4。

.. image:: media/image4.png
   :align: center
   :alt: 图 35‑4 命令与响应交互
   :name: 图35_4

图 35‑4 命令与响应交互

SD数据是以块(Black)形式传输的，SDHC卡数据块长度一般为512字节，数据可以从主机到卡，也可以是从卡到主机。数据块需要CRC位来
保证数据传输成功。CRC位由SD卡系统硬件生成。STM32控制器可以控制使用单线或4线传输，本开发板设计使用4线传输。
图35_5_ 为主机向SD卡写入数据块操作示意。

.. image:: media/image5.png
   :align: center
   :alt: 图 35‑5 多块写入操作
   :name: 图35_5

图 35‑5 多块写入操作

SD数据传输支持单块和多块读写，它们分别对应不同的操作命令，多块写入还需要使用命令来停止整个写入操作。数据写入前需要检测SD卡忙状态，因为SD卡在接收到数据后编程到存储区过程需要一定操作时间。SD卡忙状态通过把D0线拉低表示。

数据块读操作与之类似，只是无需忙状态检测。

使用4数据线传输时，每次传输4bit数据，每根数据线都必须有起始位、终止位以及CRC位，CRC位每根数据线都要分别检查，并把检查结果汇总然后在数据传输完后通过D0线反馈给主机。

SD卡数据包有两种格式，一种是常规数据(8bit宽)，它先发低字节再发高字节，而每个字节则是先发高位再发低位，4线传输示意如
图35_6_。

.. image:: media/image6.png
   :align: center
   :alt: 图 35‑6 8位宽数据包传输
   :name: 图35_6

图 35‑6 8位宽数据包传输

4线同步发送，每根线发送一个字节的其中两个位，数据位在四线顺序排列发送，DAT3数据线发较高位，DAT0数据线发较低位。

另外一种数据包发送格式是宽位数据包格式，对SD卡而言宽位数据包发送方式是针对SD卡SSR(SD状态)寄存器内容发送的，SSR寄存器总共有512bit，在
主机发出ACMD13命令后SD卡将SSR寄存器内容通过DAT线发送给主机。宽位数据包格式示意见
图35_7_。

.. image:: media/image7.png
   :align: center
   :alt: 图 35‑7 宽位数据包传输
   :name: 图35_7

图 35‑7 宽位数据包传输

命令
^^^^

SD命令由主机发出，以广播命令和寻址命令为例，广播命令是针对与SD主机总线连接的所有从设备发送的，寻址命令是指定某个地址设备进行命令传输。

命令格式
''''''''

SD命令格式固定为48bit，都是通过CMD线连续传输的（数据线不参与），见
图35_8_。

.. image:: media/image8.png
   :align: center
   :alt: 图 35‑8 SD命令格式
   :name: 图35_8

图 35‑8 SD命令格式

SD命令的组成如下：

-  起始位和终止位：命令的主体包含在起始位与终止位之间，它们都只包含一个数据位，起始位为0，终止位为1。

-  传输标志：用于区分传输方向，该位为1时表示命令，方向为主机传输到SD卡，该位为0时表示响应，方向为SD卡传输到主机。

命令主体内容包括命令、地址信息/参数和CRC校验三个部分。

-  命令号：它固定占用6bit，所以总共有64个命令(代号：CMD0~CMD63)，每个命令都有特
   定的用途，部分命令不适用于SD卡操作，只是专门用于MMC卡或者SD
   I/O卡。

-  地址/参数：每个命令有32bit地址信息/参数用于命令附加内容，例如，广播命令没有地址
   信息，这32bit用于指定参数，而寻址命令这32bit用于指定目标SD卡的地址。

-  CRC7校验：长度为7bit的校验位用于验证命令传输内容正确性，如果发生外部干扰导致传输
   数据个别位状态改变将导致校准失败，也意味着命令传输失败，SD卡不执行命令。

命令类型
''''''''

SD命令有4种类型：

-  无响应广播命令(bc)，发送到所有卡，不返回任务响应；

-  带响应广播命令(bcr)，发送到所有卡，同时接收来自所有卡响应；

-  寻址命令(ac)，发送到选定卡，DAT线无数据传输；

-  寻址数据传输命令(adtc)，发送到选定卡，DAT线有数据传输。

另外，SD卡主机模块系统旨在为各种应用程序类型提供一个标准接口。在此环境中，需要有特定的客户/应用程序功能。为实现这些功能，在标准中定义了两种类型的通用命令：特定应用命令(ACMD)和常规命令(GEN_CMD)。要使用SD卡制造商特定的ACMD命令如ACMD6，需要在发送该命令之前无发送CMD55命令，告知SD卡接下来的命令为特定应用命令。CMD55命令只对紧接的第一个命令有效，SD卡如果检测到CMD55之后的第一条命令为ACMD则执行其特定应用功能，如果检测发现不是ACMD命令，则执行标准命令。

命令描述
''''''''

SD卡系统的命令被分为多个类，每个类支持一种“卡的功能设置”。表
35‑2列举了SD卡部分命令信息，更多详细信息可以参考SD简易规格文件说明，表中填充位和保留位都必须被设置为0。

虽然没有必须完全记住每个命令详细信息，但越熟悉命令对后面编程理解非常有帮助。

.. _表35_2:

表 35‑2 SD部分命令描述

.. image:: media/table1.png
   :align: center

响应
^^^^

响应由SD卡向主机发出，部分命令要求SD卡作出响应，这些响应多用于反馈SD卡的状态。SDIO总共有7个响应类型(代号：R1~R7)，其中SD卡没有R4、R5类型响应。特定的命令对应有特定的响应类型，比如当主机发送CMD3命令时，可以得到响应R6。与命令一样，SD卡的响应也是通过CMD线连续传输的。根据响应内容大小可以分为短响应和长响应。短响应是48bit长度，只有R2类型是长响应，其长度为136bit。各个类型响应具体情况如表
35‑3。

除了R3类型之外，其他响应都使用CRC7校验来校验，对于R2类型是使用CID和CSD寄存器内部CRC7。

表 35‑3 SD卡响应类型

.. image:: media/table2.png
   :align: center

SD卡的操作模式及切换
~~~~~~~~~~~~~~~~~~~~

SD卡的操作模式
^^^^^^^^^^^^^^

SD卡有多个版本，STM32控制器目前最高支持《Physical Layer Simplified
Specification
V2.0》定义的SD卡，STM32控制器对SD卡进行数据读写之前需要识别卡的种类：V1.0标准卡、V2.0标准卡、V2.0高容量卡或者不被识别卡。

SD卡系统(包括主机和SD卡)定义了两种操作模式：卡识别模式和数据传输模式。在系统复位后，主机处于卡识别模式，寻找总线上可用的SDIO设备；同时，SD卡也处于卡识别模式，直到被主机识别到，即当SD卡接收到SEND_RCA(CMD3)命令后，SD卡就会进入数据传输模式，而主机在总线上所有卡被识别后也进入数据传输模式。在每个操作模式下，SD卡都有几种状态，参考表
35‑4，通过命令控制实现卡状态的切换。

   表 35‑4 SD卡状态与操作模式

==================================== ================================
  操作模式                             SD卡状态
无效模式(Inactive)                   无效状态(Inactive State)
卡识别模式(Card identification mode) 空闲状态(Idle State)
\                                    准备状态(Ready State)
\                                    识别状态(Identification State)
数据传输模式(Data transfer mode)     待机状态(Stand-by State)
\                                    传输状态(Transfer State)
\                                    发送数据状态(Sending-data State)
\                                    接收数据状态(Receive-data State)
\                                    编程状态(Programming State)
\                                    断开连接状态(Disconnect State)
==================================== ================================

卡识别模式
^^^^^^^^^^

在卡识别模式下，主机会复位所有处于“卡识别模式”的SD卡，确认其工作电压范围，识别SD卡类型，并且获取SD卡的相对地址(卡相对地址较短，便于寻址)。
在卡识别过程中，要求SD卡工作在识别时钟频率FOD的状态下。卡识别模式下SD卡状态转换如
图35_9_。

.. image:: media/image9.png
   :align: center
   :alt: 图 35‑9 卡识别模式状态转换图
   :name: 图35_9

图 35‑9 卡识别模式状态转换图

主机上电后，所有卡处于空闲状态，包括当前处于无效状态的卡。主机也可以发送GO_IDLE_STATE(CMD0)让所有卡软复位从而进入空闲状态，但当前处于无效状态的卡并不会复位。

主机在开始与卡通信前，需要先确定双方在互相支持的电压范围内。SD卡有一个电压支持范围，主机当前电压必须在该范围可能才能与卡正常通信。SEND_IF_COND(CMD8)命令就是用于验证卡接口操作条件的(主要是电压支持)。卡会根据命令的参数来检测操作条件匹配性，如果卡支持主机电压就产生响应，否则不响应。而主机则根据响应内容确定卡的电压匹配性。CMD8是SD卡标准V2.0版本才有的新命令，所以如果主机有接收到响应，可以判断卡为V2.0或更高版本SD卡。

SD_SEND_OP_COND(ACMD41)命令可以识别或拒绝不匹配它的电压范围的卡。ACMD41命令的VDD电压参数用于设置主机支持电压范围，卡响应会返回卡支持的电压范围。对于对CMD8有响应的卡，把ACMD41命令的HCS位设置为1，可以测试卡的容量类型，如果卡响应的CCS位为1说明为高容量SD卡，否则为标准卡。卡在响应ACMD41之后进入准备状态，不响应ACMD41的卡为不可用卡，进入无效状态。ACMD41是应用特定命令，发送该命令之前必须先发CMD55。

ALL_SEND_CID(CMD2)用来控制所有卡返回它们的卡识别号(CID)，处于准备状态的卡在发送CID之后就进入识别状态。之后主机就发送SEND_RELATIVE_ADDR(CMD3)命令，让卡自己推荐一个相对地址(RCA)并响应命令。这个RCA是16bit地址，而CID是128bit地址，使用RCA简化通信。卡在接收到CMD3并发出响应后就进入数据传输模式，并处于待机状态，主机在获取所有卡RCA之后也进入数据传输模式。

数据传输模式
^^^^^^^^^^^^

只有SD卡系统处于数据传输模式下才可以进行数据读写操作。数据传输模式下可以将主机SD时钟频率设置为FPP，默认最高为25MHz，频率切换可以通
过CMD4命令来实现。数据传输模式下，SD卡状态转换过程见
图35_10_。

.. image:: media/image10.png
   :align: center
   :alt: 图 35‑10 数据传输模式卡状态转换
   :name: 图35_10

图 35‑10 数据传输模式卡状态转换

CMD7用来选定和取消指定的卡，卡在待机状态下还不能进行数据通信，因为总线上可能有多个卡都是出于待机状态，必须选择一个RCA地址目标卡使其进入传输状态才可以进行数据通信。同时通过CMD7命令也可以让已经被选择的目标卡返回到待机状态。

数据传输模式下的数据通信都是主机和目标卡之间通过寻址命令点对点进行的。卡处于传输状态下可以使用表
35‑2中面向块的读写以及擦除命令对卡进行数据读写、擦除。CMD12可以中断正在进行的数据通信，让卡返回到传输状态。CMD0和CMD15会中止任何数据编程操作，返回卡识别模式，这可能导致卡数据被损坏。

STM32的SDMMC功能框图
~~~~~~~~~~~~~~~~~~~~~~~~

STM32控制器有一个SDMMC，由两部分组成：SDMMC适配器和AHB接口，见
图35_11_。SDMMC适配器提供SDMMC主机功能，可以提供SD时钟、发送命令和进行数据传输。AHB接口用于控制器访问SDMMC适配器寄存器。

.. image:: media/image11.png
   :align: center
   :alt: 图 35‑11 SDMMC功能框图
   :name: 图35_11

图 35‑11 SDMMC功能框图

SDMMC使用两个时钟信号，一个是SDMMC适配器时钟(SDMMCCLK=48MHz)，另外一个是AHB2总线时钟(HCLK2，一般为200MHz)。

STM32控制器的SDMMC是针对MMC卡和SD卡的主设备，所以预留有8根数据线，对于SD卡最多用四根数据线。

SDMMC适配器是SD卡系统主机部分，是STM32控制器与SD卡数据通信中间设备。SDMMC适配器由四个单元组成，分别是控制单元、命令路径单元、数据路径单元以及CLKMUX单元。

控制单元
^^^^^^^^^^

控制单元包含电源管理和时钟管理功能，结构如
图35_13_。电源管理部件会在系统断电和上电阶段禁止SD卡总线输出信号。时钟管理部件控制CLK线时钟信号生成。可以使用时钟PPL1Q和PPL2R输入。

.. image:: media/image13.png
   :align: center
   :alt: 图 35‑13 SDMMC适配器控制单元
   :name: 图35_13

图 35‑13 SDMMC适配器控制单元

命令路径
^^^^^^^^^^

命令路径控制命令发送，并接收卡的响应，结构见 图35_14_。

.. image:: media/image14.png
   :align: center
   :alt: 图 35‑14 SDMMC适配器命令路径
   :name: 图35_14

图 35‑14 SDMMC适配器命令路径

关于SDIO适配器状态转换流程可以参考
图35_9_，当SD卡处于某一状态时，SDIO适配器必然处于特定状态与之对应。STM32控制器以命令路径状态机(CPSM)来描述SDIO适配器的状态变化，并加入了等待
超时检测功能，以便退出永久等待的情况。CPSM的描述见 图35_15_。

.. image:: media/image15.png
   :align: center
   :alt: 图 35‑15 CPSM状态机描述图
   :name: 图35_15

图 35‑15 CPSM状态机描述图

数据路径
^^^^^^^^^^

数据路径部件负责与SD卡相互数据传输，内部结构见 图35_16_。

.. image:: media/image16.png
   :align: center
   :alt: 图 35‑16 SDMMC适配器数据路径
   :name: 图35_16

图 35‑16 SDMMC适配器数据路径

SD卡系统数据传输状态转换参考 图35_10_，SDMMC适配器以数据路径状态机(DPSM)来描述SDMMC适配器状态变化情况。
并加入了等待超时检测功能，以便退出永久等待情况。发送数据时，DPSM处于等待发送(Wait_S)状态，如果数据FIFO不为空，
DPSM变成发送状态并且数据路径部件启动向卡发送数据。接收数据时，DPSM处于等待接收状态，当DPSM收到起始位时变成接收状态，
并且数据路径部件开始从卡接收数据。DPSM状态机描述见
图35_17_。

.. image:: media/image17.png
   :align: center
   :alt: 图 35‑17 DPSM状态机描述图
   :name: 图35_17

图 35‑17 DPSM状态机描述图

数据FIFO
^^^^^^^^^^

数据FIFO(先进先出)部件是一个数据缓冲器，带发送和接收单元。控制器的FIFO包含宽度为32bit、深度为32字的数据缓冲器和发送/接收逻辑。如果写入的数据长度不等于4的整数倍，则最后剩下的数据会按照字的格式传输。对于读取来说，则会通过添0，使数据膨胀为字的长度。

根据FIFO空或满状态会把相应的寄存器位值1，并可以产生中断和DMA请求。STM32H7的SDMMC外设可以选择两个DMA，分别是MDMA和IDMA。IDMA用于传输数据或者是接受数据，可以通过将寄存器的IDMAEN位置1来使能。IDMA传输完成所有数据以及DPSM完成发送过程时，标志位DATAEND会被置1。注意，SDMMC的IDMA是通过AXI总线相连的，所以数据只能存放在AXISRAM中。见 图35_17_1_ 总线的内部连接的红色方框。SDMMC通过发送DMA请求，来使能MDMA发送和接受数据，当MDMA传输完成时，DATAEND标志位会被置1。

.. image:: media/image12.png
   :align: center
   :alt: 图 35‑17 总线的内部连接
   :name: 图35_17_1

CLKMUX单元
^^^^^^^^^^^^^^^^^^^^

CLKMUX主要用来选择sdmmc_rx_ck时钟，用于接受数据响应和命令响应，通过修改时钟控制寄存器的位SELCLKRX，见 图35_17_2_ CLKMUX功能框图。

.. image:: media/image12_1.png
   :align: center
   :alt: 图 35‑17-1 CLKMUX功能框图
   :name: 图35_17_2

STM32H7支持DS（Default Speed）、HS（High Speed），SDR以及DDR50等多种模式，见下表格SDIO总线速度模式。

CLKMUX的三个时钟源分别对应不同的工作模式：HS和DS模式下，选择sdmmc_io_ck时钟；SDR12，SDR25，SDR50和DDR50下，选择SDMMC_CKIN时钟；SDR104模式，注意该模式需要启用DLYB外设，选择sdmmc_fb_ck时钟

表格 - SDIO总线速度模式

+---------------------+--------------------------+---------------------+--------+
| SDIO总线速度模式    | 最大总线速度（Mbytes/s） | 最大时钟频率（MHz） | 电压值 |
|                     |                          |                     |        |
|                     |                          |                     | （V）  |
+=====================+==========================+=====================+========+
| DS（Default Speed） | 12.5                     | 25                  | 3.3    |
+---------------------+--------------------------+---------------------+--------+
| HS（High Speed）    | 25                       | 50                  | 3.3    |
+---------------------+--------------------------+---------------------+--------+
| SDR12               | 12.5                     | 25                  | 1.8    |
+---------------------+--------------------------+---------------------+--------+
| SDR25               | 25                       | 50                  | 1.8    |
+---------------------+--------------------------+---------------------+--------+
| DDR50               | 50                       | 50                  | 1.8    |
+---------------------+--------------------------+---------------------+--------+
| SDR50               | 50                       | 100                 | 1.8    |
+---------------------+--------------------------+---------------------+--------+
| SDR104              | 104                      | 208                 | 1.8    |
+---------------------+--------------------------+---------------------+--------+


SDMMC初始化结构体
~~~~~~~~~~~~~~~~~~

HAL库函数对SDMMC外设建立了三个初始化结构体，分别为SDMMC外设管理结构体SD_HandleTypeDef、SDMMC命令初始化结构体SDMMC_CmdInitTypeDef和SDMMC数据初始化结构体SDMMC_DataInitTypeDef。这些结构体成员用于设置SDMMC工作环境参数，并由SDMMC相应初始化配置函数或功能函数调用，这些参数将会被写入到SDMMC相应的寄存器，达到配置SDMMC工作环境的目的。

初始化结构体和初始化库函数配合使用是标准库精髓所在，理解了初始化结构体每个成员意义基本上就可以对该外设运用自如了。初始化结构体定义在stm32h7xx_hal_sd.h文件中，初始化库函数定义在stm32h7xx_hal_sd.c文件中，编程时我们可以结合这两个文件内注释使用。

SDMMC外设管理结构体，主要用于管理SDMMC外设，包括初始化，工作状态，SD卡的信息等等，见 代码清单35_1_ SDMMC外设管理结构体。

代码清单 35‑1  SDMMC外设管理结构体（文件stm32h7xx_hal_sd.h）

.. code-block:: c
   :name: 代码清单35_1

    typedef struct {
        SD_TypeDef               *Instance;        /*!< SDMMC寄存器基地址*/
        SD_InitTypeDef            Init;             /*!< SD初始化结构体*/
        HAL_LockTypeDef           Lock;             /*!< SD锁资源*/
        uint32_t                 *pTxBuffPtr;      /*!< 存放发送数据地址的指针*/
        uint32_t                  TxXferSize;       /*!< 发送数据的大小 */
        uint32_t                 *pRxBuffPtr;      /*!< 存放接受数据地址的指针*/
        uint32_t                  RxXferSize;       /*!< 接受数据的大小*/
        __IO uint32_t             Context;          /*!< SDMMC的工作模式 */
        __IO HAL_SD_StateTypeDef  State;           /*!< SD卡的状态值*/
        __IO uint32_t             ErrorCode;       /*!< SD错误操作返回值*/
        HAL_SD_CardInfoTypeDef    SdCard;           /*!< SD卡的信息*/
        uint32_t                  CSD[4];           /*!< SD卡的CSD寄存器值*/
        uint32_t                  CID[4];           /*!< SD卡的CID寄存器值*/
    } SD_HandleTypeDef;

各结构体成员的作用介绍如下：

(1)	Instance基地址：SDMMC寄存器基地址指针，所有参数都是指定基地址后才能正确写入寄存器。

(2)	Init初始化结构体：SDMMC的初始化结构体，下面会详细讲解每一个成员。

(3)	Lock：SDMMC锁资源。

(4)	pTxBuffPtr：用来存放发送数据地址的指针。

(5)	TxXferSize：用来指定需要发送数据的大小。

(6)	pRxBuffPtr：用来存放接受数据地址的指针。

(7)	RxXferSize：用来指定需要接受数据的大小。

(8)	Context：用来查看SDMMC的工作模式，可以是读取单个块，读取多个块，写单个块，写多个块等等。

(9)	State：SDMMC的工作状态。

(10) ErrorCode：SD卡的错误操作值，提供给用户排查错误。

(11) SdCard：用来保存SD卡的信息。主要有SD的类型，卡的版本号，多少个扇区和扇区的大小等等。

(12) CSD[4]：用来保存SD卡的CSD寄存器值，具体的值的内容请参考SD简易规格文件。

(13) CID[4]：用来保存SD卡的CIS寄存器值，具体的值的内容请参考SD简易规格文件。

SDMMC初始化结构体用于配置SDMMC基本工作环境，比如时钟分频、时钟沿、数据宽度等等。它被HAL_SD_Init函数使用。

代码清单 35‑1-1  SDMMC初始化结构体（文件stm32h7xx_hal_sd.h）

.. code-block:: c
   :name: 代码清单35_1_1

    typedef struct {
        uint32_t ClockEdge;            //时钟沿
        uint32_t ClockPowerSave;       //节能模式
        uint32_t BusWide;              //数据宽度
        uint32_t HardwareFlowControl;  //硬件流控制
        uint32_t ClockDiv;             //时钟分频
    } SDMMC_InitTypeDef;

各结构体成员的作用介绍如下：

(1)	ClockEdge：主时钟SDMMCCLK产生CLK引脚时钟有效沿选择，可选上升沿或下降沿，它设定SDMMC时钟控制寄存器(SDMMC_CLKCR)的NEGEDGE位的值，一般选择设置为高电平。

(2)	ClockPowerSave：节能模式选择，可选使能或禁用，它设定SDMMC_CLKCR寄存器的PWRSAV位的值。如果使能节能模式，CLK线只有在总线激活时才有时钟输出；如果禁用节能模式，始终使能CLK线输出时钟。

(3)	BusWide：数据线宽度选择，可选1位数据总线、4位数据总线或8为数据总线，系统默认使用1位数据总线，操作SD卡时在数据传输模式下一般选择4位数据总线。它设定SDMMC_CLKCR寄存器的WIDBUS位的值。

(4)	HardwareFlowControl：硬件流控制选择，可选使能或禁用，它设定SDMMC_CLKCR寄存器的HWFC_EN位的值。硬件流控制功能可以避免FIFO发送上溢和下溢错误。

(5)	ClockDiv：时钟分频系数，它设定SDMMC_CLKCR寄存器的CLKDIV位的值，设置SDMMCCLK与CLK线输出时钟分频系数：

CLK线时钟频率=SDMMCCLK/([CLKDIV+2])。

SDMMC命令初始化结构体
~~~~~~~~~~~~~~~~~~~~~~~

SDMMC命令初始化结构体用于设置命令相关内容，比如命令号、命令参数、响应类型等等。它被SDMMC_SendCommand函数使用。

代码清单 35‑2 SDMMC命令初始化接口

.. code-block:: c
   :name: 代码清单35_2

    typedef struct {
        uint32_t Argument; // 命令参数
        uint32_t CmdIndex; // 命令号
        uint32_t Response; // 响应类型
        uint32_t WaitForInterrupt; // 等待使能
        uint32_t CPSM;     // 命令路径状态机
    } SDMMC_CmdInitTypeDef;

各个结构体成员介绍如下：

(1) Argument：作为命令的一部分发送到卡的命令参数，它设定SDMMC参数寄存器(SDMMC_ARG)的值。

(2) CmdIndex：命令号选择，它设定SDMMC命令寄存器(SDMMC_CMD)的CMDINDEX位的值。

(3) Response：响应类型，SDMMC定义两个响应类型：长响应和短响应。根据命令号选择对应的响应类型。
    SDMMC定义了四个32位的SDMMC响应寄存器(SDMMC_RESPx,x=1..4)，短响应只用到SDMMC_RESP1。

(4) WaitForInterrupt：等待类型选择，有三种状态可选，一种是无等待状态，超时检测功能启动；
    一种是等待中断，另外一种是等待传输完成。它设定SDMMC_CMD寄存器的WAITPEND位和WAITINT位的值。

(5)	CPSM：命令路径状态机控制，可选使能或禁用CPSM。它设定SDMMC_CMD寄存器的CPSMEN位的值。

SDMMC数据初始化结构体
~~~~~~~~~~~~~~~~~~~~~~~~~~~

SDMMC数据初始化结构体用于配置数据发送和接收参数，比如传输超时、数据长度、传输模式等等。它被SDMMC_DataConfig函数使用。

代码清单 35‑3 SDMMC数据初始化结构体

.. code-block:: c
   :name: 代码清单35_3

    typedef struct {
        uint32_t DataTimeOut;    // 数据传输超时
        uint32_t DataLength;     // 数据长度
        uint32_t DataBlockSize;  // 数据块大小
        uint32_t TransferDir;    // 数据传输方向
        uint32_t TransferMode;   // 数据传输模式
        uint32_t DPSM;           // 数据路径状态机
    } SDMMC_DataInitTypeDef;

各结构体成员介绍如下：

(1) DataTimeOut：设置数据传输以卡总线时钟周期表示的超时周期，它设定SDMMC数据定时器寄存器(SDMMC_DTIMER)的值。
    在DPSM进入Wait_R或繁忙状态后开始递减，直到0还处于以上两种状态则将超时状态标志置1.

(2) DataLength：设置传输数据长度，它设定SDMMC数据长度寄存器(SDMMC_DLEN)的值。

(3) DataBlockSize：设置数据块大小，有多种尺寸可选，不同命令要求的数据块可能不同。
    它设定SDMMC数据控制寄存器(SDMMC_DCTRL)寄存器的DBLOCKSIZE位的值。

(4) TransferDir：数据传输方向，可选从主机到卡的写操作，或从卡到主机的读操作。它设定SDMMC_DCTRL寄存器的DTDIR位的值。

(5) TransferMode：数据传输模式，可选数据块或数据流模式。对于SD卡操作使用数据块类型。它设定SDMMC_DCTRL寄存器的DTMODE位的值。

(6) DPSM：数据路径状态机控制，可选使能或禁用DPSM。它设定SDMMC_DCTRL寄存器的DTEN位的值。
    要实现数据传输都必须使能SDMMC_DPSM。

SD卡读写测试实验
~~~~~~~~~~~~~~~~

SD卡广泛用于便携式设备上，比如数码相机、手机、多媒体播放器等。对于嵌入式设备来说是一种重要的存储数据部件。
类似与SPI Flash芯片数据操作，可以直接进行读写，也可以写入文件系统，然后使用文件系统读写函数，
使用文件系统操作。本实验是进行SD卡最底层的数据读写操作，直接使用SDIO对SD卡进行读写，会损坏SD卡原本内容，
导致数据丢失，实验前请注意备份SD卡的原内容。由于SD卡容量很大，我们平时使用的SD卡都是已经包含有文件系统的，
一般不会使用本章的操作方式编写SD卡的应用，但它是SD卡操作的基础，对于原理学习是非常有必要的，在它的基础上移植文件系统到SD卡的应用将在下一章讲解。

硬件设计
^^^^^^^^

STM32控制器的SDMMC引脚是被设计固定不变的，开发板设计采用四根数据线模式。对于命令线和数据线须需要加一个上拉电阻。

.. image:: media/image18.png
   :align: center
   :alt: 图 35‑18 SD卡硬件设计
   :name: 图35_18

图 35‑18 SD卡硬件设计

软件设计
^^^^^^^^

这里只讲解核心的部分代码，有些变量的设置，头文件的包含等没有全部罗列出来，完整的代码请参考本章配套的工程。有了之前相关SDMMC知识基础，我们就可以着手开始编写SD卡驱动程序了，根据之前内容，可了解操作的大概流程：

-  初始化相关GPIO及SDMMC外设；

-  配置SDMMC基本通信环境进入卡识别模式，通过几个命令处理后得到卡类型；

-  如果是可用卡就进入数据传输模式，接下来就可以进行读、写、擦除的操作。

虽然看起来只有三步，但它们有非常多的细节需要处理。实际上，SD卡是非常常用外设部件，ST公司在其测试板上也有板子SD卡卡槽，并提供了完整的驱动程序，我们直接参考移植使用即可。类似SDMMC、USB这些复杂的外设，它们的通信协议相当庞大，要自行编写完整、严谨的驱动不是一件轻松的事情，这时我们就可以利用ST官方例程的驱动文件，根据自己硬件移植到自己开发平台即可。

在“初识STM32
HAL库”章节我们重点讲解了HAL库的源代码及启动文件和库使用帮助文档这两部分内容，实际上“Utilities”文件夹内容是非常有参考价值的，该文件夹包含了基于ST官方实验板的驱动文件，比如LCD、SDRAM、SD卡、音频解码IC等等底层驱动程序，
另外还有第三方软件库，如emWin图像软件库和FatFs文件系统。虽然，我们的开发平台跟ST官方实验平台硬件设计略有差别，但移植程序方法是完全可行的。学会移植程序可以减少很多工作量，加快项目进程，更何况ST官方的驱动代码是经过严格验证的。

GPIO初始化和DMA配置
'''''''''''''''''''

SDMMC用到CLK线、CMD线和4根DAT线，使用之前必须初始化相关的GPIO，并设置复用模式为SDMMC的类型。而SDMMC外设又支持生成DMA请求，使用DMA传输可以提高数据传输效率，因此在SDMMC的控制代码中，可以把它设置为DMA传输模式或轮询模式，STM32HAL库提供SDMMC示例中针对这两个模式做了区分处理。由于应用中一般都使用DMA传输模式，所以接下来代码分析都以DMA传输模式介绍。

SDMMC底层驱动初始化
========================================

代码清单 35‑4 SDMMC底层驱动初始化（文件bsp_sdio_sd.c）

.. code-block:: c
   :name: 代码清单35_4

    /**
    * @brief  初始化SD外设
    * @param  无
    * @param  无
    * @retval None
    */
    static void BSP_SD_MspInit(void)
    {
        GPIO_InitTypeDef GPIO_InitStruct;

        /* 使能 SDMMC 时钟 */
        __HAL_RCC_SDMMC1_CLK_ENABLE();

        /* 使能 GPIOs 时钟 */
        __HAL_RCC_GPIOC_CLK_ENABLE();
        __HAL_RCC_GPIOD_CLK_ENABLE();

        GPIO_InitStruct.Pin = GPIO_PIN_8|GPIO_PIN_9|GPIO_PIN_10|GPIO_PIN_11
                            |GPIO_PIN_12;
        GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;
        GPIO_InitStruct.Pull = GPIO_NOPULL;
        GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_VERY_HIGH;
        GPIO_InitStruct.Alternate = GPIO_AF12_SDIO1;
        HAL_GPIO_Init(GPIOC, &GPIO_InitStruct);

        GPIO_InitStruct.Pin = GPIO_PIN_2;
        GPIO_InitStruct.Mode = GPIO_MODE_AF_PP;
        GPIO_InitStruct.Pull = GPIO_NOPULL;
        GPIO_InitStruct.Speed = GPIO_SPEED_FREQ_VERY_HIGH;
        GPIO_InitStruct.Alternate = GPIO_AF12_SDIO1;
        HAL_GPIO_Init(GPIOD, &GPIO_InitStruct);
        //禁用WIFI模块
        WIFI_PDN_INIT();

        HAL_NVIC_SetPriority(SDMMC1_IRQn,0,0);  //配置SDMMC1中断
        HAL_NVIC_EnableIRQ(SDMMC1_IRQn);        //使能SDMMC1中断
    }

由于SDMMC对应的IO引脚都是固定的，所以这里没有使用宏定义方式给出，直接使用GPIO引脚，该函数初始化引脚之后，因为SD卡和WIFI共用一个SD主机，所以这里调用了WIFI_PDN_INIT函数，禁用WIFI模块。例程选择IDMA进行数据的传输，只需要开启SDMMC的中断就可以了，通过调用HAL_SD_ReadBlocks_DMA和HAL_SD_WriteBlocks_DMA即可。

SD卡初始化
'''''''''''''''''''

SD卡初始化过程主要是卡识别和相关SD卡状态获取。整个初始化函数可以实现 图 中的功能。

.. image:: media/image19.png
   :align: center
   :alt: 图35_18_1 SD卡初始化和识别流程
   :name: 图35_18_1

SD卡初始化函数
====================

代码清单35_5 BSP_SD_Init函数（文件bsp_sdio_sd.c）

.. code-block:: c
   :name: 代码清单35_5 BSP_SD_Init函数（文件bsp_sdio_sd.c）

    static HAL_StatusTypeDef BSP_SD_Init(void)
    {
        HAL_StatusTypeDef sd_state = HAL_OK;
    
        /* 定义SDMMC句柄 */
        uSdHandle.Instance = SDMMC1;
        uSdHandle.Init.ClockEdge           = SDMMC_CLOCK_EDGE_RISING;
        uSdHandle.Init.ClockPowerSave      = SDMMC_CLOCK_POWER_SAVE_DISABLE;
        uSdHandle.Init.BusWide             = SDMMC_BUS_WIDE_4B;
        uSdHandle.Init.HardwareFlowControl = SDMMC_HARDWARE_FLOW_CONTROL_DISABLE;
        uSdHandle.Init.ClockDiv            = 0;
    
        /* 初始化SD底层驱动 */
        BSP_SD_MspInit();
    
        /* HAL SD 初始化 */
        if (HAL_SD_Init(&uSdHandle) != HAL_OK) {
            sd_state = HAL_OK;
        }
    
        /* 配置SD总线位宽 */
        if (sd_state == HAL_OK) {
            /* 配置为4bit模式 */
        if (HAL_SD_ConfigWideBusOperation(&uSdHandle, uSdHandle.Init.BusWide) != HAL_OK) {
                sd_state = HAL_ERROR;
            } else {
                sd_state = HAL_OK;
            }
        }
    
        return  sd_state;
    }

该函数的部分执行流程如下：

(1)	配置SD外设参数，初始化SD外设。

(2)	执行BSP_SD_MspInit函数，其功能是对底层SDMMC引脚进行初始化以及开启相关时钟，该函数在之前已经讲解。

(3)	HAL_SD_Init函数用于初始化SDMMC外设接口，识别SD卡，流程包括初始化卡上外设接口的默认配置，识别卡工作电压，初始化当前的 SD卡并将其置于空闲状态，读取 CSD/CID 寄存器获取SD卡信息，选择卡，配置 SDMMC 外设接口。

(4)	配置SD接口位宽为4bit用于数据传输。 

代码清单 SD_POWERON函数（文件stm32h7xx_hal_sd.c）

.. code-block:: c
   :name: 代码清单 SD_POWERON函数（文件stm32h7xx_hal_sd.c）

    static uint32_t SD_PowerON(SD_HandleTypeDef *hsd)
    {
        __IO uint32_t count = 0;
        uint32_t response = 0, validvoltage = 0;
        uint32_t errorstate = HAL_SD_ERROR_NONE;
    #if (USE_SD_TRANSCEIVER != 0U)
        uint32_t tickstart = HAL_GetTick();
    #endif

        /* CMD0: GO_IDLE_STATE */
        errorstate = SDMMC_CmdGoIdleState(hsd->Instance);
        if (errorstate != HAL_SD_ERROR_NONE) {
            return errorstate;
        }

        /* CMD8: SEND_IF_COND: Command available only on V2.0 cards */
        errorstate = SDMMC_CmdOperCond(hsd->Instance);
        if (errorstate != HAL_SD_ERROR_NONE) {
            hsd->SdCard.CardVersion = CARD_V1_X;
        } else {
            hsd->SdCard.CardVersion = CARD_V2_X;
        }

        /* SEND CMD55 APP_CMD with RCA as 0 */
        errorstate = SDMMC_CmdAppCommand(hsd->Instance, 0);
        if (errorstate != HAL_SD_ERROR_NONE) {
            return HAL_SD_ERROR_UNSUPPORTED_FEATURE;
        } else {
            /* SD CARD */
            /* Send ACMD41 SD_APP_OP_COND with Argument 0x80100000 */
            while ((!validvoltage) && (count < SDMMC_MAX_VOLT_TRIAL)) {
                /* SEND CMD55 APP_CMD with RCA as 0 */
                errorstate = SDMMC_CmdAppCommand(hsd->Instance, 0);
                if (errorstate != HAL_SD_ERROR_NONE) {
                    return errorstate;
                }

                /* Send CMD41 */
                errorstate = SDMMC_CmdAppOperCommand(hsd->Instance, 
                            SDMMC_VOLTAGE_WINDOW_SD | SDMMC_HIGH_CAPACITY 
                            | SD_SWITCH_1_8V_CAPACITY);
                if (errorstate != HAL_SD_ERROR_NONE) {
                    return HAL_SD_ERROR_UNSUPPORTED_FEATURE;
                }

                /* Get command response */
                response = SDMMC_GetResponse(hsd->Instance, SDMMC_RESP1);

                /* Get operating voltage*/
                validvoltage = (((response >> 31) == 1) ? 1 : 0);

                count++;
            }

            if (count >= SDMMC_MAX_VOLT_TRIAL) {
                return HAL_SD_ERROR_INVALID_VOLTRANGE;
            }

            if ((response & SDMMC_HIGH_CAPACITY) == SDMMC_HIGH_CAPACITY) { 
                /* (response &= SD_HIGH_CAPACITY) */
                hsd->SdCard.CardType = CARD_SDHC_SDXC;
    #if (USE_SD_TRANSCEIVER != 0U)
                if ((response & SD_SWITCH_1_8V_CAPACITY) == 
                    SD_SWITCH_1_8V_CAPACITY) {
                    hsd->SdCard.CardSpeed = CARD_ULTRA_HIGH_SPEED;

                    /* Start switching procedue */
                    hsd->Instance->POWER |= SDMMC_POWER_VSWITCHEN;

                    /* Send CMD11 to switch 1.8V mode */
                    errorstate = SDMMC_CmdVoltageSwitch(hsd->Instance);
                    if (errorstate != HAL_SD_ERROR_NONE) {
                        MODIFY_REG(hsd->Instance->POWER, 
                                SDMMC_POWER_VSWITCHEN, 0);
                        return HAL_SD_ERROR_NONE;
                    }

                    /* Check to BusyD0 and CKSTOP */
                    while (( hsd->Instance->STA & SDMMC_FLAG_CKSTOP) != 
                        SDMMC_FLAG_CKSTOP) {
                        if ((HAL_GetTick() - tickstart) >=  
                            SDMMC_DATATIMEOUT) {
                            return HAL_SD_ERROR_TIMEOUT;
                        }
                    }

                    if (( hsd->Instance->STA & SDMMC_FLAG_BUSYD0) != 
                        SDMMC_FLAG_BUSYD0) {
                        /* Error when activate Voltage Switch in SDMMC IP 
                        */
                        return SDMMC_ERROR_UNSUPPORTED_FEATURE;
                    } else {
                        /* Enable Trasciver Switch PIN */
                        HAL_SD_DriveTransciver_1_8V_Callback(SET);

                        /* Switch ready */
                        hsd->Instance->POWER |= SDMMC_POWER_VSWITCH;

                        /* Check VSWEND Flag */
                        while (( hsd->Instance->STA & SDMMC_FLAG_VSWEND) 
                                != SDMMC_FLAG_VSWEND) {
                            if ((HAL_GetTick() - tickstart) >=  
                                SDMMC_DATATIMEOUT) {
                                return HAL_SD_ERROR_TIMEOUT;
                            }
                        }
                        /* Check BusyD0 status */
                        if (( hsd->Instance->STA & SDMMC_FLAG_BUSYD0) == 
                            SDMMC_FLAG_BUSYD0) {
                            /* Error when enabling 1.8V mode */
                            return HAL_SD_ERROR_INVALID_VOLTRANGE;
                        }
                        /* Switch to 1.8V OK */
    
                        /* Disable VSWITCH FLAG from SDMMC IP */
                        hsd->Instance->POWER = 0x13;
                        /* Clean Status flags */
                        hsd->Instance->ICR = 0xFFFFFFFF;
                    }
    
                    hsd->SdCard.CardSpeed = CARD_ULTRA_HIGH_SPEED;
    
                }
    #endif /* USE_SD_TRANSCEIVER  */
            }
    
        }
    
        return HAL_SD_ERROR_NONE;
    }

SD_POWERON函数执行流程如下：

(1)	配置SDIO_InitStructure结构体变量成员并调用SDIO_Init库函数完成SDIO外设的基本配置，注意此处的SDIO时钟分频，由于处于卡识别阶段，其时钟不能超过400KHz。

(2)	调用SDMMC_PowerState_ON函数控制SDMMC的电源状态，给SDMMC提供电源，并调用__HAL_SD_SDMMC_DISABLE库函数使能SDMMC时钟。

(3)	发送命令给SD卡，首先发送CMD0，复位所有SD卡， CMD0命令无需响应，所以调用SD_CmdError函数检测错误即可。SD_CmdError函数用于无需响应的命令发送检测，带有等待超时检测功能，它通过不断检测SDIO_STA寄存器的CMDSENT位即可知道命令发送成功与否。如果遇到超时错误则直接退出SDMMC_PowerState_ON函数。如果无错误则执行下面程序。

(4)	发送CMD8命令，检测SD卡支持的操作条件，主要就是电压匹配，CMD8的响应类型是R7，使用SD_CmdResp7Error函数可获取得到R7响应结果，它是通过检测SDMMC_STA寄存器相关位完成的，并具有等待超时检测功能。如果SD_CmdResp7Error函数返回值为SD_OK，即CMD8有响应，可以判定SD卡为V2.0及以上的高容量SD卡，如果没有响应可能是V1.1版本卡或者是不可用卡。

(5)	使用ACMD41命令判断卡的具体类型。在发送ACMD41之前必须先发送CMD55，CMD55命令的响应类型的R1。如果CMD55命令都没有响应说明是MMC卡或不可用卡。在正确发送CMD55之后就可以发送ACMD41，并根据响应判断卡类型，ACMD41的响应号为R3，SD_CmdResp3Error函数用于检测命令正确发送并带有超时检测功能，但并不具备响应内容接收功能，需要在判定命令正确发送之后调用SDMMC_GetResponse函数才能获取响应的内容。实际上，在有响应时，SDMMC外设会自动把响应存放在SDMMC_RESPx寄存器中，SDMMC_GetResponse函数只是根据形参返回对应响应寄存器的值。通过判定响应内容值即可确定SD卡类型。

(6)	执行SD_PowerON函数无错误后就已经确定了SD卡类型，并说明卡和主机电压是匹配的，SD卡处于卡识别模式下的准备状态。退出SD_Power_ON函数返回HAL_SD_Init函数，执行接下来代码。判断执行SD_PowerON函数无错误后，执行下面的SD_Initialize_Cards函数进行与SD卡相关的初始化，使得卡进入数据传输模式下的待机模式。

代码清单 SD_InitializeCards函数（文件stm32h7xx_hal_sd.c）

.. code-block:: c
   :name: 代码清单 SD_InitializeCards函数（文件stm32h7xx_hal_sd.c）

    HAL_StatusTypeDef HAL_SD_InitCard(SD_HandleTypeDef *hsd)
    {
        uint32_t errorstate = HAL_SD_ERROR_NONE;
        SD_InitTypeDef Init;
    
        /* Default SDMMC peripheral configuration for SD card initialization  */
        Init.ClockEdge           = SDMMC_CLOCK_EDGE_RISING;
        Init.ClockPowerSave      = SDMMC_CLOCK_POWER_SAVE_DISABLE;
        Init.BusWide             = SDMMC_BUS_WIDE_1B;
        Init.HardwareFlowControl = SDMMC_HARDWARE_FLOW_CONTROL_DISABLE;
        Init.ClockDiv            = SDMMC_INIT_CLK_DIV;
    
    #if (USE_SD_TRANSCEIVER != 0U)
        /* Set Transceiver polarity */
        hsd->Instance->POWER |= SDMMC_POWER_DIRPOL;
    #endif /* USE_SD_TRANSCEIVER  */
    
        /* Initialize SDMMC peripheral interface with default configuration */

        SDMMC_Init(hsd->Instance, Init);
    
        /* Set Power State to ON */
        SDMMC_PowerState_ON(hsd->Instance);
    
    
        /* Identify card operating voltage */
        errorstate = SD_PowerON(hsd);
        if (errorstate != HAL_SD_ERROR_NONE) {
            hsd->State = HAL_SD_STATE_READY;
            hsd->ErrorCode |= errorstate;
            return HAL_ERROR;
        }
    
        /* Card initialization */
        errorstate = SD_InitCard(hsd);
        if (errorstate != HAL_SD_ERROR_NONE) {
            hsd->State = HAL_SD_STATE_READY;
            hsd->ErrorCode |= errorstate;
            return HAL_ERROR;
        }
    
        return HAL_OK;
    } 

SD_Initialize_Cards函数执行流程如下：

(1)	判断SDMMC电源是否启动，如果没有启动电源返回错误。

(2)	SD卡不是SD I/O卡时会进入if判断，执行发送CMD2，CMD2是用于通知所有卡通过CMD线返回CID值，执行CMD2发送之后就可以使用CmdResp2Error函数获取CMD2命令发送情况，发送无错误后即可以使用SDMMC_GetResponse函数获取响应内容，它是个长响应，我们把CMD2响应内容存放在CID数组内。

(3)	发送CMD2之后紧接着就发送CMD3，用于指示SD卡自行推荐RCA地址，CMD3的响应为R6类型，SD_CmdResp6Error函数用于检查R6响应错误，它有两个形参，一个是命令号，这里为CMD3，另外一个是RCA数据指针，这里使用rca变量的地址赋值给它，使得在CMD3正确响应之后rca变量即存放SD卡的RCA。R6响应还有一部分位用于指示卡的状态，SD_CmdResp6Error函数通用会对每个错误位进行必要的检测，如果发现有错误存在则直接返回对应错误类型。执行完SD_CmdResp6Error函数之后返回到SD_Initialize_Cards函数中，如果判断无错误说明此刻SD卡已经处于数据传输模式。

(4)	发送CMD9给指定RCA的SD卡使其发送返回其CSD寄存器内容，这里的RCA就是在SD_CmdResp6Error函数获取得到的rca。最后把响应内容存放在CSD数组中。

执行SD_Initialize_Cards函数无错误后SD卡就已经处于数据传输模式下的待机状态，退出SD_Initialize_Cards后会返回前面的HAL_SD_Init函数，执行接下来代码，以下是HAL_SD_Init函数的后续执行过程：

(1)	重新配置SDMMC外设，提高时钟频率，之前的卡识别模式都设定CMD线时钟为小于400KHz，进入数据传输模式可以把时钟设置为小于25MHz，以便提高数据传输速率。

(2)	调用SD_Initialize_Cards函数获取SD卡信息，它需要一个指向SD_CardInfo类型变量地址的指针形参，这里赋值为SDCardInfo变量的地址。SD卡信息主要是CID和CSD寄存器内容，这两个寄存器内容在SD_InitializeCards函数中都完成读取过程并将其分别存放在CID数组和CSD数组中，所以HAL_SD_Get_CardInfo函数只是简单的把这两个数组内容整合复制到SDCardInfo变量对应成员内。正确执行HAL_SD_Get_CardInfo函数后，SDCardInfo变量就存放了SD卡的很多状态信息，这在之后应用中使用频率是很高的。

(3)	调用SD_Select_Deselect函数用于选择特定RCA的SD卡，它实际是向SD卡发送CMD7。执行之后，卡就从待机状态转变为传输模式，可以说数据传输已经是万事俱备了。

(4)	扩展数据线宽度，之前的所有操作都是使用一根数据线传输完成的，使用4根数据线可以提高传输性能，调用可以设置数据线宽度，函数有两个形参，一个指定句柄另外一个用于指定数据线宽度。在HAL_SD_WideBusOperation_Config函数中，调用了SD_WideBus_Enable函数使能使用宽数据线，然后传输SDIO_InitTypeDef类型变量并使用SDMMC_Init函数完成使用4根数据线配置。

至此，BSP_SD_Init函数已经全部执行完成。如果程序可以正确执行，接下来就可以进行SD卡读写以及擦除等操作。虽然bsp_sdio_sd.c文件看起来非常长，但在BSP_SD_Init函数分析过程就已经涉及到它差不多一半内容了，另外一半内容主要就是读、写或擦除相关函数。

SD卡数据操作
'''''''''''''

SD卡数据操作一般包括数据读取、数据写入以及存储区擦除。数据读取和写入都可以分为单块操作和多块操作。

擦除函数
==============

代码清单 35‑9 SD_Erase函数（文件stm32h7xx_hal_sd.c）

.. code-block:: c
   :name: 代码清单35_9

    HAL_StatusTypeDef HAL_SD_Erase(SD_HandleTypeDef *hsd, uint32_t 
                                BlockStartAdd, uint32_t BlockEndAdd)
    {
        uint32_t errorstate = HAL_SD_ERROR_NONE;

        if (hsd->State == HAL_SD_STATE_READY) {
            hsd->ErrorCode = HAL_DMA_ERROR_NONE;

            if (BlockEndAdd < BlockStartAdd) {
                hsd->ErrorCode |= HAL_SD_ERROR_PARAM;
                return HAL_ERROR;
            }

            if (BlockEndAdd > (hsd->SdCard.LogBlockNbr)) {
                hsd->ErrorCode |= HAL_SD_ERROR_ADDR_OUT_OF_RANGE;
                return HAL_ERROR;
            }

            hsd->State = HAL_SD_STATE_BUSY;

            /* Check if the card command class supports erase command */
            if (((hsd->SdCard.Class) & SDMMC_CCCC_ERASE) == 0U) {
                /* Clear all the static flags */
                __HAL_SD_CLEAR_FLAG(hsd, SDMMC_STATIC_FLAGS);
                hsd->ErrorCode |= HAL_SD_ERROR_REQUEST_NOT_APPLICABLE;
                hsd->State = HAL_SD_STATE_READY;
                return HAL_ERROR;
            }

            if ((SDMMC_GetResponse(hsd->Instance, SDMMC_RESP1) & 
                SDMMC_CARD_LOCKED) == SDMMC_CARD_LOCKED) {
                /* Clear all the static flags */
                __HAL_SD_CLEAR_FLAG(hsd, SDMMC_STATIC_FLAGS);
                hsd->ErrorCode |= HAL_SD_ERROR_LOCK_UNLOCK_FAILED;
                hsd->State = HAL_SD_STATE_READY;
                return HAL_ERROR;
            }

            /* Get start and end block for high capacity cards */
            if (hsd->SdCard.CardType != CARD_SDHC_SDXC) {
                BlockStartAdd *= 512U;
                BlockEndAdd   *= 512U;
            }

            /* According to sd-card spec 1.0 ERASE_GROUP_START (CMD32) and 
                                                                erase_group
                                                                _end(CMD33)
                                                                */
            if (hsd->SdCard.CardType != CARD_SECURED) {
                /* Send CMD32 SD_ERASE_GRP_START with argument as addr  */
                errorstate = SDMMC_CmdSDEraseStartAdd(hsd->Instance, 
                            BlockStartAdd);
                if (errorstate != HAL_SD_ERROR_NONE) {
                    /* Clear all the static flags */
                    __HAL_SD_CLEAR_FLAG(hsd, SDMMC_STATIC_FLAGS);
                    hsd->ErrorCode |= errorstate;
                    hsd->State = HAL_SD_STATE_READY;
                    return HAL_ERROR;
                }

                /* Send CMD33 SD_ERASE_GRP_END with argument as addr  */
                errorstate = SDMMC_CmdSDEraseEndAdd(hsd->Instance, 
                            BlockEndAdd);
                if (errorstate != HAL_SD_ERROR_NONE) {
                    /* Clear all the static flags */
                    __HAL_SD_CLEAR_FLAG(hsd, SDMMC_STATIC_FLAGS);
                    hsd->ErrorCode |= errorstate;
                    hsd->State = HAL_SD_STATE_READY;
                    return HAL_ERROR;
                }
            }

            /* Send CMD38 ERASE */
            errorstate = SDMMC_CmdErase(hsd->Instance);
            if (errorstate != HAL_SD_ERROR_NONE) {
                /* Clear all the static flags */
                __HAL_SD_CLEAR_FLAG(hsd, SDMMC_STATIC_FLAGS);
                hsd->ErrorCode |= errorstate;
                hsd->State = HAL_SD_STATE_READY;
                return HAL_ERROR;
            }

            hsd->State = HAL_SD_STATE_READY;

            return HAL_OK;
        } else {
            return HAL_BUSY;
        }
    }

HAL_SD_Erase函数用于擦除SD卡指定地址范围内的数据。该函数接收三个参数，一个是SD外设的句柄，一个是擦除的起始地址，另外一个是擦除的结束地址。对于高容量SD卡都是以块大小为512字节进行擦除的，所以保证字节对齐是程序员的责任。HAL_SD_Erase函数的执行流程如下：

(1)	检查SD卡是否支持擦除功能，如果不支持则直接返回错误。为保证擦除指令正常进行，要求主机一个遵循下面的命令序列发送指令：CMD32->CMD33->CMD38。如果发送顺序不对，SD卡会设置ERASE_SEQ_ERROR位到状态寄存器。

(2)	HAL_SD_Erase函数发送CMD32指令用于设定擦除块开始地址，在执行无错误后发送CMD33设置擦除块的结束地址。

(3)	发送擦除命令CMD38，使得SD卡进行擦除操作。SD卡擦除操作由SD卡内部控制完成，不同卡擦除后是0xff还是0x00由厂家决定。擦除操作需要花费一定时间，这段时间不能对SD卡进行其他操作。

(4)	通过SD_IsCardProgramming函数可以检测SD卡是否处于编程状态(即卡内部的擦写状态)，需要确保SD卡擦除完成才退出HAL_SD_Erase函数。IsCardProgramming函数先通过发送CMD13命令SD卡发送它的状态寄存器内容，并对响应内容进行分析得出当前SD卡的状态以及可能发送的错误。

数据写入操作
================

数据写入可分为单块数据写入和多块数据写入，这里只分析单块数据写入，多块的与之类似。SD卡数据写入之前并没有硬性要求擦除写入块，
这与SPI Flash芯片写入是不同的。ST官方的SD卡写入函数包括扫描查询方式和DMA传输方式，我们这里只介绍DMA传输模式。

代码清单 35‑10 SD_WriteBlock函数（文件stm32h7xx_hal_sd.c）

.. code-block:: c
   :name: 代码清单35_10

    HAL_StatusTypeDef HAL_SD_WriteBlocks_DMA(SD_HandleTypeDef *hsd, uint8_t 
    *pData, uint32_t BlockAdd, 
                                            uint32_t NumberOfBlocks)
    {
        SDMMC_DataInitTypeDef config;
        uint32_t errorstate = HAL_SD_ERROR_NONE;
    
        if (NULL == pData) {
            hsd->ErrorCode |= HAL_SD_ERROR_PARAM;
            return HAL_ERROR;
        }
    
        if (hsd->State == HAL_SD_STATE_READY) {
            hsd->ErrorCode = HAL_DMA_ERROR_NONE;
    
            if ((BlockAdd + NumberOfBlocks) > (hsd->SdCard.LogBlockNbr)) {
                hsd->ErrorCode |= HAL_SD_ERROR_ADDR_OUT_OF_RANGE;
                return HAL_ERROR;
            }
    
            hsd->State = HAL_SD_STATE_BUSY;
    
            /* Initialize data control register */
            hsd->Instance->DCTRL = 0U;
    
            hsd->pTxBuffPtr = (uint32_t*)pData;
            hsd->TxXferSize = BLOCKSIZE * NumberOfBlocks;
    
            if (hsd->SdCard.CardType != CARD_SDHC_SDXC) {
                BlockAdd *= 512U;
            }
    
            /* Set Block Size for Card */
            errorstate = SDMMC_CmdBlockLength(hsd->Instance, BLOCKSIZE);
            if (errorstate != HAL_SD_ERROR_NONE) {
                /* Clear all the static flags */
                __HAL_SD_CLEAR_FLAG(hsd, SDMMC_STATIC_FLAGS);
                hsd->ErrorCode |= errorstate;
                hsd->State = HAL_SD_STATE_READY;
                return HAL_ERROR;
            }
            /* Configure the SD DPSM (Data Path State Machine) */
            config.DataTimeOut   = SDMMC_DATATIMEOUT;
            config.DataLength    = BLOCKSIZE * NumberOfBlocks;
            config.DataBlockSize = SDMMC_DATABLOCK_SIZE_512B;
            config.TransferDir   = SDMMC_TRANSFER_DIR_TO_CARD;
            config.TransferMode  = SDMMC_TRANSFER_MODE_BLOCK;
            config.DPSM          = SDMMC_DPSM_DISABLE;
            SDMMC_ConfigData(hsd->Instance, &config);
    
            /* Enable transfer interrupts */
            __HAL_SD_ENABLE_IT(hsd, (SDMMC_IT_DCRCFAIL | SDMMC_IT_DTIMEOUT | 
    SDMMC_IT_TXUNDERR | 
                                SDMMC_IT_DATAEND));
    
            __SDMMC_CMDTRANS_ENABLE( hsd->Instance);
    
            hsd->Instance->IDMACTRL  = SDMMC_ENABLE_IDMA_SINGLE_BUFF;
            hsd->Instance->IDMABASE0 = (uint32_t) pData ;
    
            /* Write Blocks in Polling mode */
            if (NumberOfBlocks > 1U) {
                hsd->Context = (SD_CONTEXT_WRITE_MULTIPLE_BLOCK | 
    SD_CONTEXT_DMA);
    
                /* Write Multi Block command */
                errorstate = SDMMC_CmdWriteMultiBlock(hsd->Instance, 
    BlockAdd);
            } else {
                hsd->Context = (SD_CONTEXT_WRITE_SINGLE_BLOCK | 
    SD_CONTEXT_DMA);
    
                /* Write Single Block command */
                errorstate = SDMMC_CmdWriteSingleBlock(hsd->Instance, 
    BlockAdd);
            }
            if (errorstate != HAL_SD_ERROR_NONE) {
                /* Clear all the static flags */
                __HAL_SD_CLEAR_FLAG(hsd, SDMMC_STATIC_FLAGS);
                __HAL_SD_DISABLE_IT(hsd, (SDMMC_IT_DCRCFAIL | 
    SDMMC_IT_DTIMEOUT | SDMMC_IT_TXUNDERR | 
                                    SDMMC_IT_DATAEND));
                hsd->ErrorCode |= errorstate;
                hsd->State = HAL_SD_STATE_READY;
                return HAL_ERROR;
            }
    
            return (HAL_StatusTypeDef)errorstate;
        } else {
            return HAL_BUSY;
        }
    
    } 

HAL_SD_WriteBlocks_DMA函数用于向指定的目标地址写入块的数据，它有五个形参，分别为指SDMMC外设句柄，向待写入数据的首地址的指针变量、目标写入地址和块大小，块数量。块大小一般都设置为512字节。HAL_SD_WriteBlocks_DMA写入函数的执行流程如下：

(1)	对SD卡进行数据读写之前，都必须发送CMD16指定块的大小，对于标准卡，要写入BlockSize长度字节的块；对于SDHC卡，写入512字节的块。

(2)	接下来就可以发送块写入命令CMD24通知SD卡要进行数据写入操作，并指定待写入数据的目标地址。

(3)	来调用__HAL_SD_SDMMC_ENABLE_IT函数使能相关中断，包括数据CRC失败中断、数据超时中断、数据结束中断等等。

(4)	利用SDMMC_DataInitTypeDef结构体类型变量配置数据传输的超时、块数量、数据块大小、数据传输方向等参数并使用SDMMC_DataConfig函数完成数据传输环境配置。

执行完以上代码后，SDMMC外设会自动生成DMA发送请求，将指定数据使用DMA传输写入到SD卡内。

数据读取操作
==================

同向SD卡写入数据类似，从SD卡读取数据可分为单块读取和多块读取。这里这介绍单块读操作函数，多块读操作类似理解即可。

代码清单 35‑12 SD_ReadBlock函数

.. code-block:: c
   :name: 代码清单35_12

    HAL_StatusTypeDef HAL_SD_ReadBlocks_DMA(SD_HandleTypeDef *hsd, uint8_t 
                                            *pData, uint32_t BlockAdd, 
                                            uint32_t NumberOfBlocks)
    {
        SDMMC_DataInitTypeDef config;
        uint32_t errorstate = HAL_SD_ERROR_NONE;

        if (NULL == pData) {
            hsd->ErrorCode = HAL_SD_ERROR_PARAM;
            return HAL_ERROR;
        }

        if (hsd->State == HAL_SD_STATE_READY) {
            hsd->ErrorCode = HAL_DMA_ERROR_NONE;

            if ((BlockAdd + NumberOfBlocks) > (hsd->SdCard.LogBlockNbr)) {
                hsd->ErrorCode = HAL_SD_ERROR_ADDR_OUT_OF_RANGE;
                return HAL_ERROR;
            }

            hsd->State = HAL_SD_STATE_BUSY;

            /* Initialize data control register */
            hsd->Instance->DCTRL = 0U;

            hsd->pRxBuffPtr = (uint32_t*)pData;
            hsd->RxXferSize = BLOCKSIZE * NumberOfBlocks;

            hsd->State = HAL_SD_STATE_BUSY;

            if (hsd->SdCard.CardType != CARD_SDHC_SDXC) {
                BlockAdd *= 512U;
            }

            /* Configure the SD DPSM (Data Path State Machine) */
            config.DataTimeOut   = SDMMC_DATATIMEOUT;
            config.DataLength    = BLOCKSIZE * NumberOfBlocks;
            config.DataBlockSize = SDMMC_DATABLOCK_SIZE_512B;
            config.TransferDir   = SDMMC_TRANSFER_DIR_TO_SDMMC;
            config.TransferMode  = SDMMC_TRANSFER_MODE_BLOCK;
            config.DPSM          = SDMMC_DPSM_DISABLE;
            SDMMC_ConfigData(hsd->Instance, &config);

            /* Set Block Size for Card */
            errorstate = SDMMC_CmdBlockLength(hsd->Instance, BLOCKSIZE);
            if (errorstate != HAL_SD_ERROR_NONE) {
                /* Clear all the static flags */
                __HAL_SD_CLEAR_FLAG(hsd, SDMMC_STATIC_FLAGS);
                hsd->ErrorCode = errorstate;
                hsd->State = HAL_SD_STATE_READY;
                return HAL_ERROR;
            }

            /* Enable transfer interrupts */
            __HAL_SD_ENABLE_IT(hsd, (SDMMC_IT_DCRCFAIL | SDMMC_IT_DTIMEOUT 
                            | SDMMC_IT_RXOVERR | SDMMC_IT_DATAEND));

            __SDMMC_CMDTRANS_ENABLE( hsd->Instance);
            hsd->Instance->IDMACTRL  = SDMMC_ENABLE_IDMA_SINGLE_BUFF;
            hsd->Instance->IDMABASE0 = (uint32_t) pData ;

            /* Read Blocks in DMA mode */
            if (NumberOfBlocks > 1U) {
                hsd->Context = (SD_CONTEXT_READ_MULTIPLE_BLOCK | 
                            SD_CONTEXT_DMA);

                /* Read Multi Block command */
                errorstate = SDMMC_CmdReadMultiBlock(hsd->Instance, 
                            BlockAdd);
            } else {
                hsd->Context = (SD_CONTEXT_READ_SINGLE_BLOCK | 
                            SD_CONTEXT_DMA);

                /* Read Single Block command */
                errorstate = SDMMC_CmdReadSingleBlock(hsd->Instance, 
                            BlockAdd);
            }
            if (errorstate != HAL_SD_ERROR_NONE) {
                /* Clear all the static flags */
                __HAL_SD_CLEAR_FLAG(hsd, SDMMC_STATIC_FLAGS);
                __HAL_SD_DISABLE_IT(hsd, (SDMMC_IT_DCRCFAIL | 
                                    SDMMC_IT_DTIMEOUT | SDMMC_IT_RXOVERR | 
                                    SDMMC_IT_DATAEND));
                hsd->ErrorCode = errorstate;
                hsd->State = HAL_SD_STATE_READY;
                return HAL_ERROR;
            }

            return HAL_OK;
        } else {
            return HAL_BUSY;
        }
    }

数据读取操作与数据写入操作编程流程是类似，只是数据传输方向改变，使用到的SD命令号也有所不同而已。HAL_SD_ReadBlocks_DMA函数有五个形参，分别为指SDMMC外设句柄，向待写入数据的首地址的指针变量、目标写入地址和块数量。HAL_SD_ReadBlocks_DMA函数执行流程如下：

(1)	调用__HAL_SD_ENABLE_IT函数使能相关中断，包括数据CRC失败中断、数据超时中断、数据结束中断等等。对于高容量的SD卡要求块大小必须为512字节，程序员有责任保证目标读取地址与块大小的字节对齐问题。

(2)	对SD卡进行数据读写之前，都必须发送CMD16指定块的大小，对于标准卡，读取BlockSize长度字节的块；对于SDHC卡，读取512字节的块。

(3)	利用SDMMC_DataInitTypeDef结构体类型变量配置数据传输的超时、块数量、数据块大小、数据传输方向等参数并使用SDMMC_DataConfig函数完成数据传输环境配置。

(4)	最后控制器向SD卡发送单块读数据命令CMD17，SD卡在接收到命令后就会通过数据线把数据传输到控制器数据FIFO内，并自动生成DMA传输请求。

SDMMC中断服务函数
'''''''''''''''''''''''

在进行数据传输操作时都会使能相关标志中断，用于跟踪传输进程和错误检测。如果是使用DMA传输，也会使能DMA相关中断。为简化代码，加之SDMMC中断服务函数内容一般不会修改，将中断服务函数放在bsp_sdio_sd.c文件中，而不是放在常用于存放中断服务函数的stm32h7xx_it.c文件。

代码清单 35‑14 SDMMC中断服务函数

.. code-block:: c
   :name: 代码清单35_14

    /**
    * @brief  This function handles SD card interrupt request.
    * @param  hsd: Pointer to SD handle
    * @retval None
    */
    void HAL_SD_IRQHandler(SD_HandleTypeDef *hsd)
    {
        uint32_t errorstate = HAL_SD_ERROR_NONE;
        uint32_t tickstart = HAL_GetTick();

        /* Check for SDMMC interrupt flags */
        if (__HAL_SD_GET_FLAG(hsd, SDMMC_IT_DATAEND) != RESET) {
            __HAL_SD_CLEAR_FLAG(hsd, SDMMC_FLAG_DATAEND);

            __HAL_SD_DISABLE_IT(hsd, SDMMC_IT_DCRCFAIL | SDMMC_IT_DTIMEOUT 
                                | SDMMC_IT_RXOVERR\
                                | SDMMC_IT_TXUNDERR | SDMMC_IT_DATAEND | 
                                SDMMC_FLAG_IDMATE\
                                | SDMMC_FLAG_TXFIFOHE | 
                                SDMMC_FLAG_RXFIFOHF);

            if ((hsd->Context & SD_CONTEXT_DMA) != RESET) {

                __HAL_SD_CLEAR_FLAG(hsd, SDMMC_STATIC_FLAGS);
                __SDMMC_CMDTRANS_DISABLE( hsd->Instance);

                hsd->Instance->DLEN = 0;
                hsd->Instance->DCTRL = 0;
                hsd->Instance->IDMACTRL = SDMMC_DISABLE_IDMA ;

                /* Stop Transfer for Write Multi blocks or Read Multi 
                blocks */
                if (((hsd->Context & SD_CONTEXT_READ_MULTIPLE_BLOCK) != 
                    RESET) || ((hsd->Context & 
                    SD_CONTEXT_WRITE_MULTIPLE_BLOCK) != RESET)) {
                    errorstate = SDMMC_CmdStopTransfer(hsd->Instance);
                    if (errorstate != HAL_SD_ERROR_NONE) {
                        hsd->ErrorCode = errorstate;
                        HAL_SD_ErrorCallback(hsd);
                    }
                }

                if (((hsd->Context & SD_CONTEXT_WRITE_SINGLE_BLOCK) != 
                    RESET) || ((hsd->Context & 
                    SD_CONTEXT_WRITE_MULTIPLE_BLOCK) != RESET)) {
                    while ((HAL_SD_GetCardState(hsd) != 
                        HAL_SD_CARD_TRANSFER) && ((HAL_GetTick() - 
                        tickstart) <=  SDMMC_MAX_TRIAL)) {
                        /* Wait until SD CARD Status goes to TRANSFER 
                        STATE or Timeout */
                    }

                    HAL_SD_TxCpltCallback(hsd);
                }
                if (((hsd->Context & SD_CONTEXT_READ_SINGLE_BLOCK) != 
                    RESET) || ((hsd->Context & 
                    SD_CONTEXT_READ_MULTIPLE_BLOCK) != RESET)) {
                    HAL_SD_RxCpltCallback(hsd);
                }


                hsd->State = HAL_SD_STATE_READY;

            }

            if ((hsd->Context & SD_CONTEXT_IT) != RESET) {
                if ((hsd->Context & SD_CONTEXT_READ_MULTIPLE_BLOCK) != 
                    RESET) {
                    errorstate = SDMMC_CmdStopTransfer(hsd->Instance);
                    if (errorstate != HAL_SD_ERROR_NONE) {
                        hsd->ErrorCode = errorstate;
                        HAL_SD_ErrorCallback(hsd);
                    }

                    hsd->State = HAL_SD_STATE_READY;

                    HAL_SD_RxCpltCallback(hsd);
                } else if (((hsd->Context & SD_CONTEXT_WRITE_SINGLE_BLOCK) 
                        != RESET) || ((hsd->Context & 
                        SD_CONTEXT_WRITE_MULTIPLE_BLOCK) != RESET)) {
                    if ((hsd->Context & SD_CONTEXT_WRITE_MULTIPLE_BLOCK) 
                        != RESET) {
                        errorstate = SDMMC_CmdStopTransfer(hsd->Instance);
                        if (errorstate != HAL_SD_ERROR_NONE) {
                            hsd->ErrorCode = errorstate;
                            HAL_SD_ErrorCallback(hsd);
                        }
                    }


                    /* Clear all the static flags */
                    __HAL_SD_CLEAR_FLAG(hsd, SDMMC_STATIC_FLAGS);

                    hsd->State = HAL_SD_STATE_READY;

                    HAL_SD_TxCpltCallback(hsd);
                }
            }
        }
    
        else if (__HAL_SD_GET_FLAG(hsd, SDMMC_IT_TXFIFOHE) != RESET) {
            __HAL_SD_CLEAR_FLAG(hsd, SDMMC_FLAG_TXFIFOHE);
    
            SD_Write_IT(hsd);
        }
    
        else if (__HAL_SD_GET_FLAG(hsd, SDMMC_IT_RXFIFOHF) != RESET) {
            __HAL_SD_CLEAR_FLAG(hsd, SDMMC_FLAG_RXFIFOHF);

            SD_Read_IT(hsd);
        }

        else if (__HAL_SD_GET_FLAG(hsd, SDMMC_IT_DCRCFAIL) != RESET) {
            __HAL_SD_CLEAR_FLAG(hsd, SDMMC_FLAG_DCRCFAIL);

            __HAL_SD_DISABLE_IT(hsd, SDMMC_IT_DCRCFAIL);

            HAL_SD_ErrorCallback(hsd);
        }

        else if (__HAL_SD_GET_FLAG(hsd, SDMMC_IT_DTIMEOUT) != RESET) {
            __HAL_SD_CLEAR_FLAG(hsd, SDMMC_FLAG_DTIMEOUT);

            __HAL_SD_DISABLE_IT(hsd, SDMMC_IT_DTIMEOUT);

            HAL_SD_ErrorCallback(hsd);
        }

        else if (__HAL_SD_GET_FLAG(hsd, SDMMC_IT_RXOVERR) != RESET) {
            __HAL_SD_CLEAR_FLAG(hsd, SDMMC_FLAG_RXOVERR);

            __HAL_SD_DISABLE_IT(hsd, SDMMC_IT_RXOVERR);

            HAL_SD_ErrorCallback(hsd);
        }

        else if (__HAL_SD_GET_FLAG(hsd, SDMMC_IT_TXUNDERR) != RESET) {
            __HAL_SD_CLEAR_FLAG(hsd, SDMMC_FLAG_TXUNDERR);

            __HAL_SD_DISABLE_IT(hsd, SDMMC_IT_TXUNDERR);

            HAL_SD_ErrorCallback(hsd);
        }

        else if (__HAL_SD_GET_FLAG(hsd, SDMMC_IT_IDMATE) != RESET) {
            __SDMMC_CMDTRANS_DISABLE( hsd->Instance);
            __HAL_SD_CLEAR_FLAG(hsd, SDMMC_FLAG_IDMATE);

            __HAL_SD_DISABLE_IT(hsd, SDMMC_IT_IDMATE);

            HAL_SD_ErrorCallback(hsd);
        } else if (__HAL_SD_GET_FLAG(hsd, SDMMC_IT_IDMABTC) != RESET) {
            if (READ_BIT(hsd->Instance->IDMACTRL, SDMMC_IDMA_IDMABACT) == 
                SD_DMA_BUFFER0) {
                /* Current buffer is buffer0, Transfer complete for 
                buffer1 */
                if ((hsd->Context & SD_CONTEXT_WRITE_MULTIPLE_BLOCK) != 
                    RESET) {
                    HAL_SDEx_Write_DMADoubleBuffer1CpltCallback(hsd);
                } else { /* SD_CONTEXT_READ_MULTIPLE_BLOCK */
                    HAL_SDEx_Read_DMADoubleBuffer1CpltCallback(hsd);
                }
            } else { /* SD_DMA_BUFFER1 */
                /* Current buffer is buffer1, Transfer complete for 
                buffer0 */
                if ((hsd->Context & SD_CONTEXT_WRITE_MULTIPLE_BLOCK) != 
                    RESET) {
                    HAL_SDEx_Write_DMADoubleBuffer0CpltCallback(hsd);
                } else { /* SD_CONTEXT_READ_MULTIPLE_BLOCK */
                    HAL_SDEx_Read_DMADoubleBuffer0CpltCallback(hsd);
                }
            }
            __HAL_SD_CLEAR_FLAG(hsd, SDMMC_IT_IDMABTC);
        }
    } 

SDMMC中断服务函数HAL_SD_IRQHandler通过多个if判断语句分辨中断源，并对传输错误标志变量TransferError赋值以指示当前传输状态，每个状态都有一个中断回调函数供客户编写用户代码。最后禁用SDMMC中断。

代码清单 SDMMC1发送完成回调函数（文件bsp_sdio_sd.c）

.. code-block:: c

    void HAL_SD_TxCpltCallback(SD_HandleTypeDef *hsd)
    {
        TX_Flag=1; 
    }

代码清单 SDMMC1接受完成回调函数（文件bsp_sdio_sd.c）

.. code-block:: c

    void HAL_SD_RxCpltCallback(SD_HandleTypeDef *hsd)
    {
        SCB_InvalidateDCache_by_Addr((uint32_t*)Buffer_Block_Rx, 
    MULTI_BUFFER_SIZE/4);
        RX_Flag=1;
    }

SDMMC1接受完成回调函数，当DMA数据传输完成时，标志位Rx_Flag置1。同时调用HAL_SD_RxcpltCallback函数，使数据存放地址相应的DCache失效。

至此，我们已经介绍了SD卡初始化、SD卡数据操作的基础功能以及SDIO相关中断服务函数内容，很多时候这些函数已经足够我们使用了。接下来我们就编写一些简单的测试程序验证移植的正确性。

测试函数
''''''''

测试SD卡部分的函数是我们自己编写的，存放在sdio_test.c文件中。

SD卡测试函数
==============

代码清单 35‑15 SD_Test

.. code-block:: c
   :name: 代码清单35_15

    void SD_Test(void)
    {

        LED_BLUE;
    /*------------------------------ SD 初始化 ------------------------ */
    /* SD卡使用SDIO中断及DMA中断接收数据，中断服务程序位于bsp_sdio_sd.c文件尾*/
        if (BSP_SD_Init() != MSD_OK) {
            LED_RED;
        printf("SD卡初始化失败，请确保SD卡已正确接入开发板，或换一张SD卡测试！\n");
        } else {
            printf("SD卡初始化成功！\n");

            LED_BLUE;
            /*擦除测试*/
            SD_EraseTest();

            LED_BLUE;
            /*single block 读写测试*/
            SD_SingleBlockTest();

            LED_BLUE;
            /*muti block 读写测试*/
            SD_MultiBlockTest();
        }
    }

测试程序以开发板上LED灯指示测试结果，同时打印相关测试结果到串口调试助手。测试程序先调用BSP_SD_Init函数完成SD卡初始化，该函数具体代码参考 代码清单35_15_，
如果初始化成功就可以进行数据操作测试。

SD卡擦除测试
=================

代码清单 35‑16 SD_EraseTest

.. code-block:: c
   :name: 代码清单35_16

    void SD_EraseTest(void)
    {
        /*------------------- Block Erase ---------------------------------*/
        if (Status == SD_OK) {
            /* Erase NumberOfBlocks Blocks of WRITE_BL_LEN(512 Bytes) */
            Status = BSP_SD_Erase(0x00, (BLOCK_SIZE * NUMBER_OF_BLOCKS));
        }

    if (Status == SD_OK) { Status = BSP_SD_ReadBlocks_DMA(
            Buffer_MultiBlock_Rx, 0x00, BLOCK_SIZE, NUMBER_OF_BLOCKS);
        }

        /* Check the correctness of erased blocks */
        if (Status == SD_OK) {
        EraseStatus = eBuffercmp(Buffer_MultiBlock_Rx, MULTI_BUFFER_SIZE/4);
        }

        if (EraseStatus == PASSED) {
            LED_GREEN;
            printf("SD卡擦除测试成功！\n");
        } else {
            LED_RED;
            printf("SD卡擦除测试失败！\n");
            printf("温馨提示：部分SD卡不支持擦除测试，若SD卡能通过下面的single读写测试，即表示SD卡能够正常使用。\n");
        }
    }

SD_EraseTest函数主要编程思路是擦除一定数量的数据块，接着读取已擦除块的数据，把读取到的数据与0xff或者0x00比较，得出擦除结果。

HAL_SD_Erase函数用于擦除指定地址空间，源代码参考 代码清单35_16_，
它接收三个参数，分别是SDMMC的外设管理结构体，擦除空间的起始地址和终止地址。如果HAL _SD_Erase函数返回正确，表示擦除成功则执行数据块读取；如果HAL _SD_Erase函数返回错误，表示SD卡擦除失败，并不是所有卡都能擦除成功的，部分卡虽然擦除失败，但数据读写操作也是可以正常执行的。这里使用Wait_SDCARD_Ready函数，用来等待SDMMC擦除完成。

单块读写测试
==================

代码清单 35‑17 SD_SingleBlockTest函数（文件bsp_sdio_sd.c）

.. code-block:: c
   :name: 代码清单35_17

    void SD_SingleBlockTest(void)
    {
        HAL_StatusTypeDef Status =  HAL_OK;
        HAL_StatusTypeDef TransferStatus1 = HAL_ERROR;
        /* Fill the buffer to send */
        Fill_Buffer(Buffer_Block_Tx, BLOCK_SIZE/4, 0);
        SCB_CleanDCache_by_Addr((uint32_t*)Buffer_Block_Tx, BLOCK_SIZE/4);
        if (Status == HAL_OK) {
            /* Write block of 512 bytes on address 0 */
            Status = HAL_SD_WriteBlocks_DMA(&uSdHandle, Buffer_Block_Tx, 0x00, 1);
            while (TX_Flag == 0);
        }
        /* Fill the buffer to reception */
        Fill_Buffer(Buffer_Block_Rx, BLOCK_SIZE/4, 0);
        SCB_CleanDCache_by_Addr((uint32_t*)Buffer_Block_Rx, BLOCK_SIZE/4);
        if (Status == HAL_OK) {
            /* Read block of 512 bytes from address 0 */
            Status = HAL_SD_ReadBlocks_DMA(&uSdHandle, Buffer_Block_Rx,0, 1);
            while (RX_Flag == 0);
        }
        if (Status == HAL_OK) {
            TransferStatus1 = Buffercmp(Buffer_Block_Tx, Buffer_Block_Rx, BLOCK_SIZE/4);
        }
        if (TransferStatus1 == HAL_OK) {
            LED_GREEN;
            printf("Single block 测试成功！\n");
        } else {
            LED_RED;
            printf("Single block 测试失败，请确保SD卡正确接入开发板，或换一张SD卡测试！\n");
        }
    } 

SD_SingleBlockTest函数主要编程思想是首先填充一个块大小的存储器，通过写入操作把数据写入到SD卡内，然后通过读取操作读取数据到另外的存储器，然后在对比存储器内容得出读写操作是否正确。

SD_SingleBlockTest函数一开始调用Fill_Buffer函数用于填充存储器内容，
它只是简单实用for循环赋值方法给存储区填充数据，它有三个形参，分别为存储区指针、
填充字节数和起始数选择，这里的起始数选择参数对本测试没有实际意义。
BSP_SD_WriteBlocks_DMA函数和BSP_SD_ReadBlocks_DMA函数分别执行数据写入和读取操作，
具体可以参考 代码清单35_10_ 和 代码清单35_12_。
Buffercmp函数用于比较两个存储区内容是否完全相等，它有三个形参，分别为第一个存储区指针、第二个存储区指针和存储器长度，该函数只是循环比较两个存储区对应位置的两个数据是否相等，只有发现存在不相等就报错退出。

SD_MultiBlockTest函数与SD_SingleBlockTest函数执行过程类似，这里就不做详细分析。

主函数
==========

代码清单 35‑18 main函数

.. code-block:: c
   :name: 代码清单35_18

    int main(void)
    {
        /* 系统时钟初始化成400MHz */
        SystemClock_Config();
        CPU_CACHE_Enable();
        LED_GPIO_Config();
        LED_BLUE;
        /* 初始化USART1 配置模式为 115200 8-N-1 */
        DEBUG_USART_Config();
        /* 初始化独立按键 */
        Key_GPIO_Config();
        printf("\r\n欢迎使用秉火  STM32 H743 开发板。\r\n");
        printf("在开始进行SD卡基本测试前，请给开发板插入32G以内的SD卡\r\n");
        printf("本程序会对SD卡进行非文件系统方式读写，会删除SD卡的文件系统\r\n");
        printf("实验后可通过电脑格式化或使用SD卡文件系统的例程恢复SD卡文件系统\r\n");
        printf("\r\n 但sd卡内的原文件不可恢复，实验前务必备份SD卡内的原文件！！！\r\n");
        printf("\r\n 若已确认，请按开发板的KEY1按键，开始SD卡测试实验....\r\n");

        while (1) {
            if (  Key_Scan(KEY1_GPIO_PORT,KEY1_PIN) == KEY_ON) {
                printf("\r\n开始进行SD卡读写实验\r\n");
                SD_Test();
            }
        }
    }

测试过程中有用到LED灯、独立按键和调试串口，所以需要对这些模块进行初始化配置。在无限循环中不断检测按键状态，如果有被按下就执行SD卡测试函数。

下载验证
^^^^^^^^

把Micro SD卡插入到开发板右侧的卡槽内，使用USB线连接开发板上的“USB TO
UART”接口到电脑，电脑端配置好串口调试助手参数。编译实验程序并下载到开发板上，程序运行后在串口调试助手可接收到开发板发过来的提示信息，按下开发板左下边沿的K1按键，开始执行SD卡测试，测试结果在串口调试助手可观察到，板子上LED灯也可以指示测试结果。
