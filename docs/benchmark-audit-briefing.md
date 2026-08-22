# ReaFlow 三配置规划评测审计报告

汇报对象：导师

评测范围：18 篇论文、3 种配置、54 个完成 Run

评测类型：Plan-only 实验规划评测

日期：2026-08-22

## 汇报结论

本次测试表明 Agent 能生成覆盖完整、结构规范的实验计划。它没有证明这些计划可以成功执行，也没有验证算法的实验结论。

三个核心判断是：

1. **规划内容得分很高，规划效率只有中等水平。** 有语料库配置的规划事实为 10.778/11，科学质量为 28.389/30，但规划效率只有 3.000/5。
2. **语料库没有明显损害科学完整性，主要问题是检索后的实验取舍。** 语料库提供了更多有依据的实验候选，Agent 却倾向于全部保留，导致组合膨胀和关键试验延后。
3. **没有发现主动读取 Judge 或历史成绩，但评测不是严格零泄漏。** 目标论文曾通过普通网络搜索进入部分 Agent 的上下文，模型是否受到影响无法排除。

因此，本次结果适合作为一次规划能力和评测流程的探索性审计，不适合作为算法有效性证明、正式排名或严格无泄漏的基准结论。

## 一、实验是如何设计的

### 1.1 评测目的

评测希望回答两个问题：

1. ReaFlow 的实验规划流程是否优于原生 Codex；
2. 在相同 ReaFlow 流程下，加入 ReaScholar 语料检索是否改善实验规划。

每篇论文只向 Agent 提供公开研究动机和目标算法代码。Agent 需要在 48 GPU-hours 的上限内设计完整实验计划，并说明预算减半到 24 GPU-hours 时保留哪些决定性实验。

### 1.2 三种配置

| 配置 | Agent 获得的能力 | ReaScholar 状态 |
| --- | --- | --- |
| Native Codex | 只使用任务提示和原生 Codex 工作方式，不加载 ReaFlow Agent Prompt 与技能运行时 | 不适用 |
| ReaFlow 无语料库 | 使用 ReaFlow Experiment Agent、实验设计规范和本地技能，但禁用 ReaScholar | 禁用 |
| ReaFlow 有语料库 | 与无语料库版本使用相同 ReaFlow Agent 和 revision，额外启用冻结的 ReaScholar 检索能力 | 启用 |

两种 ReaFlow 配置的 Prompt 差异被限制为三个冻结的 ReaScholar capability clause。每篇论文三种配置的公开输入和任务提示哈希一致。

### 1.3 模型、样本和 Judge

| 项目 | 设置 |
| --- | --- |
| 论文数量 | 18 |
| 配置数量 | 3 |
| 完成 Agent Run | 54/54 |
| 完成 Judge Evaluation | 54/54 |
| Agent 模型 | `gpt-5.6-sol`，`xhigh`，provider 为 `llmmelon` |
| Judge 模型 | `gpt-5.6-sol`，`xhigh`，provider 为 `llmmelon` |
| 每篇每配置生成次数 | 1 次最终选定生成 |
| 评测对象 | 规划文档，不执行实验 |
| 计算约束 | 48 GPU-hours 上限，要求给出 24 GPU-hours 缩减方案 |

Agent 和 Judge 使用相同模型。这保证了模型能力一致，但也会带来相似表达偏好导致的评分膨胀风险。

### 1.4 ReaScholar 是否真正使用

| 配置 | enabled | queried | succeeded | used |
| --- | ---: | ---: | ---: | ---: |
| Native Codex | 0/18 | 0/18 | 0/18 | 0/18 |
| ReaFlow 无语料库 | 0/18 | 0/18 | 0/18 | 0/18 |
| ReaFlow 有语料库 | 18/18 | 18/18 | 18/18 | 18/18 |

这里的 `used` 不是“调用过接口”就算使用。只有最终计划明确引用证据文件，并记录文献的接受或拒绝理由和协议用途时，才计为使用。

### 1.5 运行如何选中

报告生成代码没有按得分选择最高分 Run：

- Run 必须状态完整、身份匹配、模型和 benchmark revision 匹配；
- 对普通案例选择最新的合格完成 Run；
- Evaluation 必须完整、包含 19 个评分项且没有 `judge_error`；
- 对合格 Evaluation 选择最新编号，不读取分数择优；
- Provider、runner 和 Judge 失败记录被保留，没有覆盖删除。

因此，没有发现通过反复运行后挑选最高成绩的证据。但每篇每种配置只有一个最终样本，仍然不足以估计生成方差。

## 二、成绩到底如何

### 2.1 平均分

三个分数相互独立，不能相加为一个综合总分。

| 配置 | 规划事实（原始） | 规划事实（归一化） | 科学质量（原始） | 科学质量（归一化） | 规划效率（原始） | 规划效率（归一化） |
| --- | ---: | ---: | ---: | ---: | ---: | ---: |
| Native Codex | 10.889/11 | 0.990 | 29.556/30 | 0.985 | 3.278/5 | 0.656 |
| ReaFlow 无语料库 | 10.667/11 | 0.970 | 28.500/30 | 0.950 | 3.389/5 | 0.678 |
| ReaFlow 有语料库 | 10.778/11 | 0.980 | 28.389/30 | 0.946 | 3.000/5 | 0.600 |

三个维度分别表示：

- **规划事实**：11 个二元检查项，检查科学问题、实验、预算、优先级和停止规则等内容是否存在；
- **规划科学质量**：6 个维度，每个维度 0–5 分，总分 30，评价主实验、消融、分析、数据、基线和指标；
- **规划效率**：单独一个 0–5 分项目，评价单位算力获得的决策信息、实验顺序、成本核算和停止规则。

### 2.2 为什么看起来“几乎都是满分”

| 配置 | 规划事实满分 11/11 | 科学质量满分 30/30 | 规划效率满分 5/5 |
| --- | ---: | ---: | ---: |
| Native Codex | 16/18 | 14/18 | 1/18 |
| ReaFlow 无语料库 | 12/18 | 10/18 | 2/18 |
| ReaFlow 有语料库 | 14/18 | 7/18 | 0/18 |

这张表说明，满分主要集中在事实检查和科学覆盖，不在规划效率。

有语料库配置的效率分布是：

| 效率分 | 2/5 | 3/5 | 4/5 | 5/5 |
| --- | ---: | ---: | ---: | ---: |
| 论文数量 | 5 | 8 | 5 | 0 |

因此，“几乎都是满分”并不准确。更准确的说法是：**事实检查和科学质量出现了明显高分饱和，但严格的资源效率评分仍然能够区分计划。**

### 2.3 成绩应该如何解释

本次成绩支持以下判断：

- Agent 能生成完整、可证伪、带预算的实验规划；
- Agent 能覆盖多数科学比较角色；
- Agent 对实验优先级、组合规模和完整成本核算的控制明显弱于内容生成能力。

本次成绩不支持以下判断：

- 实验已经执行；
- 计划中的数据和基线可以直接运行；
- 预算估计经过实际校准；
- 算法效果已经得到证实。

## 三、为什么规划事实和科学质量分数这么高

高分主要来自任务提示、ReaFlow Agent Prompt 和 Judge Rubric 的结构性对齐。

| 任务提示要求 | ReaFlow 再次要求 | Judge 对应评分 |
| --- | --- | --- |
| 数据、基线、指标 | 证据来源与接受/拒绝表 | 数据、基线、指标三个维度 |
| 科学问题与替代解释 | 假设和证伪条件 | 主实验与消融维度 |
| 参数、机制和稳定性 | 机制实验与边界条件 | 分析实验维度 |
| 48 小时预算和优先级 | 资源合同与成本记录 | 规划效率 |
| 停止和删除规则 | `ready`/`blocked` 执行合同 | 可证伪性和资源意识 |
| 24 小时缩减方案 | 保留决定性实验 | 效率 4–5 分条件 |

这形成了“检查表饱和”：

1. 任务提示明确告诉 Agent 应包含哪些项目；
2. ReaFlow 工作规范再次要求这些项目；
3. Judge 按相同项目检查计划；
4. 能力较强的模型逐项覆盖后，事实与科学质量很容易接近满分。

此外，本次评测只检查规划文本。Judge 不要求 Agent 实际下载数据、实现基线或运行实验。现实执行中最容易失败的环节没有进入评分，因此高分门槛进一步降低。

同模型 Agent/Judge 可能提高彼此对表达方式的认可。这属于评测有效性风险，不是主动作弊的证据。

## 四、语料库在规划过程中起了什么作用

### 4.1 语料库提供了更多具体实验选项

以论文 `2310.09804` 为例，ReaScholar 为 Agent 提供了：

- BR-LSVRG 等相关文献基线；
- `a9a` 等 LIBSVM 数据集；
- 16 个工作节点、3 个 Byzantine 节点等设置；
- BF、LF、ALIE、IPM 等攻击方式；
- 压缩、鲁棒聚合、通信和收敛指标；
- 文献中的实验设置和代码位置。

Agent 在计划中明确接受或拒绝这些来源，并将其中的数据、攻击、基线和指标写入实验协议。因此，语料库不是形式上启用，而是实际改变了计划内容。

### 4.2 为什么更多证据没有提高平均科学分

有语料库相对无语料库的平均差异是：

- 规划事实：`+0.0101`；
- 科学质量：`-0.0037`；
- 规划效率：`-0.0778`。

科学质量原本已经接近上限，新增信息很难继续加分。与此同时，更多候选增加了实验取舍的难度。

当前流程缺少一个强制的“检索后收缩”步骤：每个新增实验不需要证明自己会改变哪项决策。Agent 为了保证覆盖完整，倾向于保留有文献依据的实验，最终形成攻击、Byzantine 数量、方法、压缩器和参数之间的大量组合。

底层机制可以概括为：

> 检索增加候选集合，评分又奖励完整覆盖；在缺少强制收缩规则时，Agent 更容易增加实验，而不是删除低优先级实验。

## 五、为什么有语料库的规划效率下降

### 5.1 总体结果

18 个有语料库与无语料库配对中：

- 8 篇效率下降；
- 5 篇保持不变；
- 5 篇效率提高。

无语料库平均效率为 3.389/5，有语料库为 3.000/5，平均下降 0.389 个原始分，即归一化下降 0.0778。

这不是普遍下降。它由若干大幅下降案例拉低，同时存在提高案例。

### 5.2 四个代表案例

| 论文 | 无语料库 | 有语料库 | 变化 | 主要原因 |
| --- | ---: | ---: | ---: | --- |
| `1905.13727` | 4/5 | 2/5 | -2 | 有语料库计划在加入决定性的匹配通信基线和通信受限测量前，先安排了机制消融、三档 rank 扫描和鲁棒性条件。 |
| `2307.09421` | 5/5 | 3/5 | -2 | 有语料库计划在 24 小时方案中没有保留关键的双连通性测试，并在稳定性边界前安排代表性任务验证。 |
| `2310.09804` | 5/5 | 2/5 | -3 | 有语料库计划先做 10 小时良性宽扫描，随后安排多种攻击、Byzantine 数量、控制组和 15 小时参数交叉，关键强攻击试验被延后。 |
| `2110.03294` | 3/5 | 4/5 | +1 | 有语料库计划反而保留了基础方法、无状态控制和全精度控制三件套，并让决定性控制在 24 小时方案中继续存在。 |

### 5.3 `2310.09804` 的具体对照

无语料库计划：

- 第一个实验直接比较良性环境、ALIE/IPM 强攻击、目标方法、已有鲁棒基线和压缩对照；
- 16 小时完成主实验，7 小时完成动量机制；
- 后续 error-feedback 和异质性实验由前置结果决定是否继续；
- 24 小时方案保留主实验和动量机制，不缩减关键种子数。

有语料库计划：

- 先花 10 小时在两个任务、三种方法和三档压缩率上做无攻击宽扫描；
- 再测试两个设置、5 个 Byzantine 数量、4 类攻击和3种聚合控制；
- 再花 15 小时交叉扫描 `gamma`、`p`、`a` 和多个压缩器；
- 即使缩减到 24 小时，仍然保留三种攻击，而不是先解决一个最有区分度的强攻击。

Judge 因此认为有语料库计划虽然内容完整，但存在冗余、关键证据延后和成本核算不足，只给 2/5。

### 5.4 能否认定语料库导致下降

不能做强因果判断，原因包括：

- 每篇每种配置只有一次最终生成；
- 没有多随机种子重复 Agent 生成；
- Agent 输出存在采样波动；
- Agent 与 Judge 使用相同模型；
- 普通互联网搜索也会改变无语料库和 Native 的可见信息。

当前最稳妥的结论是：

> 本批次观察到有语料库配置的平均规划效率下降。原始计划和 Judge 理由支持“检索后缺少实验收缩”是主要降分机制之一，但现有样本不足以证明稳定的因果效应。

## 六、有没有作弊或信息泄漏

### 6.1 审计需要区分四个问题

| 问题 | 审计发现 | 判断 |
| --- | --- | --- |
| Agent 是否读取 Judge、Rubric 或已有成绩 | 54 份已选轨迹的命令扫描未发现访问 `rubrics.json`、`results.json`、`evaluations/` 或汇总报告 | 未发现 |
| 是否按分数挑选最高 Run | 选择代码按身份、完成状态和编号选择最新合格 Run/Evaluation，不读取分数择优 | 未发现 |
| 目标论文是否进入模型上下文 | 4 个已选 Run、涉及 3 篇论文，普通网络搜索返回目标论文；部分结果包含摘要 | 已确认 |
| 目标论文是否被正式接受为证据 | 最终 workspace 未发现目标论文被接受为正式依据；可见案例通常明确拒绝 post-cutoff 内容 | 未发现正式采用，但影响无法排除 |

### 6.2 已确认的目标论文暴露

精确标题或编号扫描发现：

- `2307.09421` Native Run：OpenAlex 搜索结果暴露目标论文标题元数据；
- `2307.09421` 无语料库 Run：arXiv 搜索结果暴露目标论文编号、标题和摘要；
- `2310.09804` 无语料库 Run：OpenAlex 搜索结果暴露目标论文编号和标题；
- `2410.08760` 无语料库 Run：arXiv 搜索结果暴露目标论文标题和摘要，计划随后将该结果标记为超过截止日期并拒绝；
- `2011.08474` 有语料库 Run：检索词与目标标题 `Federated Composite Optimization` 相同。该短语可以从公开研究动机推导，因此原公平性审计将其记为 `public_input_derived_query_collision`，没有判为未解决违规。

这些暴露大多来自普通互联网搜索，而不是 ReaScholar。因此不能把它们解释为“有语料库配置作弊”，但它们说明整体评测环境没有实现目标论文完全不可见。

### 6.3 为什么原报告仍然显示 `valid=18`

`valid=18` 主要表示 Judge 没有在最终 workspace 中发现目标论文被引用或明显使用。它不是对完整 transcript 的零泄漏证明。

原公平性摘要中的严格 visibility 审计重点检查有语料库配置：

- strict visibility passed：17；
- public-input-derived query collision：1；
- unresolved failure：0。

该统计没有完整覆盖 Native 和无语料库配置通过普通互联网搜索看到目标论文的情况。因此，`valid=18` 不能解释为“18 篇都没有任何目标论文暴露”。

### 6.4 最终完整性判断

可以确认：

- 没有发现主动读取 Judge、Rubric 或历史成绩；
- 没有发现按得分选择最高 Run；
- 没有发现目标论文被正式接受为计划证据；
- 目标论文确实进入过部分 Agent 的搜索上下文。

最终判定是：

> 没有主动作弊证据，但确认存在目标论文暴露和评测污染风险，因此不能宣称严格零泄漏。

## 七、对本次测试的最终评价

### 7.1 可以肯定的部分

- Agent 能生成结构完整、可证伪、带预算的实验计划；
- ReaFlow 能要求 Agent 记录证据来源、接受和拒绝理由及执行合同；
- ReaScholar 在 18/18 个有语料库 Run 中成功进入最终规划；
- 完整 transcript、workspace、Judge 结果和失败记录已经公开，支持后续复核。

### 7.2 主要问题

- 规划事实和科学质量评分存在明显的检查表饱和；
- Agent 更擅长增加实验，不擅长在检索后删除低优先级实验；
- Agent 与 Judge 使用相同模型；
- 每篇每配置只有一次生成，无法估计随机波动；
- 普通互联网检索绕过了目标论文可见性控制；
- 本次评测没有执行实验，无法验证可运行性和成本估计。

### 7.3 最终定位

本次评测能够支持的结论是：

> ReaFlow 具备较强的实验规划文本生成能力，但当前检索增强没有转化为更高的平均规划效率。下一阶段的重点不是继续增加检索内容，而是增加检索后的优先级收缩、严格可见性隔离和重复生成评测。

## 八、下一轮测试如何改进

正文只保留三个最高优先级改进：

1. **增加重复生成。** 每篇每配置至少运行 3–5 次，报告均值、方差和配对区间。
2. **切断普通互联网旁路。** 所有配置只允许使用受控检索接口，并在内容到达模型前按目标 ID、标题别名和发布日期硬过滤。
3. **加入检索后收缩门。** 每个新增实验必须说明它会改变哪项科学决策；不能改变决策的实验必须删除或延后。Judge 应由不同模型和人工抽查进行校准。

## 九、5–10 分钟汇报顺序

| 时间 | 内容 | 必须讲清楚的结论 |
| --- | --- | --- |
| 0:00–0:45 | 问题和结论 | 这是规划评测，不是实验执行；内容强、效率一般、不是零泄漏 |
| 0:45–1:45 | 实验设计 | 18 篇、3 配置、54 Run；有无 ReaScholar 是主要对照；每配置只有一次生成 |
| 1:45–3:00 | 成绩 | 高分集中在事实和科学覆盖；有语料库效率只有 3/5，无效率满分 |
| 3:00–4:15 | 为什么高分 | Prompt、Agent 规范和 Judge Rubric 高度对齐，产生检查表饱和 |
| 4:15–6:15 | 为什么语料库降分 | 8降5平5升；重点讲 `2310.09804`，再用 `2110.03294` 说明不是必然下降 |
| 6:15–8:00 | 作弊与泄漏 | 无 Judge/成绩访问，无挑分；有目标论文暴露，所以不是严格零泄漏 |
| 8:00–9:00 | 最终判断与改进 | 重复生成、切断普通互联网、加入检索后收缩门 |

## 十、审查材料

- [三配置完整汇总报告](https://github.com/HuangZhengyang-2004/reasflow-experiment-benchmark/blob/main/reports/three-profile-plan-v2-leak-filter-v1/final/report_zh.md)
- [最终审计 Manifest](https://github.com/HuangZhengyang-2004/reasflow-experiment-benchmark/blob/main/reports/three-profile-plan-v2-leak-filter-v1/final/manifest.json)
- [选中运行清单](https://github.com/HuangZhengyang-2004/reasflow-experiment-benchmark/blob/main/reports/three-profile-plan-v2-leak-filter-v1/final/selected_runs.csv)
- [选中 Evaluation 清单](https://github.com/HuangZhengyang-2004/reasflow-experiment-benchmark/blob/main/reports/three-profile-plan-v2-leak-filter-v1/final/selected_evaluations.csv)
- [发布文件完整性审计](https://github.com/HuangZhengyang-2004/reasflow-experiment-benchmark/blob/main/reports/three-profile-plan-v2-leak-filter-v1/final/publication_audit.md)
- [逐文件发布 Manifest](https://github.com/HuangZhengyang-2004/reasflow-experiment-benchmark/blob/main/reports/three-profile-plan-v2-leak-filter-v1/final/publication_manifest.json)
- [`2310.09804` 无语料库计划](https://github.com/HuangZhengyang-2004/reasflow-experiment-benchmark/blob/main/runs/2310.09804/plan-v2-budget/reasflow-no-reascholar/run-002/workspace/Alg_Exp/document/experiment_plan.md)
- [`2310.09804` 有语料库计划](https://github.com/HuangZhengyang-2004/reasflow-experiment-benchmark/blob/main/runs/2310.09804/plan-v2-budget/reasflow-with-reascholar/run-003/workspace/Alg_Exp/document/experiment_plan.md)
- [`2310.09804` 无语料库 Judge 结果](https://github.com/HuangZhengyang-2004/reasflow-experiment-benchmark/blob/main/runs/2310.09804/plan-v2-budget/reasflow-no-reascholar/run-002/evaluations/coverage_v2_budget/evaluation-001/results.json)
- [`2310.09804` 有语料库 Judge 结果](https://github.com/HuangZhengyang-2004/reasflow-experiment-benchmark/blob/main/runs/2310.09804/plan-v2-budget/reasflow-with-reascholar/run-003/evaluations/coverage_v2_budget/evaluation-001/results.json)
- [`2310.09804` ReaScholar 检索证据](https://github.com/HuangZhengyang-2004/reasflow-experiment-benchmark/blob/main/runs/2310.09804/plan-v2-budget/reasflow-with-reascholar/run-003/workspace/Alg_Exp/evidence/experiment_evidence.json)
- [目标论文暴露代表性轨迹](https://github.com/HuangZhengyang-2004/reasflow-experiment-benchmark/blob/main/runs/2307.09421/plan-v2-budget/reasflow-no-reascholar/run-002/trace.jsonl)
- [报告选择与汇总代码](https://github.com/HuangZhengyang-2004/reasflow-experiment-benchmark/blob/main/tools/build_three_profile_report.py)

---

本报告基于已发布的原始 transcript、workspace、Judge 结果、失败记录和汇总 Manifest。所有比较均为规划结果比较，不构成算法实验结论或因果证明。
