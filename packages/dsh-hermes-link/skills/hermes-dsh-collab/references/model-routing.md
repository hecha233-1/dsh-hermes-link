# 模型分层路由细则

> 真源：style-museum `output/dsh-pipeline/observations.md`（08-14 交叉验证定）与 patch 文件本体（2026-09-10 起只剩 `pro-model.patch.yml`，且该档当前崩溃暂停）。本文件提炼判断规则；patch 内容以文件为准，用路径引用，别抄。

## 三档路由（effort 全开 max）

| 档位 | 模型 | 适用任务 | 派单 |
|---|---|---|---|
| 常规 | deepseek/deepseek-v4.1-flash（max 档） | 单文件修改 / 参数调整 / 写测试 / 简单 UI / 看设计稿截图对照 | 默认，无需 patch |
| 复杂 | ~~deepseek/deepseek-v4-pro~~（⚠️ 通道 2026-09-10 起崩溃暂停，复杂任务先用 Flash-max 顶） | 多文件重构 / 长内容提炼 / 复杂调试 | ~~`--patch ~/.dsh/profiles/headless/pro-model.patch.yml`~~（现状与回退见 `ai-coding-cli-setup` skill 的 dsh profile 排障段） |
| 视觉 | 无独立档位（原 qwen3.7-plus 已下线） | 看设计稿 / 截图对照 / 截图自检 | 主模型原生多模态：直接 `read_image`，**不带任何 patch** |

## 判断规则

1. **默认 Flash-max**。依据：Writer 论文（委派可靠性：强模型 0.85 vs 弱模型 0.45）+ COPE（大 planner 提升小 executor）+ Kilo 实测（Flash 复杂编码工作流质量降级）；Flash 的 max 推理档可弥补大部分差距。
2. **问三个问题**（任一为是 → 本来该升 Pro，但 Pro 通道暂停期间按下面第 3 条处理）：改动横跨多文件且互相牵连？要读大量材料做长提炼？调试涉及跨层推理？
3. **拿不准就试跑**：Flash 派一轮，质量门全绿 + 零返工 = 组合成立；返工 = 升级 Pro 重派（Pro 通道可用时；暂停期间改用 Flash-max + 拆小阶段，或换别的执行体跑该阶段）。**不是失败才升级，是返工才升级**——Flash 一次通过就说明档位合适。
4. **视觉不挂 patch、也不降档**：主模型 v4.1-flash 原生多模态，看图直接 `read_image`（图片一律用 `read_image`，`read` 碰二进制会报 `binary file`）；原 `vision-model.patch.yml` 与独立视觉模型（qwen/glm）2026-09-10 起已全部下线——任务书里不要再写视觉 patch 相关字段。

## 实战验证样本（08-14）

- **Flash 常规 3/3 通过**：候选面板加标题、教学闭环三项、统计页行动处方——均一次通过、零返工、TDD 规范（红→绿）、质量门全绿（43-44 pytest + 18-30 node + build）、浏览器走查通过。
- **Pro 复杂 1/1 通过且体现差异化价值**：读全 74 件教材（约 60 万 token）产出词表校准报告，抓出跨 74 件的系统性错误（整条自击方向反向、全宇宙特征词误作判别词）——这正是 Flash 长上下文弱项。
- 待验证：视觉密集实现、复杂调试的更多样本。

## patch 机制与两个坑

**机制**：`--patch <file>` 是 patch-list overlay，`llm-pi-ai.providers` 下写什么就整体替换什么。

1. **整段替换语义（非深度合并）**：providers 的 `<provider>` 键必须写完整定义（displayName / apiKeyEnv / api / baseURL / models），只写局部字段（如仅 `reasoning: max`）会把 profile 里完整 provider 定义整个顶掉 → `Provider is not configured: <provider>`。
2. **patch 只改模型、不声明模态就不认图片**：会话模型所在 provider 没写 `input: [text, image]` 时 `read_image` 报 `model "<id>" does not declare image input`——别重试、别去找已下线的视觉链路（vision-bridge），换回声明了图片输入的默认会话读图。

**已作废的历史坑（2026-09-10 前，独立视觉模型时代）**：3. qwen3.7-plus 不支持 `reasoning: max` → `UNSUPPORTED_REASONING_EFFORT`（视觉模型不配推理档）；4. vision patch 不改 `agent-default-model`，只声明 `input: [text, image]` 让 read_image 自动路由到视觉模型。两条随视觉 patch 删除一并失效，留作追溯；现行做法见上面第 2 条。

## 验证姿势（换模型必须双确认）

1. `dsh --patch <patch> "只输出你的主模型 id"`——自报确认配置生效
2. 真任务跑通（质量门通过）
- 两个确认都过了才认定档位切换成功；自报错了但任务过了、或反之，都要查 patch。

## Hermes 主控侧

- Hermes 日常用 Flash 不变（`commandcode/deepseek-v4.1-flash`）；管线主控需要 Pro 时用 `hermes chat -m commandcode/deepseek-v4-pro` 起 Pro 会话（2026-09-01 起 provider 已从 opencode-go 切到 commandcode）——**会话边界切模型，零副作用**（不污染其他会话）。
- 会话策略：**每阶段新会话为主**（质量门独立、上下文纯净）；headless 无 resume（每次调用随机 session UUID），所以回炉 = 新任务书重派，不是 resume。
