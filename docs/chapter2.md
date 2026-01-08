# 第2章：数据包处理实例

![image_chapter2.png](image_chapter2.png)

上图为完整的数据报文处理流程图。报文从左侧进入交换机硬件 。接着被解析成多个头部字段，包括以太网，ipv4等。通过元数据metadata获取报文头部所携带的信息，接着进入到线程thread内处理报文，处理过程为将元数据和头部字段通过多个block（包括table+function以及control函数），处理完成后重新组合字段构建数据包发送，元数据被丢弃。

在深入了解语法细节之前，我们先通过一个经典的“数据包镜像”实例，来完整体验一次 NPC++ 的开发流程。本章将采用**代码拆解**的方式，一步步剖析如何在数据平面实现从报文接收、解析、查表到转发的完整逻辑。

## 2.1 简单实例：NPC++镜像功能实现详解

### 2.1.1 功能概述

本教程讲解如何使用NPC++语言实现**网络报文镜像（Mirroring）**功能。镜像功能常用于：
- 流量监控与分析
- 安全审计
- 故障排查
- 网络性能监控

**总体思路**：将接收到的原始报文复制一份，修改副本报文的目标地址，从指定端口发送出去，实现”一收多发”的效果。

### 2.1.2 代码结构解析

#### 数据结构定义（Header Definition）

```cpp
typedef uint<48> MAC_ADDR_T;
typedef uint<32> IPV4_ADDR_T;
#define ETHERTYPE_IPV4 0x0800

struct ETHER_HDR_S {
    MAC_ADDR_T DstMac;
    MAC_ADDR_T SrcMac;
    uint<16> EtherType;
};

struct IPV4_HDR_S {
    uint<4>  Version;
    uint<4>  Ihl;
    uint<8>  Tos;
    uint<16> TotalLen;
    uint<16> Id;
    uint<1>  FlagR;
    uint<1>  FlagDF;
    uint<1>  FlagMF;
    uint<13> FrgOffset;
    uint<8>  Ttl;
    uint<8>  Protocal;
    uint<16> Checksum;
    IPV4_ADDR_T Sip;
    IPV4_ADDR_T Dip;
};
NPCHeader <ETHER_HDR_S>  ethHdr;
NPCHeader <IPV4_HDR_S>   ipv4Hdr;
```

**要点说明**：
1. `uint<N>`表示N位的无符号整数，如`uint<48>`对应6字节MAC地址
2. 使用`struct`定义报文头部结构，与协议格式一一对应
3. `NPCHeader<>`将结构体实例化为可操作的报文头部对象

#### 线性表定义（Linear Table）

```cpp
// 定义线性表返回的数据结构
#define TBL_SIZE_8BIT 1<<8  //表大小
struct MIRROR_CFG_S {
    uint<8>  MirrorPort; // 镜像出接口
    uint<32> DestIp;     // 镜像报文修改后的目的IP
    uint<48> DestMac;   // 镜像报文修改后的目的MAC
};

table MIRROR_LITBL() {
    key = {
        // 使用报文源MAC的低8位作为线性表索引
        ethHdr.SrcMac[7:0]: index;
    }
    STATUS_T     _status();
    MIRROR_CFG_S _lookup();

    actions = {
        _lookup: MirrorReadProc(); // 绑定查找后的处理函数
    }
    size = TBL_SIZE_8BIT;
}
```

**核心概念**：
1. **线性表（Linear Table）**：通过索引直接访问的表结构，查找时间复杂度O(1)
2. **键（Key）**：使用源MAC地址的低8位作为索引值（共256个表项）
3. **动作（Actions）**：查表成功后执行的处理函数
4. **表项结构**：每个表项包含镜像端口、目的IP、目的MAC三个字段

#### 镜像处理函数（Mirror Processing）

```cpp
struct NPC_INTRINSIC_OUTPUT_METADATA_S {
    uint<9>  OutSubChan;  // 报文在芯片上的出口接口 ID（非面板端口号）
    uint<16> Sqid;        // 用户队列 ID
    uint<1>  DropFlag;    // 丢包标志
    uint<4>  Qindex;      // 队列优先级（决定调度队列）
    uint<2>  Color;       // 报文丢弃优先级（队列满时红色报文优先丢弃）
};
NPC_INTRINSIC_OUTPUT_METADATA_S outputs; //声明输出元数据
void MirrorReadProc(STATUS_T sta, MIRROR_CFG_S rsp) {
    if (sta == 0) { // 查表成功（HIT状态）
        // --- A. 配置原始报文 ---
        outputs.OutSubChan = 0;

        // --- B. 复制报文 ---
        npc_pkt_replicate();

        // --- C. 配置副本（镜像）报文 ---
        outputs.OutSubChan = rsp.MirrorPort;
        ipv4Hdr.Dip = rsp.DestIp;
        ethHdr.DstMac = rsp.DestMac;

        // 发送副本报文
        npc_pkt_send();
        npc_exit();
    }
}
MIRROR_LITBL mirr; //声明一个全局的mirror表
```

**关键技术点**：

1. **报文复制**：
    
    ```cpp
    npc_pkt_replicate();
    ```
    
    - 复制当前报文，生成副本
    - 复制前的操作作用于原始报文
    - 复制后当前线程继续处理副本报文
2. **通道控制**：
    
    ```cpp
    npc_intrinsic_metadata.OutSubChan = rsp.MirrorPort;
    ```
    
    - 控制报文从哪个物理端口发送
    - 原始报文发往端口0，副本发往镜像端口
3. **报文修改**：
    
    ```cpp
    ipv4Hdr.Dip = rsp.DestIp;
    ethHdr.DstMac = rsp.DestMac;
    ```
    
    - 修改副本报文的目的地址
    - 原始报文保持原样转发

#### 解析器与控制流

```cpp
// 4. 解析器 (Parser)
@parser void ParseEthernet() {
    _extract(ethHdr);
    if (ethHdr.EtherType == 0x0800) {
        _extract(ipv4Hdr);
    }
    return accept;
}

// 5. 控制流 (Control Flow)
@control void ControlFlow() {
    // 执行线性表查找
    MIRROR_LITBL._lookup();

    // 原始报文默认后续处理
    npc_pkt_add_sys_hdr();
    npc_pkt_deparser();
    npc_pkt_send();
    npc_exit();
}

NPCProgram(ParseEthernet(), ControlFlow()) main;
```

**处理流程**：
1. **解析阶段**：提取以太网头和IPv4头
2. **查表阶段**：根据源MAC查镜像配置表
3. **镜像处理**：如果命中表项，复制并修改报文
4. **转发阶段**：原始报文和镜像报文分别发送

### 2.1.3 关键API详解

#### 报文操作API

```cpp
// 复制报文，生成副本
npc_pkt_replicate();

// 发送当前处理的报文
npc_pkt_send();

// 添加系统头部
npc_pkt_add_sys_hdr();

// 反解析，将头部组装回报文
npc_pkt_deparser();
```

#### 元数据操作

```cpp
// 设置报文出端口
npc_intrinsic_metadata.OutSubChan = port_num;
```

## **2.2 复杂实例：构建基础三层转发器**

**场景假设**：

我们需要在一台拥有8个以太网端口的交换机上实现基础的 IPv4 路由转发功能。

**处理逻辑**：

1. **解析**：提取报文的 Ethernet 和 IPv4 头部。
2. **检查**：检查报文的目的 MAC 是否匹配本机（IPAT表），若不匹配则丢弃。
3. **路由**：根据目的 IP 查找路由表（FIB表），获取出端口和下一跳信息。
4. **封装**：修改报文的源/目的 MAC 地址。
5. **转发**：将报文从指定出端口发送出去。

### 第一步：定义报文头格式 (Header Definition)

数据面编程的第一步是告诉芯片“报文长什么样”。我们利用 struct 和 typedef 定义数据结构。

> 注意：NPC++ 使用 uint<N> 来精确控制位宽，例如 uint<48> 对应 6 字节的 MAC 地址。
> 

```cpp
//首先是头文件模块
//注意以太网地址是48字节，ipv4地址是32字节
//利用typedef 自定义名称 IPV4_ADDR_T
typedef uint<48> MAC_ADDR_T;
typedef uint<32> IPV4_ADDR_T;
struct ETHER_HDR_S {
    MAC_ADDR_T DstMac; 
    MAC_ADDR_T SrcMac;
    uint<16> EtherType;
};
#define ETHERTYPE_IPV4 0x0800
struct IPV4_HDR_S {
    uint<4>  Version;
    uint<4>  Ihl;
    uint<8>  Tos;
    uint<16> TotalLen;
    uint<16> Id;
    uint<1>  FlagR;
    uint<1>  FlagDF;
    uint<1>  FlagMF;
    uint<13> FrgOffset;
    uint<8>  Ttl;
    uint<8>  Protocal;
    uint<16> Checksum;
    IPV4_ADDR_T Sip;
    IPV4_ADDR_T Dip;
};
NPCHeader <ETHER_HDR_S>  ethHdr;  //通过NPCHeader把声明的结构体具体化一个实例ethHdr
NPCHeader <IPV4_HDR_S>   ipv4Hdr;
```

### 第二步：编写解析器 (Parser)

解析器是一个有限状态机，负责从比特流中提取字段。@parser 关键字标识了这是解析逻辑。

```cpp
//其次是解析器模块，解析以太头和ipv4头，属于两个C函数，注意前面表明该功能是@parser

@parser void ParseEthernet()  //解析提取以太网头部
{
    _extract(ethHdr);
    return switch(ethHdr.EtherType)
    {
        case ETHERTYPE_IPV4:
            ParseIpv4;
        default:
            accept;
    }
}

@parser void ParseIpv4() //解析提取ipv4头部
{
    _extract(ipv4Hdr);
    return accept;
}
```

### 第三步：定义匹配-动作表 (Tables & Actions)

这是数据面的核心业务逻辑。我们需要定义三张表：

1. **IPAT (Ingress Port Associate Table)**：入端口关联表，用于验证目的 MAC。
2. **FIB (Forwarding Information Base)**：路由表，用于查找目的 IP 对应的出端口。
3. **EPAT (Egress Port Associate Table)**：出端口关联表，用于获取发包时的源 MAC。

```cpp
//接着声明匹配表模块，当数据包内的某些字段与表项匹配，则进行相应的action
#define TBL_SIZE_8BIT 8
#define FIBV4_SIZE 8
table IPAT_TBL()
{
    key = {
        g_ctrlMd.InChnnl: exact;
    }
    IPAT_RSP_S _lookup();
    actions = {
        _lookup: no_action; //不执行额外操作
    }
    size = TBL_SIZE_8BIT;
}

table FIBV4_TBL()
{
    key = {
        g_flowKey.VrfId: lpm;
        g_flowKey.Dip: lpm;
    }
    STATUS_T _status();
    LPM_RSP_S _lookup();
    actions = {
        _lookup: Fibv4Proc(); //匹配成功后执行相应的action
    }
    size = FIBV4_SIZE;
}
//Fibv4Proc是上面FIBV_TBL对应表项匹配成功后执行的action操作
void Fibv4Proc(STATUS_T sta, LPM_RSP_S rsp)
{
    g_ctrlMd.OpStatus = sta;
    g_ctrlMd.LinearIndex = rsp.idx;
}

table EPAT_TBL()
{
    key = {
        g_ctrlMd.OutChnnl: exact;
    }
    EPAT_RSP_S _lookup();
    actions = {
        _lookup: no_action;
    }
    size = TBL_SIZE_8BIT;
}
```

### 第四步：构建控制流

控制流将上述组件串联起来，形成完整的处理管线。

```cpp
//最后是控制流模块，解析完数据包后，通过如下流程实现整体上的数据包匹配和发送操作
//声明一些全局变量
struct CTRL_META_DATA_S{
		uint<8> OpStatus;//状态
		uint<8> LinearIndex; //索引
		uint<8> InChnnl; //入端口
		uint<8> OutChnnl; //出端口
};
struct FLOW_KEY_S{
		uint<8> VrfId; //找到匹配的索引
		uint<32> Dip; //目的ip
};
struct IPAT_RSP_S{
		uint<48> Mac; //目的MAC地址
		uint<8> VrfId; //找到匹配的索引
};
struct LPM_RSP_S{
    uint<8> idx; //找到匹配的索引
		uint<48> DstMac; //目的MAC地址	
};
struct EPAT_RSP_S{
    uint<8> idx; //找到匹配的索引
		uint<48> Mac; //目的MAC地址	
};

CTRL_META_DATA_S   g_ctrlMd;
FLOW_KEY_S  g_flowKey;

IPAT_TBL ipat;  //这个是声明的table
FIBV4_TBL fibv4; // 这个是声明的table
EPAT_TBL epat; //这个是声明的table

@control void ControlFlow()
{
    g_ctrlMd.InChnnl = npc_intrinsic_metadata.InSubChan[7:0];
    //匹配表项，查找目的mac地址，如果mac地址不对，则丢弃
    //IPAT_RSP_S 这个代表要写入的内容，即匹配成功后将内容写入到对应变量，后续带RSP_S的均为写入
    IPAT_RSP_S ipatRsp = ipat._lookup();
    if (ipatRsp.Mac[47:32] != ethHdr.DstMac[47:32] || ipatRsp.Mac[31:0] != ethHdr.DstMac[31:0]) {
        npc_pkt_drop();
        npc_exit();
    }
    //获取匹配到的VrfId和Dip，接着匹配
    g_flowKey.VrfId = ipatRsp.VrfId;
    g_flowKey.Dip = ipv4Hdr.Dip;
    
    //如果匹配成功，获取出端口 
    LPM_RSP_S rsp = fibv4._lookup();
    g_ctrlMd.OutChnnl = rsp.idx;
    
    //依照出端口获取到对应的mac地址
    EPAT_RSP_S epatRsp = epat._lookup();
    
    //重新封装数据包的mac地址
    ethHdr.DstMac = rsp.DstMac;
    ethHdr.SrcMac = epatRsp.Mac;
    //发送数据包
    npc_pkt_add_sys_hdr();
    npc_pkt_deparser();
    npc_pkt_send();
    npc_exit();
}

NPCProgram(ParseEthernet(), ControlFlow()) main;
```

## 2.3 编译与工具链

完成代码编写后，我们需要使用华为提供的编译器 icomp 将 NPC++ 源码编译为硬件可执行的二进制文件。

### 1. 工具安装

确保 icomp 编译器文件已上传至开发环境（如 Ubuntu 系统）。

### 2. 编译命令

在终端中执行以下命令进行赋权和编译：

```jsx
# 赋予编译器执行权限
chmod 777 icomp 

# 编译 npc_test.npc 文件 (假设源码文件名为 npc_test.npc)
# 输出结果通常包含二进制固件及相关日志
./icomp npc_test.npc
```

## 2.4 测试环境搭建

（此处预留测试代码位置，通常涉及使用发包工具如 Scapy 或 TCPreplay 向模拟端口发送数据包，并验证出端口捕获的报文 MAC 地址是否已被正确修改。）