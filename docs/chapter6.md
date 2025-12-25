# 第6章    配套控制面接口(API)

本章聚焦控制面接口，全部基于C语言在CPU上实现。

6.1节介绍NDI（Network Device Interface）接口，作为控制面核心，提供标准化的设备互通能力，涵盖函数定义、参数规范及调用示例。

6.2节说明HAL接口中的启动初始化流程。

6.3节阐述SACLE接口，用于实现基于五元组（源/目的IP、端口、协议号）的最长前缀匹配ACL规则查询与动作执行。

6.4节介绍CAR（Committed Access Rate）接口，控制面可配置流量监管策略，实现精细化带宽控制与资源管理。

### 6.1. NDI接口说明

NDI（Network Device Interface） 接口作为控制面核心功能接口之一，完全遵循 C 语言的开发规范与编码标准进行设计和实现。

NDI接口作为网络音视频传输的核心标准化接口，其函数接口体系直接决定了设备间互联互通的稳定性与效率。本节将聚焦核心函数的功能定义、参数规范及调用示例，为开发适配提供清晰的技术指引。

### 6.1.1 相关函数接口参考

| 函数声明 | 函数说明 |
| --- | --- |
| ndi_status_t ndi_module_init(const char *json_path); | json解析接口,入参json_path为json路径。 |
| ndi_status_t ndi_info_get(const ndi_info_hdl **info_hdl_ret); | 获取ndiinfo实例,该实例和之前调用ndi_module_init加载的json关联。 |
| ndi_status_t ndi_table_from_name_get(const char *table_name, const ndi_table_hdl **ndi_table_hdl_ret); | 通过表项名字获取表项实例handle。 |
| ndi_status_t ndi_table_from_id_get(uint32_t id, const ndi_table_hdl **ndi_table_hdl_ret); | 通过表项id获取表项实例handle。 |
| ndi_status_t ndi_table_name_to_id(const char *table_name, uint32_t *id_ret); | 获取名字为table_name的表项的id。 |
| ndi_status_t ndi_table_info_get(const ndi_table_hdl *table_hdl, const ndi_table_info_hdl **table_info_hdl); | 获取表项实例table_info_hd的表项信息handle。 |
| ndi_status_t ndi_table_size_get(const ndi_table_hdl *table_hdl, size_t *size); | 获取表项实例table _hdl的条目数量。 |
| ndi_status_t ndi_table_name_get(const ndi_table_info_hdl *table_info_hdl, const char **table_name_ret); | 获取表项信息的表项名字 |
| ndi_status_t ndi_table_usage_get(const ndi_table_hdl *table_hdl, uint32_t *type); | 获取表项的类型 |
| ndi_status_t ndi_key_field_id_get(const ndi_table_info_hdl *table_info_hdl, const char *name, uint32_t *field_id); | 通过key域段的名字获取域段id。 |
| ndi_status_t ndi_key_field_size_get(const ndi_table_info_hdl *table_info_hdl, const uint32_t field_id, size_t *size); | 获取id为field_id的key域段的位宽。 |
| ndi_status_t ndi_table_key_allocate(const ndi_table_hdl *table_hdl, const ndi_table_key_hdl **key_hdl_ret); | 申请表项table_hdl的key操作handle指针,此操作会申请内存,申请出来的key handle里的所有域段初始值为0。调用者需保证后续使用完handle之后将handle进行释放,否则会有内存泄露。 |
| ndi_status_t ndi_table_key_deallocate(ndi_table_key_hdl *key_hdl); | 释放key_hdl指针。 |
| ndi_status_t ndi_table_data_allocate(const ndi_table_hdl *table_hdl, const ndi_table_data_hdl **data_hdl_ret);  | 申请表项table_hdl的data操作handle指针,此操作会申请内存,申请出来的data handle里的所有域段初始值为0。调用者需保证后续使用完handle之后将handle进行释放,否则会有内存泄露。 |
| ndi_status_t ndi_table_data_deallocate(ndi_table_data_hdl *data_hdl); | 释放data_hdl指针。 |
| ndi_status_t ndi_key_field_name_get(const ndi_table_info_hdl *table_info_hdl, const uint32_t field_id, const char **name); | 获取key handle里id为field_id的域段的名字。 |
| ndi_status_t ndi_key_field_id_list_size_get(const ndi_table_info_hdl *table_info_hdl, uint32_t *num); | 获取table_info_hdl里key field的数量。 |
| ndi_status_t ndi_key_field_id_list_get(const ndi_table_info_hdl *table_info_hdl, uint32_t *id_arr); | 获取table_info_hdl里所有key field的id编号。 |
| ndi_status_t ndi_data_field_id_get(const ndi_table_info_hdl *table_info_hdl, const char *name, uint32_t *field_id); | 获取table_info_hdl里名字为name的data field的id。 |
| ndi_status_t ndi_data_field_name_get(const ndi_table_info_hdl *table_info_hdl, const uint32_t field_id, const char **name); | 获取table_info_hdl里id为field_id的key feld的名字。 |
|  ndi_status_t ndi_data_field_id_list_size_get(const ndi_table_info_hdl *table_info_hdl, uint32_t *num); | 获取table_info_hdl里data field的数量。 |
| ndi_status_t ndi_data_field_list_get(const ndi_table_info_hdl *table_info_hdl, uint32_t *id_arr); | 获取table_info_hdl里所有data field的id编号。 |
| ndi_status_t ndi_data_field_size_get(const ndi_table_info_hdl *table_info_hdl, const uint32_t field_id, size_t *size); | 获取table_info_hdl里id为field_id的data field的bit位宽 |
| ndi_status_t ndi_key_field_set_value(ndi_table_key_hdl *key_hdl, const uint32_t field_id, const uint64_t value); | 将key_hdl里id为field_id的域段的值设置为value,用于EM表key域段赋值。 |
| ndi_status_t ndi_key_field_set_value_ptr(ndi_table_key_hdl *key_hdl, const uint32_t field_id, const uint8_t *value, const size_t size); | 将value字节数组赋值给key_hdl中id为field_id的域段,用于EM表的key域段赋值。 |
| ndi_status_t ndi_key_field_set_value_and_mask(ndi_table_key_hdl *key_hdl, const uint32_t field_id, uint64_t value, uint64_t mask); | 将value和mask赋值给key_hdl中id为field_id的域段,用于Ternary表的key域段赋值。
 |
| ndi_status_t ndi_key_field_set_value_and_mask_ptr(ndi_table_key_hdl*key_hdl, const uint32_t field_id, const uint8_t *value, const uint8_t *mask, const size_t size); | 将value和mask字节数组赋值给key_hdl中id为field_id的域段,用于Ternary表的key域段赋值,用于Ternary表的key域段赋值。 |
| ndi_status_t ndi_key_field_set_value_lpm(ndi_table_key_hdl *key_hdl, const uint32_t field_id, const uint64_t value, const uint16_t p_length); | 将value和p_length赋值给key_hdl中id为field_id的域段,用于LPM表的key域段赋值,其中p_length为域段对应的prefix匹配长度。 |
| ndi_status_t ndi_key_field_set_value_lpm_ptr(ndi_table_key_hdl *key_hdl, const uint32_t field_id, const uint8_t *value, const uint16_t p_length, const size_t size); | 将value字节数组和p_length赋值给key_hdl中id为field_id的域段,用于LPM表的key域段赋值。
 |
| ndi_status_t ndi_data_field_set_value(ndi_table_data_hdl *data_hdl, const uint32_t field_id, const uint64_t value); | 将value赋值给data_hdl中id为field_id的域段。 |
| ndi_status_t ndi_data_field_set_value_ptr(ndi_table_data_hdl *data_hdl, const uint32_t field_id, const uint8_t *value, const size_t size); | 将value字节数组赋值给data hdl中id为field_id的域段。 |
| ndi_status_t ndi_key_field_get_value(const ndi_table_key_hdl *key_hdl, const uint32_t field_id, uint64_t *value); | 获取key里id为field_id的field的数值。 |
| ndi_status_t ndi_key_field_get_value_ptr(const ndi_table_key_hdl *key_hdl, const uint32_t field_id, const size_t size, uint8_t *value); | 获取key_hdl中id为field_id的域段的数值,存放到字节数组返回,用于EM表。 |
| ndi_status_t ndi_key_field_get_value_and_mask(const ndi_table_key_hdl*key_hdl, const uint32_t field_id, uint64_t *value, uint64_t *mask); | 获取key_hdl中id为field_id的域段的数值和mask,用于Ternary表。 |
| ndi_status_t ndi_key_field_get_value_and_mask_ptr(const ndi_table_key_hdl *key_hdl, const uint32_t field_id, const size_t size, uint8_t *value, uint8_t *mask); | 获取key_hdl中id为field_id的field的数值和掩码,以字节数组(深度为入参size)返回。
 |
| ndi_status_t ndi_key_field_get_value_lpm(const ndi_table_key_hdl *key_hdl, const uint32_t field_id, uint64_t*value, uint16_t *p_length); | 获取key_hdl中id为field_id的域段的数值和p_length,用于LPM表。 |
| ndi_status_t ndi_key_field_get_value_lpm_ptr(const ndi_table_key_hdl *key_hdl, const uint32_t field_id, const size_t size, uint8_t *value, uint16_t *p_length); | 获取key_hdl中id为field_id的域段的数值和p_length,返回的数值通过字节数组返回,用于LPM表。 |
| ndi_status_t ndi_data_field_get_value(ndi_table_data_hdl *data_hdl, const uint32_t field_id, uint64_t *value); | 获取data hdl中id为field_id的field的数值。 |
| ndi_status_t ndi_data_field_get_value_ptr(ndi_table_data_hdl *data_hdl, const uint32_t field_id, const size_t size, uint8_t *value); | 获取data hdl中id为field_id的field的数值,返回的数值通过字节数组返回。
 |
| ndi_status_t ndi_table_entry_add(ndi_table_hdl *table_hdl, const ndi_table_key_hdl *key, const ndi_table_data_hdl *data); | 表项add操作,往表项table_hdl中添加一条entry。 |
| ndi_status_t ndi_table_entry_get(ndi_table_hdl *table_hdl, const ndi_table_key_hdl *key, ndi_table_data_hdl *data); | 表项get操作,获取表项table_hdl一条entry。 |
| ndi_status_t ndi_table_entry_mod(ndi_table_hdl *table_hdl, const ndi_table_key_hdl *key, const ndi_table_data_hdl *new_data, const ndi_table_data_hdl *old_data); | 表项modify操作,更新表项一条entry。**注意:**当表项为ternary表时,需要传入旧的data,即old_data不能为空;当表项为线性表、EM表、LPM表时,old_data无需关注,即old_data可以为空。 |
| ndi_status_t ndi_table_entry_del(ndi_table_hdl *table_hdl, const ndi_table_key_hdl *key, const ndi_table_data_hdl *data); | 表项delete操作,删除表项一条entry。**注意:**当表项为ternary表时,需要传入旧的data,即data不能为空;当表项为线性表、EM表、LPM表时,data无需关注,即old data可以为空。 |
|  ndi status_t ndi_table_clear(ndi_table_hdl *table_hdl); | 清理一个表,只支持线性表清空,不支持算法表清空。 |
| ndi status_t ndi_linear_tables_reset(const ndi_info_hdl *ndi_hdl); | 清理所有线性表,将所有线性表清空。 |
| ndi status_t ndi_set_log file(const char *filePath, uint32_t fileNum, uint32_t size); | 设置ndi日志路径、数量、文件大小(MB单位)。 |
| ndi status_t ndi_set_log_enable(bool enable); | ndi日志打印开关,enable为true时打开日志记录,否则关闭日志。 |
| ndi_status_t ndi_set_log_level(uint32_t level); | 设置日志打印级别,支持以下级别:
#define NDI LOG LEVEL TRACE 0
#define NDI LOG LEVEL DEBUG 1
#define NDI LOG LEVEL INFO 2
#define NDI LOG LEVEL WARN 3
#define NDI LOG LEVEL ERROR 4
#define NDI LOG LEVEL CRITICAL 5
#define NDI LOG LEVEL OFF 6 |

### 6.1.2 使用示例(C代码)

```cpp
//所有头文件代码
#include<stdio.h>
#include<stdlib.h>
#include<string.h>
#include<sys/socket.h>
#include<netinet/in.h>
#include<netpacket/packet.h>
#include<net/ethernet.h>
#include<net/if.h>
#include<arpa/inet.h>
#include<unistd.h>
#include<sys/ioctl.h>
#include <linux/if.h>
#include <errno.h>
#include <linux/if ether.h>
#define BUFFER SIZE 2048
```

添加表项entry，这些都写进某个函数里

```cpp
std::string tblName = "icib"; // EM table
ndi_table_hdl *table_hdl = nullptr;
ndi_table_key_hdl *key_hdl = nullptr;
ndi_table_data_hdl *data_hdl = nullptr;

auto ret = ndi_table_from_name_get(tblName.c_str(), (const ndi_table_hdl **)&table_hdl);
EXPECT_EQ(ret, NDI_OK);
ret = ndi_table_key_allocate(table_hdl, (const ndi_table_key_hdl **)&key_hdl);
EXPECT_EQ(ret, NDI_OK);
ret = ndi_table_data_allocate(table_hdl, (const ndi_table_data_hdl **)&data_hdl);
EXPECT_EQ(ret, NDI_OK);
ret = ndi_key_field_set_value_and_mask(key_hdl, 0, 0x03, 0);
EXPECT_EQ(ret, NDI_OK);
ret = ndi_key_field_set_value_and_mask(key_hdl, 1, 0x011, 0);
EXPECT_EQ(ret, NDI_OK);
ndi_data_field_set_value(data_hdl, 0, 0x0012);
EXPECT_EQ(ret, NDI_OK);
ndi_data_field_set_value(data_hdl, 1, 0x3);
EXPECT_EQ(ret, NDI_OK);
ret = ndi_table_entry_add(table_hdl, (const ndi_table_key_hdl *)key_hdl, (const ndi_table_data_hdl *)data_hdl);
EXPECT_EQ(ret, NDI_OK);
ret = ndi_table_key_deallocate(key_hdl);
EXPECT_EQ(ret, NDI_OK);
ret = ndi_table_data_deallocate(data_hdl);
EXPECT_EQ(ret, NDI_OK);
```

查询表项entry

```cpp
std::string tblName = "fibv4"; // LPM table
ndi_table_hdl *table_hdl = nullptr;
ndi_table_key_hdl *key_hdl = nullptr;
ndi_table_data_hdl *data_hdl = nullptr;

ret = ndi_table_from_name_get(tblName.c_str(), (const ndi_table_hdl **)&table_hdl);
EXPECT_EQ(ret, NDI_OK);
ret = ndi_table_key_allocate(table_hdl, (const ndi_table_key_hdl **)&key_hdl);
EXPECT_EQ(ret, NDI_OK);
ret = ndi_table_data_allocate(table_hdl, (const ndi_table_data_hdl **)&data_hdl);
EXPECT_EQ(ret, NDI_OK);
ret = ndi_key_field_set_value_lpm(key_hdl, 0, 0x1111, 0);
EXPECT_EQ(ret, NDI_OK);
ret = ndi_key_field_set_value_lpm(key_hdl, 1, 0x22222222, 0);
EXPECT_EQ(ret, NDI_OK);
ret = ndi_table_entry_get(table_hdl, (const ndi_table_key_hdl *)key_hdl,(ndi_table_data_hdl *)data_hdl);
EXPECT_EQ(ret, NDI_OK);
ret = ndi_table_key_deallocate(key_hdl);
EXPECT_EQ(ret, NDI_OK);
ret = ndi_table_data_deallocate(data_hdl);
EXPECT_EQ(ret, NDI_OK);
```

删除表项entry

```cpp
std::string tblName = "ternary_test_table"; // ternary表
ndi_table_hdl *table_hdl = nullptr;
ndi_table_key_hdl *key_hdl = nullptr;
ndi_table_data_hdl *data_hdl = nullptr;

auto ret = ndi_table_from_name_get(tblName.c_str(), (const ndi_table_hdl **)&table_hdl);
EXPECT_EQ(ret, NDI_OK);
ret = ndi_table_key_allocate(table_hdl, (const ndi_table_key_hdl **)&key_hdl);
EXPECT_EQ(ret, NDI_OK);
ret = ndi_table_data_allocate(table_hdl, (const ndi_table_data_hdl **)&data_hdl);
EXPECT_EQ(ret, NDI_OK);
ret = ndi_key_field_set_value_and_mask(key_hdl, 0, 0x25322110, 0);
EXPECT_EQ(ret, NDI_OK);
ndi_data_field_set_value(data_hdl, 0, 0x111);
EXPECT_EQ(ret, NDI_OK);
// add
ret = ndi_table_entry_add(table_hdl, (const ndi_table_key_hdl *)key_hdl, (const ndi_table_data_hdl *)data_hdl);
EXPECT_EQ(ret, NDI_OK);
ret = ndi_table_data_deallocate(data_hdl);
EXPECT_EQ(ret, NDI_OK);
// delete
ret = ndi_table_entry_del(table_hdl, (const ndi_table_key_hdl *)key_hdl, (ndi_table_data_hdl *)data_hdl);
EXPECT_EQ(ret, NDI_OK);
ret = ndi_table_key_deallocate(key_hdl);
EXPECT_EQ(ret, NDI_OK);
ret = ndi_table_data_deallocate(data_hdl);
EXPECT_EQ(ret, NDI_OK);
```

修改表项entry

```cpp
std::string tblName = "ternary_test_table"; // ternary表
ndi_table_hdl *table_hdl = nullptr;
ndi_table_key_hdl *key_hdl = nullptr;
ndi_table_data_hdl *data_hdl = nullptr;

auto ret = ndi_table_from_name_get(tblName.c_str(), (const ndi_table_hdl **)&table_hdl);
EXPECT_EQ(ret, NDI_OK);
ret = ndi_table_key_allocate(table_hdl, (const ndi_table_key_hdl **)&key_hdl);
EXPECT_EQ(ret, NDI_OK);
ret = ndi_table_data_allocate(table_hdl, (const ndi_table_data_hdl **)&data_hdl);
EXPECT_EQ(ret, NDI_OK);
ret = ndi_key_field_set_value_and_mask(key_hdl, 0, 0x25300110, 0);
EXPECT_EQ(ret, NDI_OK);
ndi_data_field_set_value(data_hdl, 0, 0x125);
EXPECT_EQ(ret, NDI_OK);
ret = ndi_table_entry_add(table_hdl, (const ndi_table_key_hdl *)key_hdl, 
                          (const ndi_table_data_hdl *)data_hdl);
EXPECT_EQ(ret, NDI_OK);
// alloc new data handler
ndi_table_data_hdl *new_data_hdl = nullptr;
ret = ndi_table_data_allocate(table_hdl, (const ndi_table_data_hdl **)&new_data_hdl);
EXPECT_EQ(ret, NDI_OK);
ndi_data_field_set_value(new_data_hdl, 0, 0x036);
EXPECT_EQ(ret, NDI_OK);
// modify entry using old data and new data
ret = ndi_table_entry_mod(table_hdl, (const ndi_table_key_hdl *)key_hdl, 
                          (const ndi_table_data_hdl *)new_data_hdl,
                          (ndi_table_data_hdl *)data_hdl);
EXPECT_EQ(ret, NDI_OK);
ret = ndi_table_data_deallocate(new_data_hdl);
EXPECT_EQ(ret, NDI_OK);
ret = ndi_table_key_deallocate(key_hdl);
EXPECT_EQ(ret, NDI_OK);
ret = ndi_table_data_deallocate(data_hdl);
EXPECT_EQ(ret, NDI_OK);
```

## 6.2  HAL接口说明

### 6.2.1  启动初始化（这部分是控制面初始化代码，可直接c编译执行，同时添加相关的NDI接口对表项进行增删改查）

```cpp
typedef enum tagHalOasBoardType{
    HAL_BOARD_TYPE_F1A=0,
} HalOasBoardType;
typedef enum tagHalOasBootType{
    HAL_BOOT_TYPE_BOARD_REBOOT=0, /* 设备重启启动模式 */
    HAL_BOOT_TYPE_PROCESS_REBOOT=1, /* 进程重启模式，能力预留 */
} HalOasBootType;
typedef enum tagHalOasProcessType{
    HAL_PROCESS_TYPE_MAIN=0, /* 主进程 */
    HAL_PROCESS_TYPE_SLAVE=1, /* 从进程 */
} HalOasProcessType;

typedef struct tagHalOasInitInfo {
    HalOasBoardType boardType; /* 单板类型 */
    HalOasBootType bootType; /* 启动类型，0：设备重启启动，1：进程重启启动 */
    HalOasProcessType processType; /* 新增：支持多进程模式，0：主进程，1：从进程 */
    
    void (*feLogDiagFunc)(char *szExp, ...); /* 诊断日志文件 */
    void (*feLogBootFunc)(char *szExp, ...); /* 关键日志 */
    void (*pDebugPrintFunc)(char *szExp, ...); /* 私有日志文件记录 */
    void (*pSystemPrintFunc)(char *szExp, ...); /* 系统日志文件记录 */
    void (*pSendMsg2ICFunc)(char *pcBuf); /* debugging命令行打印到主控 */
    void *(*pHALMallocFunc)(uint32_t len); /* HAL申请内存 */
    void (*pHALFreeFunc)(void *p_data); /* HAL释放内存 */
    void *(*pCDKMallocFunc)(uint32_t len); /* CDK申请内存 */
    void (*pCDKFreeFunc)(void *p_data); /* CDK释放内存 */
    void (*pReportAlarm)(uint32_t param); /* 底层故障通告给上层 */
} HalOasInitInfo;

/* 加载表项配置文件  
参数：
[in]  feId芯片号，全都芯片可传入0xffff  
[in]  filename 表项json文件的路径+文件名
[in]  mode 加载/卸载
执行成功返回FE_RC_OK, 执行失败返回FE_RC_ERROR*/
uint32_t HAL_OAS_LoadTableCfgFile(uint32_t feId, char *filename, HalOasLoadTableMode mode);
typedef enum tagHalOasMicrocodeLoadMode {
    HAL_LOAD_MICRO_MODE_LOAD = 0, /* 加载 */
    HAL_LOAD_MICRO_MODE_UPGRADE = 1, /* 升级 */
} HalOasMiloadMode;
/* 开始执行背景线程任务
参数
[in]  feId 芯片号，全都芯片可传入0xffff
[in]  filename 表项json文件的路径+文件名
[in]  mode 加载/热升级
执行成功返回FE_RC_OK, 执行失败返回FE_RC_ERROR */
uint32_t HAL_OAS_LoadMicBinFile(uint32_t feId, char *filename, HalOasMicLoadMode mode);

/* 打开端口流量
参数
[in]  feId 芯片号，全都芯片可传入全F
执行成功返回FE_RC_OK, 执行失败返回FE_RC_ERROR */
uint32_t HAL_OAS_BootOk(uint32_t feId);

/* 启动背景线程任务
参数
[in]  feId 芯片号，全都芯片可传入0xffff
[in]  taskName: 背景线程名称
执行成功返回FE_RC_OK, 执行失败返回FE_RC_ERROR */
uint32_t HAL_OAS_StartBackgroundTask(uint32_t feId, const char *taskName);

/* 停止背景线程任务
参数
[in]  feId: 芯片号，全都芯片可传入0xffff
[in]  taskName: 背景线程名称
执行成功返回FE_RC_OK, 执行失败返回FE_RC_ERROR */
uint32_t HAL_OAS_EndBackgroundTask(uint32_t feId, const char *taskName);

/* 初始化 */
uint32_t HAL_OAS_InitPhase1(HalOasInitInfo *halInitInfo);
uint32_t HAL_OAS_InitPhase2(void);
```

### 6.2.2 加载数据面编译结果

```cpp
/*
feId: 芯片号，全都芯片可传入0xffff
filename: 微码bin文件的路径+文件名
mode: 加载/热升级
*/
uint32_t HAL_OAS_LoadMicBinFile(uint32_t feId, char *filename, HalOasMicLoadMode mode);

typedef enum tagHalOasLoadTableMode{
    HAL_LOAD_TABLE_MODE_LOAD = 0,
    HAL_LOAD_TABLE_MODE_UNLOAD = 1,
} HaloasLoadTableMode;

/*
feId: 芯片号，全都芯片可传入0xffff
filename: 表项json文件的路径+文件名
mode: 加载/卸载
*/
uint32_t HAL_OAS_LoadTableCfgFile(uint32_t feId, char *filename, HalOasLoadTableMode mode);

typedef enum tagHalOasMicrocodeLoadMode{
    HAL_LOAD_MICRO_MODE_LOAD = 0,
    HAL_LOAD_MICRO_MODE_UPGRADE = 1,
} HalOasMicLoadMode;

```

## 6.3 SACLE接口

SACLE是最长前缀匹配规则算法的接口，即匹配ACL规则，ACL是常见的五元组规则，包括源ip，目的ip，源端口，目的端口，协议号，通过最长前缀匹配方法对相应的数据包进行查询，命中后执行function

### 6.3.1 相关函数接口参考

| 函数声明 | 函数说明 |
| --- | --- |
| sacle_status_type_e sacle_alg_init(uint32_t en_print); | sacle算法初始化接口,入参为打印使能。
en_print=0:不打印
en_print=1:打印简单log
en_print=2:打印详细log |
| sacle_status_type_e sacle_uninit(); | sacle算法空间释放。 |
| sacle_status_type_e sacle_add_rule(sacle_rule_s *rule); | sacle添加单条规则。 |
| sacle_status_type_e sacle_batch_add_rule(sacle_rule_s *rules, uint32_t rules_num); | sacle批量添加规则,入参为规则数组和规则数目。 |
| sacle_status_type_e sacle_del_rule(uint32_t rule_id); | sacle删除单条规则 |
| sacle_status_type_e sacle_batch_del_rule(uint32_t *rules, uint32_t rules_num); | sacle批量删除规则,入参为规则ID的数组和规则数目。 |
| sacle_status_type_e sacle_clear_rule(); | sacle清空所有下过的规则。 |
| sacle_status_type_e sacle_update_rule(sacle_rule_s *rule); | sacle更新规则的ID,入参为需要修改ID的规则结构体指针。 |
| sacle_status_type_e sacle_lookup(sacle_ip_type_e ip_type, sacle_tuple_s src_tuple, sacle_tuple_s dst_tuple, uint32_t *rule_id); | sacle规则硬查接口,入参为IP类型,源tuple和目的tuple,返回为规则ID。 |
| sacle_status_type_e sacle_rule_s *sacle_get_rule(uint32_t rule_id); | sacle获取规则接口,通过rule id获取规则的结构体 |
| sacle_status_type_e sacle_set_print(uint32_t en_print); | sacle设置打印使能
en_print=0:接口调用之后的不打印
en_print=1:接口调用之后的打印简单log
en_print=2:接口调用之后的打印详细log |
| sacle_status_type_e sacle_cancel_initial_hbm(); | sacle取消硬件初始化,在sacle_alg_init之前调用,可以减少初始化时间。调试非规则下发与查找的功能时,可以使用。 |
| sacle_status_type_e sacle_debug_enable(uint32_t en_debug); | sacle设置转发面寄存器。
en_print=0:关闭转发面寄存器打印
en_print=1:打开转发面寄存器打印; |

### 6.3.2 使用示例

规则结构体赋值

```cpp
int test_num = 10;
int rules_num = test_num;
sacle_rule_s *rules = (sacle_rule_s *)malloc_size(sizeof(sacle_rule_s) * rules_num);
memset(rules, 0, sizeof(sacle_rule_s) * rules_num);

for (int i = 0; i < rules_num; i++) {
    rules[i].rule_id = i;
    for (int j = 0; j < 2; j++) {
        for (int k = 0; k < 2; k++) {
            rules[i].tuple_num[j][k] = 1;
            rules[i].tuple[j][k] = (sacle_rule_tuple_s *)
                malloc_size(sizeof(sacle_rule_tuple_s) * rules[i].tuple_num[j][k]);

            for (int p = 0; p < rules[i].tuple_num[j][k]; p++) {
                rules[i].tuple[j][k][p].ip[0] = i + 1;
                rules[i].tuple[j][k][p].ip[1] = 0;
                rules[i].tuple[j][k][p].ip[2] = 0;
                rules[i].tuple[j][k][p].ip[3] = 0;
                rules[i].tuple[j][k][p].mask_len = 32;
                rules[i].tuple[j][k][p].port_start = i + 1;
                rules[i].tuple[j][k][p].port_end = i + 1;
                rules[i].tuple[j][k][p].protocol = i + 1;
                rules[i].tuple[j][k][p].protocol_mask = 0xFF;
            }
        }
    }
}

int traces_num = test_num;
sacle_tuple_s *src_tuple = (sacle_tuple_s *)malloc_size(sizeof(sacle_tuple_s) * traces_num);
for (int i = 0; i < traces_num; i++) {
    src_tuple[i].ip[0] = i + 1;
    src_tuple[i].port = i + 1;
    src_tuple[i].protocol = i + 1;
}

sacle_tuple_s *dst_tuple = (sacle_tuple_s *)malloc_size(sizeof(sacle_tuple_s) * traces_num);
for (int i = 0; i < traces_num; i++) {
    dst_tuple[i].ip[0] = i + 1;
    dst_tuple[i].port = i + 1;
    dst_tuple[i].protocol = i + 1;
}
```

API返回值

```cpp
typedef enum {
    SACLE_OK = 0,        // 算法执行成功
    SACLE_ERROR,         // 算法执行出现错误情况
    SACLE_NOT_FOUND,     // 查找结果未找到匹配规则
    SACLE_RULE_EXIST,    // 插入规则时，该规则id已存在。此次操作不执行
    SACLE_RULE_NOT_EXIST,// 删除规则时，该规则id不存在。此次操作不执行
    SACLE_RULE_REPEAT,   // 批量插入或删除规则时，列表中有相同id的规则。此次操作不执行
    SACLE_RULE_REDUNDANCY,// 添加的规则中，tuple信息有冗余，仅为提示信息
    SACLE_RULE_FULL      // 此次添加操作会导致tuple总数超过限制。此次操作不执行
} sacle_status_type_e;
```

SCALE算法的增删改查接口使用

```cpp
算法初始化
sacle_alg_init(1);

添加单条表项
uint32_t ret;
ret = sacle_add_rule(&rules[i]); // 下发章节2.1定义的结构体数组rules的第i条规则

批量添加表项
uint32_t ret;
ret = sacle_batch_add_rule(rules, rules_num);

删除单条表项
uint32_t ret;
ret = sacle_del_rule(rules[i].rule_id);

批量删除表项
uint32_t *del_rules_id = (uint32_t *)malloc_size(sizeof(uint32_t) * rules_num);
for (int i = 0; i < rules_num; i++) {
    del_rules_id[i] = rules[i].rule_id;
}
uint32_t ret;
ret = sacle_batch_del_rule(del_rules_id, rules_num);

删除所有规则
sacle_clear_rule();

更新规则ID
uint32_t new_rule_id = 100;
rules[0].rule_id = new_rule_id;

uint32_t ret;
ret = sacle_update_rule(&rules[0]); // 更新rules[0]的ID为100

获取规则信息
sacle_rule_s *rule;
uint32_t get_rule_id = 100;
rule=sacle_get_rule(get_rule_id);//获取规则id=100的信息

规则查找(硬查)
sacle_rule_tuple_s src_tuple, dst_tuple;
src_tuple. ip[0] = 0x10101010;
src_tuple. port_start = 15000;
src_tuple. protocol = 0x06;
dst_tuple.ip[0] = 0x01010101;
dst_tuple. port_start = 15001;
dst_tuple. protocol = 0x06;
uint32_t rule_id;
uint32_t ret;
ret = sacle_lookup(SACLE_IPV4, src_tuple, dst_tuple, &rule_id);
```

## 6.4 CAR接口（查看系统环境）

在数据面使用npc_meter_run后会生成car_profile和car_context表，用户可在控制面下发car配置，示例如下：

```cpp
Car_profile carProfile;
Car_profile::Entry entry;
entry.key.profile_index = 0;   // car profile index
entry.data.cbs = 1024;
entry.data.cir = 10000;
entry.data.pir = 10000;
entry.data.pbs = 10000;
entry.data.carIndex = 0;
entry.data.colorMode = 0;  // 颜色模式
entry.data.carType = 5;    // CAR类型
entry.data.copId = 11;     // 无需关注
entry.data.ptpIndex = 31;
entry.data.p1Th = 1;  // cbs/xbs burst size threshold for policing with priority 1
entry.data.p2Th = 1;  // 优先级2
entry.data.p3Th = 1;  // 优先级3
entry.data.p4Th = 1;  // 优先级4
entry.data.p5Th = 1;  // 优先级5
entry.data.p6Th = 1;  // 优先级6
entry.data.p7Th = 1;  // 优先级7
entry.data.udf = 0;
entry.data.direction = 0;
entry.data.pipelineId = 0;
entry.data.polEn = 0;        // 使能标志
entry.data.polMode = 0;    // 0:byte模式 1:报文模式
entry.data.hierar = 0;     // 层次化的CAR
entry.data.compLen = 0;    // 包长补偿
carProfile.EntryAdd(entry);
```