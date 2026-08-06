# Plonky3 深入原理讲解 PPT — 规划 TODO（v3）

> 生成 PPT **之前**的规划。基于已完成的 `plonky3-guide/` 10 章 md 文档。
> 工具：`codex-ppt` skill（image-based，每页一张图，最终 `assemble_ppt.py` 合成 `.pptx`）。
> **本文档仅为规划，尚未生成任何幻灯片或 .pptx。**

## ⚙ 已确认的决策

- ✅ **风格**：科研答辩风，**配色明快、丰富**（见 §3 调色板）。
- ✅ **不嵌入现有 SVG**：所有页（含机制图）一律由图像模型在统一风格内生成（也符合 codex-ppt 禁用本地绘图的硬规则）。
- ✅ **默认画图模型 = Qwen Image 3.0**（`qwen-image-3.0`）：已把 `~/.codex-ppt-skill/.env` 的 `CODEX_PPT_IMAGE_MODEL` 改为 `qwen-image-3.0`，并对 codex-ppt 做了最小补丁以支持 Qwen（见 §2.2）；**已实测出图成功**（2048×1152，16:9）。
- ⏳ 待定：页数、输出位置、是否立即开工（见 §7）。

---

## 1. 目标 · 受众 · 范围

| 项 | 说明 |
|---|---|
| **目标** | 把 Plonky3 从地基（有限域）到塔尖（STARK 全流程）讲深讲透，重**原理与过程** |
| **受众** | 有一定密码学/工程背景、想读懂现代 STARK 内部的开发者/研究者 |
| **源材料** | `plonky3-guide/01..10-*.md`（已订正素数、引用真实 trait/函数签名） |
| **形态** | image-based PPT，16:9，中文，图文并茂，统一"明快科研答辩"视觉 |
| **规模** | 目标 **约 40 页**（可精简 ~30 / 深挖 ~50） |
| **输出** | `plonky3-ppt/Plonky3深入原理.pptx`（待确认） |

---

## 2. 模型、约束与风险（Qwen Image 3.0）

### 2.1 中文/公式/机制图的精确度
每页由 Qwen Image 3.0 生成。**Qwen 没有 `quality` 参数**（`--quality` 会被静默忽略），所有页都是模型原生保真度。因此"质量控制"改为靠：**prompt 细致度 + `prompt_extend` 开关 + 尺寸 + 多轮重试**，而非质量档位。

| 内容类型 | 风险 | 对策 |
|---|---|---|
| 纯概念/分隔/封面（少文字） | 低 | 简洁 prompt，`prompt_extend` 可开 |
| 文字+要点页（中文短句、表格） | 中 | 每页≤5 短要点 |
| **公式/代码/机制图页**（FRI 折叠、FFT 蝶形、circle 群、流水线等） | **高** | 极细致 prompt + `prompt_extend=false`（避免自动改写偏离结构）+ 预留 1–2 轮重试 |

> **Qwen 的强项是中文文字渲染**（显著优于 gpt-image-2），这对中文原理讲解 PPT 是利好。但精密几何示意图仍难，QA 阶段逐页核对，必要时重生成；个别极精密图若多轮仍不达标，会**如实标注限制**而非糊弄。

### 2.2 已做的 codex-ppt 补丁（为支持 Qwen）
codex-ppt 原本硬编码只支持 gpt-image。为让 Qwen 作为默认模型跑通，做了**最小、保留 gpt-image 行为不变**的补丁：
- `scripts/image_gen.py`：`_validate_model` 放开（允许 `qwen-*`）；`_validate_size` 增加 Qwen 分支（接受 512–2048 范围任意 WxH）。
- `scripts/image_providers/atlascloud.py`：`atlascloud_model_for_operation` 不再给无 vendor 前缀的模型强加 `openai/`（Qwen id 为 `qwen-image-3.0`）；`_atlas_payload` 为 Qwen 转尺寸格式 `W*H`、并剔除 Qwen 不支持的 `quality`/`output_format`。
- ⚠️ **注意**：这些是改的 skill 安装目录下的文件，**codex-ppt 升级可能覆盖**。若被覆盖需重打。`prompt_extend` 开关目前 image_gen.py 未暴露，生成阶段如需关闭我会再加一个小透传。

### 2.3 质量与成本
- Qwen 单页约 **90–120s**（比 gpt-image-2 的 ~42s 慢约 2–3 倍）。
- 40 页全量串行约 **70–80 分钟**；codex-ppt 子代理可并行，但 AtlasCloud 并发可能受限，实际视情况。
- 单价约 $0.04/张（据模型列表）。

---

## 3. 视觉风格：明快丰富的科研答辩风

**调性**：科研答辩的结构感与严谨 + 明快多色的活力。浅底为主、章节页深色反衬；信息分块、网格规整、数据感强；多色用于区分概念/层级，而非装饰堆砌。

**调色板（统一全 deck）**
```
背景       #FFFFFF  / 面板 #F4F7FB / 分隔线 #E2E8F2
主色 靛蓝   #3B5BFF   （标题、主结构）
青         #06B6D4     翠绿 #10B981     活力橙 #F97316
品红       #EC4899     金 #F59E0B       紫 #7C3AED
文字主     #0E1525   文字次 #51607A
章节/封面   深靛 #16235A → #3B5BFF 渐变 + 亮色字
```
**版式**：16:9；标题大号无衬线粗体；每页一个核心观点；色块/图标区分要点；公式代码用等宽面板；机制图按多色体系绘制；不重复同一布局。
**尺寸**：Qwen 上限 2048，16:9 用 **2048×1152**（替代 gpt-image-2 时代的 2560×1440；中文文字渲染仍清晰）。

---

## 4. 视觉资产计划（全部 Qwen 生成，无 SVG / 无 Mermaid 外挂）

- 不复用现有 SVG，不单独渲染 Mermaid。
- 所有页由 Qwen 在 §3 风格内生成；机制图/流程图靠**精细 prompt** 让 Qwen 在 deck 风格内绘制（多色、分块、标注）。
- 高精密示意图页（FFT 蝶形、circle 群、FRI 折叠、11 步流水线）列为**重点页**：细致 prompt + `prompt_extend=false` + 多轮。

---

## 5. 详细分镜大纲（约 40 页）

> 每页：**标题 · 核心要点 · 视觉**。视觉 `[Qwen生成]`；标 ⚠ 的为精密图重点页（多轮）。

### 开篇（2）
- **P1 封面**：Plonky3 深入原理讲解 · 副标题 · `[Qwen生成]`（深靛渐变 + 亮色标题 + 概念图形）
- **P2 导览与学习路径**：8 部分 + 推荐路径 · `[Qwen生成]`（路径/流程图）

### 第一部分 · 背景与定位（4）
- **P3 可验证计算**：为什么"证明算对了"；简洁 vs 零知识 · `[Qwen生成]`
- **P4 SNARK vs STARK**：透明/后量子/可信设置对比 · `[Qwen生成]`（对比表）
- **P5 什么是 PIOP**：多项式+承诺+随机挑战；AIR+PCS+Challenger · `[Qwen生成]`
- **P6 Plonky3 是什么/不是什么**：toolkit 非 zkVM；域×哈希×PCS 三轴 · `[Qwen生成]`

### 第二部分 · 有限域（7）
- **P7 为什么用有限域 + 小域直觉** · `[Qwen生成]`
- **P8 四个主力域**：素数/2-adicity/扩域表（含订正） · `[Qwen生成]`（四色表格）
- **P9 2-adicity 为什么关键** · `[Qwen生成]`
- **P10 Mersenne31 的麻烦**：基域 2-adicity=1 · `[Qwen生成]`
- **P11 circle 群救场**：阶 p+1=2³¹、倍映射 (x,y)→(2x²−1,2xy) · `[Qwen生成]` ⚠（单位圆+倍映射几何）
- **P12 扩域塔**：M31→Complex→QM31 · `[Qwen生成]`（分层塔图）
- **P13 SIMD 打包**：一条指令并行 8 元素 · `[Qwen生成]`（寄存器分槽图）

### 第三部分 · 多项式、编码与 FFT（4）
- **P14 一列数 = 一个多项式** · `[Qwen生成]`
- **P15 Reed–Solomon 低度扩展 + 码率 ρ** · `[Qwen生成]`（n→m 冗余图）
- **P16 FFT 蝶形**：O(n log n)、twiddle · `[Qwen生成]` ⚠（8 点蝶形信号流）
- **P17 插值与数据流** · `[Qwen生成]`

### 第四部分 · AIR（4）
- **P18 AIR 直觉**：trace 表 + 约束 · `[Qwen生成]`（表格+窗口）
- **P19 三个 trait**：BaseAir/AirBuilder/Air · `[Qwen生成]`
- **P20 一个 eval 三处复用**：symbolic/prover/verifier · `[Qwen生成]`
- **P21 选择器 when_* + RowLogicAir** · `[Qwen生成]`

### 第五部分 · 多项式承诺与 FRI（7）
- **P22 什么是 PCS**：透明/后量子/低度测试 · `[Qwen生成]`
- **P23 承诺抽象三层**：Pcs/MultilinearPcs/Mmcs · `[Qwen生成]`
- **P24 Merkle 承诺 + 多重打开**：去重 multiproof · `[Qwen生成]`（树+鉴权路径）
- **P25 FRI commit 阶段**：反复折叠降次 · `[Qwen生成]` ⚠（折叠树+β+公式）
- **P26 FRI query 阶段**：随机抽查重算 + final poly · `[Qwen生成]`
- **P27 DEEP-FRI**：域外点 ζ + 商 + soundness 公式 · `[Qwen生成]` ⚠（曲线+域外点几何）
- **P28 STIR/Whir/Brakedown 对比** · `[Qwen生成]`（对比表）

### 第六部分 · STARK 全流程（5）
- **P29 配置 StarkConfig** · `[Qwen生成]`
- **P30 11 步证明流水线** · `[Qwen生成]` ⚠（流水线图）
- **P31 商多项式核心恒等式**：C/Z_H ⟺ 约束满足 · `[Qwen生成]`
- **P32 验证器检查清单** · `[Qwen生成]`
- **P33 multi-stark / batch-stark** · `[Qwen生成]`

### 第七部分 · 辅助协议（4）
- **P34 Challenger**：Fiat–Shamir 4 trait · `[Qwen生成]`
- **P35 Sumcheck**：2^m 求和归约到单点 · `[Qwen生成]`（逐轮折叠图）
- **P36 LogUp 查找**：对数导数 · `[Qwen生成]`（乘积→求和对照）
- **P37 三者协同** · `[Qwen生成]`

### 第八部分 · 工程与选型（4）
- **P38 性能四大支柱 + target-cpu=native** · `[Qwen生成]`
- **P39 哈希生态**：代数 vs 原生 · `[Qwen生成]`
- **P40 Plonky3 vs Plonky2/Halo2/Groth16** · `[Qwen生成]`（对比表）
- **P41 安全坑 + 选型决策树** · `[Qwen生成]`（决策树）

### 收尾（2）
- **P42 全链路回顾**：域→FFT→AIR→PCS/FRI→STARK · `[Qwen生成]` ⚠（总览图）
- **P43 参考资料 + 下一步 / Q&A** · `[Qwen生成]`

> 合计 **43 页**。可精简（合并 P10/P11、P25/P26、P34/P35 → ~36）或深挖（FRI/流水线/sumcheck 各拆 1–2 页 → ~50）。

---

## 6. codex-ppt 工作流 TODO 清单（确认后执行）

> 遵循 skill 审批门，每阶段产物需你确认再进下一阶段。

- [ ] **阶段 0 · 决策确认**（本 TODO）
  - [x] 风格：科研答辩风·明快丰富（§3）
  - [x] 不嵌入 SVG，全模型生成（§4）
  - [x] 默认模型 Qwen Image 3.0（已改 + 补丁 + 实测，§2.2）
  - [ ] 页数 / 输出位置 / 是否开工（§7）
- [ ] **阶段 1 · 准备**
  - [ ] 建项目目录 `plonky3-ppt/`
  - [ ] 基于本大纲写 `outline.md`（逐页：标题/要点/视觉/源章节）
  - [ ] 确认后端：`scripts/image_gen.py`（atlascloud provider，模型 qwen-image-3.0，尺寸 2048×1152）
  - [ ] （如需）给 image_gen.py 加 `prompt_extend` 透传，供精密图页关闭自动改写
- [ ] **阶段 2 · 样张**（强制门）
  - [ ] 选 1 页代表性内容页生成样张（建议 P25 FRI 折叠 或 P30 流水线，验证 Qwen 中文+精密图）
  - [ ] **等你确认**风格/中文清晰度/精密图质量
  - [ ] 记录 `deck_spec.json`（backend/sample 方法）
- [ ] **阶段 3 · 全量生成**
  - [ ] `prepare_slide_prompts.py` 生成每页 prompt job
  - [ ] 子代理并行生成剩余页（⚠ 精密图页优先、多轮）
  - [ ] `record_slide_result.py` 记录状态
- [ ] **阶段 4 · QA 与修复**
  - [ ] 逐页检查：中文错字/糊、要点遗漏、截断、风格一致、精密图正确性
  - [ ] 严重失败重生成；局部问题 edit
- [ ] **阶段 5 · 讲稿与合成**
  - [ ] 写 `speech.md`（每页讲者备注，映射 Slide N）
  - [ ] `assemble_ppt.py`（16:9）合成 `.pptx`
  - [ ] 最终报告：路径/页数/后端/限制

---

## 7. 待你确认的剩余决策点

1. **页数**：~40（推荐）/ 精简 ~30 / 更深 ~50？
2. **输出位置**：`plonky3-ppt/Plonky3深入原理.pptx` 可否？
3. **是否现在开工**？确认后我按 codex-ppt 流程：建目录 + 写 `outline.md` → **先出 1 页样张**给你审（建议样张选风险最高的精密图页，如 FRI 折叠或 11 步流水线，先验证 Qwen 能否达标）→ 通过后再全量生产。

> 说明：Qwen 无 `quality` 参数，故不再有 low/medium/high 分档；精密图页改为"细致 prompt + 关闭 prompt_extend + 多轮重试"。
