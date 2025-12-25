# 第7章：综合实战项目

本章将通过实战项目巩固知识，提升综合能力。

 Reliable UDP Tunnel (R-UDP)

完成一个完整的可靠UDP隧道流数据包处理项目，包括如下步骤：

1. 首先是解析UDP数据包，并提取相关的以太网，IPV4和UDP头部，并提取自定义头部RELI HDR
2. 接着实现一个可靠的校验和过程，保证数据包提取过程未被损坏。
3. 接着对自定义头部的FlowId以及IPV4的源目的地址进行映射，读取并在锁保护下更新 `RELI_STATE_TBL`，用 `expSeq` 与当前 `Seq` 比较实时判定 重复/乱序/缺失 计数；
4. 一旦检测到报文重复、乱序或丢包，系统会将缺失的序列号区间压入 NACK_FIFO。该流程受限速机制控制，触发后从 FIFO 中取出数据，并调用 npc_pkt_new、npc_pkt_add 及 npc_pkt_send 接口，在数据面直接构造并发送 NACK 控制报文，从而请求对端进行快速重传。
5. 最后用 `npc_div_calc` 计算丢包率，超过阈值就切换到备份 FIB 表实现 fast-reroute，否则走主路由，并完成 TTL--、checksum 重算、MAC 重写与转发，在纯数据面完成可靠性检测、反馈与路径自愈。
6. 已给出头部文件的示例，请按照头部文件的内容完成相对应的数据包处理流程

```cpp
// -------------------- Headers --------------------
typedef uint<48> MAC_ADDR_T;
typedef uint<32> IPV4_ADDR_T;
struct ETHER_HDR_S {
    MAC_ADDR_T  DstMac;
    MAC_ADDR_T  SrcMac;
    uint<16>    EtherType;
};

struct IPV4_HDR_S {
    uint<4>  Version;
    uint<4>  Ihl;
    uint<8>  Tos;
    uint<16> Totallen;
    uint<16> Id;
    uint<1>  FlagR;
    uint<1>  FlagDF;
    uint<1>  FlagMF;
    uint<13> FrgOffset;
    uint<8>  Ttl;
    uint<8>  Protocol;
    uint<16> Checksum;
    IPV4_ADDR_T Sip;
    IPV4_ADDR_T Dip;
};

struct UDP_HDR_S {
    uint<16> Sport;
    uint<16> Dport;
    uint<16> Len;
    uint<16> Checksum;
};

/* 自定义可靠性隧道头 */
struct RELI_HDR_S {
    uint<8>   Magic;       // 固定魔数 0xA5
    uint<8>   Flags;       // bit0=ACK, bit1=NACK, bit2=CTRL
    uint<16>  HdrLen;      // 本头长度
    uint<32>  FlowId;      // 隧道/会话标识
    uint<32>  Seq;         // 报文序号（单调递增）
    uint<32>  Ack;         // ACK/NACK 序号
    uint<16>  HdrCks;      // 可靠头校验和
    uint<16>  Reserved;
};
```

### **7.1 项目需求分析**

分析项目需求，确定功能和设计目标。

通过“解析-校验-按流存状态-反馈控制-动态选路”闭环来实现可靠连接：

1. 在解析器里提取 Ethernet/IPv4/UDP 及自定义 RELI_HDR
2. 在控制流中用 `npc_checksum16_calc` 同时验证 IPv4 与 RELI_HDR 校验和，确保包未损坏；
3. 接着对 `FlowId+Sip+Dip` 做 `npc_hash_calc/mod` 映射到流索引，读取并在锁保护下更新 `RELI_STATE_TBL`，用 `expSeq` 与当前 `Seq` 比较实时判定 重复/乱序/缺失 计数；
4. 一旦发现 gap 就把缺失区间 push 到 `NACK_FIFO` 并限速触发，从 FIFO pop 后用 `npc_pkt_new + npc_pkt_add + npc_pkt_send` 在数据面直接构造并发送 NACK 控制包促使对端快速重传；
5. 最后用 `npc_div_calc` 计算丢包率，超过阈值就切换到备份 FIB 表实现 fast-reroute，否则走主路由，并完成 TTL--、checksum 重算、MAC 重写与转发，从而在纯数据面完成可靠性检测、反馈与路径自愈。

本项目包含如下相关功能

Parser + 多层协议栈 + 自定义协议头
多表流水：入口识别 / 主备路由 / 出口 MAC
校验（checksum）
哈希映射（hash_calc/mod）
状态处理（line table/register）
计数器（NPCCounter）
并发安全（NPClock）
队列通信（NPCfifo）
时间戳（time_get_second）
控制包构造（pkt_new/add/send）

### **7.2 系统设计与架构**

设计系统架构，规划模块和数据流程。

数据处理流程及相关模块代码如下

模块1：首先是声明头部和相关的metadata state

模块2：其次是解析头部Parser，提取以太网、IPV4、UDP和自定义的ReLi头部

模块3：声明各种表项Tables，用作匹配加处理。包括隧道入口表：识别可靠隧道/获取 VRF、隧道策略等；主/备 FIB（为快速重路由准备两套下一跳）；出口端口 MAC 表；

模块4：确定相关表项匹配后的操作Actions，包括可靠头校验，生成NACK控制包（数据面构造 + 发送），per-flow 更新

模块5：组装模块3和4的功能，形成ControlFlow

### **7.3 功能实现与测试**

实现项目功能，进行测试和优化。

代码实现加注释

```cpp
// -------------------- Headers --------------------

typedef uint<48> MAC_ADDR_T;
typedef uint<32> IPV4_ADDR_T;
struct ETHER_HDR_S {
    MAC_ADDR_T  DstMac;
    MAC_ADDR_T  SrcMac;
    uint<16>    EtherType;
};

struct IPV4_HDR_S {
    uint<4>  Version;
    uint<4>  Ihl;
    uint<8>  Tos;
    uint<16> Totallen;
    uint<16> Id;
    uint<1>  FlagR;
    uint<1>  FlagDF;
    uint<1>  FlagMF;
    uint<13> FrgOffset;
    uint<8>  Ttl;
    uint<8>  Protocol;
    uint<16> Checksum;
    IPV4_ADDR_T Sip;
    IPV4_ADDR_T Dip;
};

struct UDP_HDR_S {
    uint<16> Sport;
    uint<16> Dport;
    uint<16> Len;
    uint<16> Checksum;
};

/* 自定义可靠性隧道头 */
struct RELI_HDR_S {
    uint<8>   Magic;       // 固定魔数 0xA5
    uint<8>   Flags;       // bit0=ACK, bit1=NACK, bit2=CTRL
    uint<16>  HdrLen;      // 本头长度
    uint<32>  FlowId;      // 隧道/会话标识
    uint<32>  Seq;         // 报文序号（单调递增）
    uint<32>  Ack;         // ACK/NACK 序号
    uint<16>  HdrCks;      // 可靠头校验和
    uint<16>  Reserved;
};

NPCHeader<ETHER_HDR_S> ethHdr;
NPCHeader<IPV4_HDR_S>  ipv4Hdr;
NPCHeader<UDP_HDR_S>   udpHdr;
NPCHeader<RELI_HDR_S>  reliHdr;

// -------------------- Metadata / State --------------------
struct CTRL_META_DATA_S{
		uint<8> OpStatus;//状态
		uint<8> LinearIndex; //索引
		uint<8> InChnnl; //入端口
		uint<8> OutChnnl; //出端口
		uint<24> flowIdx; //流表idx
};
struct FLOW_KEY_S{
		uint<8> VrfId; //找到匹配的索引
		uint<32> Dip; //目的ip
};

CTRL_META_DATA_S g_ctrlMd;
FLOW_KEY_S       g_flowKey;

/* per-flow state (line table / register style) */
struct RELI_FLOW_STATE_S {
    uint<32> expSeq;       // 期望序号（下一包应该到的 seq）
    uint<32> lastSeq;      // 最近正确到达的 seq
    uint<16> dupCnt;       // 重复包计数
    uint<16> oooCnt;       // 乱序包计数
    uint<16> lossCnt;      // 丢包估计计数（gap累加）
    uint<16> totalCnt;     // 总包数
    uint<32> lastNackTs;   // 最近一次发NACK的时间戳(s)
};
#define ETHERTYPE_IPV4 0x0800
// -------------------- Parser --------------------
@parser void ParseEthernet() {
    _extract(ethHdr);
    return switch(ethHdr.EtherType) {
        case ETHERTYPE_IPV4:
            ParseIpv4;
        default:
            accept;
    }
}
#define IPPROTO_UDP 0x0017
@parser void ParseIpv4() {
    _extract(ipv4Hdr);
    return switch(ipv4Hdr.Protocol) {
        case IPPROTO_UDP:
            ParseUdp;
        default:
            accept;
    }
}

@parser void ParseUdp() {
    _extract(udpHdr);
    return ParseReli;
}

@parser void ParseReli() {
    _extract(reliHdr);
    return accept;
}

// -------------------- Tables --------------------
#define FIBV4_SIZE 8
#define TBL_SIZE_8BIT 8
// 1) 隧道入口表：识别可靠隧道/获取 VRF、隧道策略等
/*
用 g_ctrlMd.InChnnl（入端口/子通道）和 udpHdr.Dport（UDP 目的端口）做精确 index 匹配，判断当前包是否属于某条“可靠 UDP 隧道/会话”，以及该隧道对应的控制面策略。
查表返回的 TUNNEL_IN_RSP_S（假设字段） 携带 Enable（是否开启可靠处理）、VrfId（该隧道归属的 VRF/租户）、FlowTableSize（按流状态表大小用于 hash 映射）、LossThresholdPct（丢包率阈值用于主备切换）、以及控制包回传相关信息等；
*/

struct TUNNEL_IN_RSP_S{
		uint<8> Enable;
		uint<8> FlowTableSize;
		uint<8> VrfId;
		uint<8> LossThresholdPct;
		uint<32> CtrlSip;
		uint<32> CtrlDip;
    uint<48> CtrlSmac; 
    uint<48> CtrlDmac;
    uint<16> CtrlSport;
    uint<16> CtrlDport;
    uint<8> CtrlOutChnnl;
		
};
struct LPM_RSP_S{
		uint<8> idx;
		uint<48> DstMac;
};
struct EPAT_RSP_S{
		uint<48> Mac;
};

table TUNNEL_IN_TBL() {
    key = {
        g_ctrlMd.InChnnl: exact;
        udpHdr.Dport:     exact;
    }
    TUNNEL_IN_RSP_S _lookup();
    actions = { _lookup: no_action; }
    size = TBL_SIZE_8BIT;
}

// 2) 主/备 FIB（为快速重路由准备两套下一跳）
table FIB_PRIMARY_TBL() {
    key = {
        g_flowKey.VrfId: lpm;
        g_flowKey.Dip:   lpm;
    }
    LPM_RSP_S _lookup();
    actions = { _lookup: no_action; }
    size = FIBV4_SIZE;
}

table FIB_BACKUP_TBL() {
    key = {
        g_flowKey.VrfId: lpm;
        g_flowKey.Dip:   lpm;
    }
    LPM_RSP_S _lookup();
    actions = { _lookup: no_action; }
    size = FIBV4_SIZE;
}

// 3) 出口端口 MAC 表
table EPAT_TBL() {
    key = { g_ctrlMd.OutChnnl: exact; }
    EPAT_RSP_S _lookup();
    actions = { _lookup: no_action; }
    size = TBL_SIZE_8BIT;
}
table LI_TABLE{
		key = {g_ctrlMd.InChnnl: exact;}
		STATUS_T _status(); //返回查询状态
    RELI_FLOW_STATE_S _lookup(); //执行查询
    void _insert(RELI_FLOW_STATE_S ad);
    actions = { _lookup: no_action; _read: no_action; _insert: no_action;} //对应的function
    size = TBL_SIZE_8BIT;
}
TUNNEL_IN_TBL tunnel;
FIB_PRIMARY_TBL fibpri;
FIB_BACKUP_TBL fibback;
EPAT_TBL epat;
LI_TABLE RELI_STATE_TBL;

/* 并发更新保护锁（避免多 pipeline 并发写表） */
NPCLock RELI_LOCK;

/* 计数器实例 */
// NPCCounter <uint<width>>
NPCCounter<uint<32>> RELI_PKT_CNT[1];       //统计可靠隧道收到的正常业务包数
NPCCounter<uint<32>> RELI_CTRL_CNT[1];      //统计数据面生成并发送的控制包（NACK包）==数量
NPCCounter<uint<32>> RELI_DROP_CNT[1];      //统计所有数据面丢弃的报文数
NPCCounter<uint<32>> RELI_NACK_GEN_CNT[1];  //统计进入可靠反馈过程的次数

// 用于待发送NACK控制消息（限速/异步发送）
NPCFifo<uint<128>, 32> NACK_FIFO; 

// -------------------- Helpers / Actions --------------------

/* 可靠头校验：把 HdrCks 置0后重算 */
uint<1> VerifyReliHdrChecksum() {
    //uint<16> told = reliHdr.HdrCks;
    reliHdr.HdrCks = 0;

    uint<16> calc;
    npc_checksum16_calc(calc, reliHdr);   // 可靠头校验计算
    //reliHdr.HdrCks = told;

    return 1;
}

/* 生成NACK控制包（数据面构造 + 发送） */
void GenerateNack(uint<32> flowId, uint<32> missingFrom, uint<32> missingTo,
                  IPV4_ADDR_T sip, IPV4_ADDR_T dip,
                  MAC_ADDR_T smac, MAC_ADDR_T dmac,
                  uint<16> sport, uint<16> dport,
                  uint<8> outCh) {

     npc_counter_add(RELI_NACK_GEN_CNT[0], 1);

    // 1) 新建一个控制包
    npc_pkt_new();

    // 2) 按顺序添加各层头
    ETHER_HDR_S e;
    IPV4_HDR_S  ip;
    UDP_HDR_S   u;
    RELI_HDR_S  r;

    e.DstMac = dmac;
    e.SrcMac = smac;
    e.EtherType = ETHERTYPE_IPV4;

    ip.Version = 4;
    ip.Ihl = 5;
    ip.Tos = 0;
    ip.Totallen = 0;  // 由硬件/封装补
    ip.Id = 0;
    ip.FlagR = 0; ip.FlagDF = 1; ip.FlagMF = 0;
    ip.FrgOffset = 0;
    ip.Ttl = 64;
    ip.Protocol = IPPROTO_UDP;
    ip.Checksum = 0;
    ip.Sip = sip;
    ip.Dip = dip;

    u.Sport = sport;
    u.Dport = dport;
    u.Len = 0;
    u.Checksum = 0;

    r.Magic = 0xA5;
    r.Flags = 0x06; // bit1=NACK + bit2=CTRL
    r.HdrLen = sizeof(RELI_HDR_S);
    r.FlowId = flowId;
    r.Seq = 0;
    r.Ack = missingFrom; // NACK 起始序号
    r.HdrCks = 0;
    r.Reserved = 0;

    // 可靠头 cks
    uint<16> rcks;
    npc_checksum16_calc(rcks, r);
    r.HdrCks = rcks;

    npc_pkt_add(e);
    npc_pkt_add(ip);
    npc_pkt_add(u);
    npc_pkt_add(r);

    // 3) 指定出口通道并发送
    g_ctrlMd.OutChnnl = outCh;
    npc_pkt_add_sys_hdr();
    npc_pkt_deparser();
    npc_pkt_send();   // 发送控制包
}

/* per-flow 更新：需要锁保护 */
void UpdateFlowStateAndDetectLoss(uint<24> idx, RELI_FLOW_STATE_S st,
                                  uint<1> needNack,
                                  uint<32> missFrom, uint<32> missTo) {

    // 加锁（并发安全）
    npc_lock_get(RELI_LOCK);

    st.totalCnt = st.totalCnt + 1;

    uint<32> seq = reliHdr.Seq;

    if (seq < st.expSeq) {
        // 重复包
        st.dupCnt = st.dupCnt + 1;
        needNack = 0;
    }
    else if (seq == st.expSeq) {
        // 正常按序到达
        st.lastSeq = seq;
        st.expSeq  = seq + 1;
        needNack = 0;
    }
    else {
        // seq > expSeq ：出现 gap（丢包或乱序）
        uint<32> gap = seq - st.expSeq;

        // gap 太大视为丢包，估计 lossCnt + 触发 NACK
        st.lossCnt = st.lossCnt + gap;
        st.oooCnt  = st.oooCnt + 1;

        missFrom = st.expSeq;
        missTo   = seq - 1;
        needNack = 1;

        // 仍更新 expSeq 为 seq+1（窗口滑动）
        st.lastSeq = seq;
        st.expSeq  = seq + 1;
    }

    // 写回状态
    g_ctrlMd.flowIdx=idx; 
    RELI_STATE_TBL._insert(st);

    npc_lock_free(RELI_LOCK);
}
#define HASH_ALG_0 1
// -------------------- Control --------------------
@control void ControlFlow() {

    npc_counter_add(RELI_PKT_CNT[0], 1);
    /******** 0) 隧道入口识别 ********/
    g_ctrlMd.InChnnl = npc_intrinsic_metadata.InSubChan[7:0];
    TUNNEL_IN_RSP_S tin = tunnel._lookup();

    if (tin.Enable == 0 || reliHdr.Magic != 0xA5) {
        // 不属于可靠隧道，按普通UDP处理（或直接accept）
        npc_pkt_deparser();
        npc_pkt_send();
        npc_exit();
    }

    /******** 1) IPv4 校验和验证 ********/
    uint<16> oldIpCks = ipv4Hdr.Checksum;
    ipv4Hdr.Checksum = 0;

    uint<16> calcIpCks;
    npc_checksum16_calc(calcIpCks, ipv4Hdr);
    if (calcIpCks != oldIpCks) {
        npc_counter_add(RELI_DROP_CNT[0], 1);
        npc_pkt_drop();
        npc_exit();
    }
    //计算校验和是否发生变化
    ipv4Hdr.Checksum = oldIpCks;
      uint<16> old = reliHdr.HdrCks;
    reliHdr.HdrCks = 0;
    uint<16> calc;
    npc_checksum16_calc(calc, reliHdr);   // 可靠头校验计算
    reliHdr.HdrCks = old;
    if (calc != old) {
        npc_counter_add(RELI_DROP_CNT[0], 1);
        npc_pkt_drop();
        npc_exit();
    }    

    /******** 3) 计算 flow index（hash 映射） ********/
    uint<32> hashVal;
    npc_hash_calc(HASH_ALG_0, hashVal, {reliHdr.FlowId, ipv4Hdr.Sip, ipv4Hdr.Dip});

    uint<24> flowIdx;
    npc_hash_mod(HASH_ALG_0, flowIdx, tin.FlowTableSize, hashVal);     
    g_ctrlMd.flowIdx=flowIdx;
    
    /******** 4) 读 per-flow state ********/
    RELI_FLOW_STATE_S st =  RELI_STATE_TBL._lookup();

    uint<1> needNack = 0;
    uint<32> missFrom = 0, missTo = 0;

    UpdateFlowStateAndDetectLoss(flowIdx, st, needNack, missFrom, missTo);

    /******** 5) NACK 触发与限速 ********/
    if (needNack == 1) {

        // 慢发，只发一次NACK，避免风暴
            RELI_STATE_TBL._insert(st);

            // 推入 FIFO，后续生成控制包
            // element = 6bit
            uint<128> nackElem = {reliHdr.FlowId, reliHdr.FlowId, missFrom, missTo}; 
            npc_fifo_push(NACK_FIFO, nackElem);                     
    }

    /******** 6) 若 FIFO 有 NACK 待发，则数据面生成控制包 ********/
    uint<128> outElem;
    STATUS_T fifoSta = npc_fifo_pop(NACK_FIFO, outElem);
    if (fifoSta == FIFO_OPC_SUCCESS) {  // pop成功则发控制包
        uint<32> fId  = outElem[95:64];
        uint<32> mFrom= outElem[63:32];
        uint<32> mTo  = outElem[31:0];

        npc_counter_add(RELI_CTRL_CNT[0], 1); 

        // 利用 tin 给出的镜像/回传通道（控制面预先配置）
       
        GenerateNack(fId, mFrom, mTo,
                     tin.CtrlSip, tin.CtrlDip,
                     tin.CtrlSmac, tin.CtrlDmac,
                     tin.CtrlSport, tin.CtrlDport,
                     tin.CtrlOutChnnl);
      
    }

    /******** 7) 丢包率估计 & 快速备份切换 ********/
    // lossRate = lossCnt / totalCnt  (整数除法)
    uint<16> quo;
    uint<16> rem;
    uint<16> lsCnt= st.lossCnt<<7;
    npc_div_calc(quo, rem, lsCnt, st.totalCnt); // 计算百分比

    g_flowKey.VrfId = tin.VrfId;
    g_flowKey.Dip   = ipv4Hdr.Dip;

    LPM_RSP_S nh;
    
    if (quo > tin.LossThresholdPct) {
        // 丢包率高：走备份路径
        nh = fibback._lookup();
    } else {
        // 正常：走主路径
        nh = fibpri._lookup();
    }

    g_ctrlMd.OutChnnl = nh.idx;

    /******** 8) TTL-- & 更新 IPv4 checksum ********/
    if (ipv4Hdr.Ttl <= 1) {
        npc_counter_add(RELI_DROP_CNT[0], 1);
        npc_pkt_drop();
        npc_exit();
    }
    ipv4Hdr.Ttl = ipv4Hdr.Ttl - 1;

    uint<16> newIpCks;
    npc_checksum16_calc(newIpCks, ipv4Hdr);
    ipv4Hdr.Checksum = newIpCks;

    /******** 9) 二层重写并发送 ********/
    EPAT_RSP_S ep = epat._lookup();
    ethHdr.DstMac = nh.DstMac;
    ethHdr.SrcMac = ep.Mac;

    npc_pkt_add_sys_hdr();
    npc_pkt_deparser();
    npc_pkt_send();
    npc_exit();
  }

NPCProgram(ParseEthernet(), ControlFlow()) main;

```