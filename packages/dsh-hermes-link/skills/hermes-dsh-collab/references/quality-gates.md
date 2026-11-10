# 质量门与回炉流程（主控执行）

> 真源：style-museum `AGENTS.md` 验证纪律 + `output/dsh-pipeline/observations.md` 会话策略/踩坑 + checkpoint.json 结构。本文件是质量门执行细则；具体测试命令以目标项目为准。

## 质量门四步（顺序执行，全绿才 commit）

| 步骤 | 动作 | 判定标准 |
|---|---|---|
| 1. 测试全绿 | 独立重跑测试命令（后端 pytest / 前端 node --test，以项目实际为准） | 全部通过；不采信 dsh 自报 |
| 2. build 成功 | 独立重跑 build 命令（如 `npm run build`） | 构建无错误 |
| 3. diff 审查 | `git diff`（或 diff --stat）逐文件核对 | **改动面与任务书范围声明完全一致**；越界改动即 FAIL |
| 4. 浏览器走查 | 涉及 UI 时真浏览器点击验证（主控执行） | 每个验收点的实际页面行为正确 |

## 不信自报（核心原则）

- dsh 的「全绿」「完成」报告只当**线索**，不当**证据**。
- 独立验证手段：`curl` 打真后端接口、真浏览器点击链路、色值/内容 diff 对照设计稿。
- 数字声明必须可验证或带标签——不能验证的声明要标注「待验证」。
- 实战例：dsh 曾用 curl 真后端 POST 独立验证「错判不泄露答案」逻辑（未判用户响应 count=0），把「不信自报」内化到执行侧。

## 回炉与升级链

- 质量门 FAIL → **把失败原因写进新任务书重派**（新会话），不是 resume、不是让 dsh 原地重试。
- 同一阶段重试 > 2 次 → 沿升级降级链升级：dsh → Codex → 人工。
- 每轮回炉在 checkpoint（如 `output/dsh-pipeline/checkpoint.json`）记录 retry_count；FAIL 判定 = 验收标准不满足，内容质量问题走**定向修复**（附失败日志摘要），不是原样重跑。
- 阶段全部 PASS 才由主控统一 git commit + tag。

## 会话与进程管理

- **每阶段新会话为主**：质量门独立、上下文纯净；一个会话攒多个阶段会互相污染上下文。
- **headless 无 resume**（源码级确认：每次调用随机 session UUID）——设计如此，不要试图 resume。
- **主控一律用 terminal(background=true) 管理进程，不用 nohup**：nohup 曾被 guard 拦截连带 pkill 未执行 → 旧代码占端口 8797，「测试全绿」验到旧进程、验证被误导。验证前先确认端口上跑的是新代码。

## 环境坑（走查时警惕）

- Browserbase stealth 环境偶发 localStorage 被禁（页面初始化失败）——重载页面即恢复，属环境问题非应用 bug；判 FAIL 前先排除。
- 后端进程残留会让测试「假绿」——改代码后没重启服务，pytest 对旧进程跑。
