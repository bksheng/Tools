# SWA Bounded Replay 双仓库分析报告

> **联网核验修订：2026-11-12（北京时间）**。本文核对论文原文、GitHub PR 状态/正文/讨论和固定版本源码，已直接改写确认错误的段落。原报告并非全部准确，尤其存在论文误读、P/D 逻辑反转、版本能力混淆和 PR 清单漏计。
> 固定代码基线：Ascend `5e40bc723165bfc8d8b1ec245ce5908a45dfd9d2`；上游 vLLM `v0.33.0`（`4ba787ec9959e566d3b587a796dff07673324da5`）。最新 PR 进展另列，不能自动归入这些旧快照。
> 论文：[DeepSeek-V4.1-Flash，arXiv:2609.19969v1](https://arxiv.org/html/2609.19969v1)；arXiv 提交历史本次仍仅列 v1。
> [修订前完整备份](../backups/SWA%20Bounded%20Replay%20双仓库分析报告_联网核验前_20261112.md) · [原始来源清单](../references/swa-audit-20261112/manifest.json) · [网页证据清单](../references/swa-audit-20261112/page-manifest.json)。第 8 章汇总修正与验证范围。

---

## 第 0 章：核验口径、版本与核心结论

### 0.1 证据等级

- **源码事实**：能在固定 commit/tag 的公开源码中定位。它证明静态实现，不证明所有硬件、配置和输入均正确。
- **社区记录**：PR 状态、合入时间、作者报告的测试和限制。作者自报成绩不等于本次复现，也不等于独立精度验收。
- **分析建议**：由源码和工作负载推导的移植取舍，明确与社区已实现能力区分。
- **未复核历史记录**：原工作区 `D:\Project\code\REPO\...`、`TASK/report/...` 和图/注释文件不在当前交付目录中；不能继续以“本机已验证”表述。相关内容保留为来源记录，不当成当前事实。

本次通过联网获取固定版本源文件核对关键链路；当前目录不是原双仓库 Git 工作树，**没有重新执行原报告所称的 `merge-base --is-ancestor` 全集验证**。原文“全部”“唯一”“完全互证”等结论已缩小到实际证据范围。PR 状态优先看 `merged_at`，`closed` 和 `merge_commit_sha` 单独都不能证明合入。

### 0.2 版本边界

| 对象 | 本次核验结果 | 证据与限制 |
| --- | --- | --- |
| Ascend 固定快照 | commit `5e40bc723…` 是 `releases/v0.31.0rc` 的 #19696 回移提交，提交时间 2026-10-26 11:08:51 +08:00 | [commit 页面](https://github.com/vllm-project/vllm-ascend/commit/5e40bc723165bfc8d8b1ec245ce5908a45dfd9d2)；不能把 10-19 的 replay 收拢日当作 HEAD 日期 |
| Ascend 与 vLLM 版本 | `vllm_version_is("0.31.0")` 证明存在针对该版本的兼容分支，**不足以单独证明依赖被精确 pin** | [cache_config.py](https://github.com/vllm-project/vllm-ascend/blob/5e40bc723165bfc8d8b1ec245ce5908a45dfd9d2/vllm_ascend/models/deepseek_v41/cache_config.py#L84)；本报告用 v0.31.0 作兼容对照，实际部署仍需锁定安装包/镜像中的 vLLM SHA |
| vLLM v0.31.0 | encoder replay 已有；`decoder_replay_layers.py` 不存在；`amd/model.py` 已存在 | [v0.31.0 attention](https://github.com/vllm-project/vllm/blob/v0.31.0/vllm/models/deepseek_v41/attention.py#L519)、[AMD 模型文件](https://github.com/vllm-project/vllm/blob/v0.31.0/vllm/models/deepseek_v41/amd/model.py)；原文把 AMD 文件整体写成不存在是错误的 |
| vLLM v0.32.0 | `decoder_replay_layers.py` 已存在 | [版本文件](https://github.com/vllm-project/vllm/blob/v0.32.0/vllm/models/deepseek_v41/decoder_replay_layers.py)；这是已核对版本间的文件边界，不声称遍历了所有 tag |
| vLLM v0.33.0 | 本报告上游实现基线，包含 encoder/decoder、图执行和若干 PCP/打分修复 | 以固定 tag 源码为准，不称“所有 replay 修复全集” |
| 发布日期 | v0.31.0 GitHub release 显示 2026-10-05；v0.33.0 页面显示 tagged 2026-11-01 | [v0.31.0 release](https://github.com/vllm-project/vllm/releases/tag/v0.31.0)、[v0.33.0 页面](https://github.com/vllm-project/vllm/releases/tag/v0.33.0)；原文 10-02/10-27 不可未经区分地叫“发布日期”，commit 时间、tag 时间和 release 时间不同 |

### 0.3 术语和机制底线

PCP 是 **Prefill Context Parallelism**；APC 是 **Automatic Prefix Caching**。EPD 是 **Encoder/Prefill/Decode 分离**，其中 E 指多模态编码处理，不是 draft，也不要与 CED 的语言模型 encoder 层混为一谈。

Encoder replay 在采用 prefix hit 且缺少需要的 SWA 窗口时，回卷命中尾部并与新后缀一起执行；decoder replay 则压缩晚层处理的 query 行。二者共享开关和部分状态，但触发条件、后端可用性和 KV 写入对象不同。

**不参与 prefix caching ≠ 不存 KV**。运行中请求仍需保存 SWA KV 供后续 chunk/decode 读取。**已有共享 global KV 不被覆写 ≠ 新后缀状态数学精确**：论文明确承认新后缀生成的 global/SWA KV 可随命中位置变化。正文不再使用“所有近似绝不进入持久状态”这一过强概括。

---

## 第 1 章：Ascend 固定快照的实现

### 1.1 主要 PR 脉络

原报告重点分析的六项，公开 PR 内容与当前快照实现相符；这里列出其职责，不把原工作区祖先测试记录伪装成本次执行结果。

| PR | 合入 commit（短 SHA） | 职责 |
| --- | --- | --- |
| [#17742](https://github.com/vllm-project/vllm-ascend/pull/17742) | `3a2e4a057` | encoder replay、SWA spec、逐 token 可见长度、共享 KV 写保护 |
| [#18322](https://github.com/vllm-project/vllm-ascend/pull/18322) | `b59bfc57c` | vLLM 0.30 legacy 兼容回移；后来删除 shim |
| [#17496](https://github.com/vllm-project/vllm-ascend/pull/17496) | `a58da927a` | decoder replay 尾部子批 |
| [#18750](https://github.com/vllm-project/vllm-ascend/pull/18750) | `05abc83bf` | FULL 图 padding 请求行对应的 replay metadata 长度与清零 |
| [#19034](https://github.com/vllm-project/vllm-ascend/pull/19034) | `5c9b1134d` | PCP 本地分段 metadata 容量；容量修复不等于 PCP replay 已支持 |
| [#18936](https://github.com/vllm-project/vllm-ascend/pull/18936) | `e8b7bc990` | V1/V2 执行面收拢和 V1 resumed request 的 replay_start 载荷补齐 |

上游 [#56227](https://github.com/vllm-project/vllm/pull/56227) 提供 encoder 协议；[#58132](https://github.com/vllm-project/vllm/pull/58132) 提供 decoder 框架。Ascend 在 v0.31.0 兼容环境中自带 decoder 实现是可观察事实；不能只凭合入日早于/晚于 release 日解释分支祖先关系。探测 `_mark_prefix_replay` 是确认 encoder 协议存在，并不是检测 decoder 协议。

### 1.2 四层链路与目录

```text
Scheduler prefix hit / replay_start
        ↓
AscendSlidingWindowMLASpec
  prefix_cacheable = not bounded_replay
  prefix_replay_tokens = sliding_window if bounded_replay else 0
        ↓
V1/V2 runner：请求边界、逐批 metadata、共享组 slot 写保护
        ↓
build_replay_swa_visible_lens → ori_topk_length
        ↓
band 模式向量 mask：left = max(left, right - rowLen + 1)
```

| 层次 | 固定快照代码 | 已核实职责 |
| --- | --- | --- |
| spec/入口守卫 | [core/kv_cache_interface.py](https://github.com/vllm-project/vllm-ascend/blob/5e40bc723165bfc8d8b1ec245ce5908a45dfd9d2/vllm_ascend/core/kv_cache_interface.py#L112) | 支持架构、配置开关、PCP>1 降级；不是原注释清单写的“无前缀缓存/PCP/窗口 0”三守卫 |
| 调度器兼容 | [platform.py](https://github.com/vllm-project/vllm-ascend/blob/5e40bc723165bfc8d8b1ec245ce5908a45dfd9d2/vllm_ascend/platform.py#L1471) | recompute、dyntra、profiling_chunk、balance_scheduling 等旧 schedule 路径关闭 replay 并 warning |
| V1 payload | [patch_swa_bounded_replay.py](https://github.com/vllm-project/vllm-ascend/blob/5e40bc723165bfc8d8b1ec245ce5908a45dfd9d2/vllm_ascend/patch/platform/patch_swa_bounded_replay.py#L17) | 注入 CachedRequestData.replay_start，恢复请求二次回卷时同步 worker；不重写上游回卷算法 |
| 可见区间 | [attention/dsa_v41.py](https://github.com/vllm-project/vllm-ascend/blob/5e40bc723165bfc8d8b1ec245ce5908a45dfd9d2/vllm_ascend/attention/dsa_v41.py#L96) | 按同一坐标计算每 token 可见长度；后续将其交给 band metadata |
| V1 执行 | [worker/v1/model_runner.py](https://github.com/vllm-project/vllm-ascend/blob/5e40bc723165bfc8d8b1ec245ce5908a45dfd9d2/vllm_ascend/worker/v1/model_runner.py#L3328) | replay 请求簿记、decoder 子批、DP 协调等 |
| V2 执行 | [worker/v2/model_states/default.py](https://github.com/vllm-project/vllm-ascend/blob/5e40bc723165bfc8d8b1ec245ce5908a45dfd9d2/vllm_ascend/worker/v2/model_states/default.py)、[engram/model_state.py](https://github.com/vllm-project/vllm-ascend/blob/5e40bc723165bfc8d8b1ec245ce5908a45dfd9d2/vllm_ascend/models/deepseek_v41/engram/model_state.py) | metadata 及 decoder state 接入 |
| cacheable 组写保护 | [worker/block_table.py](https://github.com/vllm-project/vllm-ascend/blob/5e40bc723165bfc8d8b1ec245ce5908a45dfd9d2/vllm_ascend/worker/block_table.py#L533) | replay 范围内共享组 slot 置 PAD；SWA 本身继续构建 |
| decoder 尾层执行 | [decoder_replay_layers.py](https://github.com/vllm-project/vllm-ascend/blob/5e40bc723165bfc8d8b1ec245ce5908a45dfd9d2/vllm_ascend/models/deepseek_v41/decoder_replay_layers.py#L84) | gather→运行晚层→scatter；本快照从 `cut+1` 起裁剪 |
| decoder 状态 | [decoder_replay_state.py](https://github.com/vllm-project/vllm-ascend/blob/5e40bc723165bfc8d8b1ec245ce5908a45dfd9d2/vllm_ascend/models/deepseek_v41/decoder_replay_state.py#L95) | eager/捕获/全行保留/DP 等条件与子批 metadata |
| gather kernel | [ops/triton/decoder_replay.py](https://github.com/vllm-project/vllm-ascend/blob/5e40bc723165bfc8d8b1ec245ce5908a45dfd9d2/vllm_ascend/ops/triton/decoder_replay.py#L39) | 保留行、SWA 起点、padding 行 slot；与 cacheable 组写保护不是同一件事 |

### 1.3 截断公式的真正算子落点

令请求内 query 绝对位置为 `i`、重放起点为 `s`、窗口大小为 `W`：

```text
lower(i) = max(0, s, i-W+1)
visible_len(i) = i-lower(i)+1
```

`build_replay_swa_visible_lens()` 实现上式；普通非回放请求的下界为 0，因而仍是正常因果 SWA。`ori_win_left=W-1` 给出窗口宽度上限，`ori_topk_length` 提供逐 token 的额外限制。

**实际 band 消费点在** [sparse_flash_mla_swa_block_vector.h](https://github.com/vllm-project/vllm-ascend/blob/5e40bc723165bfc8d8b1ec245ce5908a45dfd9d2/csrc/attention/sparse_flash_mla/op_kernel/arch22/sparse_flash_mla_swa_block_vector.h#L648)：有长度输入且 mask mode=4 时，以 `right-rowLen+1` 抬高左界，再把区间外分数置为屏蔽值。模式常量见 [sparse_flash_mla_tiling.h](https://github.com/vllm-project/vllm-ascend/blob/5e40bc723165bfc8d8b1ec245ce5908a45dfd9d2/csrc/attention/sparse_flash_mla/op_host/sparse_flash_mla_tiling.h#L94)。

原文的 `GetOriSparseActualSeqLen()` 不能作为 dense band 截断的最终证据：它先检查 `hasOriSparseIndices`，服务显式稀疏索引路径；dense band 走另一分支。见 [kernel 分支](https://github.com/vllm-project/vllm-ascend/blob/5e40bc723165bfc8d8b1ec245ce5908a45dfd9d2/csrc/attention/sparse_flash_mla/op_kernel/arch22/sparse_flash_mla_swa_kernel.h#L887)。读到 null block 即使值为零也会污染 softmax 权重；同时不能假定所有后端的 null KV 始终为零。

这条链路证明该 arch22 实现具有相应 mask 逻辑；未在本次 NPU 上运行，不能外推所有 A3/A5 算子分派和打包格式均已验证。

### 1.4 Decoder 裁剪和支持边界

`cut=max(kv_source_layer_ids)`；本快照 `decoder_replay_start=cut+1`，最后 KV-source 层仍对整批执行。后来的 [#19926](https://github.com/vllm-project/vllm-ascend/pull/19926) 才把 cut 层“整批写 KV”与“裁剪剩余计算”拆开。原表“从 cut 层起裁剪”不适用于本快照。

`make_decoder_replay()` 会检查 SP without DSA-CP、PP、PCP、DCP、microbatching、图模式、cut 后 Engram、draft 类型/窗口/层类型和 LoRA 等条件。允许图模式为 NONE 或 FULL_DECODE_ONLY，**这是 prefill eager 的约束，不等于普通 decode 完全不能用图**。配置开关开启也不保证每一步发生 trim。

运行时还检查捕获/profiling、是否能同步 DP、是否存在可裁剪行，以及 prompt-logprob 等全行消费者。短 query 或必须保留全行的请求不会压到更短。scatter 后未保留行置零仅是输出布局处理，**不能用它证明 SWA KV 没有保存或不落 prefix cache**；后者须看 cache spec 和写入路径。

**DP 修正**：固定快照 V1 `_prepare_decoder_replay()` 和 V2 `DecoderReplayState.prepare()` 已有 `data_parallel_size>1` 握手与 token-count 协调，工厂也没有一刀切的 DP>1 禁用。因此原文“此版本无 DP，#20210 首次支持 DP”错误。[#20210](https://github.com/vllm-project/vllm-ascend/pull/20210) 具体扩展的是 **MRV2 PCP×DP 组合、不均衡和 idle 布局**。存在旧 DP 代码仍不代表任意拓扑都通过验证。

**PCP 修正**：本快照 `resolve_bounded_replay()` 和 decoder 工厂明确拒收 PCP>1。[#19034](https://github.com/vllm-project/vllm-ascend/pull/19034) 的 2× metadata 容量修复与后来的 [#19321](https://github.com/vllm-project/vllm-ascend/pull/19321) PCP 使能必须分开。

### 1.5 论文到实现的映射

| 论文/设计点 | 核验后结论 |
| --- | --- |
| SWA 区间 `[max(s,i−W+1),i]` | Python 可见长度与 arch22 band 向量 mask 对应；见 1.3 |
| 回放尾部 W，而非 L×W | 上游调度器回卷量取 `min(W,hit_tokens)`；短 hit 另有全量重算保护。W 是该模型的配置，不是所有场景无条件恰好执行 W 行 |
| encoder replay 与新后缀一起处理 | 论文 §3.2.2 本来就如此描述；框架拼接前向并非偏离论文的另一条原生路线 |
| 复用命中的 global KV | cacheable 组 slot PAD 保证不覆写；不能由此推出所有 global 投影算术都被跳过 |
| SWA 退出 prefix caching | spec 对应；运行中窗口仍保存在设备 KV cache，不能写成“完全不存” |
| decoder replay | 晚层保留尾窗 query；实际执行受图模式、打分/路由消费者和并行条件约束 |
| DRAM 短期池与 global 长期存储 | 论文服务系统的部署策略；所审 replay 模块没有实现完整的 10% host DRAM/分钟 TTL 管理，不能因此否定整个 connector 生态的主存池能力 |
| train-aware adaptation | 论文在 decoder replay 段说明 post-training 模拟；推理代码没有能力替代训练验证 |
| SWA 精度 | 论文 §2.4.4 保留 SWA FP8、global 用 FP4。Ascend A5 packed 分支明确设 FP8 payload+BF16 scale，非“完全由模型配置选择、框架从不硬编码” |
| W=128 | 论文 §4.2.1 的模型设置；框架读配置并受实现约束 |
| global 约 1/4、persistent 约 1/8 | 是相对 DeepSeek-V4-Flash 的不同统计口径及组合结果；不能机械拆成 replay 单独贡献统一的“再减半”，更不是任意服务实例实测值 |

论文依据：[§3.2.1–3.2.2](https://arxiv.org/html/2609.19969v1#S3.SS2.SSS1)、[§2.4.4](https://arxiv.org/html/2609.19969v1#S2.SS4.SSS4)、[§4.2.1](https://arxiv.org/html/2609.19969v1#S4.SS2.SSS1)。A5 格式依据：[make_swa_cache_spec](https://github.com/vllm-project/vllm-ascend/blob/5e40bc723165bfc8d8b1ec245ce5908a45dfd9d2/vllm_ascend/models/deepseek_v41/cache_config.py#L44)。

### 1.6 不能宣称“完全对应”的边界

1. **系统存储策略不等价**：移出 prefix cache 不意味着所有会话每个 chunk 都要 miss→replay；连续运行请求仍保留窗口，P/D 可能直接传窗口。论文的短期主存池是一套独立部署设计。
2. **近似边界必须说准确**：保护的是已命中的共享 global KV；新后缀 KV 可受回放影响。原文所谓“论文 §4.1 三条纪律”是二次归纳且过度泛化，论文 §4.1 实际是 Data Construction。
3. **删去错误的 hidden-state 路线对比**：论文没有提出“必须缓存每层输入 hidden states、只算 KV 投影、不执行 attention”作为 encoder replay 原生路线。若采用这类工程方案，应单独论证状态来源与成本，不能归于论文。
4. **训练与运行验证分开**：单元测试和少量生成结果不证明对任意模型、RL/logprob 或长文本工作负载无损。原文把 SMMU fault 归因于“未训练适配”没有因果证据，已删除；硬件地址错误需要独立定位。
5. **文档可能滞后于代码**：固定快照教程 §12 仍介绍 legacy v0.30 shim，而实际补丁已改为 native encoder 协议下的 V1 payload 补齐；教程 §11 的“只支持 V1”也与快照已有 V2 实现不一致。不能把教程一句话当成该 SHA 的完整能力矩阵。[教程快照](https://github.com/vllm-project/vllm-ascend/blob/5e40bc723165bfc8d8b1ec245ce5908a45dfd9d2/docs/source/tutorials/models/DeepSeek-V4.1-Flash.md#L1827)

---

## 第 2 章：上游 vLLM v0.33.0 实现

### 2.1 已核对的主要演进项

以下是机制相关的代表项，按 **GitHub merged_at 转北京时间** 排序；不再称“15 个全集”。原清单把无关 ViT 图 replay 和 KV 事件消息重放混入，同时漏列真正有关的 ROCm decoder PR。

| PR | 合入 commit | 合入日（北京时间） | 内容 |
| --- | --- | --- | --- |
| [#56227](https://github.com/vllm-project/vllm/pull/56227) | `1dc2d854c1` | 2026-09-19 | encoder 基础协议：请求边界、调度回卷、spec 与写保护；v0.31.0 已有。 |
| [#57906](https://github.com/vllm-project/vllm/pull/57906) | `04c1f4a407` | 2026-09-21 | 禁用 ROCm 的 encoder prefix-cache replay；不能与后续 decoder 使能混淆。 |
| [#58132](https://github.com/vllm-project/vllm/pull/58132) | `4b2e1cfa7b` | 2026-09-29 | decoder 子批裁剪、状态与图支持基础；v0.31.0 没有对应文件，v0.32.0 已有。 |
| [#59532](https://github.com/vllm-project/vllm/pull/59532) | `eb2901b735` | 2026-10-06 | replay 层 CUDA 图、PIECEWISE 阈值切换及 DP 协调。 |
| [#59197](https://github.com/vllm-project/vllm/pull/59197) | `ef63c23d35` | 2026-10-08 | 加载带齐 SWA 窗口时跳过回放；P/D 的 N−1 布局配套。 |
| [#59894](https://github.com/vllm-project/vllm/pull/59894) | `ad73a4740d` | 2026-10-09 | 最后 KV-source 层先写整批 KV，再裁剪该层其余计算及后续层。 |
| [#61012](https://github.com/vllm-project/vllm/pull/61012) | `2339bd39d6` | 2026-10-11 | Mooncake 按 EAGLE replay 边界保留 SWA 尾部，属于 connector 邻接修复。 |
| [#61041](https://github.com/vllm-project/vllm/pull/61041) | `ffec221276` | 2026-10-12 | replay input_ids 的 dtype 与 FlashInfer MegaMoE 对齐。 |
| [#59039](https://github.com/vllm-project/vllm/pull/59039) | `c077dc663d` | 2026-10-13 | 开启 ROCm decoder replay；此时 encoder prefix-cache replay 仍关闭。 |
| [#61643](https://github.com/vllm-project/vllm/pull/61643) | `d12d5460a5` | 2026-10-15 | 请求返回 routed experts 时保留全部路由行；不是“防负载失衡”的通用修复。 |
| [#62183](https://github.com/vllm-project/vllm/pull/62183) | `fc47cc8fff` | 2026-10-19 | PCP 下 APC 与 encoder replay，APC=Automatic Prefix Caching。 |
| [#62652](https://github.com/vllm-project/vllm/pull/62652) | `85752a43c3` | 2026-10-21 | prompt_logprob_token_ids 目标 token 打分保留行。 |
| [#62834](https://github.com/vllm-project/vllm/pull/62834) | `ddc3f417d4` | 2026-10-22 | EPD 的多模态 encoder embedding 就绪检查覆盖回放前缀尾部。 |
| [#62384](https://github.com/vllm-project/vllm/pull/62384) | `c7476c7a6f` | 2026-10-24 | PCP 下 decoder replay 与 APC。 |

排除项：[#58499](https://github.com/vllm-project/vllm/pull/58499) 是 ViT CUDA graph 元数据同步；[#63224](https://github.com/vllm-project/vllm/pull/63224) 是 ZMQ KV-event 历史消息重放发送队列，不是 SWA 重算。它们不能计入 SWA bounded replay 实现数量。[#63155](https://github.com/vllm-project/vllm/pull/63155) 等后续图内存优化也说明仅靠 `git log --grep=replay` 不能宣称穷尽机制。

### 2.2 Encoder 协议链

`CacheConfig.swa_bounded_replay` → SWA spec 的 `prefix_cacheable/prefix_replay_tokens` → scheduler 记录 `Request.replay_start` 并回卷请求 computed 计数 → 请求载荷传 worker → metadata 下界和 cacheable 组写保护。模型和后端决定特性是否真正启用；不是“从配置到算子全部跨后端共享”。

[v0.33.0 scheduler](https://github.com/vllm-project/vllm/blob/v0.33.0/vllm/v1/core/sched/scheduler.py#L3438) 中：

```text
若没有 replay window：不回卷
若 connector 已加载全部所需 SWA 组：replay_start=0，不回卷
否则：replay_tokens=min(window, hit_tokens)
      replay_start=hit_tokens-replay_tokens
```

`ec_computed_tokens` 用于多模态 encoder embedding 就绪检查，不是“每个 global KV 组单独的 computed 计数”。已有 global KV 靠 slot 写保护复用。短命中不值得回放时重算、失败加载后的恢复、完全命中保留采样 token等是独立边界。

[attention.py](https://github.com/vllm-project/vllm/blob/v0.33.0/vllm/models/deepseek_v41/attention.py#L538) 显示该 tag 的 encoder replay 需要 MRV2；PCP 还受 prefix caching 条件约束；ROCm 在这个 tag 中仍关闭 encoder 路径。**上游 V1 没有因此“退役”**，只是这个特性不对其启用。

### 2.3 Decoder 执行

| 层次 | 代码 | 要点 |
| --- | --- | --- |
| 子批执行器 | [DecoderReplayLayers](https://github.com/vllm-project/vllm/blob/v0.33.0/vllm/models/deepseek_v41/decoder_replay_layers.py#L1) | gather/scatter、forward context、图断点；这是模型执行适配层，不是引擎 scheduler |
| NVIDIA 模型装配 | [nvidia/model.py](https://github.com/vllm-project/vllm/blob/v0.33.0/vllm/models/deepseek_v41/nvidia/model.py#L855) | cut 层先整批写 KV，再把该层剩余计算及后续层裁剪 |
| AMD 模型装配 | [amd/model.py](https://github.com/vllm-project/vllm/blob/v0.33.0/vllm/models/deepseek_v41/amd/model.py#L580) | decoder 由 cache_config 开关与 MRV2 等条件独立控制，不依赖 encoder 的 forced-off 标志 |
| 共享状态 | [nvidia/model_state.py](https://github.com/vllm-project/vllm/blob/v0.33.0/vllm/models/deepseek_v41/nvidia/model_state.py#L281) | AMD 的 get_model_state_cls 也返回此类；目录名不代表仅 NVIDIA 使用 |
| ROCm 可见范围 | [amd/ops/swa_decode_meta.py](https://github.com/vllm-project/vllm/blob/v0.33.0/vllm/models/deepseek_v41/amd/ops/swa_decode_meta.py#L87) | 每 query 起点与 replay_start 取 max；存在该 kernel 不代表所有 ROCm encoder 路径已经使能 |

在 v0.33.0，FULL 和小 PIECEWISE 图保整批；达到阈值的 PIECEWISE 可通过断点进入 replay 子批，阈值默认 768，具体由配置控制。普通 eager 步在有收益时裁剪。需要 prompt logprobs、`prompt_logprob_token_ids` 或 routed experts 的请求保留全部所需行。DP 任一 rank 需 replay 时协调执行，但仍保留各 rank 的布局和计数约束。

[#61643](https://github.com/vllm-project/vllm/pull/61643) 的目的在于返回完整路由记录；[#62652](https://github.com/vllm-project/vllm/pull/62652) 的目标是 prompt target-token scoring。原文把它们概括为“避免专家负载失衡”和“MTP/EPD verify 掉行”均不准确。

### 2.4 P/D 与 Mooncake：原文反转逻辑的修正

v0.33.0 的判断是：**加载本请求自身的 KV 且包含全部 replay SWA 组，就保留窗口并跳过 encoder bounded replay**；prefix-cache store/offload 若不带这些组，则仍需回放。决定因素是 `get_loaded_kv_cache_group_ids(request)`，不能一概说“SWA 从不过 connector”。

Mooncake 的 `_bounded_replay` 从 KV 配置识别是否存在 replay group，**不是判断当前 batch 一定有正在回放的请求**。`_get_remote_prefill_token_count()` 是 D 侧计数辅助函数，N>1 时可返回 N−1；配套 P 侧请求截短，让 D 用按 N−1 正确布局的窗口计算最后 token。它不是“P 端随意扔掉最后 KV，因此 D 缺窗口重放 W token”。

来源：[scheduler 窗口恢复判定](https://github.com/vllm-project/vllm/blob/v0.33.0/vllm/v1/core/sched/scheduler.py#L3438)、[Mooncake N−1 说明](https://github.com/vllm-project/vllm/blob/v0.33.0/vllm/distributed/kv_transfer/kv_connector/v1/mooncake/mooncake_connector.py#L1002)、[#59197](https://github.com/vllm-project/vllm/pull/59197)。该修复晚于 v0.31.0 对照源码，**不能把 v0.33.0 的 window_loaded 协议写成 Ascend v0.31.0 已直接继承**。

Mooncake 名称也不保证底层一定是 SSD；内存池、传输、offload/store 的具体部署需看 connector 类型及配置。

### 2.5 固定版本对照

| 项目 | vLLM v0.31.0 | vLLM v0.33.0 | Ascend `5e40bc723…` |
| --- | --- | --- | --- |
| encoder 基础协议 | 有，MRV2 路径 | 有并增补 connector/PCP 边界 | 接上游协议，自有 spec 与执行/算子适配；额外 V1 payload patch |
| decoder trim | 对照文件不存在 | 有 | 自有实现；从 cut+1 起，不能与新 main 的 cut 层拆分混用 |
| PCP replay | 原守卫关闭 | 有相关实现及条件限制 | PCP>1 关闭 |
| DP replay | 不在本表推断完整支持矩阵 | 状态层有协调 | V1/V2 已有协调；后续 #20210 增加 PCP×DP，不能叫旧版“无 DP” |
| ROCm | encoder 关闭；AMD 模型文件已存在 | encoder 仍关闭，decoder 已开启 | 不适用 |
| replay prefill 图执行 | 无 decoder replay 文件 | PIECEWISE/独立 replay 图支持 | 固定快照 replay prefill 要 eager；普通 decode 可使用 FULL_DECODE_ONLY |
| 后续 bugfix | 不能用最新 PR 反推 | 不保证“全部修完” | #20394 scoring、#19926 cut/compile 等后续修复需单独核对 |

### 2.6 对应论文的结论

核心近似窗口和两类执行策略有静态实现对应；分布式缓存寿命策略、训练适配和全面运行验证不由这些代码自动保证。ROCm 两类 replay 的支持边界不同，且新主线已继续演进（第 7 章）。不能再把“上游 v0.33.0”描述为“双后端全部功能/所有 bugfix 全集”。

---

## 第 3 章：原配图交付记录与当前可用性

原报告称在旧工作区 `TASK/report/diagrams/` 生成 5 张图、每图 PNG/SVG/Excalidraw/Excalidraw.md 四格式：总体链路、encoder、decoder、论文映射、降级矩阵。

**本次未找到该目录对应的原始 20 文件及 QA 日志，不能复核“5/5 PASS、nativeIssues=0、全部 PNG 3000 宽”等历史结果。** 原报告的 `diagrams/...` 相对链接在当前 `hw_md` 下没有目标，已去除失效交付链接。当前工程另有 `swa_diagrams/` 等产物，但没有证据证明它们就是这五张图，不能混用。

旧图若再次用于汇报，至少需同步本文修正：SWA prefix caching 与运行时 KV 的区别、cut 与 cut+1、band 真正消费点、P/D 带窗口跳过回放、DP 支持边界。本次只修订目标报告，没有冒称已同步修图。

## 第 4 章：原注释版文件记录与证据边界

原报告记录旧双仓库旁有 22 份 `.annotated`（Ascend 16、vLLM 6），包括 scheduler/spec、runner、decoder state、attention 和 csrc。当前工作区没有这些原始文件，因此无法确认其数量、提取范围、内容与“零改既有源码”的历史执行结果。

本次下载的公开源码位于 `references/swa-audit-20261112/`，用于复核本文，**不是重新生成的那 22 份注释版**。若恢复原注释资产，特别需要修正：

- `resolve_bounded_replay` 的守卫描述应为架构、开关、PCP，而非旧表的“无 prefix caching/PCP/窗口 0”。
- `patch_swa_bounded_replay.py` 实际函数是 `_upstream_carries_replay_start`、`_patch_make_cached_request_data` 等；原表 `_load_replay_start` 不能作为此文件的现行函数锚点。
- dense band 的最终消费证据要加入 `sparse_flash_mla_swa_block_vector.h`，不能只引用 sparse-length 函数。
- Ascend 旧快照已经有 DP 协调，不能把 DP 一律记成后续版本新增。

---

## 第 5 章：omniinfer 移植记录与可采纳建议

### 5.1 原侦察记录的限制

原对象为 `D:\Project\code\REPO\GITEE\omniinfer_lyh_ced_dev\omniinfer`，分支 `feature/ced_dev`、HEAD `713a7556`；本次该路径不可用，未获得可公开对应的同 SHA 源码。因此“omni_npu 插件、VLLMPatch、ShareKVSlidingWindowSpec、npu_pangu.py 特定行”等均是**原报告的历史侦察记录，尚未独立复核**，不能据此保证当前 omniinfer 行为。

| 原结论 | 修订后的证据要求 |
| --- | --- |
| 基线一定是 v0.25.x，docs v0.9.0 一定过时 | 注释提到 0.25.1 和 import 风格只是线索；需该环境锁文件、镜像构建配置、实际安装版本及完整 SHA |
| 字段“首现 v0.33.0” | 错误；上游 #56227 已引入，v0.31.0 源码已有。不等于已核实私有 v0.25.1 树完全没有回移 |
| getattr 引用意味着已有协议 | 错误；getattr 的默认值只能说明容错，必须检查字段定义和传输链 |
| CED 与 bounded replay 天生互斥 | 错误；论文 decoder replay 正依托 CED。旧私有实现如果设置互斥，那是该实现的局部限制 |
| Python 一定无法表达窗口地板，必须改算子 | 证据不足；需核对具体 CANN/torch_npu ABI、mask 能力、ragged 长度、块表 rebasing、prefix/chunk 语义和性能约束 |

### 5.2 推荐的移植拆分

1. **锁定环境**：模型结构、完整源码 SHA、vLLM/CANN/torch_npu 版本、目标 NPU、runner 和图模式。
2. **选择变体**：encoder 重点是命中回卷、共享 KV 写保护和窗口下界；decoder 重点是晚层裁剪、KV-source 边界和全行消费者。原报告第 5 章写“一期 encoder”，后文却写“一期 decoder”，无法从缺失的决策报告确定最终拍板；此处不擅自选定。
3. **验证协议链**：调度、请求载荷、spec/cache manager、runner metadata 同步；旧版缺失时按其扩展机制适配。
4. **检查全部消费者**：prompt logprobs、目标 token scoring、返回 routed experts、draft hidden states、PP/PCP/SP/DP、Engram 和图捕获。
5. **分级验证**：短 hit、W 边界、跨 chunk、恢复/抢占、混合批、idle DP 和 P/D window-loaded。需要与冷前向比较近似影响，不强求论文未承诺的逐位相同。

可先用全前缀重算验证管线和数据结构，但这只是一种参考/兜底模式，**不证明 W-token bounded replay 数值等价或性能收益**。

### 5.3 算子候选需求（待目标环境确认）

| 候选项 | 定位 |
| --- | --- |
| R1：per-request replay_start 或 per-token 可见长度 | 当现有接口确实无法表达不同请求的左界时，考虑新增；不是已确认必须改 ABI |
| R2：tiling metadata/图缓冲与 R1 同步 | 如果 R1 改动影响 metadata，需一起设计和验证 |
| R3：稀疏 top-k 与窗口地板交集 | 需要用目标算子的具体索引/坐标语义论证，不能只靠同名 sparse_mode=4 推断 |

sink、量化、块布局和 null/PAD 语义需逐项对齐。“两个算子都有 band 模式=4”不证明其 ABI 等价。输入 `s=0` 应恢复标准 SWA 掩码，但不自动推出所有硬件结果逐位相同。

### 5.4 历史交付物

旧路径 `TASK/report/20261109_135724_SWA_Bounded_Replay_omniinfer移植工作计划.md` 与 `TASK/report/20261109_141949_SWA_Bounded_Replay_算子需求报告.md` 本次不可用；本报告不能声称已经同步修正这两份文件，也不能把其决策号 D0/D6/T3.5 当成当前已确认实施范围。

---

## 第 6 章：SP × Replay 两个候选的核验比较

截至 2026-11-12，至少确认两个直接相关 PR：[#20266](https://github.com/vllm-project/vllm-ascend/pull/20266) 与 [#21129](https://github.com/vllm-project/vllm-ascend/pull/21129)，均未合入。原文“有且只有一个”来自标题关键词漏检；不能用这种检索结果证明全社区唯一。

| 维度 | #20266 | #21129 |
| --- | --- | --- |
| 目标分支 | main | releases/v0.32.0rc |
| 核心方式 | all-gather 原 SP states 后选行、pad、再分片 | 本地选保留行后变长 all-to-all；目的端 pad，逆交换恢复 |
| 适用范围（PR 自述） | MRV2 eager，含 PCP/DP 组合，仍受其他守卫限制 | MRV2、PCP=1、eager prefill；SP+PCP、MRV1 SP、adaptive verification 不支持 |
| 验证证据 | 作者报告 8 NPU：TP4/DP2/PCP1、TP2/PCP2/DP2；每组 E2E 18/18；固定 48 题分别 43/48、44/48 | 当前作者报告 macOS CPU harness **78 passed**，400 个随机 ownership 用例与 Gloo 通信；未报告本地复现原生 Ascend 崩溃或完成 NPU E2E |
| 最新提交状态 | 页面 1 commit；仍有冲突提示，不能直接当作可合版本 | 页面 **8 commits**，11-11 ready for review；11-12 增补类型注解提交，末项短 SHA `618d079` |
| 编译行为 | 需按其 main 基线审查 | active SP/non-SP replay 显式绕过 torch.compile 的模型入口；普通 warmup/decode 保持编译 |

上述 4096→128、TP8 每 rank 16 行只是 #21129 的**布局示例**，不是端到端通信量统一降低 32 倍，更不是实测加速；最终输出 gather 等成本仍在。其 CPU 测试修复包括 fork worker 禁用 vLLM `PIN_MEMORY`，不能据此推断作者是否拥有 NPU。

**建议**：同 `v0.32.0rc` 且 PCP=1 时可优先阅读 #21129 的紧凑交换设计；需要 PCP 组合时 #20266 更相关。二者代码和测试不可互相代替：#20266 跑通不能证明 #21129 的 HCCL 路径正确。没有同环境、同模型、同输入的 A/B，就不能裁定哪一个更快、更成熟或更可能合入。作者履历、commit 数、分支名带 `codex` 都不是技术正确性的证据。

原文把 #21129 的 torch.compile runtime dispatch 等同 ACLGraph 捕获问题也过强：两者相关但分属不同执行边界；该 PR 自述 replay prefill 保持编译不在范围内。需要 PIECEWISE replay 图时另看 #19286，不能把它当成 #21129 已提供的能力。

---

## 第 7 章：社区清单与截至 2026-11-12 的增量

### 7.1 Ascend 已核对清单

本表选取 **30 项直接 replay 实现、修复和部署文档 PR：已合入 13，open 11，关闭未合入 6**。范围是本次已核对清单，不声称穷尽。搜索 `replay in:title` 本次返回 96 项，名称相同但机制不同的内容不自动纳入。

原表实际有 26 行而非 25，历史状态按其行内容是 10 merged/13 open/3 closed-unmerged；还漏掉正文已有的 #19034，以及 #19901/#20264。当前表排除 standalone MTP 边界 #19666、补入上述三项和 #21477/#21478，并更新 #21151 已关闭。

| PR | 创建日（北京时间） | 本次状态 | 标题 | 核验说明 |
| --- | --- | --- | --- | --- |
| [#17365](https://github.com/vllm-project/vllm-ascend/pull/17365) | 2026-09-23 | open | [WIP][Feature] Support encoder-side SWA Bounded Replay for DeepSeek-V4.1-Flash | encoder WIP；与合入方案重叠，不能据长期 open 断言已被放弃。 |
| [#17496](https://github.com/vllm-project/vllm-ascend/pull/17496) | 2026-09-24 | 已合入 2026-10-15 | [Feature][Model] Support DeepSeek V4.1 decoder bounded replay | decoder 尾窗裁剪初版。 |
| [#17742](https://github.com/vllm-project/vllm-ascend/pull/17742) | 2026-09-29 | 已合入 2026-10-11 | [Feature][Model] Support DeepSeek V4.1 Endoder SWA bounded replay | encoder 协议适配、可见长度与写保护。 |
| [#18221](https://github.com/vllm-project/vllm-ascend/pull/18221) | 2026-10-10 | 关闭未合入 | [Feature][Model] Support DeepSeek V4.1 SWA bounded replay on the Ascend V1 runner | 关闭未合入；讨论指向 #17742。 |
| [#18244](https://github.com/vllm-project/vllm-ascend/pull/18244) | 2026-10-10 | 关闭未合入 | [Feature][Model] Support DeepSeek V4.1 bounded replay on Model Runner V2 | 关闭未合入；不能当作已交付 MRV2 支持。 |
| [#18286](https://github.com/vllm-project/vllm-ascend/pull/18286) | 2026-10-11 | open（draft） | [Feat][Model] Integrate DeepSeek V4.1 bounded replay and A5 inference compatibility | replay 与 A5 兼容的大集成，不能整体视为稳定基线。 |
| [#18290](https://github.com/vllm-project/vllm-ascend/pull/18290) | 2026-10-11 | open | [Feature] Support DeepSeek V4.1 decoder bounded replay on MRV1 and MRV2 | MRV1/MRV2 rework，仍是未合入候选。 |
| [#18322](https://github.com/vllm-project/vllm-ascend/pull/18322) | 2026-10-11 | 已合入 2026-10-12 | [BugFix][Model] Backport SWA bounded replay to vLLM 0.30 | vLLM 0.30 兼容回移；后续代码已删除 legacy shim。 |
| [#18360](https://github.com/vllm-project/vllm-ascend/pull/18360) | 2026-10-12 | open（draft） | [BugFix][Model] Fix decoder replay correctness and vLLM 0.30 compatibility | 依赖未合入 rework 的兼容/正确性候选。 |
| [#18750](https://github.com/vllm-project/vllm-ascend/pull/18750) | 2026-10-14 | 已合入 2026-10-15 | [BugFix][MRV2] Keep replay metadata aligned with FULL graph padding | FULL 图请求 padding 与 replay metadata 对齐。 |
| [#18936](https://github.com/vllm-project/vllm-ascend/pull/18936) | 2026-10-16 | 已合入 2026-10-19 | [Feature][Model] Extend DeepSeek V4.1 decoder bounded replay across model runners | V1/V2 执行路径及 resumed request payload 收拢。 |
| [#19034](https://github.com/vllm-project/vllm-ascend/pull/19034) | 2026-10-16 | 已合入 2026-10-19 | [BugFix][PCP] Size DSV4.1 replay metadata for local segments | PCP 本地前后段的 metadata 容量修复；不能等同 PCP replay 已使能。 |
| [#19286](https://github.com/vllm-project/vllm-ascend/pull/19286) | 2026-10-20 | open | [Feature][Model] Support DeepSeek V4.1 decoder replay PIECEWISE ACL graphs | PIECEWISE ACL replay 图候选；有早期 A5 小样本测试，最终 head 性能验证仍待完成。 |
| [#19321](https://github.com/vllm-project/vllm-ascend/pull/19321) | 2026-10-21 | 已合入 2026-10-23 | [Feature][PCP] Support DeepSeek V4.1 SWA bounded replay with PCP | PCP replay 扩展；随后 #20210 放开 PCP×DP 组合。 |
| [#19901](https://github.com/vllm-project/vllm-ascend/pull/19901) | 2026-10-27 | 关闭未合入 | [Perf][DSv4.1] Trim the last KV-source layer in decoder replay | 旧 cut-layer trim 候选，10-29 关闭未合入；原全景遗漏。 |
| [#19926](https://github.com/vllm-project/vllm-ascend/pull/19926) | 2026-10-27 | 已合入 2026-11-03 | [Performance][DSv4.1][no piecewise] Trim the last KV-source layer in decoder replay | 最后 KV-source 层裁剪，并包含 #20475 的 compiled replay 修复。 |
| [#20124](https://github.com/vllm-project/vllm-ascend/pull/20124) | 2026-10-28 | open（draft） | [Performance][DSV4.1] Use single-token decoder replay for DeepSeek-V4.1 prefill | 单行 replay 草案；原参照 #62817 已关闭，现应对照上游合入的 #65524。 |
| [#20210](https://github.com/vllm-project/vllm-ascend/pull/20210) | 2026-10-29 | 已合入 2026-10-29 | [Feature][PCP] Enable DeepSeek V4.1 SWA bounded replay with DP | MRV2 PCP×DP 组合及不均衡/idle 批布局；不是首次引入所有 DP 逻辑。 |
| [#20264](https://github.com/vllm-project/vllm-ascend/pull/20264) | 2026-10-29 | 关闭未合入 | [BugFix][Model] Restore DeepSeek V4.1 decoder replay with torch.compile | compiled replay 候选，11-02 关闭转向 #20475；原全景遗漏。 |
| [#20266](https://github.com/vllm-project/vllm-ascend/pull/20266) | 2026-10-29 | open | [Feature] Support DeepSeek V4.1 sequence-parallel SWA bounded replay | main 基线的 SP gather→裁剪方案；作者报告 8 NPU 测试，仍未合入。 |
| [#20394](https://github.com/vllm-project/vllm-ascend/pull/20394) | 2026-10-31 | 已合入 2026-11-05 | [BugFix][Model] Preserve target-token scoring rows during DeepSeek-V4.1 replay | 保留 prompt_logprob_token_ids 所需行；不是泛指 MTP verify。 |
| [#20475](https://github.com/vllm-project/vllm-ascend/pull/20475) | 2026-10-31 | 关闭未合入 | [Bugfix][DeepSeek-V4.1] Keep bounded replay state dynamic under torch.compile | 关闭未合入，相关实现被 #19926 包含。 |
| [#20645](https://github.com/vllm-project/vllm-ascend/pull/20645) | 2026-11-02 | 已合入 2026-11-06 | [BugFix][KVOffload] Offload only prefix-cacheable KV cache groups | 合入 releases/v0.32.0rc；offload 组过滤及顺序对齐。 |
| [#20911](https://github.com/vllm-project/vllm-ascend/pull/20911) | 2026-11-04 | open（draft） | [BugFix][Model] Guard invalid DeepSeek V4.1 replay cuts | 草案：空 KV-source 守卫和 compressed cut 回退；未完成 NPU 验证。 |
| [#21129](https://github.com/vllm-project/vllm-ascend/pull/21129) | 2026-11-06 | open | [Feature][Model][v0.32.0rc] Enable compact SP replay and fix compiled replay for DeepSeek V4.1 | release 基线的 compact SP；当前 8 commits、作者报告 CPU harness 78 passed。 |
| [#21151](https://github.com/vllm-project/vllm-ascend/pull/21151) | 2026-11-06 | 关闭未合入 | [BugFix][ModelRunner] Bound idle-DP dummy extent during decoder replay | 11-12 已关闭且 merged_at=null；不能推断问题已经修复。 |
| [#21244](https://github.com/vllm-project/vllm-ascend/pull/21244) | 2026-11-07 | open | [Performance][MRV2] Reuse V4.1 SWA replay visible lengths | 可见长度构建复用的性能候选。 |
| [#21250](https://github.com/vllm-project/vllm-ascend/pull/21250) | 2026-11-07 | open | [BugFix][KVOffload] Bound DeepSeek V4.1 replay CPU cache allocations | CPU offload 分配边界候选。 |
| [#21477](https://github.com/vllm-project/vllm-ascend/pull/21477) | 2026-11-10 | 已合入 2026-11-11 | [Doc] Enable SWA bounded replay for A5 V4.1 colocated deployment | 11-11 合入 main：A5 同机部署示例显式加 replay 参数；仅文档变更。 |
| [#21478](https://github.com/vllm-project/vllm-ascend/pull/21478) | 2026-11-10 | 已合入 2026-11-11 | [Doc][v0.32.0rc] Enable SWA bounded replay for A5 V4.1 colocated deployment | 11-11 合入 releases/v0.32.0rc：同上文档回移。 |

状态依据：[GitHub search/issues 原始响应](../references/swa-audit-20261112/search-replay.json)及各 PR 页面；#20645 由合入事件补核。网页搜索缓存可能滞后，例如 #21151 的缓存页面仍显示 open，而本次直接 API 的 `closed_at=2026-11-12T07:22:44Z`、`merged_at=null` 已明确关闭。

### 7.2 版本归属和关闭关系

原“10 个 merged 中 8 个进 main”算术有误，其列举实际是 9 个 main+1 个 release。新增清单口径已变，不能沿用旧合计。

[#20645](https://github.com/vllm-project/vllm-ascend/pull/20645) 的页面显示 base 由 main 改为 `releases/v0.32.0rc` 后合入，不能无证据叫“release 专属 cherry-pick”；“合入哪个分支”也不等于该变更永远不在其他分支。[#21477](https://github.com/vllm-project/vllm-ascend/pull/21477) 和 [#21478](https://github.com/vllm-project/vllm-ascend/pull/21478) 的目标分支与合入事件则分别明确为 main 与 release。

原报告“v0.31/v0.32/main 包含哪些 SHA”的旧工作区表需要完整 Git 对象及固定分支 SHA 才能重新认证。本文已核对固定源码的行为边界；不把随时变化的 `releases/v0.32.0rc` 分支名当成不可变 release tag。

已核实的关闭关系：#20264 讨论明确转向 #20475；#20475 内容被 #19926 包含。#21151 关闭未合入的事实**不说明问题已由另一 PR 解决**。长期 open 的 WIP 也不宜称“僵死线”。

### 7.3 新合入的文档与邻接修复

- **A5 部署示例**：11-11，#21477/main 与 #21478/release 显式增加 `--swa-bounded-replay`。本次查看 diff 均为文档参数新增，不能解读为首次实现 runtime 功能。[main diff](https://github.com/vllm-project/vllm-ascend/pull/21477/files)、[release diff](https://github.com/vllm-project/vllm-ascend/pull/21478/files)
- **SWA block size**：[#21243](https://github.com/vllm-project/vllm-ascend/pull/21243) 11-09 合入 main，[#21355](https://github.com/vllm-project/vllm-ascend/pull/21355) 11-10 回移 release；让 V4.1 使用配置 block size，原表中 32/64/128 值映射相同，不代表新增值在所有硬件上已全面验证。
- **KV Pool 保存边界**：[#20853](https://github.com/vllm-project/vllm-ascend/pull/20853) 与 release 回移 [#21557](https://github.com/vllm-project/vllm-ascend/pull/21557)、v0.30 回移 [#21556](https://github.com/vllm-project/vllm-ascend/pull/21556) 于 11-12 合入；刷新被释放 SWA block ID，并等待非 layerwise 保存队列，防止旧 key 指向已复用槽。该问题涉及可缓存 SWA/RSWA，属于相关缓存正确性工作，不能笼统算作 bounded replay 核心算法；等待队列的性能影响仍需测量。[检索原始证据](../references/swa-audit-20261112/search-swa.json)

### 7.4 上游的新进展（不能回写到旧 tag）

| PR | 本次状态 | 对本报告的影响 |
| --- | --- | --- |
| [#64094](https://github.com/vllm-project/vllm/pull/64094) | 2026-10-31 合入 | gfx950/CDNA4 开启 encoder prefix-cache replay；与早先 #59039 的 decoder replay 区分 |
| [#65249](https://github.com/vllm-project/vllm/pull/65249) | 2026-11-06 合入 | 再扩展到 gfx942/CDNA3；不能继续将“ROCm encoder 全关”写成当前主线状态 |
| [#65524](https://github.com/vllm-project/vllm/pull/65524) | 2026-11-09 合入 | 单行 prefill replay 的简化实现；原参照 #62817 已关闭未合入，Ascend #20124 应与该最终合入路线比较 |
| [#66397](https://github.com/vllm-project/vllm/pull/66397) | 2026-11-09 合入 | replay attention builders 提前到 warmup，避免首个实际步中的同步构建；共享 state 也影响 CUDA |
| [#64921](https://github.com/vllm-project/vllm/pull/64921) | open | 非法 replay cut 守卫；Ascend #20911 仍是 draft，不是已验证适配 |
| [#65064](https://github.com/vllm-project/vllm/pull/65064) | open，堆叠候选 | batch invariance 下扩展重放范围与 staged exact replay；不能把提案语义当成现行 W-token 近似重放已精确化 |
| [#67076](https://github.com/vllm-project/vllm/pull/67076) | draft/open | ShardQ 前置流 token sharding 的新探索；不能等同 Ascend SP 两方案已经合流 |

状态及正文见 [上游搜索原始响应](../references/swa-audit-20261112/search-upstream-dsv41.json)。单行 replay 不只是把 W 改成 1：需要约束离 prompt 尾部的距离、后续 chunk 的 KV floor、图阈值和多模态/优先级等边界；移植应读 #65524 完整代码，不能照标题实现。

### 7.5 图执行验证的新增线索

[Ascend issue #21461](https://github.com/vllm-project/vllm-ascend/issues/21461) 将 P 侧 decoder replay tail 的 ACL 图性能验证关联到 #19286。它是 **issue，不是已合入 PR**。报告的 profiling 指向 tail 20–39 层周围的 Host 准备与 rank 到达差异；短尾计算会暴露已有 Host 开销，不能说 bounded replay 新增了全部间隙，也不能假定图回放能消除所有同步/通信成本。

#19286 最新正文提供早期 A5 的固定 48 题冒烟对比，但承认最终 PR head 尚未完成硬件/性能复验。原文把它仅称为“方向”信息不够完整；把它称作“性能已验证”同样不成立。

---

## 第 8 章：修正清单、来源与验证范围

### 8.1 主要修正

| 原问题 | 修正位置与结论 |
| --- | --- |
| “三纪律在论文 §4.1”“近似绝不进入持久状态” | 0.3、1.5–1.6：删除误引；区分共享旧 global KV 与新后缀 KV |
| “论文原生只算 KV 投影、不跑 attention” | 1.5–1.6：原文无此路线，论文明确一起处理 replay 与后缀 |
| “SWA 完全不存，每次/每 chunk 都 miss” | 0.3、1.6、2.4：仍有活动窗口与可携带窗口的 P/D |
| band kernel 证据引用错误 | 1.3：改为 vector mask 的实际下界钳制 |
| 旧 Ascend cut 起点错误 | 1.4：旧版 cut+1；cut 本层拆分是后续优化 |
| “旧 Ascend 无 DP” | 1.4、2.5：已有协调；#20210 是 PCP×DP 组合扩展 |
| “P/D 不带窗口所以必须 replay” | 2.4：窗口带齐则跳过，N−1 为布局/最终 token 配套 |
| ROCm encoder/decoder 混为一谈 | 2.3、2.5、7.4：分别说明旧 tag 与新主线 |
| “v0.31 无 amd/model.py”“V1 已退役” | 0.2、2.2：文件存在；只是特性有 MRV2 限制 |
| 版本 pin/发布日期/全部祖先验证过度断言 | 0.1–0.2、7.2：区分源码兼容、实际依赖、commit/tag/release 与历史验证 |
| MoE routing、target scoring、PCP/APC/EPD 释义错误 | 0.3、2.1–2.3：按代码与 PR 实际用途修正 |
| 15 个全集、25 个全景、8 个 main 等数量错误 | 2.1、7.1–7.2：排除同词异义、补漏，明确选集口径 |
| SP 唯一、作者履历推断优劣、另一个 PR 测试可背书 | 第 6 章：两候选分别评估，硬件结果不互相继承 |
| #21129 5 commits/77 tests 与 #21151 open 已过时 | 第 6–7 章：8 commits/78 passed；#21151 关闭未合入 |
| 移植“字段首现 v0.33”“CED 天生互斥”“必须改算子” | 第 5 章：改为已知上游历史与待环境确认的条件建议 |
| 缺失图/注释文件被称作当前已验收 | 第 3–4 章：仅保留历史记录，去除失效交付链接 |

### 8.2 本次执行的检查

- 全文阅读原报告，对论文、固定 Ascend 关键源码、vLLM v0.31/v0.32/v0.33 关键版本文件做对照；直接核对 band mask、cut 边界、DP 协调、P/D 与打分行。
- GitHub PR API、搜索 API 与网页正文交叉核对；遇到 API 限流使用公开 PR 页面补证，未把请求失败当成 PR 不存在。保存来源 URL、HTTP 状态与抓取时间；补充清单见证据目录中的各 `*-manifest.json`。
- 使用固定 SHA/tag 链接支撑历史行为；新分支和 PR 信息带本次日期。表格条目数量由脚本统计，不凭人工估算。
- 本次没有安装/运行 vLLM、Ascend 算子或模型服务，没有 NPU 性能/精度复现；也没有重新验收旧图和私有 omniinfer 源码。

**可信度结论**：核心机制和本次指出的错误已有一手原文/源码支持；运行正确性、性能数字、私有移植环境与历史产物不能保证“全部准确”。正文已将这些未证实内容降为明确的限制或待确认建议，避免继续把推断当事实。
