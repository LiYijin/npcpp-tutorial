# 附录2 常用函数参考

## 

提供参考资料和实用资源，方便读者查阅和拓展学习。

### **附录A：常用函数与库参考**

整理常用函数和库，方便查阅。

| 函数声明 | 函数说明 | 输入说明 | 输出说明 |
| --- | --- | --- | --- |
| STATUS_T npc_bit_get(LI_BITMAP bitmap, uint<24> index, uint<1> &oldbit); | 获取线性表的单bit值 | bitmap：线性表实例index：预期操作的bit索引 | oldbit：索引位置的bit值返回的左值：操作是否成功，参考npc_arch.h中的TBL_BIT_OPC_STATUS_E定义。 |
| STATUS_T npc_bit_set(LI_BITMAP bitmap,uint<24> index, uint<1> bitVal, uint<1> &oldbit); | 设置线性表的单bit值并返回原值 | bitmap：线性表实例index：预期操作的bit索引bitVal：设置的目标bit值 | oldbit：索引位置的bit原值返回的左值：操作是否成功，参考npc_arch.h中的TBL_BIT_OPC_STATUS_E定义。 |
| uint<1> npc_pkt_get_valid(packet_header); | 获取指定NPCHeader对应的有效位 | packet_header：NPCHeader类型的变量（报文头） | 返回的左值：指定NPCHeader对应的有效位 |
| void npc_pkt_set_valid(packet_header, uint<1> validFlag); | 设置指定NPCHeader对应的有效位，用于增加报文协议头 | packet_header：NPCHeader类型的变量（报文头） | NA |
| void npc_pkt_new(); | 实例化一个报文，申请报文资源，需要与npc_pkt_send/npc_pkt_drop配对使用 | NA | NA |
| void npc_pkt_add(data_struct); | 添加报文数据 | data_struct：报文数据段 | NA |
| void npc_pkt_truncate(uint<0> length); | 根据用户指定长度保留后续报文payload，超出部分将删除 | length：保留的payload长度，单位字节 | NA |
| void npc_pkt_replicate(); | 拷贝报文，原始报文编辑后发送，拷贝出来的报文继续用户自定义的后续处理 | NA | NA |
| void npc_pkt_send(); | 发送报文，释放报文资源 | NA | NA |
| void npc_pkt_drop(); | 丢弃报文，释放报文资源 | NA | NA |
| void npc_exit(); | 报文处理结束，等待新报文 | NA | NA |
| STATUS_T npc_fifo_push(NPCFifo Fifo, uint Element); | 将element数据push进入队列 | Fifo：NPCFifo实例Element：push进入队列的数据，允许以{part1, part2}的列表形式输入，但是列表内元素的宽度需要是8bit整倍数，列表总宽度不超过fifo元素位宽，低位补0 | 返回的左值：操作是否成功，参考npc_arch.h中的FIFO_OPC_STATUS_E定义。 |
| STATUS_T npc_fifo_pop(NPCFifo Fifo, uint &OutElement); | 从队列pop出数据 | Fifo：NPCFifo实例 | OutElement：从队列pop出来的数据返回的左值：操作是否成功，参考npc_arch.h中的FIFO_OPC_STATUS_E定义。 |
| STATUS_T npc_lock_get(NPCLock LockDemo); | 获取锁 | LockDemo：NPCLock实例 | 返回的左值：操作是否成功，参考npc_arch.h中的LOCK_OPC_STATUS_E定义。 |
| STATUS_T npc_lock_free(NPCLock LockDemo); | 释放锁 | LockDemo：NPCLock实例 | 返回的左值：操作是否成功，参考npc_arch.h中的LOCK_OPC_STATUS_E定义。 |
| STATUS_T npc_counter_add(NPCCounter Counter, uint<16> addend); | 计数器加 | Counter：NPCCounter实例addend：计数器加数 | 返回的左值：操作是否成功，参考npc_arch.h中的COUNTER_OPC_STATUS_E定义。 |
| STATUS_T npc_counter_add(NPCCounteCounter, uint<16> addend, uint &cntVal); | 计数器加并返回更新后的值 | Counter：NPCCounter实例addend：计数器加数 | cntVal：更新后的计数器值返回的左值：操作是否成功，参考npc_arch.h中的COUNTER_OPC_STATUS_E定义。 |
| STATUS_T npc_counter_sub(NPCCounte Counter, uint<16> subtraction); | 计数器减 | Counter：NPCCounter实例subtraction：计数器减数 | 返回的左值：操作是否成功，参考npc_arch.h中的COUNTER_OPC_STATUS_E定义。 |
| STATUS_T npc_counter_sub(NPCCounter Counter, uint<16> subtraction, uint &cntVal); | 计数器减并返回更新后的值 | Counter：NPCCounte实例subtraction：计数器减数 | cntVal：更新后的计数器值返回的左值：操作是否成功，参考npc_arch.h中的COUNTER_OPC_STATUS_E定义。 |
| STATUS_T npc_counter_read(NPCCounter Counter, uint &cntVal); | 读计数器 | Counter：NPCCounter实例 | cntVal：计数器值返回的左值：操作是否成功，参考npc_arch.h中的COUNTER_OPC_STATUS_E定义。 |
| STATUS_T npc_counter_read_clr(NPCCounter Counter, uint &cntVal); | 读计数器后清零 | Counter：NPCCounter实例 | cntVal：清零前的计数器值返回的左值：操作是否成功，参考npc_arch.h中的COUNTER_OPC_STATUS_E定义。 |
| STATUS_T npc_hash_calc(HASH_ALG_E HashAlg, uint<0> &OutResult, uint<0> Key); | Hash计算 | HashAlg：使用的Hash算法，支持0~2共3种算法。可以选择一次性返回3个算法的Hash结果.Key：Hash计算的输入数据 | OutResult：返回的Hash值。返回的左值：操作是否成功，参考npc_arch.h中的HASH_OPC_STATUS_E定义。 |
| STATUS_T npc_hash_mod(HASH_ALG_E HashAlg, uint<0> &OutResult, uint<0> Divisor, uint<0> Key); | Hash计算并求余 | HashAlg：指示使用Hash算法，支持0~2共3种算法。只能选择其中一种算法，不可以选择全部。Divisor：除数，求余运算的输入。Key：Hash计算的输入数据。 | OutResult:返回的求余结果。根据选择的Hash算法，计算Hash值，再对Divisor求余，返回余数。返回的左值：操作是否成功，参考npc_arch.h中的HASH_OPC_STATUS_E定义。 |
| STATUS_T npc_div_calc(uint<0> &outQuote, uint<0> &outRemainder, uint<0> dividend, uint<0> divisor); | 整数除法计算，返回商和余数 | dividend：被除数，整数除法的输入。divisor：除数，整数除法的输入。 | outQuotient：商，整数除法的输出。outRemainder：余数，整数除法的输出。返回的左值：操作是否成功，参考npc_arch.h中的DIV_OPC_STATUS_E定义。 |
| STATUS_T npc_div_quo(uint<0> &outQuotient, uint<0> dividend, uint<0> divisor); | 整数除法计算，返回商 | dividend：被除数，整数除法的输入。divisor：除数，整数除法的输入。 | outQuotient：商，整数除法的输出。返回的左值：操作是否成功，参考npc_arch.h中的DIV_OPC_STATUS_E定义。 |
| STATUS_T npc_div_mod(uint<0> &outRemainder, uint<0> dividend, uint<0> divisor); | 整数除法计算，返回余数 | dividend：被除数，整数除法的输入。divisor：除数，整数除法的输入。 | outRemainder：余数，整数除法的输出。返回的左值：操作是否成功，参考npc_arch.h中的DIV_OPC_STATUS_E定义。 |
| void npc_checksum16_calc(uint<16> &cksResult, uint<0> inputData); | Checksum计算 | inputData：输入的待计算数据，要求位宽必须是16 bit的整数倍。 | cksResult：返回的校验和计算结果，位宽16 bit。 |
| void npc_checksum16_inc(uint<16> &cksResult, uint<16> initCks, uint<0> inputData); | Checksum增量计算 | nitCks：初始校验和值。inputData：输入的待计算数据，要求位宽必须是16 bit的整数倍。 | cksResult：返回的校验和计算结果，位宽16 bit。 |
| void npc_checksum16_dec(uint<16> &cksResult, uint<16> initCks, uint<0> inputData); | Checksum减量计算 | initCks：初始校验和值。inputData：输入的待计算数据，要求位宽必须是16 bit的整数倍。 | cksResult：返回的校验和计算结果，位宽16 bit。 |
| uint<32> npc_time_get_second(); | 返回当前时间戳的秒部分 | NA | 返回的左值：获取到的当前秒时间 |
| uint<32> npc_time_get_nano_second(); | 返回当前时间戳的纳秒部分 | NA | 返回的左值：获取到的当前纳秒时间 |
| void npc_time_sleep(uint<32> cycles); | 延迟用户指定的cycles个时钟周期 | cycles：将要延迟的时钟周期个数 | NA |
| void npc_time_timer(TIMER_MODE_E TimeMode, uint<32> CycleTime); | 等待用户指定的时间。 | TimeMode：计时的时间单位，如果TimeMode设置为0，则以纳秒计时，如果TimeMode设置为1，则以秒计时。CycleTime：立即数或变量，等待若干个单位的时间。 | NA |
| void npc_random_get(uint<32> &value); | 获取随机数 | NA | value：获取到的随机数 |
| uint<8> npc_pkt_get_data(T headerLeftValue, uint<8> buffer[124]); | 获取当前报文窗中指定报文协议头后的报文数据 | headerLeftValue：指定从某个NPCHeader后开始获取报文。 | buffer：报文数据缓存的buffer，需要是字节数组。返回的左值：buffer数组的有效长度。 |
| uint<8> npc_pkt_get_next_data(uint<8> offset, uint<8> buffer[124]); | 获取下一段报文窗的数据 | offset：从当前报文窗的某个位置开始，获取下一段报文窗数据 | buffer：报文数据缓存的buffer，需要是字节数组。返回的左值：buffer数组的有效长度。 |
| void npc_pkt_copy_to_var(uint0 &dst, uint<8> onset, uint<8> size); | 从报文窗的指定偏移开始获取指定长度的数据拷贝到变量中 | offset：当前报文窗的某个位置，单位字节。size：拷贝的报文数据长度，单位字节。 | dst：拷贝报文数据的目的变量。 |

### **附录B：错误代码与解决方案**

列举常见错误代码及解决方案。

### **附录C：相关工具与资源推荐**

推荐学习工具、社区、开源项目等资源。

了解数据包转发处理基础：[协议无关的可编程包处理器](https://zhuanlan.zhihu.com/p/19947977947) 

基础数据包处理语言P4：[P4~16~ Language Specification](https://p4lang.github.io/p4-spec/docs/P4-16-v1.0.0-spec.html)