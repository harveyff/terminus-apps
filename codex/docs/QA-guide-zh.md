# 用 Olares 市场里的 Codex 分析用户问题（测试指南）

面向测试同事：用市场应用 **Codex CLI**，结合预装的 Olares Skills，分析用户反馈的 **Olares 系统问题** 或 **应用问题**，并整理可转交研发的结论。

前提：你已有 **ChatGPT Plus / Pro / Business / Edu / Enterprise** 订阅，用 **Sign in with ChatGPT** 登录即可。

---

## 1. 这套流程能做什么

| 用户常见描述 | Codex 适合做的事 |
|---|---|
| 应用装不上 / 升级卡住 / 一直 pending | 查市场状态、是否在排队下载、失败原因 |
| 应用崩溃、反复重启 | 查 Pod 状态、日志、事件 |
| 显示 running 但打不开 / 白屏 / 超时 | 区分「入口可达」和「业务真健康」 |
| 系统或应用很慢、内存/磁盘告警 | 查资源占用、压力、GPU 绑定 |
| 镜像拉不下来 | 查 ImagePull、架构不匹配等 |

Codex 在 Olares 上已预装 `olares-cli` 与一整套 Olares Skills（`olares-doctor`、`olares-market`、`olares-cluster` 等）。你用自然语言描述工单，它会按 Skills 选对命令、收集证据、给出根因判断。

### 查哪台机器？

- **默认查本机**：Codex 装在哪台 Olares，排查就针对这台（以及当前 `olares-cli` 已登录的本机账号）。日常复现、自测问题都按本机处理即可。
- **查其他机器**：必须先用 `olares-cli profile login` **重新登录那台机器上的账号**，再让 Codex 分析；否则查到的仍是本机数据。

---

## 2. 一次性准备（约 10 分钟）

### 2.1 从市场安装 Codex

1. 打开本机 Olares Desktop → **Market（应用市场）**。
2. 搜索 **Codex CLI** / `codex`，安装到你的账号。
3. 等到状态为 **running**，在「我的应用」里能看到入口 **Codex CLI**。

环境变量保持默认即可，无需改动。

### 2.2 打开终端并完成 ChatGPT 订阅登录

1. 点击入口 **Codex CLI**，进入终端窗口。
2. 执行：

```bash
codex
```

3. 按提示选择 **Sign in with ChatGPT**，在浏览器用你的 ChatGPT 订阅账号完成登录。
4. 登录成功后，终端里出现 Codex 会话界面即可。

以后同一安装一般会记住登录态；若提示未登录，再跑一次 `codex`，重新 Sign in with ChatGPT。

### 2.3 用 olares-cli 登录本机（查本机时）

排查本机问题时，在 Codex 终端里登录**本机当前用户**的 Olares ID：

```bash
# 查看是否已有 profile
olares-cli profile list

# 登录本机账号（换成你的 Olares ID）
olares-cli profile login --olares-id alice@olares.com
```

按提示输入密码；若开了 2FA，再输入 TOTP。

登录后确认：

```bash
olares-cli profile list
olares-cli market list --mine
```

能列出本机已安装应用，说明登录正确。之后日常查本机，一般不用再登。

### 2.4 查其他机器时：重新登录那个账号

要把排查目标切到**另一台 Olares / 另一个用户**时：

1. 拿到那台机器上的 Olares ID（及密码 / 2FA；或对方提供的可登录测试账号）。
2. 在 Codex 终端执行：

```bash
olares-cli profile login --olares-id <那台机器上的OlaresID>
# 若该账号以前登过，也可：
olares-cli profile list
olares-cli profile use <对应profile名>
```

3. 用下面命令确认已经切到目标环境，再开 `codex` 分析：

```bash
olares-cli profile list
olares-cli market list --mine
```

| 场景 | 怎么做 |
|---|---|
| 查本机问题 | 保持本机 profile，直接分析 |
| 查其他机器 / 其他用户 | `profile login`（或已有则 `profile use`）切到那个账号后再查 |
| 需要看系统级 Pod（os-framework 等） | 目标机器上需要 **admin** 账号；普通用户常会 403 |

---

## 3. 日常分析流程（接到工单后）

### 步骤 A：先确认「查本机还是查别机」

1. **现象出在本机** → 确认当前 profile 是本机账号即可。
2. **现象出在其他机器** → 先按 2.4 重新 `olares-cli profile login` 登录那个账号，再继续。

同时尽量凑齐工单信息：

1. **是否本机**；若否，目标 **Olares ID**
2. **应用名**（市场标题或 appid，如 `openwebui`、`n8n`）
3. **现象**（装不上 / 崩溃 / 打不开 / 慢 …）
4. **发生时间**、是否刚升级、是否刚改过设置
5. **截图 / 错误原文**（可粘贴进 Codex）
6. 若是系统问题：是整机卡顿、无法进 Desktop，还是某个系统功能失败

### 步骤 B：启动 Codex 并说明目标

在 Codex CLI 入口终端执行 `codex`，然后用清晰指令，例如：

```text
请用 Olares Skills（优先 olares-doctor）分析这个问题。
先确认当前 olares-cli profile（是不是我要查的那台/那个账号），再收集证据，最后给出：
1) 根因判断（高/中/低置信度）
2) 证据摘要（命令与关键输出）
3) 建议下一步（给用户的临时处理 / 给研发的复现与修复方向）
不要擅自卸载或升级应用，除非我明确同意。
```

### 步骤 C：把工单贴进去（推荐模板）

直接复制改写：

```text
【用户问题】
- 环境: 本机 / 其他机器（Olares ID: <xxx@olares.com>）
- 应用: <应用名 / appid>
- 现象: <一句话>
- 时间: <大约何时>
- 附加: <截图文字 / 错误日志粘贴>

【请你做】
1. 先确认当前 profile 是否对应上述环境；不对就停下来告诉我如何 login / use。
2. 用 olares-doctor 按症状路由排查（必要时配合 market / cluster / dashboard）。
3. 输出结构化报告（见下方格式）。
4. 只做只读排查；需要 stop/upgrade/uninstall 时先问我。
```

### 步骤 D：审核结论后再回复用户 / 提单

Codex 会跑 `market status`、`cluster pod logs` 等命令。你需要：

- 核对查的是不是正确环境、正确应用
- 把「根因 + 证据」整理进工单或研发群
- 涉及破坏性操作（卸载、改 env、重启系统组件）必须人工确认

---

## 4. 按场景的提问示例

### 4.1 安装 / 升级卡住

```text
用户说应用 X 装了很久还是 pending/downloading。
请按 olares-doctor「install stuck」路径排查：
- 是否有别的应用正在 downloading（单下载队列）
- market status / list --mine
- 若已失败，给出失败原因与日志要点
```

### 4.2 崩溃 / CrashLoop

```text
应用 X 反复重启。请查：
- market status
- 对应 namespace 的 pod 状态、events、最近日志
判断是配置错误、权限、依赖中间件，还是镜像/架构问题。
```

### 4.3 running 但打不开

```text
应用 X 状态是 running，但入口打不开 / 5xx / 白屏。
请按「running but unhealthy」路径：从入口连通性到容器日志逐级查，给出最可能根因。
```

### 4.4 慢 / 资源不足

```text
用户反馈整机或应用 X 很卡。请用 dashboard / cluster 看 CPU、内存、磁盘、GPU 压力，
以及该应用是否资源打满或调度失败（含 node-pressure）。
```

### 4.5 系统级问题（Desktop / 登录 / 系统服务）

```text
用户反馈 Olares 系统问题：<具体现象>。
请先确认当前 profile 是否 admin；
能查的系统组件就查，权限不够就明确写出需要 admin 才能继续的步骤，
并先基于用户空间可见信息给出临时结论。
```

---

## 5. 建议的输出报告格式（可直接贴工单）

让 Codex 按此格式收尾；你复制到飞书/工单即可：

```markdown
## 结论
- 环境：本机 / <Olares ID>
- 根因：…
- 置信度：高 / 中 / 低
- 影响范围：单应用 / 用户空间 / 整机

## 证据
- 应用状态：…
- Pod / 事件：…
- 关键日志摘录：…
- 资源情况（如有）：…

## 建议
- 给用户：…（可自助操作）
- 给研发：…（复现条件、怀疑模块、建议修复）

## 未完成项
- 因权限/信息不足未能确认的点：…
```

---

## 6. 内置 Skills 速查（知道即可，不必背命令）

| Skill | 何时会用到 |
|---|---|
| `olares-shared` | 登录、切 profile、鉴权失败恢复 |
| `olares-doctor` | **主入口**：按症状查根因 |
| `olares-market` | 安装状态、我的应用、生命周期信息 |
| `olares-cluster` | Pod、日志、事件、workload |
| `olares-dashboard` | CPU / 内存 / 磁盘 / GPU 等 |
| `olares-settings` | 入口域名、env、策略等配置 |
| `olares-files` | 用户文件 / 存储相关问题 |

对 Codex 说「用 olares-doctor 分析」通常就够；它会自行加载相关 Skills。

---

## 7. 安全与协作约定

1. **默认只读**：分析阶段不要让 Codex 自动卸载、升级、改生产配置。
2. **隐私**：工单里的密码、Token、Cookie 不要原样贴进会话；需要时可打码。
3. **先对环境**：默认本机；查别机必须先重新 `olares-cli` 登录那个账号，再分析。
4. **admin**：系统 Pod / 框架组件日志往往需要管理员；普通账号查不到时写明「需 admin」。
5. **结论人工过一眼**：AI 可能误判「排队下载」为「卡住」；以证据为准。

---

## 8. 常见卡点

| 现象 | 处理 |
|---|---|
| `codex` 提示未登录 ChatGPT | 再跑 `codex`，选 Sign in with ChatGPT |
| `olares-cli` 报未登录 / token 无效 | `olares-cli profile login --olares-id ...` |
| 查别机却看到本机应用列表 | 还没切账号：对目标机器重新 `profile login` 或 `profile use` |
| `market list` 为空但用户说装了 | 登错 Olares ID / 仍停在本机 profile |
| 查系统组件 403 | 用目标机器的 admin 账号重新 login，或请有权限同事协助 |
| Codex 想执行危险操作 | 明确回复：「先不要执行，只给出命令让我确认」 |
| 应用入口打不开 Codex 本身 | 在市场看 codex 是否 running；必要时重启应用后重登 ChatGPT |

---

## 9. 最短上手清单

- [ ] 市场安装 **Codex CLI** 且为 running  
- [ ] 打开入口 → `codex` → **Sign in with ChatGPT**（订阅账号）  
- [ ] `olares-cli profile login` 登录**本机**账号（查本机）  
- [ ] 若查其他机器：对那个账号再执行一次 `olares-cli profile login`（或 `profile use`）  
- [ ] `olares-cli market list --mine` 确认应用列表对得上目标环境  
- [ ] 用第 3 节模板把工单交给 Codex  
- [ ] 审核报告后回复用户 / 转研发  

完成以上步骤后，大部分「应用装不上 / 起不来 / 打不开 / 很慢」类用户问题，都可以在 Codex 里用同一套话术完成初筛与证据收集。
