# 第5章：元数据与数据面接口(API)

本章介绍了NPC++编程模型中数据面的拓展使用方法。首先，5.1节阐述了元数据的概念，包括系统固有的只读输入/输出元数据和用户可自定义的元数据，用于在报文处理过程中携带控制与状态信息。5.2节详述了四类表结构及其用途：线性表、精确匹配表、最长匹配表和三态匹配表，适用于配置存储、流表、路由和ACL等场景。5.3节聚焦报文处理API，涵盖报文头的有效位操作、发送与丢弃机制。最后，5.4节简要介绍了FIFO、Lock、Counter和CAR四类辅助编程接口，用于实现队列管理、同步、计数和流量控制等高级功能。

### 5.1 元数据

网络芯片处理数据报文时，可以定义全局的NPC结构体，被称作元数据。

包括系统固有的元数据和用户自定义元数据

系统固有元数据（仅作了解）：

可编程网络芯片在处理报文时，会携带一组固有的只读元数据 NPC_INTRINSIC_METADATA_S，用于描述报文的输入特征；一组传递至 OUTPUT 阶段的元数据NPC_INTRINSIC_OUTPUT_METADATA_S，用于描述输出行为，如下所示。

```cpp
输入元数据结构：NPC_INTRINSIC_METADATA_S
struct NPC_INTRINSIC_METADATA_S {
    uint<9>  InSubChan;     // 报文进入芯片时的接口 ID（非面板端口号）
    uint<4>  PortType;      // 接口类型：0-网络接口，1-CPU接口，2-loopback接口，3-组播loopback接口
    uint<14> PacSize;       // 报文长度
    uint<32> TimestampS;    // 秒级时间戳
    uint<32> TimestampNs;   // 纳秒级时间戳
};
```

InSubChan 表示报文到达芯片的入口接口编号，PortType 指明接口类型，PacSize 为数据包长度，TimestampS 与 TimestampNs 分别表示报文到达时的秒与纳秒时间戳。

```cpp
输出元数据结构：NPC_INTRINSIC_OUTPUT_METADATA_S
struct NPC_INTRINSIC_OUTPUT_METADATA_S {
    uint<9>  OutSubChan;  // 报文在芯片上的出口接口 ID（非面板端口号）
    uint<16> Sqid;        // 用户队列 ID
    uint<1>  DropFlag;    // 丢包标志
    uint<4>  Qindex;      // 队列优先级（决定调度队列）
    uint<2>  Color;       // 报文丢弃优先级（队列满时红色报文优先丢弃）
};
```

OutSubChan 指示报文出口接口，Sqid 为对应的用户队列标识，DropFlag 表示是否丢包，Qindex 用于调度优先级选择，Color 决定丢弃优先级。

用户自定义元数据：

我们以第2章的数据包处理实例为例子，支持自定义，如果需要记录报文内的一些信息，均可以声明元数据：

```cpp
//声明一些元数据（全局变量）
struct CTRL_META_DATA_S{   //自定义元数据，用于读取报文内的一些信息
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
```

### 5.2 table编程API

| **table类型** | **关键字** | **用途** |
| --- | --- | --- |
| 线性表 | index | 基于线性存储和寻址的线性存储表，在网络设备中常用于存储配置信息、封装信息等。 |
| 精确匹配表 | exact | 基于hash算法的非线性存储表，常用于定义流表，能够精确匹配特定的key。 |
| 最长匹配表 | lpm | 基于最长匹配算法的非线性存储表，常用于定义路由转发表。 |
| 三态匹配/掩码匹配表 | ternary | 基于三态内容寻址原理的存储表，常用于定义**访问控制列表（ACL）**，相对于精确匹配表，可以认为是一种能灵活定义规则的流表。 |

5.2.1 线性表 （Linear Table）

线性表通过 index 关键字定义，按索引位置直接存取。表大小（size）需与 key 的位宽对应。

```cpp
//所有未出现的变量需要相应的技术文档接口
#define TBL_SIZE_8BIT (1 << 8)
TEST_CTRL_META_DATA_S g_ctrlMd; //基础元数据的定义，声明以后其报文的元数据可以被读取使用
table TEST_LT_128BIT_LMEM_TBL() {
    key = { g_ctrlMd.LinearIndex_8bit: index; } //获取元数据的线性Index进行匹配
    STATUS_T _status(); //返回查询状态
    DATA_128BIT_S _lookup(); //执行查询
    void _write(DATA_128BIT_S inputData); //写入相关的inputData
    actions = { _lookup: LifbIReadProc(); _write: no_action; } //对应的function
    size = TBL_SIZE_8BIT;
}
void LifbIReadProc(STATUS_T sta, DATA_128BIT_S tblrsp) { g_ctrlMd.OpStatus = sta; }//更新元数据的status
```

线性表内置函数：
_lookup()：以声明的 key 变量为索引读取表项；
_write(data)：以 key 为索引写入数据；
_status()：返回表操作状态。

```cpp
读写示例：
//内置的变量
TEST_LT_128BIT_LMEM_TBL testLifbLmem;

// lookup 这个是声明的function函数，即匹配key成功后，执行这部分function
void TestLinearRead() {
    g_ctrlMd.LinearIndex_8bit = 0x3F;
    DATA_128BIT_S data = testLifbLmem._lookup();//内置函数
    g_ctrlMd.OpStatus = testLifbLmem._status();
}

// write
void TestLinearWrite() {
    g_ctrlMd.LinearIndex_8bit = 0x7F;
    DATA_128BIT_S data = 0xAABB;
    testLifbLmem._write(data); //内置函数
    g_ctrlMd.OpStatus = testLifbLmem._status();
}

```

5.2.2 精确匹配表（Exact Match Table）

精确匹配表是进行精确的匹配的，即例如数据报文的地址是0x0123，当表项里的Key也有0x0123，这样能精确匹配后可以执行后续的function

精确匹配表以 exact 声明 key。内置函数包括：
_lookup()：精确匹配查找；
_insert()：插入表项；
_delete()：删除表项；
_status()：获取操作状态。

```cpp
typedef uint<8> EM_INDEX_T;
TEST_FLOW_KEY_S g_flowKey;

table TEST_EM_AD128BIT_TBL() {
    key = {
        g_flowKey.Vsid: exact;
        g_flowKey.OuterVlan: exact;
        g_flowKey.DmacH16: exact;
        g_flowKey.DmacL32: exact;
        g_flowKey.InnerVlan: exact;
        g_flowKey.SmacH16: exact;
        g_flowKey.SmacL32: exact;
    }
    STATUS_T _status();
    DATA_128BIT_S_lookup();
    EM_INDEX_T _delete(EM_DELETE_OP_TYPE_E optype);
    actions = { _lookup: no_action; _delete: no_action; }
    size = TBL_SIZE_8BIT;
}

Lookup 示例：
void TestEmLookup() {
    ConstructEmKey(); // 根据报文组建Key
    DATA_128BIT_S ad = testEmTb1._lookup();
    g_ctrlMd.OpStatus = testEmTb1._status();
}

Read / Write 示例：
void TestEmReadWrite() {
    EM_INDEX_T idx = 0x3F;
    EM_READ_KT240BIT_S rsp = testEmIdx._read(idx);
    g_ctrlMd.OpStatus = testEmIdx._status();
    rsp.Add[7:0] += 1;
    g_ctrlMd.OpStatus = testEmIdx._status();
}

Insert 示例：
void TestEmInsert() {
    ConstructEmKey();
    DATA_128BIT_S ad = 0xAA;
    EM_INDEX_T idx = testEmTbl._insert(ad);
    g_ctrlMd.OpStatus = testEmTbl._status();
}

Delete 示例：
void TestEmDeleteByKey() {
    ConstructEmKey();
    testEmTbl._lookup();
    g_ctrlMd.OpStatus = testEmTbl._status();
    testEmTbl._delete(DELETE_BY_KEY);
    g_ctrlMd.OpStatus = testEmTbl._status();
}
```

5.2.3 最长匹配表（LPM Table）
最长匹配表以 lpm 声明 key，常用于路由查找。内置 _lookup() 函数执行最长匹配。

例如报文是0x0123，匹配的key是0x012*，*代表任意匹配，即可匹配成功0x0123的报文

```cpp
struct RSP_ADINDEX_S { uint<21> AdIndex; };
table TEST_LPM_TBL() {
    key = {
        g_flowKey.Vrfid: lpm;
        g_flowKey.DipPart1: lpm;
        g_flowKey.DipPart2: lpm;
        g_flowKey.DipPart3: lpm;
        g_flowKey.DipPart4: lpm;
    }
    RSP_ADINDEX_S _lookup();
    STATUS_T _status();
    actions = { _lookup: no_action; }
    size = TBL_SIZE_21BIT;
}

Lookup 示例：
void TestLpmLookup() {
    // 报文组Key  //rsp为要写入的内容，这个是华为的固定写法估计
    RSP_ADINDEX_S rsp = testLpmTbl._lookup();
    testRsp.Lw0 = rsp.AdIndex;
    testRsp.Lw1 = testLpmTbl._status();
}
```

5.2.4 掩码匹配表（Ternary Table）
掩码匹配表以 ternary 声明 key，执行掩码匹配（key & mask）。当命中多条 entry 时，AdIndex 小的优先级高。

```cpp
table TEST_ACL_TBL() {
    key = {
        g_flowKey.Vrfid : ternary;
        g_flowKey.SipPart1~4 : ternary;
        g_flowKey.DipPart1~4 : ternary;
        g_flowKey.Protocol : ternary;
        g_flowKey.SrcPort : ternary;
        g_flowKey.DstPort : ternary;
        g_flowKey.ExtOpt : ternary;
        g_flowKey.FlowId : ternary;
    }
    RSP_ADINDEX_S _lookup();
    STATUS_T _status();
    actions = { __lookup : no_action; }
    size = TBL_SIZE;
}

Lookup 示例：
void TernaryLookup() {
    // 报文组Key
    RSP_ADINDEX_S rsp = testAclTbl._lookup();
    testRsp3.Lw0 = rsp.AdIndex;
    testRsp3.Lw1 = testAclTbl._status();
}

```

### 5.3 报文处理API

介绍 NPC++ 报文编辑、发送与交互的相关 API，包括报文编辑、自发包、深度解析以及 CPU/环回交互。

**报文编辑**
转发程序中的报文编辑：在转发场景中，编辑基于 parse 模块输出的 NPCHeader 及其有效位。通过设置或清除有效位并调用 deparser 可实现协议头的增删改。
接口定义如下：
uint<l> npc_pkt_get_valid(packet_header); 

//这个packet_header是NPCHeader类型，即头文件里声明的
void npc_pkt_set_valid(packet_header, uint<l> validFlag);

```cpp
void SamplePktEditSend(ATB_S Arp, ECTB_S Ectb, EPAT_S Epat)
{
    if (npc_pkt_get_valid(outerIpv4Hdr) == 1) {
        npc_pkt_set_valid(innerIpv4Hdr, true);  // 增加 innerIpv4 头
        innerIpv4Hdr = outerIpv4Hdr;
    } else {
        npc_pkt_drop();
    }

    npc_pkt_set_valid(outerIpv4Hdr, true);  //这里可以看到相关代码的用法
    outerIpv4Hdr.Proto = IP_PROTOCOL_IPV4;
    outerIpv4Hdr.Totallen = innerIpv4Hdr.Totallen + IPV4_HDR_LENGTH;
    outerIpv4Hdr.SrcAddr = rsp.Sip;
    outerIpv4Hdr.DstAddr = rsp.Dip;
    npc_pkt_add_sys_hdr();
    npc_pkt_deparser();
    npc_pkt_send();
    npc_exit();
}
```

自发包：是在无输入报文的情况下由程序主动生成并发送的报文，不依赖 parser。
接口定义如下：
void npc_pkt_new();         // 申请报文资源
void npc_pkt_add(data_struct);
void npc_pkt_replicate();   // 复制当前报文
void npc_pkt_send();        // 发送并释放报文
void npc_pkt_drop();        // 丢弃并释放报文

使用流程：
调用 npc_pkt_new() 申请报文资源。
依次使用 npc_pkt_add() 添加各层报头。
如需批量发送，循环中调用 npc_pkt_replicate()；结束后调用 npc_pkt_drop()。
单报文发送则直接调用 npc_pkt_send()。

```cpp
void SampleSendMsg()
{
			    
    npc_pkt_new();
    uint<8> counter = 0;
    do {
        if (counter < 10) {
            CPU_ETH_HDR_S eth_hdr;
            CPU_PESUDO_IP_HDR_S ip_prehdr;
            eth_hdr.dmach16 = 0x2021;
            eth_hdr.dmac132 = 0xQAQAQAAAA;
            eth_hdr.smach16 = 0xAAA1;
            eth_hdr.smac132 = counter; 
            npc_pkt_add(eth_hdr);
            ip_prehdr.ver = 4;
            npc_pkt_add(ip_prehdr);
            npc_intrinsic_output_metadata.OutSubChan = CPU_CHANNEL0;
            npc_pkt_replicate();
            counter = counter + 1;
        } else {
            npc_pkt_drop(); 
            npc_exit();
        }
    } while(1);
}
```

**报文截断（Truncate）：用于删除报文尾部数据。**
void npc_pkt_truncate(uint<8> length);

```cpp
void SampleSetTruncAdd()
{
    UDF_HDR_S udfHdr;
    npc_pkt_add_sys_hdr();
    npc_pkt_deparser();
    npc_pkt_truncate(0);
    udfHdr.Lw0 = 0x0021;
    udfHdr.Lw1 = 0x0022;
    udfHdr.Lw2 = 0x0023;
    udfHdr.Lw3 = 0x0024;
    npc_pkt_add(udfHdr);
    npc_pkt_send();
    npc_exit();
}
```

**报文发送与丢弃**
报文发送接口：
void npc_pkt_send();  // 发包并释放资源

发包时隐式使用元数据指定出接口和丢弃标志，参见“元数据”章节。
报文丢弃接口：
void npc_pkt_drop();  // 丢弃并释放资源

```cpp
void SampleDrop()
{
    if (Ipv4Hdr.Ttl < 1) {
        npc_pkt_drop();
        npc_exit();
    }
}
```

```cpp

```

### **5.4 Fifo / Lock / Counter / CAR 编程 API  (拓展内容，参考着使用即可)**

介绍 NPC++ 中常用的四类基础数据结构和控制机制，包括 FIFO（先进先出队列）、Lock（互斥锁）、Counter（计数器） 及 CAR（流量控制计量器），并给出具体接口定义与示例。

5.4.1 FIFO
NPC++ 支持先进先出（FIFO）结构，用关键字 NPCFifo 声明。

```cpp
相关接口定义如下：
enum FIFO_OPC_STATUS_E {
    FIFO_OPC_SUCCESS = 0,
    FIFO_PUSH_FULL   = 1,
    FIFO_POP_EMPTY   = 1
};

NPCFifo <uint<width>, FIFO_DEPTH> Fifo;

STATUS_T npc_fifo_push(NPCFifo Fifo, uint<width> Element);
STATUS_T npc_fifo_pop(NPCFifo Fifo, uint<width> &OutElement);
参数说明：
uint<width>：FIFO 元素位宽。
FIFO_DEPTH：FIFO 深度，为立即数。
Element：push 输入，可用 {part1, part2} 形式，内部按 8bit 对齐，不足部分低位补 0。
OutElement：pop 输出，位宽需与 FIFO 元素宽度一致。
STATUS_T：返回状态可选，支持无状态调用。
```

```cpp
示例：
#define FIFO_DEPTH 1024
NPCFifo <uint<32>, FIFO_DEPTH> testfifo_32bit;

void SampleInfoPushPop()
{
    STATUS_T outStatus;
    uint<32> inputData = 0x11223344;
    uint<32> outputData;

    outStatus = npc_fifo_push(testfifo_32bit, inputData);
    outStatus = npc_fifo_pop(testfifo_32bit, outputData);  // 输出 0x11223344

    // Push 数据位宽小于 FIFO 元素宽度，低位补 0
    npc_fifo_push(testfifo_32bit, {inputData[31:(32-24)]}); // 24bit
    npc_fifo_pop(testfifo_32bit, outputData);               // 输出 0x11223300
}
```

5.4.2 Lock
NPC++ 提供锁机制（Lock），用于多线程或并行访问保护。

```cpp
声明与接口如下：
enum LOCK_OPC_STATUS_E {
    LOCK_OPC_SUCCESS   = 0,
    LOCK_GET_FALLURE   = 1
};

NPCLock LockDemo[LOCK_NUM];

STATUS_T npc_lock_get(NPCLock LockDemo);
STATUS_T npc_lock_free(NPCLock LockDemo);
```

```cpp
示例：
NPCLock FlowLock[4096];

void FlowReadProc(FLOW_KEY_S FlowKey, uint<8> Rsp, uint<16> Index)
{
    for (i = 0; i < 10; i++) {
        STATUS_T sta = npc_lock_get(FlowLock[Index]);
        if (sta == LOCK_OPC_SUCCESS) {
            ret = FlowCounter.read(Index);
            npc_lock_free(FlowLock[Index]);
            break;
        } else {
            npc_time.sleep(100);
        }
    }
}
```

5.4.3 Counter
计数器（Counter）用于数据面统计，如报文数、字节数等，用 NPCCounter 声明。

```cpp
NPCCounter <uint<width>> Counter[ARRAY_SIZE];

STATUS_T npc_counter_add(NPCCounter Counter, uint<16> addend);
STATUS_T npc_counter_add(NPCCounter Counter, uint<16> addend, uint<width> &cntVal);
STATUS_T npc_counter_sub(NPCCounter Counter, uint<16> subtraction);
STATUS_T npc_counter_sub(NPCCounter Counter, uint<16> subtraction, uint<width> &cntVal);
STATUS_T npc_counter_read(NPCCounter Counter, uint<width> &cntVal);
STATUS_T npc_counter_read_clr(NPCCounter Counter, uint<width> &cntVal);
参数说明：
uint<width>：计数器位宽。
ARRAY_SIZE：计数器个数。
addend / subtraction：加减数值。
cntVal：返回计数值（可选，用于 add/sub；必选用于 read/read_clr）。
STATUS_T：返回状态可选。

示例：
#define CNTER_NUM 256
NPCCounter <uint<64>> Fc64bit[CNTER_NUM];

void TestCntAdd_Fc64bit()
{
    STATUS_T outStatus;
    uint<64> cntVal;

    outStatus = npc_counter_add(Fc64bit[0], 1, cntVal);
    npc_counter_add(Fc64bit[0], 1);
}

void TestCntSub_Fc64bit()
{
    uint<64> cntVal;
    npc_counter_sub(Fc64bit[1], 1, cntVal);
    npc_counter_sub(Fc64bit[1], 1);
}

void TestCntRead_Fc64bit()
{
    uint<64> cntVal;
    npc_counter_read(Fc64bit[2], cntVal);
}

void TestCntReadClear_Fc64bit()
{
    uint<64> cntVal;
    npc_counter_read_clr(Fc64bit[3], cntVal);
}
```

5.4.4 CAR（Committed Access Rate）
CAR（流量控制）用于带宽限制与优先级标记，使用 NPCMeter 声明。

```cpp
NPCMeter Meter[Profile][ArraySize];

参数说明：
Profile：模板数量（用于 CIR、PIR 等属性承载）。
ArraySize：令牌桶实例数量。

接口定义：
STATUS_T npc_meter_run(NPCMeter Meter, uint<2> OutColor, uint<16> Len, ...);

参数说明：
OutColor：输出颜色（0=绿色，1=黄色，2=红色）。
Len：处理报文长度（≤16 bit）。

可选参数：
InColor：输入颜色（色散 CAR 时使用，默认 0）。
Priority：优先级（默认 0）。
返回值：STATUS_T 表示执行状态。
限制：Meter 第二维位宽 ≤ 24bit，Len ≤ 16bit。
```

```cpp
示例：
#define METER_PROFILE 10
#define METER_SIZE 8
NPCCounter <uint<DBG_CNTER_MIDTH>> debugCounter[TEST_CODE_TOTAL];
NPCMeter debugMeter[METER_PROFILE][METER_SIZE];

@control void TestCar()
{
    uint<2> outColor, inColor, inPriority;
    uint<16> len = npc_intrinsic_metadata.PacSize;

    STATUS_T outStatus1 = npc_meter_run(debugMeter[0][0], outColor, len);
    if (outStatus1 == OUTSTATUS_SUCCESS) {
        npc_counter_add(debugCounter[3], 1); // Policy IO 成功计数
    }

    npc_counter_add(debugCounter[CAR_OUT_COLOR], outColor);

    if (outColor == CAR_COLOR_GREEN) {
        npc_counter_add(debugCounter[CAR_COLOR_GREEN], 1);
    } else if (outColor == CAR_COLOR_YELLOW) {
        npc_counter_add(debugCounter[CAR_COLOR_YELLOW], 1);
    } else if (outColor >= CAR_COLOR_RED) {
        npc_counter_add(debugCounter[CAR_COLOR_RED], 1);
        npc_pkt_drop();
    }
}
```

### **5.5 其他API   (拓展内容，参考着使用即可)**

介绍NPC++中的其他实用编程接口，包括 Hash/除法计算、Checksum校验、定时器、随机数 以及 SACLE（ACL规则匹配）API。

5.5.1 Hash 与 除法接口
NPC++ 支持常用的 Hash 计算与整数除法操作，可用于负载均衡、流表索引等场景。
Hash计算接口

```cpp
定义如下：
enum HASH_ALG_E {
    CRC0 = 0,
    CRC1 = 1,
    CRC2 = 2,
    CRC_ALL = 3
};
STATUS_T npc_hash_calc(HASH_ALG_E HashMap, uint<0> &OutResult, uint<0> Key);
STATUS_T npc_hash_mod(HASH_ALG_E HashMap, uint<0> &OutResult, uint<0> Divisor, uint<0> Key);

npc_hash_calc()：执行单纯的Hash计算。
HashMap：Hash算法选择（0~2 或 CRC_ALL）。
Key：输入数据。
OutResult：输出Hash值。若选择CRC_ALL则返回96bit。
npc_hash_mod()：执行Hash计算并求余。
HashMap：选择单一算法。
Divisor：除数。
OutResult：返回Hash值对Divisor取余的结果。
两者的返回状态STATUS_T均为可选，可省略无状态调用。

使用示例：
void SampleHashCalc() {
    uint<32> hashKey = 0x000000A1;
    uint<32> hashVal;
    uint<96> allHashVal;
    npc_hash_calc(CRC0, hashVal, hashKey);
    npc_hash_calc(CRC1, hashVal, hashKey);
    npc_hash_calc(CRC2, hashVal, hashKey);
    npc_hash_calc(CRC_ALL, allHashVal, hashKey);
}
void SampleHashMod() {
    uint<32> hashKey = 0x000000A1;
    uint<16> divisor = 16, remainder;
    npc_hash_mod(CRC0, remainder, divisor, hashKey);
    npc_hash_mod(CRC1, remainder, divisor, hashKey);
    npc_hash_mod(CRC2, remainder, divisor, hashKey);
}
```

整数除法接口
NPC++提供三种除法接口，可选择返回商、余数或两者。

```cpp
STATUS_T npc_div_calc(uint<0> &outQuotient, uint<0> &outRemainder, uint<0> dividend, uint<0> divisor);
STATUS_T npc_div_quo(uint<0> &outQuotient, uint<0> dividend, uint<0> divisor);
STATUS_T npc_div_mod(uint<0> &outRemainder, uint<0> dividend, uint<0> divisor);

dividend：被除数
divisor：除数
outQuotient：商
outRemainder：余数

示例：
void SampleDivCalc() {
    uint<32> dividend = 0x000000A1, outQuotient;
    uint<16> divisor = 3, outRemainder;
    npc_div_calc(outQuotient, outRemainder, dividend, divisor);
    npc_div_quo(outQuotient, dividend, divisor);
    npc_div_mod(outRemainder, dividend, divisor);
}

```

5.5.2 校验和（Checksum）

NPC++支持16位校验和的计算、增量与减量更新操作。

```cpp

void npc_checksum16_calc(uint<16> &cksResult, uint<0> inputData);
void npc_checksum16_inc(uint<16> &cksResult, uint<16> initCks, uint<0> inputData);
void npc_checksum16_dec(uint<16> &cksResult, uint<16> initCks, uint<0> inputData);

cksResult：输出校验和（16 bit）
inputData：输入数据（位宽为16bit的整数倍）
initCks：初始校验值（用于增量或减量更新）

示例1：IPv4头校验
void EncapIPv4() {
    if (npc_pkt_get_valid(outerIpv4Hdr)) {
        outerIpv4Hdr.Totallen += GRE_HDR_LENGTH;
        outerIpv4Hdr.Proto = IP_PROTOCOL_GRE;
        outerIpv4Hdr.HdrChecksum = 0;
        uint<16> result;
        npc_checksum16_calc(result, outerIpv4Hdr);
        outerIpv4Hdr.HdrChecksum = result;
    } else {
        npc_pkt_drop();
        npc_exit();
    }
}

示例2：增量/减量验证
void TestCksCalc() {
    uint<16> cksResult, initCks, cksNewResult;
    uint<32> inputLw0 = 0xABCDEF00, inputLw1 = 0xABCDEF01, inputLw2 = 0xABCDEF02;
    npc_checksum16_calc(cksResult, {inputLw0, inputLw1});
    npc_checksum16_calc(initCks, inputLw0);
    npc_checksum16_inc(cksNewResult, initCks, inputLw1);
    if (cksResult != cksNewResult) { npc_counter_add(debugCounter[CKS_INC_CALC_ERROR], 1); npc_pkt_drop(); npc_exit(); }
    npc_checksum16_calc(initCks, {inputLw0, inputLw1, inputLw2});
    npc_checksum16_dec(cksNewResult, initCks, inputLw2);
    if (cksResult != cksNewResult) { npc_counter_add(debugCounter[CKS_DEC_CALC_ERROR], 1); npc_pkt_drop(); npc_exit(); }
}
```

5.5.3 定时器（Timer）

```cpp
定时器相关API定义如下：
enum TIMER_MODE_E { TIMER_MODE_NS = 0, TIMER_MODE_S = 1 };
uint<32> npc_time_get_second();          // 当前时间秒
uint<32> npc_time_get_nano_second();     // 当前时间纳秒
void npc_time_sleep(uint<32> cycles);    // 延迟指定周期
void npc_time_timer(TIMER_MODE_E mode, uint<32> CycleTime); // 等待指定时间

TimeMode = 0 表示按纳秒计时，1 表示按秒计时；
CycleTime 可为立即数或变量。

示例：
void SampTimer() {
    uint<32> curTimeS = npc_time_get_second();
    uint<32> curTimeNs = npc_time_get_nano_second();
    npc_time_sleep(500);             // 空闲500周期
    npc_time_timer(TIMER_MODE_S, 5); // 等待5秒
    npc_time_timer(TIMER_MODE_NS, 600); // 等待600纳秒
}
```

5.5.4 随机数

该接口用于获取一个32位随机数。

```jsx
void npc_random_get(uint<32> &value);
```

5.5.5 SACLE（ACL规则匹配）
SACLE API根据报文五元组进行ACL规则查找，支持IPv4与IPv6报文。
功能：
根据五元组（源IP、目的IP、协议、源端口、目的端口）匹配ACL规则，支持TCP/UDP协议。

```cpp
接口定义：
void ipv4_sacl_lookup(uint<32> sip, uint<32> dip, uint<8> protocol, uint<16> sport, uint<16> dport, uint<17> rid, uint<1> hit);
void ipv6_sacl_lookup(uint<128> sip, uint<128> dip, uint<8> protocol, uint<16> sport, uint<16> dport, uint<17> rid, uint<1> hit);

sip/dip/protocol/sport/dport：五元组信息
rid：命中规则索引，未命中为0
hit：规则命中标志

示例：
@control void ControlFlow() {
    uint<17> rid;
    uint<1> hit;

    if (npc_pkt_get_valid(ipv4Hdr)) {
        uint<32> sip_v4 = ipv4Hdr.Sip, dip_v4 = ipv4Hdr.Dip;
        uint<8>  protocol = ipv4Hdr.Protocol;
        uint<16> sport = 0, dport = 0;
        if (npc_pkt_get_valid(udpHdr)) { 
		        sport = udpHdr.SrcPort; 
		        dport = udpHdr.DstPort; 
        }
        else if (npc_pkt_get_valid(tcpHdr)) { 
		        sport = tcpHdr.srcPort; 
		        dport = tcpHdr.dstPort; 
        }
        ipv4_sacl_lookup(sip_v4, dip_v4, protocol, sport, dport, rid, hit);
    } else {
        uint<128> sip_v6 = ipv6Hdr.SrcAddr;
        uint<128> dip_v6 = ipv6Hdr.DstAddr;
        uint<8>  protocol = ipv6Hdr.NextHdr;
        uint<16> sport = 0; 
        uint<16> dport = 0;
        if (npc_pkt_get_valid(udpHdr)) { 
		        sport = udpHdr.SrcPort; 
		        dport = udpHdr.DstPort; 
        }
        else if (npc_pkt_get_valid(tcpHdr)) { 
		        sport = tcpHdr.srcPort; 
		        dport = tcpHdr.dstPort; 
        }
        ipv6_sacl_lookup(sip_v6, dip_v6, protocol, sport, dport, rid, hit);
    }
    npc_pkt_drop();
    npc_exit();
}
```