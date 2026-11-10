# 常见踩坑（唯一清单家）

> 本文件是踩坑的**唯一清单家**——SKILL.md 只放速览，全量在这里。真源是 style-museum `output/dsh-pipeline/observations.md`（持续追加）；新踩坑先记真源，再提炼进这里。每条：症状 → 根因 → 对策。

## 派单/写回

1. **写回拦截（已根治）**
   - 症状：dsh 产物落在 /tmp，项目里看不到；靠镜像/patch 交接的旧时代坑（patch 带 /tmp 前缀需 python 重写 a/b/ 再用 -p2 变体 apply）。
   - 根因：dsh workspace-write 沙箱写权限 = 启动工作目录 + 平台临时根。
   - 对策：派单命令必须 `cd <项目根> && dsh --profile headless "..."`；cd 项目启动即直接写回，交接全免。

2. **patch 路径前缀（写回修复后不再出现）**：同上历史坑，不再使用镜像交接后自然消失。

## 路径解析

3. **文件工具不展开 `~`（也不展开 `$VAR`）——任务书里的路径写绝对路径**
   - 症状：dsh 会话内用 `read` / `write` / `edit` / `glob` / `grep` 传 `~/...` 或 `$DSH_HOME/...` → `cannot read "<cwd>/~/...": not found`（它把波浪号当成了工作目录下一个叫 `~` 的目录）。
   - 根因：fs provider 的解析是 `resolve(cwd, path)`，**没有调用**会展开 `~` 的 `expandHomePath`（`@deepseek-ai/dsh-home-paths` 有该函数，但 `dsh-fs-local` / `dsh-fs-sandbox` / `dsh-fs-observation-policy` 均未使用）。
   - 对策：dsh 会话内读文件、以及任务书「必读」清单里的路径，**一律写绝对路径**（`/home/<user>/.dsh/...`）；`~` 只在两类地方安全——bash 工具里（由 shell 展开）、以及主控在终端敲的 `dsh --patch ~/...` 派单命令。
   - 证据（2026-09-11 源码级 + 实测）：本机探测「读 `~/.dsh/skills/...`」4 次调用失败 2 次；fail-logger 里 `not found` 是全量最高频类（10 次），最近几次的参数全是 `~/` 开头。

## 模型 patch

4. **patch 整段替换语义**
   - 症状：`--patch` 只写局部字段（如 `reasoning: max`）后报 `Provider is not configured: opencode-go`。
   - 根因：patch config 是整段替换，非深度合并；局部字段会把 profile 里完整 provider 定义整个顶掉。
   - 对策：每个 patch 必须自包含完整 provider 定义（displayName/apiKeyEnv/api/baseURL/models）。

5. **qwen3.7-plus 不支持 reasoning effort max**（历史条目，2026-09-10 起在本机不适用）
   - 症状：vision patch 带 `reasoning: max` → `UNSUPPORTED_REASONING_EFFORT`。
   - 对策：视觉模型不配推理档；视觉靠声明 `input: [text, image]` 让 read_image 自动路由，不改 agent-default-model。
   - 现状：主模型原生化后视觉 patch 与独立视觉模型（qwen/glm）已全部下线（见第 6 条），本坑只在对其他不支持推理档的 provider 挂 patch 时复现。

## 视觉与读图（2026-09-10 起主模型原生多模态）

6. **图片一律 `read_image`；报「模型不声明图片输入」时不要去找旧视觉链路**
   - 症状：① `read` 读图片报 `binary file`；② `read_image` 报 `model "<id>" does not declare image input`。
   - 根因：① `read` 是文本工具，碰二进制直接拒；② 当前会话模型的 provider 没声明 `input: [text, image]`（例如挂了 Pro 档的 patch），或这一轮跑的是旧模型 id。
   - 对策：图片一律用 `read_image`（传本地路径）——主模型 `deepseek/deepseek-v4.1-flash` 原生多模态，**不需要任何视觉 patch**；报 ② 时不要重试、也不要去读 `vision-bridge`（该链路 2026-09-10 已下线，`~/.dsh/skills/vision-bridge/` 已改名 `.bak-20260910`），改用声明了图片输入的默认会话读图，或让主控在任务书里给文字描述。
   - 升级条件：本条生效后同类失败仍在多个项目复发 → 升级进 `$DSH_HOME/AGENTS.md` 行为约束（当前只有个位数次数，按规矩留在 skill）。

## git 纪律

7. **dsh 会自行 git commit**
   - 症状：执行者悄悄 commit（历史 186a357/29d2832，作者为 dsh 身份）。
   - 根因：任务书未写明禁止。
   - 对策：任务书执行纪律必须含「不 commit：git 唯一写者 = Hermes 主控」条款；主控提交信息中文、单 commit 一个逻辑变更、提交前 git status 确认。

## 进程管理

8. **旧后端进程残留**
   - 症状：改完代码测试「全绿」却验到旧代码（nohup 命令被 guard 拦截，连带 pkill 未执行，旧进程占 8797 误导验证）。
   - 对策：主控一律用 terminal(background=true) 管理进程，不用 nohup；验证前确认端口上跑的是新代码。

## 走查环境

9. **Browserbase localStorage 偶发被禁**
   - 症状：stealth 环境 progress 初始化失败页面全挂。
   - 对策：重载页面即恢复，非应用 bug；判 FAIL 前先排除环境因素。

## 会话与回炉

10. **headless 无 resume**：每次调用随机 session UUID——回炉必须把失败原因写进新任务书重派（新会话），不是 resume；同一阶段重试 > 2 次升级降级链（dsh → Codex → 人工）。

## 上下文预算

11. **任务书塞太满**：把项目背景/历史教训整段贴进任务书浪费上下文。对策：spec 只保留本阶段最小信息；背景与教训让 dsh 自己读项目 AGENTS.md 和观测卡（必读清单列路径即可）。

## 插件生态（仅当走 bundle 形态时才相关；本 skill 用纯目录形态，不适用）

12. **dsh plugin 安装坑**（历史记录，备查）：
    - dsh plugin 转发 pnpm，需 pnpm 在 PATH；软链有 Permission denied 怪问题，用 wrapper 脚本（`#!/bin/sh exec node .../pnpm.cjs`）绕。
    - `pnpm add` 只装依赖不激活——必须 `dsh plugin --profile X add <pkg>` 注册进 package.json 的 dsh.profile.bundles。
    - 装插件前备份 profile；guard 误判（绝对路径含 .local 触发）用 execute_code 绕。
