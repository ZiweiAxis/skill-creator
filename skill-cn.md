---
name: skill-creator
description: 创建新技能、修改并迭代优化现有技能，衡量技能表现。当用户希望从零创建技能、更新或优化技能、运行评测验证技能效果、通过方差分析做基准对比，或者优化技能 description 以提升触发准确率时使用。
---

# Skill Creator

用于创建技能并持续迭代优化的技能。

从高层看，创建技能的流程如下：

- 明确技能要解决什么问题，以及大致如何解决。
- 写出技能草稿。
- 准备几条测试提示词，并运行可访问该技能的 Claude。
- 协助用户做定性和定量评估。
  - 当运行在后台进行时，如果尚无定量评测就先起草；如果已有，可直接复用或按需要修改。然后向用户说明这些评测。
  - 使用 `eval-viewer/generate_review.py` 向用户展示结果，同时呈现量化指标。
- 根据用户反馈以及基准里暴露出的明显问题重写技能。
- 循环迭代直到满意。
- 扩大测试集并做更大规模验证。

使用本技能时，你的工作是先判断用户目前处于哪个阶段，再推动他们进入下一阶段。例如用户说“我想做一个 X 技能”，你要帮助其收敛目标、起草内容、编写测试用例、确定评估方式、运行全部提示词并迭代。

如果用户已经有技能草稿，可以直接进入评测与迭代环节。

保持灵活：若用户说“不需要一堆评测，先一起快速打磨”，也可以采用轻量流程。

技能主体完成后（顺序可灵活），还可以运行单独的 description 优化脚本，提升技能触发效果。

## 与用户沟通

Skill Creator 的使用者对技术术语的熟悉程度差异很大。有人非常懂技术，也有人刚开始接触终端和开发工具。

因此请根据上下文调整表达方式。默认可参考：

- “评估（evaluation）”“基准（benchmark）”通常可以直接使用。
- “JSON”“断言（assertion）”这类术语，除非用户明显熟悉，否则请简短解释后再用。

拿不准时，先给一句简短定义再继续，能显著降低沟通成本。

---

## 创建技能

### 1) 捕获意图

先理解用户真正想要的结果。当前对话中可能已经包含可抽象成技能的工作流（例如“把这套流程做成 skill”）。若是这样，先从历史对话抽取答案：用过哪些工具、步骤顺序、用户纠正过什么、输入输出格式是什么。缺失信息再向用户补问，并在进入下一步前确认。

1. 这个技能要让 Claude 能做什么？
2. 这个技能在什么情况下应当触发？（用户会怎么说、在什么语境）
3. 预期输出格式是什么？
4. 是否需要测试用例验证技能？  
   - 结果可客观验证的技能（文件转换、数据提取、代码生成、固定流程）建议加测试。  
   - 结果偏主观的技能（文风、艺术创作）通常不必强制测试。  
   - 给出默认建议，但最终由用户决定。

### 2) 访谈与调研

主动询问边界场景、输入输出格式、示例文件、成功标准、依赖项。把这些问题理顺后再写测试提示词。

检查可用 MCP。如果有助于调研（查文档、找类似技能、总结最佳实践），优先并行使用子代理；不具备条件时再内联调研。目标是减少用户需要反复补充背景的负担。

根据访谈结果，补全以下部分，并在 `~/.skills/` 中生成文件：

- **SKILL.md frontmatter（ `name`, `description` 等）**：触发条件与功能说明。这是技能触发的主机制，必须同时写“做什么”与“何时用”。目前 Claude 对技能存在“欠触发”倾向，建议 description 稍微“主动”一点。
- **其余技能正文内容**。

文件创建完成后，提示用户调用 `skill-manager`（或直接运行 `~/.skills/skill-manager/scripts/sync_skills.sh`），为新技能配置 `meta.yaml` 的作用域并同步符号链接。

### 技能写作指南

#### 技能目录结构

```
skill-name/
├── SKILL.md（必需）
│   ├── YAML frontmatter（至少包含 name、description）
│   └── Markdown 指令正文
├── meta.yaml（必需）
│   └── 维度定义（包含 universal 或特定的 agent 列表）
└── Bundled Resources（可选）
    ├── scripts/    - 可执行脚本，用于确定性/重复性任务
    ├── references/ - 按需加载的参考文档
    └── assets/     - 输出会用到的资源（模板、图标、字体）
```

#### 渐进式加载

技能采用三级加载：

1. **元数据**（name + description）：始终在上下文中（约 100 词）。
2. **`SKILL.md` 正文**：技能触发时加载（建议 <500 行）。
3. **捆绑资源**：按需读取（可无限扩展，脚本可在不全文加载时执行）。

字数/行数是经验值，确有必要可适度超出。

关键模式：

- `SKILL.md` 建议控制在 500 行内；接近上限时请增加层级并给出清晰跳转指引。
- 在 `SKILL.md` 中明确引用外部文件，并说明何时读取。
- 大型参考文件（>300 行）建议加目录。

多域组织建议：当技能支持多个框架或云平台时，按变体拆分：

```
cloud-deploy/
├── SKILL.md（流程 + 选择逻辑）
└── references/
    ├── aws.md
    ├── gcp.md
    └── azure.md
```

Claude 仅需读取当前场景对应的 reference 文件。

#### 最小惊讶原则

技能不得包含恶意代码、漏洞利用代码或任何可能危害系统安全的内容。技能行为必须与描述一致，不应让用户“意外中招”。不要协助创建误导性技能，或任何用于未授权访问、数据外传等恶意用途的技能。“角色扮演”类需求通常是可接受的。

#### 写作模式

指令尽量使用祈使语气。

定义输出格式可参考：

```markdown
## 报告结构
必须使用以下模板：
# [标题]
## 执行摘要
## 关键发现
## 建议
```

示例写法可参考：

```markdown
## 提交信息格式
**示例 1：**
输入：新增基于 JWT 的用户认证
输出：feat(auth): implement JWT-based authentication
```

### 写作风格

尽量解释“为什么这样做重要”，而不是堆砌生硬的强制语句。利用模型的理解能力，让技能可泛化，不要过拟合到少数例子。先写草稿，再以“新读者”视角复审并优化。

### 测试用例

技能草稿完成后，先准备 2-3 条真实用户可能会说的测试提示词。先给用户确认，例如：

“我准备先试这几条测试用例，你看是否合适，或要不要补充？”

然后再运行。

将测试保存到 `evals/evals.json`。先只写 prompt，不急着写 assertions；下一步在运行进行中再补。

```json
{
  "skill_name": "example-skill",
  "evals": [
    {
      "id": 1,
      "prompt": "User's task prompt",
      "expected_output": "Description of expected result",
      "files": []
    }
  ]
}
```

完整结构见 `references/schemas.md`（后续会添加 `assertions` 字段）。

## 运行与评估测试用例

这一节是连续流程，不要中途停下。不要使用 `/skill-test` 或其他测试技能。

结果放在与技能目录同级的 `<skill-name>-workspace/`。工作区内按迭代分目录（`iteration-1/`、`iteration-2/` 等），每个测试用例再单独分目录（`eval-0/`、`eval-1/` 等）。不要一开始就全部建完，按执行进度创建即可。

### 第 1 步：同一轮发起全部运行（with-skill 与 baseline）

每个测试用例在同一轮中拉起两个子代理：一个带技能，一个不带技能。关键点：不要先跑完 with-skill 再补 baseline，要一次性全部发起，确保接近同时完成。

**with-skill 任务模板：**

```
执行以下任务：
- Skill path: <path-to-skill>
- Task: <eval prompt>
- Input files: <eval files if any, or "none">
- Save outputs to: <workspace>/iteration-<N>/eval-<ID>/with_skill/outputs/
- Outputs to save: <what the user cares about — e.g., "the .docx file", "the final CSV">
```

**baseline 任务**（同一提示词，但 baseline 随场景而变）：

- 创建新技能：不带任何技能。输出到 `without_skill/outputs/`。
- 优化现有技能：使用旧版本。编辑前先快照：`cp -r <skill-path> <workspace>/skill-snapshot/`，baseline 子代理指向快照，输出到 `old_skill/outputs/`。

每个测试用例都要写 `eval_metadata.json`（此时 assertions 可为空）。`eval_name` 应描述测试目标，不要只写 `eval-0`。目录名也用该描述名。如果当前迭代新增或修改了提示词，要为每个新用例目录单独创建，不要假设可继承上一轮。

```json
{
  "eval_id": 0,
  "eval_name": "descriptive-name-here",
  "prompt": "The user's task prompt",
  "assertions": []
}
```

### 第 2 步：运行进行中起草 assertions

不要空等任务结束。应利用这段时间为每个测试用例起草可量化 assertions，并向用户解释。如果 `evals/evals.json` 已有 assertions，也要解释它们在检查什么。

好的 assertion 应该客观可验证、命名清晰，让人一眼就知道在验证什么。主观技能（文风、设计质量）更适合定性评估，不要强行量化。

起草完成后，更新 `eval_metadata.json` 与 `evals/evals.json`，并告知用户查看器中会看到什么（定性输出 + 定量基准）。

### 第 3 步：任务完成时立刻记录耗时数据

每个子代理完成时，你会收到含 `total_tokens` 与 `duration_ms` 的通知。请立刻写入该运行目录下的 `timing.json`：

```json
{
  "total_tokens": 84852,
  "duration_ms": 23332,
  "total_duration_seconds": 23.3
}
```

这是唯一可获取该数据的时机：它只出现在任务通知里，其他地方不会持久化。必须“来一条记一条”，不要等全部完成再批处理。

### 第 4 步：打分、聚合、启动查看器

所有运行完成后：

1. **逐个打分**  
   拉起 grader 子代理（或内联执行），读取 `agents/grader.md`，对每条 assertion 判定并保存到各运行目录的 `grading.json`。  
   `grading.json` 中 expectations 必须使用 `text`、`passed`、`evidence` 三个字段名，查看器依赖这些固定字段。  
   可程序化校验的断言应尽量写脚本自动检查，而非人工目测。

2. **聚合基准**  
   在 skill-creator 目录运行：

   ```bash
   python -m scripts.aggregate_benchmark <workspace>/iteration-N --skill-name <name>
   ```

   产出 `benchmark.json` 与 `benchmark.md`，包含各配置的 pass_rate、耗时、tokens，以及 mean ± stddev 和 delta。若手写 `benchmark.json`，必须严格遵循 `references/schemas.md` 的 schema。  
   排序上，with_skill 要放在对应 baseline 前面。

3. **做一轮分析**  
   读取基准数据，补充聚合统计难以直接看出的模式。参考 `agents/analyzer.md` 的 “Analyzing Benchmark Results” 部分，例如：
   - 不区分技能价值的断言（无论是否启用技能都总是通过）
   - 高方差用例（可能不稳定）
   - 时间/Token 的收益权衡

4. **启动查看器（含定性+定量）**

   ```bash
   nohup python <skill-creator-path>/eval-viewer/generate_review.py \
     <workspace>/iteration-N \
     --skill-name "my-skill" \
     --benchmark <workspace>/iteration-N/benchmark.json \
     > /dev/null 2>&1 &
   VIEWER_PID=$!
   ```

   若为第 2 轮及之后，追加 `--previous-workspace <workspace>/iteration-<N-1>`。

   **Cowork / 无头环境：** 若 `webbrowser.open()` 不可用或无显示环境，使用 `--static <output_path>` 输出独立 HTML，而不是启动服务。用户点击 “Submit All Reviews” 后会下载 `feedback.json`，下载后需复制到 workspace，供下一轮读取。

   注意：请使用 `generate_review.py` 生成查看器，不要自写 HTML。

5. **提示用户查看**

   可类似这样说：  
   “结果已在浏览器打开。`Outputs` 标签页可逐条查看并反馈，`Benchmark` 标签页展示量化对比。你看完后回来告诉我。”

### 用户在查看器中会看到什么

`Outputs` 标签页按测试用例逐条展示：

- **Prompt**：给模型的任务；
- **Output**：技能生成的文件（可内联则内联）；
- **Previous Output**（第 2 轮+）：折叠展示上一轮输出；
- **Formal Grades**（若已打分）：折叠展示断言通过情况；
- **Feedback**：自动保存的反馈输入框；
- **Previous Feedback**（第 2 轮+）：显示上轮反馈。

`Benchmark` 标签页展示各配置的通过率、耗时、Token 与分用例明细，以及分析备注。

支持上一条/下一条按钮与方向键导航。用户点 “Submit All Reviews” 后会保存 `feedback.json`。

### 第 5 步：读取反馈

用户说“看完了”之后，读取 `feedback.json`：

```json
{
  "reviews": [
    {"run_id": "eval-0-with_skill", "feedback": "the chart is missing axis labels", "timestamp": "..."},
    {"run_id": "eval-1-with_skill", "feedback": "", "timestamp": "..."},
    {"run_id": "eval-2-with_skill", "feedback": "perfect, love this", "timestamp": "..."}
  ],
  "status": "complete"
}
```

空反馈通常表示用户认为该条结果可接受。优化重点应放在有明确问题反馈的用例上。

查看结束后关闭服务：

```bash
kill $VIEWER_PID 2>/dev/null
```

---

## 优化技能

这是迭代闭环的核心：你已经跑完测试并拿到用户反馈，下一步是据此提升技能质量。

### 优化思路

1. **从反馈中抽象出可泛化规律**  
   你和用户可能只围绕少量示例快速迭代，但目标是让技能在大规模真实场景下可复用。  
   不要只做针对样例的“打补丁式过拟合”。遇到顽固问题时，尝试改写指令隐喻、调整流程策略，往往成本不高但收益可能很大。

2. **保持提示词精简**  
   删除不产生价值的内容。读 transcript，不只看最终产物；若技能引导模型做了大量无效步骤，要考虑删减对应指令并复测。

3. **解释“为什么”**  
   尽量把规则背后的动机讲清楚，而不是堆砌硬性命令。模型具备较强理解能力，解释原因通常比死板约束更稳健、更高效。

4. **识别跨用例重复劳动**  
   若多个测试都重复生成类似辅助脚本（例如 `create_docx.py`、`build_chart.py`），说明该脚本应内置到技能的 `scripts/` 中，避免每次重复造轮子。

这个任务很重要。请给足思考时间，先写一版修订，再回看并优化，尽力从用户真实目标出发做设计。

### 迭代循环

完成一次优化后：

1. 将改动应用到技能；
2. 在新的 `iteration-<N+1>/` 下重跑全部测试（包含 baseline）；
   - 新技能场景：baseline 固定为 `without_skill`；
   - 旧技能优化场景：可按判断使用“初始版本”或“上一轮版本”作 baseline；
3. 使用 `--previous-workspace` 指向上一轮并启动 reviewer；
4. 等用户看完并确认；
5. 读取新反馈，继续优化并重复。

结束条件：

- 用户明确表示满意；
- 反馈基本为空（整体通过）；
- 继续迭代已无明显增益。

---

## 进阶：盲评对比

当需要更严格比较两版技能（如“新版本真的更好吗”）时，可使用盲评系统。详见 `agents/comparator.md` 与 `agents/analyzer.md`。核心思路是将两份输出匿名给独立代理评判，再分析胜出原因。

该流程是可选项，且依赖子代理。多数场景下，人类评审闭环已足够。

---

## Description 优化

`SKILL.md` frontmatter 中的 `description` 是决定技能是否触发的关键。创建或优化技能后，建议提供 description 优化流程以提升触发准确率。

### 第 1 步：生成触发评测查询

准备 20 条查询，混合“应触发”和“不应触发”，保存为 JSON：

```json
[
  {"query": "the user prompt", "should_trigger": true},
  {"query": "another prompt", "should_trigger": false}
]
```

这些查询必须真实、具体，接近 Claude Code/Claude.ai 用户真实输入，而非抽象命令。可包含文件路径、职业背景、列名与取值、公司名、URL、口语、缩写、错拼、不同长度等，并强调边界场景。

反例（过于抽象）：
- `"Format this data"`
- `"Extract text from PDF"`
- `"Create a chart"`

正例（真实语境）：
- `"ok so my boss just sent me this xlsx file (its in my downloads, called something like 'Q4 sales final FINAL v2.xlsx') and she wants me to add a column that shows the profit margin as a percentage. The revenue is in column C and costs are in column D i think"`

对于 **should-trigger**（8-10 条）：

- 覆盖同一意图的不同表达（正式/口语）；
- 包含未明确点名技能但明显需要技能的场景；
- 包含与其他技能竞争但本技能应胜出的案例。

对于 **should-not-trigger**（8-10 条）：

- 优先设计“近似误触发”样本：关键词相关但本质需求不同；
- 关注相邻领域、歧义表达、应由其他工具处理的语境。

避免把负样本设计得太“离题”。例如对 PDF 技能用“写斐波那契函数”作为负例太容易，无法验证边界。

### 第 2 步：与用户共审

使用 HTML 模板让用户审阅并编辑 eval 集：

1. 读取 `assets/eval_review.html`；
2. 替换占位符：
   - `__EVAL_DATA_PLACEHOLDER__` -> eval JSON 数组（不要加引号，这是 JS 变量赋值）；
   - `__SKILL_NAME_PLACEHOLDER__` -> 技能名称；
   - `__SKILL_DESCRIPTION_PLACEHOLDER__` -> 当前技能 description；
3. 写入临时文件（如 `/tmp/eval_review_<skill-name>.html`）并打开：
   `open /tmp/eval_review_<skill-name>.html`
4. 用户可编辑 query、切换 should-trigger、增删条目，然后点 “Export Eval Set”；
5. 文件会下载到 `~/Downloads/eval_set.json`。若有多个同名副本（如 `eval_set (1).json`），取最新版本。

这一环很关键：评测集质量直接影响 description 优化质量。

### 第 3 步：运行优化循环

先告知用户：  
“这个过程会花一些时间，我会在后台运行优化循环并定期同步进度。”

保存 eval 集后，在后台运行：

```bash
python -m scripts.run_loop \
  --eval-set <path-to-trigger-eval.json> \
  --skill-path <path-to-skill> \
  --model <model-id-powering-this-session> \
  --max-iterations 5 \
  --verbose
```

`--model` 请使用当前会话真实模型 ID（与用户实际体验保持一致）。

运行期间定期查看输出并同步进度（当前迭代、分数变化等）。

该脚本会自动执行完整循环：  
将 eval 集拆分为 60% 训练 / 40% 测试；对当前 description 做评估（每条 query 跑 3 次）；调用 Claude 做增强思考并提出改进；在训练与测试集上重复评估，最多 5 轮。完成后会打开 HTML 报告并输出包含 `best_description` 的 JSON；最终以测试集得分选优，降低过拟合风险。

### 触发机制说明

理解触发机制有助于设计更有效的评测查询。技能会以 name + description 出现在 Claude 的 `available_skills` 列表中，Claude 决定是否依据 description 调用技能。

关键点：Claude 通常只在“自身不易一步完成”的任务上调用技能。像“读这个 PDF”这类简单请求，即使 description 很匹配，也可能不触发技能，因为基础工具可直接完成。复杂、多步、专业任务更容易稳定触发。

因此评测查询要足够“有工作量”，让模型确实有必要调用技能。过于简单的查询并不能有效评估 description 质量。

### 第 4 步：应用结果

从 JSON 输出中取 `best_description`，更新技能 `SKILL.md` frontmatter，并向用户展示前后对比与评分结果。

---

### 打包与交付（仅在存在 `present_files` 工具时）

先检查是否可用 `present_files`。不可用则跳过；可用则执行：

```bash
python -m scripts.package_skill <path/to/skill-folder>
```

打包后告知用户 `.skill` 文件路径，供其安装。

---

## Claude.ai 场景说明

在 Claude.ai 中，核心流程仍是“起草 -> 测试 -> 评审 -> 优化 -> 重复”，但因无子代理，执行细节需调整：

- **运行测试用例**：无法并行。每个用例都先读 `SKILL.md`，再按技能指令逐条执行。该方式不如独立子代理严格（因为你既写技能又执行），但可作为有效的快速校验。可跳过 baseline，仅用技能完成任务。
- **审阅结果**：若无法开浏览器（无显示环境或远程主机），跳过浏览器 reviewer，直接在对话里逐条展示 prompt 与输出。若产物是用户需要检查的文件（如 `.docx`、`.xlsx`），保存到文件系统并告知路径。
- **基准评测**：可跳过定量基准（无子代理时 baseline 对比意义弱），聚焦用户定性反馈。
- **迭代循环**：仍按“改进 -> 重跑 -> 收反馈”执行，只是中间不使用浏览器 reviewer。
- **description 优化**：依赖 `claude` CLI（`claude -p`），仅 Claude Code 可用；在 Claude.ai 中应跳过。
- **盲评**：依赖子代理，应跳过。
- **打包**：`package_skill.py` 在有 Python 和文件系统时均可运行，Claude.ai 也可打包并下载 `.skill`。

---

## Cowork 场景说明

若在 Cowork 环境中，请注意：

- 有子代理，主流程（并行跑测试、baseline、打分等）可用。若遇到严重超时，可退化为串行运行。
- 无浏览器/显示时，生成 reviewer 请使用 `--static <output_path>` 输出独立 HTML，再给用户可点击链接。
- 强调一次：无论在 Cowork 还是 Claude Code，跑完测试后都应先用 `generate_review.py` 生成评审查看器，再做自我修订。  
  **先让人类尽快看到结果，再改技能。**
- 反馈机制不同：无服务模式下，“Submit All Reviews” 会下载 `feedback.json`，随后从下载文件读取反馈（必要时先请求访问权限）。
- 打包可正常使用：`package_skill.py` 仅依赖 Python 与文件系统。
- description 优化（`run_loop.py` / `run_eval.py`）在 Cowork 通常可用，但建议在技能主体已稳定且用户认可后再做。

---

## 参考文件

`agents/` 目录是专用子代理说明，按需读取：

- `agents/grader.md`：如何对断言进行评分；
- `agents/comparator.md`：如何做盲评 A/B 对比；
- `agents/analyzer.md`：如何分析某版本为何胜出。

`references/` 目录为补充文档：

- `references/schemas.md`：`evals.json`、`grading.json` 等 JSON 结构定义。

---

再次强调核心闭环：

- 明确技能目标；
- 起草或编辑技能；
- 在测试提示词上运行可访问该技能的 Claude；
- 与用户共同评估输出：
  - 生成 `benchmark.json` 并运行 `eval-viewer/generate_review.py` 协助人工评审；
  - 执行定量评测；
- 反复迭代直到你和用户都满意；
- 打包最终技能并交付用户。

如果你有 TodoList，请把关键步骤写进去避免遗漏。若在 Cowork，务必将“创建 evals JSON 并运行 `eval-viewer/generate_review.py` 供人类评审”显式加入待办。

祝顺利。
