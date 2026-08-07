# OpenCodex 多模型智能路由规则（公开脱敏版）

版本：2026-08-07

这是一套面向 macOS + ChatGPT.app Codex 的“指导式智能路由”配置。它把一个任务交给 GPT-5.6 Sol max 统一编排，再按工作性质调用产品、技术实现和前端视觉专家：

| 角色 | 模型 | 主要职责 |
| --- | --- | --- |
| 主代理 / 技术架构师 | GPT-5.6 Sol max | 理解任务、确定技术架构、拆解依赖、控制风险、汇总最终答案 |
| 产品经理 | Claude Fable 5 max | PRD、用户故事、用户旅程、优先级、验收标准、产品评审 |
| 代码落地工程师 | GPT-5.6 Luna max | 后端、普通代码、调试、测试、代码评审和实现交付 |
| 原型 / 前端专家 | Kimi K3 max | 原型、UI、前端、交互和视觉设计；`k3[1m]` 不可用时降级为 `k3` |

包内只有脱敏模板和说明，不包含任何登录令牌、API Key、`auth.json`、CLI shim、运行时选择、历史记录或本机日志。

> 这是公开发布版。所有 provider endpoint 和上游代理都使用环境变量占位符：
> `OPENAI_CODEX_BASE_URL`、`KIMI_CODE_BASE_URL`、`CLAUDE_FABLE_BASE_URL`、
> `UPSTREAM_PROXY_URL`。克隆后请在目标设备的本地私密配置中填写真实值，
> 不要把真实 URL、IP 或密钥回填到 Git；没有上游代理时删除模板中的 `proxy` 行。

## 1. 背景与要解决的问题

不同工作阶段需要不同模型的优势：产品问题重在目标和验收，架构问题重在约束和风险，代码落地重在可执行性，原型与前端重在视觉和交互。如果所有问题都交给同一个模型，容易出现需求未澄清就写代码、架构与实现脱节，或者 UI 只停留在描述层。

这套方案在本机运行一个 OpenCodex 网关：ChatGPT.app 继续负责登录和 Codex 运行时，OpenCodex 负责把请求转译到 OpenAI Responses、Kimi Coding 和 Claude-compatible `/v1/messages` 三类 provider。Sol 通过 `injectionPrompt`、`subagentModels` 和 `multiAgentGuidanceEnabled` 获得角色规则，按任务类型选择子代理。

这里的“智能路由”是模型引导式编排，不是一个独立的、对所有输入做硬编码分类的规则引擎。关键任务仍可显式指定完整的 `provider/model`；模型不可用时必须按文档执行降级并记录原因。

## 2. 目标与非目标

### 目标

- 保留 ChatGPT.app 的登录态和原生 Codex 体验。
- 让 Sol max 负责技术架构和最终决策。
- 让 Fable 5 负责产品定义，不承担代码落地。
- 让 Luna max 负责可执行代码、测试和修复。
- 让 K3 保持原型、UI、前端和视觉职责。
- 用脱敏模板把可迁移部分交给更多设备和团队成员。
- 通过备份、健康检查、冒烟测试和回滚降低升级风险。

### 非目标

- 不把任何个人登录态或密钥打包给其他人。
- 不承诺模型额度、供应商可用性或网络质量永久不变。
- 不把 `ANTHROPIC_*` 变量全局写入每台机器的 shell；这可能绕过 OpenCodex 的本地 Claude 代理。
- 不把“产品评审通过”误认为代码已经实现或部署完成。

## 3. 术语和边界

- **主代理 / 编排器**：当前对话中的 Sol max，拥有最终合并权。
- **子代理**：由 Sol 通过 `spawn_agent` 委派的、职责单一的模型任务。
- **provider**：OpenCodex 配置中的 `openai`、`kimi-code`、`claude-fable`。
- **adapter**：provider 与协议的转译器：`openai-responses`、`openai-chat`、`anthropic`。
- **runtime**：ChatGPT.app 内置 Codex 可执行文件；每台设备应自行选择，不迁移旧机器路径。
- **OpenCodex 端口**：模板默认使用本地网关端口 `10100`；主机名用 `<LOCAL_GATEWAY_HOST>` 占位，不在公开模板中固定设备地址。
- **上游网络代理**：配置中的 `${UPSTREAM_PROXY_URL}`，是可选的设备级网络代理，不是 OpenCodex 端口。

## 4. 技术架构

```mermaid
flowchart LR
  User[用户 / ChatGPT.app] --> Sol[GPT-5.6 Sol max\n主代理与技术架构]
  Sol --> Fable[Claude Fable 5 max\n产品经理]
  Sol --> Luna[GPT-5.6 Luna max\n代码落地]
  Sol --> K3[Kimi K3 max\n原型 UI 前端]
  User --> Gateway[OpenCodex\n<LOCAL_GATEWAY_HOST>:10100\nwebsockets=true]
  Gateway --> OpenAI[ChatGPT.app Codex\nOpenAI Responses\nauthMode=forward]
  Gateway --> Kimi[Kimi Coding\nOpenAI Chat\nAPI Key]
  Gateway --> Claude[Claude-compatible endpoint\n/v1/messages\nAPI Key]
  Gateway -. 可选 .-> NetProxy[上游网络代理\n${UPSTREAM_PROXY_URL}]
```

请求链路分成四层：

1. ChatGPT.app 提供原生 Codex runtime 和 OpenAI 登录转发。
2. OpenCodex 在本机 `10100` 监听并按 provider/model 做协议转换。
3. `openai` 使用 `openai-responses`；`kimi-code` 使用 `openai-chat`；`claude-fable` 使用 `anthropic`，其 base URL 后接 `/v1/messages`。
4. 需要访问外部服务时，再经设备自己的上游网络代理；该代理是否存在必须按目标设备单独确认。

`websockets=true` 是当前 ChatGPT.app Codex `0.147.0-alpha.1.2` 的兼容性实测结论，用于避免 Responses 请求落入不兼容的 HTTP fallback；升级 runtime 后应重新做冒烟测试，不应把它当成所有版本的永久规则。

## 5. 路由策略与降级

| 输入特征 | 首选子代理 | effort | 降级 / 备注 |
| --- | --- | --- | --- |
| 需求澄清、PRD、用户故事、优先级、验收标准 | `claude-fable/claude-fable-5` | `max` | 当前自定义 endpoint 要求裸模型名；1M 是上下文能力标注；仍需 Sol 审核 |
| 技术架构、依赖、接口边界、风险决策 | Sol max（父代理） | `max` | 不把最终架构决策下放给子代理 |
| 普通代码、后端、调试、测试、评审 | `openai/gpt-5.6-luna` | `max` | 由 Sol 重新拆分或重试，不能把 Fable 当代码执行器 |
| 原型、UI、前端、交互、视觉 | `kimi-code/k3[1m]` | `max` | 降为 `kimi-code/k3` |
| 混合任务 | Fable → Sol → Luna/K3 | `max` | 每个子任务自包含，按依赖顺序执行 |

跨 provider 委派时覆盖模型或 effort，必须使用 `fork_turns: "none"`，并在子任务中写全背景、输入、输出格式和验收标准。不要把完整历史 fork 给 Kimi 或 Claude。OpenCodex v2 的跨 provider 加密消息在部分版本可能导致子任务正文为空；若日志显示 child body 为空，可执行 `ocx v2 mode v1` 后重启代理，再复测跨 provider 委派。

## 6. 配置实现

### 6.1 OpenCodex provider

模板中的关键片段如下，真实密钥只通过环境变量解析：

```json
"claude-fable": {
  "adapter": "anthropic",
  "baseUrl": "${CLAUDE_FABLE_BASE_URL}",
  "authMode": "key",
  "apiKey": "${CLAUDE_FABLE_API_KEY}",
  "defaultModel": "claude-fable-5",
  "models": ["claude-fable-5"],
  "selectedModels": ["claude-fable-5"],
  "liveModels": false
}
```

`${ENV_VAR}` 会在 OpenCodex 进程启动时从环境解析。`liveModels: false` 表示使用静态清单，不代表供应商额度或模型永远可用；模型、额度和 endpoint 变化后要重新验证。

注意：Claude 客户端的 `claude-fable-5[1M]` 是上下文窗口标记；当前 OpenCodex 的自定义 Anthropic adapter 会把 provider model id 原样发送到 `/v1/messages`，而本 endpoint 接受的是裸 `claude-fable-5`。因此路由和冒烟测试使用裸模型名，同时把上下文窗口记录为 1,000,000；不要在这个 provider 上显式追加 `[1M]`。

### 6.2 Shell 凭据边界

`kimi-env.zsh.example` 和 `claude-env.zsh.example` 只读取 `~/Downloads/kimi.ini`、`~/Downloads/claude.ini` 中的 `apikey:` 行，并导出 `KIMI_CODE_API_KEY`、`CLAUDE_FABLE_API_KEY`。文件应为 `chmod 600`，不要打印变量值。

如果目标设备使用 `codex-shim` 或其他非交互方式自动启动 OpenCodex，还要把 `opencodex-provider-env.sh.example` 安装为 `~/.opencodex/provider-env.sh`，并让 shim 在执行 `ocx ensure` 前 source 它；仅修改 `.zshrc` 覆盖不到 GUI/后台启动。当前原设备的 shim 已加入这段加载逻辑。

用户提供的 `ANTHROPIC_AUTH_TOKEN`、`ANTHROPIC_BASE_URL`、默认模型变量适合“直接启动 Claude 客户端”的独立 shell。使用 OpenCodex 时不要全局导出它们，否则 `ocx claude` 可能继承外部 URL，绕过本地网关；包内示例仅以注释形式保留这种直接客户端用法。

更安全的团队部署可把密钥放入 macOS Keychain 或企业 secret manager，再在启动服务时注入变量；`.zshrc` 只是兼容性示例。

## 7. 新设备迁移步骤

以下命令在目标 macOS 的终端执行。不要把当前设备的 `auth.json`、shim 或历史复制过去。

### 7.1 准备 runtime 和 OpenCodex

1. 安装或更新 ChatGPT.app，登录同一个 ChatGPT 账号，确认 Codex 可用。
2. 安装 Node.js 18+ 和 OpenCodex：

   ```bash
   npm install -g @bitkyc08/opencodex
   ```

3. 将 `opencodex-config.template.json` 复制到 `~/.opencodex/config.json`。已有文件先备份：

   ```bash
   mkdir -p ~/.opencodex
   [ -f ~/.opencodex/config.json ] && cp -p ~/.opencodex/config.json ~/.opencodex/config.json.before-migration
   cp opencodex-config.template.json ~/.opencodex/config.json
   chmod 600 ~/.opencodex/config.json
   jq empty ~/.opencodex/config.json
   ```

   先把 `opencodex-endpoints.env.example` 复制为 `~/.opencodex/endpoints.env`，
   只在该设备本地填入四个 endpoint 变量并执行 `chmod 600`；不要把填写后的文件放回 Git。
   如果不需要上游代理，删除 `config.json` 中的可选 `proxy` 行。

4. 让目标设备自己选择 ChatGPT.app runtime：

   ```bash
   ocx doctor
   ```

   预期路径类似 `/Applications/ChatGPT.app/Contents/Resources/codex`。不要复制旧机器的 `codex-runtime.json` 或 `codex-shim.json`。

### 7.2 注入密钥

推荐为每台设备创建独立、可撤销的 Kimi 和 Claude key。若使用本地 ini 文件：

```bash
chmod 600 ~/Downloads/kimi.ini ~/Downloads/claude.ini
cat kimi-env.zsh.example >> ~/.zshrc
cat claude-env.zsh.example >> ~/.zshrc
exec zsh -l
printenv KIMI_CODE_API_KEY | awk '{ print "Kimi key loaded: " (length($0) > 0 ? "yes" : "no") }'
printenv CLAUDE_FABLE_API_KEY | awk '{ print "Claude key loaded: " (length($0) > 0 ? "yes" : "no") }'
```

绝不执行 `echo $KIMI_CODE_API_KEY` 或 `echo $CLAUDE_FABLE_API_KEY`。

### 7.3 合并 Codex 默认值

只合并包中的 `codex-defaults.toml` 顶层值，不覆盖目标设备完整的 `~/.codex/config.toml`：

```toml
model = "gpt-5.6-sol"
model_reasoning_effort = "max"
```

保留目标设备自己的项目授权、MCP、hooks、skills 和其他设置。不要手写 `openai_base_url`；`ocx start` 会临时接管并在停止时恢复。

### 7.4 启动、验收和结束

```bash
ocx start
ocx health --json
```

四个最小冒烟请求（按供应商额度决定是否执行）：

```bash
codex -m gpt-5.6-sol "Reply with exactly OK"
codex -m 'claude-fable/claude-fable-5' "Reply with exactly OK"
codex -m 'openai/gpt-5.6-luna' "Reply with exactly OK"
codex -m 'kimi-code/k3[1m]' "Reply with exactly OK"
```

验收后，如果希望 ChatGPT.app 继续通过网关运行，保持 `ocx` 启动；如果明确要恢复原生直连，再执行：

```bash
ocx stop
ocx health --json
```

`ocx stop` 后 `ok:false` 是预期状态，不是配置损坏；再次使用前执行 `ocx start`。

## 8. 验收、观测与回滚

### 验收清单

- `jq empty ~/.opencodex/config.json` 成功。
- `ocx health --json` 显示 `ok:true`、端口为 `10100`。
- Sol、Fable、Luna、K3 四个 provider/model 至少各有一次成功或有明确的供应商/额度失败证据。
- 日志能区分 `openai`、`claude-fable`、`openai/gpt-5.6-luna`、`kimi-code/k3[1m]`，不记录密钥或完整敏感 prompt。
- 混合任务能看到自包含子任务和最终由 Sol 汇总的结果。
- 停止测试后按约定保留或停止代理，并记录当前状态。

### 分层排查

1. **进程层**：`ocx health --json`、`pgrep -af opencodex`，确认端口是否监听。
2. **传输层**：`websockets=true`；出现 `502 /v1/responses` 时先检查代理状态和 runtime，而不是立即换模型。
3. **provider 层**：检查模型名、adapter、上游 URL、API key 是否为空；Claude endpoint 应提供 `/v1/messages`。
4. **账号 / 额度层**：OpenAI 转发依赖目标设备 ChatGPT.app 登录态；Kimi 和 Claude 依赖各自 key、权限和额度。
5. **编排层**：child body 为空时检查 v2 跨 provider 限制，必要时按上文切换 v1；确认子任务没有 full-history fork。

### 回滚

```bash
cp -p ~/.opencodex/config.json.before-migration ~/.opencodex/config.json
ocx stop
```

本次原设备更新前的备份保存在打包目录的 `work/backups/20260806-fable-before-edit/`，仅供本机恢复，不要放进迁移 ZIP。

## 9. 实战踩坑记录与解决方案

以下记录来自本机真实迁移、重启、502 排查和四路由复测。遇到错误时先按“进程 → 请求 → provider → 账号/额度”的顺序定位，不要看到 502 就直接换模型或改密钥。

### 9.1 `10100` 和 `7897` 是两条不同链路

| 端口 | 作用 | 典型证据 | 处理方式 |
| --- | --- | --- | --- |
| `<LOCAL_GATEWAY_HOST>:10100` | Codex 必须访问的 OpenCodex 本地监听 | `lsof -nP -iTCP:10100 -sTCP:LISTEN` 有 `bun`；`ocx health --json` 为 `ok:true` | 没监听时先 `ocx service repair`，不要先换 provider |
| `${UPSTREAM_PROXY_URL}` | Clash 等软件的上游出站网络代理 | 使用目标设备的代理 URL 执行 `curl -x "${UPSTREAM_PROXY_URL}" ...` 得到 HTTP 响应（例如 405） | 只证明出站代理可达，不能代替 10100，也不能证明 provider 鉴权成功 |

曾出现过“上游代理可达但本地网关没启动”的情况，ChatGPT.app 因而显示 `502 ... <LOCAL_GATEWAY_BASE_URL>/v1/responses`。此时请求没有进入 OpenCodex，`usage.jsonl` 不会有对应记录。

### 9.2 macOS launchd 的旧式加载会制造假成功

原先使用 `launchctl load -w/unload` 时，命令可能返回成功，但重启后出现“服务已安装、未加载”，或者磁盘 plist 与 launchd 中的旧 job 不一致。解决方案是使用当前 macOS 的用户域生命周期：

```bash
ocx service repair
ocx health --json
ocx status
launchctl print gui/$(id -u)/com.opencodex.proxy
launchctl print-disabled gui/$(id -u) | rg 'com\.opencodex\.proxy'
```

当前实现使用 `bootstrap/bootout/enable`，并在 `bootout` 后等待旧 job 完全退出再重新 `bootstrap`。`KeepAlive` 能拉起意外退出的进程，但不能阻止人为执行 `ocx service stop` 或 `launchctl bootout`；如果 job 本身不存在，先修复注册，不要只看进程日志。

### 9.3 GUI/launchd 不会读取 `.zshrc`

在终端里 `printenv` 能看到 key，不代表 ChatGPT.app 或 launchd 服务能看到。解决方案是使用非交互启动钩子 `~/.opencodex/provider-env.sh`，优先从 Keychain 注入 Kimi/Claude key，必要时再读取权限为 600 的 ini 文件；不要把 key 写入 `config.json`。

如果 `service.log` 反复出现 `awk: can't open file .../kimi.ini`，不是 Kimi 上游故障，而是启动钩子在文件不存在时仍执行了 `awk`。先用 `[ -r "$path" ]` 判断文件，再读取并将 fallback 解析限定在子进程环境；不要让诊断噪音污染服务日志。

只验证变量“存在”，不要打印值：

```bash
pid=$(lsof -t -iTCP:10100 -sTCP:LISTEN | head -n1)
ps eww "$pid" | tr ' ' '\n' | rg '^KIMI_CODE_API_KEY=.'
ps eww "$pid" | tr ' ' '\n' | rg '^CLAUDE_FABLE_API_KEY=.'
```

迁移包只带环境变量模板，不带 Keychain、ini、`auth.json` 或真实日志。

### 9.4 502 必须区分“没进网关”和“上游返回 502”

- `ocx health` 失败、`10100` 没监听：这是本地服务生命周期问题，先 `ocx service repair`。
- `10100` 正常且 `usage.jsonl` 有记录：这是已经进入路由，要看 `provider`、`model`、`status`、`errorCode`、`upstreamError` 和耗时。
- 之前的 Luna 502 曾包含“输入超过模型上下文窗口”的上游错误；这和 10100 没监听完全不同，应缩短上下文或拆分任务。
- 之前 Kimi/Fable 的早期 502 出现在凭据注入修复前；Keychain/非交互环境注入后，最小 Kimi、Fable、Luna 请求均成功过。

### 9.5 用脱敏日志定位上游错误

OpenCodex 已有内置 usage debug，不需要另写会泄露密钥的抓包器：

```bash
ocx debug usage on
ocx debug usage status
ocx debug usage logs -f
```

持久文件是 `~/.opencodex/usage-debug.jsonl`，只保留滚动的 100 行，权限 600；它记录 provider/model、上游状态、响应类型和有限的脱敏错误片段，不记录入站提示词或 API Key。完整结构化路由记录仍在 `~/.opencodex/usage.jsonl`。

复现后可筛选 5xx：

```bash
ocx observe logs --status 5xx --limit 20 --jsonl
```

调试完成后关闭运行时采集：

```bash
ocx debug usage off
```

如果曾为了让 launchd 重启后仍采集而在 `provider-env.sh` 中设置 `OPENCODEX_USAGE_DEBUG=1`，还要删除该行后再执行一次 `ocx service repair`。

### 9.6 手工 Responses 探针不能随便拼请求体

当前 ChatGPT backend 的 Responses 探针要求 `input` 是列表，并要求 `store=false`、`stream=true`。直接使用旧版 `ocx access test ... --protocol responses` 可能得到 `400 Input must be a list`；这是探针请求体与当前上游契约不一致，不等于 provider 挂了。优先用真实 ChatGPT.app 请求复测，或用符合当前契约的最小请求；把上游返回的具体 400/401/429/5xx 原文记入排查记录。

### 9.7 Fable 的 `[1M]` 只用于能力标注

Claude 客户端显示的 `claude-fable-5[1M]` 不能直接作为当前自定义 Anthropic endpoint 的 wire model。曾出现 `503 model_not_found`；解决方案是发送裸 `claude-fable-5`，把 1,000,000 作为上下文窗口元数据保存。

### 9.8 一分钟排查顺序

```bash
# 1. 本地网关是否活着
ocx health --json
lsof -nP -iTCP:10100 -sTCP:LISTEN

# 2. launchd 是否真的加载了当前 plist
ocx status
launchctl print gui/$(id -u)/com.opencodex.proxy

# 3. 不活跃时重建服务
ocx service repair

# 4. 需要看上游具体原因时开启脱敏日志
ocx debug usage on
```

服务恢复后，再完全退出并重新打开 ChatGPT.app，让 app-server 重新读取 `openai_base_url`；不要为了重载客户端而执行 `ocx service stop`。

## 10. 安全、隐私和团队分发

- ZIP 只分发模板和操作说明；每台设备独立申请 key，并设置最小权限、额度和撤销流程。
- 不把 `kimi.ini`、`claude.ini`、`auth.json`、`responses-state.json`、`usage.jsonl`、Codex history 或 shim 放入 ZIP、Git、公共网盘、工单或聊天记录。
- 本地网关默认只绑定设备本机回环地址，不把 `10100` 暴露到局域网或公网；具体绑定地址由目标设备配置决定。
- 共享任务前先做数据分级和脱敏；不要把生产密钥、个人信息或未授权客户数据发送给任何 provider。
- 密钥疑似泄露时，立即停止代理、吊销/轮换 key、检查日志和 shell 历史，再从备份恢复配置。
- 分发前做 ZIP 完整性和 secret scan；接收方先核对 SHA-256，再在隔离设备完成四路由验收。

建议团队为这套配置维护版本号、模型能力矩阵、额度负责人、变更审批人和每季度复测日期；供应商模型名或 endpoint 改动时，不要只修改 README，应同步更新模板、冒烟脚本和验收记录。

## 11. 常见问题

### 为什么 ChatGPT.app 请求突然挂掉？

最常见原因是 `ocx` 已停止，或者 `openai_base_url` 没有指向目标设备的 `<LOCAL_GATEWAY_BASE_URL>/v1`。先执行 `ocx start` 和 `ocx health --json`，再重试。

### Fable 返回 401、404、503 或 TLS 错误怎么办？

确认 `CLAUDE_FABLE_API_KEY` 已加载、endpoint 可达、证书被目标设备信任，并确认服务提供 `/v1/messages`。本 endpoint 对带 `[1M]` 后缀的 wire model 返回过 `503`，应使用裸 `claude-fable-5`；`401` 则检查 key 是否有效。不要为了绕过证书错误而全局关闭 TLS 校验；先让 endpoint 管理者修复证书或提供受信 CA。

### K3 的 `k3[1m]` 不可用怎么办？

显式改用 `kimi-code/k3`，并保留原型/UI/前端职责。不要把 K3 的失败误判为 OpenAI 或 Claude provider 故障。

### 为什么文档不直接设置 `ANTHROPIC_BASE_URL`？

因为全局变量会被 OpenCodex/Claude launcher 继承，可能绕过本地网关。需要直接使用 Claude 客户端时，在独立 shell 临时设置，完成后退出该 shell。

## 12. 包内容与参考

```text
README-migration-zh-CN.md       本说明
opencodex-config.template.json   脱敏 OpenCodex 配置
codex-defaults.toml              Sol 默认模型与 effort
kimi-env.zsh.example             Kimi key 环境变量示例
claude-env.zsh.example           Claude Fable key 环境变量示例
opencodex-provider-env.sh.example 非交互启动的 key 环境变量示例
opencodex-endpoints.env.example   设备私有 endpoint 环境变量模板
```

参考：

- [OpenCodex GitHub](https://github.com/lidge-jun/opencodex)
- [OpenCodex 中文说明](https://github.com/lidge-jun/opencodex/blob/main/readme/README.zh-CN.md)
- [Kimi Code 文档](https://www.kimi.com/code/docs/)
- [Kimi Code 的 Codex 接入](https://www.kimi.com/code/docs/third-party-tools/codex.html)
