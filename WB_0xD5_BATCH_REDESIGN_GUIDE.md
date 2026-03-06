# WB `0xd5` 批量通知彻底改造指南（Host + Controller）

> 目标：在**不新增命令 opcode** 的前提下，对现有 `0xd5` 路径做彻底重构，把每 I/O 一次同步通知改成批量通知，显著降低通知开销，解除 Host 侧 IOPS 瓶颈。  
> 原则：**不考虑兼容性**，旧 `0xd5` 单条语义直接删除。

---

## 0. 结论与现状判断

你的判断是对的：当前瓶颈主要在 `nvme_wb_submit_notify()`（Host 侧 `0xd5` 提交）。

### 证据链（已观测）

- Controller 侧 `copy_done avg` 常态约 `2.2~2.5us`，即设备处理 `0xd5` 本身不慢。
- Host 侧 `notify_submit` 常态约 `58~68us`，且 `notify_qwait` 高，远高于 `copy/mcp_read/parse/complete`。
- Host 主链路已降到 `~10us`，但整体 IOPS 仍被 `notify_submit` 上限钳制。

结论：若把 `0xd5` 从“每 I/O 一次”改为“每批次一次”，理论上可显著提升吞吐并降低抖动。

---

## 1. 重构总览（新 `0xd5` 语义）

### 1.1 语义变更（破坏式）

旧语义（删除）：
- 一条 `0xd5` 仅描述一个 `(qid, cmd_id, seg_cnt)`。

新语义（保留 opcode=`0xd5`）：
- 一条 `0xd5` 表示**同一 `qid` 下多个命令完成通知**，即批量 ACK。
- `0xd5` 通过 PRP 负载传输批量条目数组。

### 1.2 新 `0xd5` 编码建议

保持 opcode 不变，仅改字段定义：

- `cmd.common.opcode = 0xd5`
- `cdw10`: `flags/version`（例如 bit0=batched 固定为1）
- `cdw11`: `qid`
- `cdw12`: `batch_cnt`
- `cdw13`: `entry_bytes`（固定结构大小）
- `dptr.prp1/prp2`: 指向批量数组

批量条目结构（Host/Controller 共识）：

```c
struct wb_d5_batch_entry {
    __le16 cmd_id;
    __le16 rsvd0;
    __le32 seg_cnt;
};
```

> 说明：同一条目只需 `cmd_id + seg_cnt`，因为 `qid` 已在 `cdw11`。

---

## 2. Host 侧改造指南（`nvme-driver-for-write-buffer`）

## 2.1 设计目标

- 将 `notify_wq` 从“同步提交器”改为“批量聚合器 + 提交器”。
- `wb_work` 只负责：copy + complete + 投递 notify-item，不再逐条提交。
- `notify_wq` 负责按策略聚合后一次提交 `0xd5`。

## 2.2 关键改动点

建议重点改这些位置：

- [host/pci.c](host/pci.c): `nvme_wb_submit_notify()`
- [host/pci.c](host/pci.c): `nvme_wb_notify_workfn()`
- [host/pci.c](host/pci.c): `nvme_wb_queue_async_notify()`
- [host/pci.c](host/pci.c): `struct nvme_wb_queue_ctx`

### A. 删除旧单条提交路径

删除/替换：
- 旧 `nvme_wb_submit_notify(...cmd_id, seg_cnt...)` 单条语义。

新增：
- `nvme_wb_submit_notify_batch(struct request_queue *q, u16 qid, struct wb_d5_batch_entry *ents, u32 cnt)`

### B. 为每个 `qid` 增加批量缓存

在 `struct nvme_wb_queue_ctx` 增加：

- `batch_buf`（预分配数组）
- `batch_cnt`
- `batch_max`（例如 64/128）
- `batch_deadline_ns`（flush deadline）
- `batch_lock`

### C. 批量策略（推荐）

触发 flush 条件（任一满足）：
1. `batch_cnt >= batch_max`
2. 距首次入批超过 `T_flush`（例如 `20~50us`）
3. 队列空闲转入强制冲刷

### D. `notify_wq` 线程逻辑改成聚合提交

伪代码：

```c
while (has_pending_or_deadline) {
    collect entries into batch_buf;
    if (need_flush)
        submit one 0xd5 with PRP payload;
}
```

### E. 错误处理策略（简化版）

- 提交失败：直接报错，中断IO。不要重试。（这是为了调试阶段能快速看到问题）

### F. Host 观测项（必须保留）

新增/保留统计：
- `notify_batch_calls`
- `notify_batch_entries`
- `notify_submit_us`
- `notify_qwait_us`
- `notify_batch_fill`（平均每批条目数）

目标观测：
- `notify_submit` 仍可能在几十微秒，但**摊到单命令应降至亚微秒级~几微秒级**。

---

## 3. Controller 侧改造指南（`FEMU`）

## 3.1 设计目标

- `femu_wb_io_notify_copy_done()` 从“处理单 cmd_id”改为“处理批量条目数组”。
- 保持现有内部处理函数不变（可复用），外层改成批处理循环。

## 3.2 关键改动点

建议重点改这些位置：

- [hw/femu/bbssd/write_buffer.c](../FEMU/hw/femu/bbssd/write_buffer.c): `femu_wb_io_notify_copy_done()`
- [hw/femu/bbssd/write_buffer.c](../FEMU/hw/femu/bbssd/write_buffer.c): `wb_parse_notify_args()`（需要重写）
- [hw/femu/nvme-io.c](../FEMU/hw/femu/nvme-io.c): `NVME_CMD_WB_NOTIFY_COPY_DONE` 路由保留

### A. 重写 `wb_parse_notify_args()`

新解析内容：
- `qid` from `cdw11`
- `batch_cnt` from `cdw12`
- `entry_bytes` from `cdw13`
- PRP 读取 `batch_cnt * entry_bytes` 到临时 buffer

### B. 批处理执行模型

在 `femu_wb_io_notify_copy_done()` 中：

- 一次加锁（`lpn_index_lock`）
- 循环处理每个 entry：
  - lookup `cmd_id` -> track
  - 校验 `seg_cnt`
  - 执行原有 copy_done 子逻辑（`mcp_release`、`mirror/index/reclaim`）
- 一次解锁

> 这样可以把锁竞争从“每命令一次”降到“每批次一次”。

### C. 内部函数拆分建议

从当前函数抽出：

- `wb_copy_done_one_locked(FemuCtrl *n, FemuWbLocal *l, u16 cmd_id, u32 seg_cnt, struct wb_copy_done_stat *s)`

批处理函数只负责循环与聚合统计。

### D. Controller 统计项增强

在现有 `WB copy_done perf(1s)` 基础上加：

- `batch_calls`
- `batch_entries`
- `avg_entries_per_batch`

用于验证批处理确实生效。

---

## 4. 分阶段实施计划（建议给团队）

## Phase 1：协议切换（破坏式）

- Host 改 `0xd5` 为 batch payload。
- Controller 仅接受新 batch 格式；旧格式代码直接删掉。
- 验收：功能正确，数据一致。

## Phase 2：批量策略调优

- 参数化 `batch_max`、`T_flush`。
- 默认建议：`batch_max=64`，`T_flush=25us`。
- 验收：`avg_entries_per_batch` 明显 > 1。

## Phase 3：性能验证

- 对比改造前后：
  - Host: `notify_submit`、`notify_qwait`、`notify_batch_fill`
  - Controller: `copy_done avg`、`lock_wait`
  - FIO: IOPS、P99/P999

---

## 5. 删除项清单（按“彻底改造”执行）

可直接删：

- Host 旧 `0xd5` 单条提交逻辑路径
- Controller 旧 `wb_parse_notify_args()` 对单条 `cmd_id/seg_cnt` 的解析分支
- 与旧单条语义绑定的日志文本和错误码分支

---

## 6. 风险与防护

主要风险：
- 批量提交失败导致 ACK 延迟扩大。
- 单条 entry 损坏影响整批处理。

防护建议：
- 批内逐条校验，坏条目跳过并计数，不拖垮整批。
- 批提交失败时重试，并输出 `qid/batch_cnt/first_cmd_id`。
- 对 `batch_cnt` 和 `entry_bytes` 做硬边界检查。

---

## 7. 预期收益（按当前日志估算）

当前 Host `notify_submit` 约 `60us/命令`。若平均每批 `N` 条：

- 摊薄后约 $T_{notify\_per\_cmd} \approx \frac{60}{N}\,us$
- 当 `N=16`，理论可降到约 `3.75us/命令`。
- 当 `N=32`，理论可降到约 `1.9us/命令`。

这会直接解除当前 IOPS 天花板（约 `1 / 60us`）的限制。

---

## 8. 交付物建议（团队执行用）

Host 团队：
- `0xd5 batch payload` 发送实现
- `notify aggregator` + 参数化策略
- 新 perf 指标日志

Controller 团队：
- `0xd5 batch payload` 解析
- 批量循环调用 `copy_done_one_locked`
- 批处理 perf 指标日志

---
