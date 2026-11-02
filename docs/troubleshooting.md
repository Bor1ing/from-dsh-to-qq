# 深度排障记录：`reading 'current'` 与 13 个坑

本文是 [README](README.md) 的技术补充。README 讲"怎么做"，这里讲"为什么"，以及出问题时**具体该看哪个文件的哪一行**。

适用版本：`dsh-onebot-qq@0.9.8` + NapCat 4.18.42 + QQ NT 9.9.36 + DSH 桌面版（Electron 外壳 44.7.0）。

---

## 1. 排障基本盘：先看 trace，再看日志

插件把所有链路节点写进一个 jsonl：

```
%USERPROFILE%\.dsh\media\onebot\qq-trace.jsonl
```

前提是配置里开了：

```yaml
traceEnabled: true
traceLevel: debug
recordInbound: true
```

实测出现过的 `stage` 取值只有六个：

| stage | 含义 |
|---|---|
| `inbound` | 收到 NapCat 上报的消息 |
| `dispatch` | 投递给某个 chat 的 agent |
| `agent` | 模型回合（真正干活的地方） |
| `outbound` | 发送回复 |
| `command` | 斜杠命令（`/status`、`/model` 等） |
| `queue` | 断线时排队，等重连补发 |

一次成功回复：

```json
{"stage":"inbound","ok":true, ...}
{"stage":"dispatch","ok":true, ...}
{"stage":"agent","ok":true, ...}
{"stage":"outbound","ok":true,"reason":"回复已发送","data":{"segments":1}}
```

**判读口诀**：哪一步最后出现，问题就在它之后那一步。

- 只有 `inbound`：消息进来了但没投递 → 权限策略（`adminUsers` / `dmPolicy` / `groupPolicy`）拦掉了
- `inbound` → `dispatch` → 停在 `agent`：模型回合崩了（本文主角）
- 到 `agent` 有 `outbound` 但 `ok:false`：发送失败，看 `reason`

---

## 2. 主角：每条消息都回 `Cannot read properties of undefined (reading 'current')`

### 2.1 症状

QQ 里收到：

```
⚠️ 运行出错：Cannot read properties of undefined (reading 'current')
（参考号 MF-xxxxxxxx，可凭此号在 qq-trace.jsonl 中检索本次故障全过程）
```

trace 里 `inbound`、`dispatch` 都 `ok:true`，`agent` 立刻失败——**8ms 内**。

这段"立刻崩"很关键：说明崩在回合启动阶段（组装 system prompt），**还没走到模型请求**。所以跟消息内容、模型选择、工作区路径全都无关，改配置没用。

### 2.2 第一反应（都错了）

| 试过的方向 | 结果 |
|---|---|
| 怀疑插件 `lib/commands.js` 里 `chat?.agent.model.current()` 三处 | 改成 `chat?.agent?.model?.current?.()`，**错误照旧**；而且崩在命令处理之前，根本走不到 |
| 重装插件 | 无效 |
| 换模型 / 换工作区 | 无效 |
| 看 DSH 主进程 stderr | 没有对应记录 |

### 2.3 卡点：错误被包装，堆栈丢失

在插件 `lib/bridge.js` 的 `turn/end` 错误分支里打印 `reason.error`：

```js
appendFileSync(logPath, String(reason.error));      // 得到 "[object Object]"
```

改成 dump 全部字段：

```js
const e = reason.error;
[
  'message=' + e.message,
  'name=' + e.name,
  'stack=' + e.stack,
  'json=' + JSON.stringify(e),
  'keys=' + Object.getOwnPropertyNames(e).join(','),
].join(' | ')
```

拿到：

```
message=Cannot read properties of undefined (reading 'current')
name=undefined
stack=undefined
json={"message":"Cannot read properties of undefined (reading 'current')","code":"UNKNOWN"}
keys=message,code
```

**DSH 把它包装成了一个裸的 `{message, code:'UNKNOWN'}`，堆栈被吞掉了。** 这条路到此为止。

> 教训：遇到这种"信息被上游吞掉"的错误，不要继续在调用方加日志，直接去读上游源码。

### 2.4 侧面证据：会话投影缓存

DSH 把每个会话的状态投影在：

```
%USERPROFILE%\.dsh\storages\session_projcache\sessions\<sessionId>.json
```

对比两份：

| 会话 | sandboxMode | modelSelection |
|---|---|---|
| `onebot-private-<你的QQ号>.json`（出问题的） | `workspace-write` | `{"lastUsed":null,"pending":null}` |
| `session-<uuid>.json`（Web 界面正常会话） | `danger-full-access` | `{"lastUsed":{"provider":"deepseek-account","model":"deepseek-flash","reasoningEffort":"low"},"pending":null}` |

QQ 会话的 `modelSelection.lastUsed` 是 **null**——于是很容易误判成"模型没选中导致的"。实际这是**结果**不是**原因**（补丁打上后，这一项依然是 null，但回合能正常跑）。

### 2.5 破局：把 DSH 的 asar 解开读源码

DSH 的核心代码在安装目录的 `resources\app.asar` 里，普通编辑器打不开。asar 的二进制布局：

```
offset 0 : u32 = 4
offset 4 : u32 = pickle 大小
offset 8 : u32
offset 12: u32 = header JSON 长度
offset 16: header JSON（文件名 → {offset, size, unpacked} 的树）
数据区起点 = 16 + 对齐后的 header 长度
```

解包脚本（Node，注意跳过 `unpacked: true` 的条目，它们没有 offset）：

```js
const fs = require('fs'), path = require('path');
const ASAR = '<DSH安装目录>\\resources\\app.asar';
const OUT  = '<解包输出目录>';

const fd = fs.openSync(ASAR, 'r');
const buf = Buffer.alloc(8 * 1024 * 1024);
fs.readSync(fd, buf, 0, buf.length, 16);          // header JSON 从 16 开始

// 花括号深度扫描找 JSON 结尾（跳过字符串里的括号）
let depth = 0, inStr = false, esc = false, end = -1;
for (let i = 0; i < buf.length; i++) {
  const c = buf[i];
  if (inStr) { if (esc) esc = false; else if (c === 92) esc = true; else if (c === 34) inStr = false; continue; }
  if (c === 34) { inStr = true; continue; }
  if (c === 123) { depth++; continue; }
  if (c === 125 && --depth === 0) { end = i + 1; break; }
}

const header = JSON.parse(buf.slice(0, end).toString('utf8'));
const base = 16 + Math.ceil(end / 4) * 4;

function walk(node, prefix) {
  for (const [name, v] of Object.entries(node.files || {})) {
    const p = prefix ? prefix + '/' + name : name;
    if (v.files) { walk(v, p); continue; }
    if (v.unpacked || v.offset === undefined) continue;   // ← 关键，否则 readSync 报 NaN
    if (!/\.(js|mjs|cjs)$/.test(p)) continue;
    const out = path.join(OUT, p);
    fs.mkdirSync(path.dirname(out), { recursive: true });
    const b = Buffer.alloc(Number(v.size));
    fs.readSync(fd, b, 0, b.length, base + Number(v.offset));
    fs.writeFileSync(out, b);
  }
}
walk(header, '');
```

在解出的树里 grep：

```bash
grep -rn "\.current\b" --include=*.js . | grep -i "model\|selection"
```

命中：

```
dsh/node_modules/@deepseek-ai/dsh-agent/lib/types/model-selection.js:47:  const selected = selection.current;
```

打开看签名：

```js
export function installModelSelection(agentCtx, selection) {
    const disposeAssembly = agentCtx.on('system-prompt/assemble', async (_assembly, _context, next) => {
        const selected = selection.current;      // ← selection 是 undefined 就炸这里
        const assembled = await next();
        selection.assembled = selected;
        ...
```

**双参数**。

再看插件调用处 `node_modules/dsh-onebot-qq/lib/registry.js`：

```js
buildSetup(recordedPreset) {
    return async (agentCtx) => {
        this.deps.installChannelScope(agentCtx);
        await this.joinPreset(agentCtx, recordedPreset);
        // dsh 0.2.2: host-style unconditional install (single-arg — it only
        // wires the model-switch notice listener; the selection itself lives
        // on agent.model). ...
        installModelSelection(agentCtx);          // ← 只传了 1 个
    };
}
```

插件作者是按 **DSH 0.2.2** 的新签名（单参数）写的，遇到双参数的老版本就必然崩。

### 2.6 修复

```js
// 备份：cp registry.js registry.js.bak-model
const selectionHolder = {};
Object.defineProperty(selectionHolder, 'current', {
    configurable: true,
    get() {
        try {
            return agentCtx.agent?.model?.current?.();
        }
        catch {
            return undefined;
        }
    },
});
installModelSelection(agentCtx, selectionHolder);
```

用 getter 让 `current` 每次现取，兼容两种 API：新版本忽略第二个参数，老版本拿到一个永远有 `.current` 的 holder，不会再炸。

对应补丁文件：`patches/dsh-onebot-qq-0.9.8-installModelSelection.patch`。

应用方式（在 profile 目录下）：

```bash
cd "%USERPROFILE%\.dsh\profiles\<profile>"
patch -p1 < dsh-onebot-qq-0.9.8-installModelSelection.patch
```

打完重启 DSH，问题消失。

> ⚠️ 补丁位于 `node_modules`，**升级或重装插件会被覆盖**。升级前先确认新版是否已修此签名（看 `registry.js` 里是不是还只有单参数）。

---

## 3. 十三个坑一览

| # | 坑 | 严重度 | 一句话解法 |
|---|---|---|---|
| 1 | `reading 'current'` 每回合必崩 | ★★★★★ | 打 `installModelSelection` 补丁 |
| 2 | 装不到最新版插件 | ★★★★ | pnpm `minimumReleaseAge` 拦截；先卸载再指定版本 |
| 3 | 普通 QQ 与 NapCat 互斥 | ★★★★ | 先 `Stop-Process QQ`，只从 launcher 起 |
| 4 | 终端里起的 NapCat 随命令被杀 | ★★★★ | 改用计划任务 |
| 5 | NapCat WebUI 登录页输不进 token | ★★★ | URL 带 `?token=` |
| 6 | 扫码登录玄学失败 | ★★★ | 用"快速登录" |
| 7 | 登录后配置被改坏（`enable:false`） | ★★★ | 网络配置页开关关一次再开 |
| 8 | 装好了谁都不理 | ★★★ | `adminUsers` 不能空 |
| 9 | 群里乱回 | ★★ | `groupPolicy: disabled` |
| 10 | 中文长回复转图失败 | ★★ | 配 `t2i` 字体，或调大阈值 |
| 11 | 脚本重启 DSH 起不来 | ★★★ | 手动开关，别用脚本 |
| 12 | Windows 沙箱 `grantWrite` 失败 | ★★ | 改文件策略为完全访问 |
| 13 | 想读 DSH 源码打不开 | ★ | 解 `app.asar`（见 2.5） |

### 坑 2：装不到最新版

**现象**：想装 0.9.10，界面黄字提示"根据 pnpm 供应链安全策略（minimumReleaseAge），发布时间未满规定时长的版本不会被自动安装"，实际装成 0.9.8。

**机制**：DSH 插件安装器有一层防供应链攻击的等待期，新发布的版本默认不让装。

**解决**：先卸载，再显式带版本号安装：`dsh-onebot-qq@0.9.10`。

### 坑 3：普通 QQ 与 NapCat 互斥

**现象**：NapCat 启动后闪退，6099 不监听。

**机制**：NapCat 靠向官方 `QQ.exe` 注入 `NapCatWinBootHook.dll` 工作。你已经用普通方式开了一个 QQ，注入就失败。

**判断**：

```powershell
# NapCat 注入成功后，QQ 的子进程命令行里会出现 napcat.mjs
Get-Process QQ | ForEach-Object { (Get-CimInstance Win32_Process -Filter "ProcessId=$($_.Id)").CommandLine }
# 模块里应该有 Hook dll
Get-Process QQ | ForEach-Object { $_.Modules | Where-Object { $_.ModuleName -like '*NapCat*' } }
```

**解决**：杀掉所有 QQ，只通过 NapCat 的 launcher 启动。日常别单独开 QQ。

### 坑 4：终端里的 NapCat 会随命令结束被杀

**现象**：明明起来了、6099 也监听了，一会儿整个 QQ/NapCat 都没了。

**机制**：在 shell / agent 里起后台进程，进程属于该命令的进程树，命令结算时被连带清理。

**解决**：用 Windows 计划任务（见 README 7.1）。用 `Get-ScheduledTaskInfo` 看真实退出码：

```
0x41301 (267009)     正在运行
0xC000013A (3221225786)  被 Ctrl+C / 控制台关闭杀掉
```

### 坑 5：WebUI 输不进 token

**现象**：打开 `/webui` 跳到 `/webui/qq_login`，提示"获取QQ列表失败: Unauthorized""获取二维码失败: Unauthorized"，输入框点不进焦点。

**解决**：token 直接拼在 URL 上，包括跳转后的页面：

```
http://127.0.0.1:6099/webui?token=<WebUI密码>
http://127.0.0.1:6099/webui/qq_login?token=<WebUI密码>
```

### 坑 6：优先快速登录

`-q <机器人QQ号>` 参数或面板里的"快速登录"走的是本机已有登录态，不弹二维码。二维码真出不来时它在 `<NapCat目录>\cache\qrcode.png`。

### 坑 7：登录成功后配置被回写

**现象**：登录成功反而连不上 DSH；`onebot11_<QQ号>.json` 里 `enable` 变成 `false`，还混进一条不认识的 `ws://localhost:8082` 客户端。

**解决**：NapCat 面板 → 网络配置 → 把 `dsh-onebot` 客户端开关**关掉再打开**，提示"更新配置成功"。之后 `8643` 出现 `ESTABLISHED`。

> 顺序建议：先起 NapCat 并登录成功，再去改 `onebot11_<QQ号>.json`，改完在面板里触发一次重载。

### 坑 8 / 坑 9：权限策略

```yaml
adminUsers:                    # 空 = 没人能用（fail-closed）
  - '<机器人QQ号>'
  - '<你自己的QQ号>'
dmPolicy: open                 # open = 只有 adminUsers 能私聊
groupPolicy: disabled          # disabled = 群里完全不理
```

### 坑 10：中文长回复转图失败

```
"stage":"outbound","ok":false,
"reason":"文字图卡片渲染失败，已降级为文本: t2i: no usable CJK font family (set t2i fontFiles/fontFamilies)"
```

超过 `textImageThreshold`（默认 150 字）会渲染成图片卡片，找不到中文字体就降级成纯文本——**功能不受影响**。要修就给插件配 `t2i.fontFiles` / `fontFamilies`，指向 `<系统盘>\Windows\Fonts\msyh.ttc` 之类。

### 坑 11：不要用脚本重启 DSH

用 PowerShell 脚本"杀掉 DSH → `Start-Process` 重启"：脚本结束时 Electron 被连带回收，`Start-Process` 看着成功，进程瞬间消失。

**结论**：重启 DSH 只能手动（关窗口再点图标）。

### 坑 12：Windows 沙箱 `grantWrite` 失败

```
Error: SetNamedSecurityInfoW failed (Win32 5): grantWrite(<工作区>)
```

工作区目录只有继承 ACE，`Authenticated Users` 只有 `(M)`，不含 `WRITE_DAC`，且进程未提权 → 沙箱无法给自己改 DACL。

**解决**：把 DSH 文件策略改成完全访问，或给工作区目录显式授予当前用户 `FullControl`。

---

## 4. 排障速查表

| 症状 | 先查 |
|---|---|
| 6099 不监听 | NapCat 没起来 → 坑 3 / 4 |
| 8643 只有 LISTENING 没有 ESTABLISHED | OneBot 客户端没启用 → 坑 7 |
| 有连接但发消息无反应 | `adminUsers` / `dmPolicy` / `groupPolicy` |
| `reading 'current'` | 坑 1，打补丁 |
| `⚠️ 连接断开，N 秒后重试` | DSH 正在重启，等几秒自动重连 |
| 重复回复 / 顺序乱 | NapCat 重连重发，`dedupWindowSeconds` 兜底 |
| 想看一次消息的完整链路 | `qq-trace.jsonl` |

---

## 5. 关键文件位置

| 用途 | 路径 |
|---|---|
| 插件配置 | `%USERPROFILE%\.dsh\profiles\<profile>\cordis.patch.yml` |
| 插件本体 | `%USERPROFILE%\.dsh\profiles\<profile>\node_modules\dsh-onebot-qq\` |
| 要打补丁 | 上面目录 `lib\registry.js`（第 ~107 行） |
| 消息链路 | `%USERPROFILE%\.dsh\media\onebot\qq-trace.jsonl` |
| 聊天↔会话映射 | `%USERPROFILE%\.dsh\media\onebot\chat-sessions.json` |
| 会话投影缓存 | `%USERPROFILE%\.dsh\storages\session_projcache\sessions\onebot-*.json` |
| NapCat 配置 | `<NapCat目录>\config\{webui,napcat}.json`、`onebot11_<机器人QQ号>.json` |
| NapCat 日志 | `<NapCat目录>\logs\`（需 `fileLog: true`） |
| DSH 源码 | `<DSH安装目录>\resources\app.asar`（解包见 2.5） |
