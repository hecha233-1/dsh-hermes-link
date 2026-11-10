---
name: hermes-dsh-collab
description: 当你在 Hermes×dsh 多 agent 协作管线中担任 dsh 执行者（或为主控 Hermes 写派单任务书、判模型档位、定质量门）时使用——给出模型分层路由（Flash 常规 / Pro 复杂·通道当前暂停 / 视觉由主模型原生多模态承担，无需 patch）、spec 三铁律（Plan 先行/测试先行/范围声明）、git 唯一写者约定、质量门与写回机制的判断指引和踩坑规避。
whenToUse: 收到任务书式派单要开工；写派单任务书；判断模型档位或视觉任务怎么派；执行质量门验证、回炉或升级降级
---

# Hermes×dsh 多 Agent 协作规范

**本 skill 是判断指引，不是清单。** 它把 style-museum 管线 14 天 30 commits 的实战规范提炼为可移植规则：**Hermes 是唯一主控**（写任务书 → 派单 → 独立验证质量门 → 统一 commit），**dsh 是执行者**（读任务书 → 按三铁律实现 → 报告，不 commit）。用它判断"这个阶段怎么派、怎么执行、怎么验收"，而不是机械过列表。

## 真源（Sources of truth）

以下文件是规范的活体，会持续更新——涉及对应主题时**读它们，不要凭本 skill 复述**：

- 实战观测卡（行为模式 + 踩坑，每阶段追加）：`/mnt/d/WorkSpace/Projects/style-museum/output/dsh-pipeline/observations.md`
- spec 模板定稿：`/mnt/d/WorkSpace/Projects/style-museum/specs/template-v2.md`；本 skill 的 `references/spec-template.md` 是它的通用化可复制件，两者冲突以项目模板为准
- 模型 patch 文件：`~/.dsh/profiles/headless/pro-model.patch.yml`（2026-09-10 起本机只剩这一个，且 pro 通道暂停中；原 `vision-model.patch.yml` 已删除）——按路径引用，别把内容抄进任务书。⚠️ 派单命令里用 `~` 没问题（shell 展开）；但**让 dsh 用文件工具去读这些文件时必须换成绝对路径**——dsh 的文件工具不展开 `~`（见 `references/pitfalls.md` 第 3 条）
- 目标项目 `AGENTS.md`（硬约束，每次动手前必读）

## 何时使用

- 收到「任务书」形式的派单，要作为 dsh 执行者开工
- 要替 Hermes 主控写一份新派单任务书
- 要判断某阶段选哪档模型、视觉任务怎么派（答案是无需 patch）
- 要执行质量门验证、判定回炉、或决定升级降级

## 阻断项（违反 = 返工）

1. **git 唯一写者 = Hermes 主控。** dsh 执行者不做任何 git 写操作（commit/add/push/amend 全禁）。历史观测：任务书不写禁止条款时 dsh 会自行 commit——派单必须显式声明「不 commit」。
2. **写回靠启动目录。** dsh 沙箱写权限 = 启动工作目录 + 平台临时根。派单命令必须是 `cd <项目根> && dsh --profile headless "任务"`；不在项目根启动，产物写不回项目。**任务书里的文件路径写绝对路径**——dsh 的文件工具不展开 `~`。
3. **三铁律缺一不派单**：Plan 先行、测试先行、范围声明（file scoping）。

## spec 三铁律（写任务书与执行都要守）

| 铁律 | 执行侧要求 | 判断要点 |
|---|---|---|
| **Plan 先行** | 动手前读「必读」+ 目标文件，先输出改动清单（文件 × 改动点），再实现 | 清单是范围第一次自查：清单里出现范围外文件 = 立刻暴露 |
| **测试先行（TDD）** | 先写/更新测试并确认失败（红），再实现到绿 | 没先红过的绿不可信——它可能什么都没测 |
| **范围声明** | 只允许改任务书列出的文件；禁止清单（数据真源/索引/未列文件）同等重要 | 主控 diff 审查按此对账；越界改动即使「顺手修好」也算违规 |

外加 **不 commit**（git 唯一写者条款，见阻断项）。完整可复制模板见 `references/spec-template.md`。

## 模型分层路由（按阶段复杂度选，不是按心情）

| 档位 | 适用 | 派单方式 |
|---|---|---|
| **Flash（max 档）** | 常规：单文件修改/参数/测试/简单 UI/看图 | 默认，无需 patch |
| **Pro**（⚠️ 通道 2026-09-10 起崩溃暂停） | 复杂：多文件重构/长内容提炼/复杂调试 | `--patch ~/.dsh/profiles/headless/pro-model.patch.yml`；暂停期用 Flash-max 拆小阶段顶 |
| **视觉** | 看设计稿/截图对照/截图自检 | **无需任何 patch**——主模型 v4.1-flash 原生多模态，`read_image` 直接读 |

判断问句：改动横跨多文件且互相牵连？要读大量材料做长提炼？调试涉及跨层推理？——任一为是 → Pro（⚠️ Pro 通道暂停期间改为 Flash-max + 拆小阶段）；单文件局部改动 → Flash。**拿不准就 Flash 试跑一轮：质量门全绿 + 零返工 = 组合成立；返工 = 升级 Pro 重派**（实战已验证 Flash-max 处理常规阶段 3/3 通过；Pro 暂停前长提炼 1/1 通过并抓出跨文件系统性错误）。

patch 机制与两个坑（整段替换语义 / 未声明图片模态就不认图）、验证姿势、以及 Pro 与视觉档的现状见 `references/model-routing.md`。

## 质量门（主控执行，不是 dsh 自报）

dsh 报告「完成」只是线索。主控必须独立验证四步：

1. 测试全绿（后端 pytest / 前端 node --test）
2. build 成功
3. `git diff` 审查：改动面与范围声明一致
4. 浏览器走查（涉及 UI）

原则：**不信自报**——用 curl 200 / 真浏览器点击 / 色值 diff 实测；数字声明必须可验证或带标签。命令细则与回炉流程见 `references/quality-gates.md`。

## 写回机制（一句话记住）

**cd 项目根启动即写回，镜像/patch 交接全免。** dsh 的 workspace-write 沙箱以启动工作目录为写权限白名单，所以派单命令自带 `cd`。任何「产物在 /tmp、需 cp 回项目」的交接方式都是历史包袱。

## 常见踩坑（速览）

细节与对策在 `references/pitfalls.md`（唯一清单家），高频四条：

1. **文件工具不展开 `~`**——dsh 会话内读文件、任务书「必读」清单一律写绝对路径（`~` 只在 bash 与主控派单命令里安全）
2. **patch 是整段替换语义**——providers 必须写完整定义，只写局部字段 → `Provider is not configured`
3. **图片一律 `read_image`**——`read` 碰二进制报 `binary file`；报 `does not declare image input` 时不要重试、不要去找已下线的 vision-bridge，换默认会话读图或让主控给文字描述（视觉无需任何 patch）
4. **旧后端进程残留**——主控用 terminal(background=true) 管理进程，不用 nohup；残留进程会让「测试全绿」验到旧代码

## What NOT to do（专门节：挡住最容易犯的错）

- **不要 git commit / add / push / amend**——你是执行者；主控验证全绿后统一提交（中文、单逻辑）
- **不要在任务书外自造范围**——「顺手修一下」相邻文件 = 范围外偏差；发现需要修的，写进报告让主控决定
- **不要照抄 patch 文件内容进任务书**——引用路径即可；抄错一份废一份，且 patch 语义本就易错
- **不要 resume 回炉**——headless 无 resume（每次调用随机新 session）；失败由主控把原因写进新任务书重派
- **不要 nohup 起进程**——用主控的 terminal(background=true) 管理
- **不要用 bundle 形态安装这个纯提示词 skill**——`dsh plugin add` 走 pnpm 依赖解析，对零代码资产纯增风险；复制目录到 skills root 即可（安装命令见项目 README.md）
- **不要复述真源**——observations.md / spec 模板 / patch 文件是活的，用路径引用
- **不要相信 dsh 的「全绿」自报**——质量门是主控的独立验证动作

## 自测（装完验证三件事）

1. **出现在目录里**：新开会话问「列出你的技能」应见 `hermes-dsh-collab`（没出现 = frontmatter 问题，查 kebab-case 与目录名一致）
2. **description 能触发**：用真实场景提问（如「帮我写一份 dsh 派单任务书」）应触发加载
3. **正文真的改变行为**：对比加载前后——加载后应能复述三铁律与唯一写者约定，并拒绝 git commit
