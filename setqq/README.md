# 从零把 DeepSeek Harness 接入 QQ

> 用 **DSH 插件 `dsh-onebot-qq` + NapCat** 把 DeepSeek Harness 接进 QQ，全程 Windows、纯本地、不花钱、不用公网服务器。
> 这份文档是自己踩完坑后整理的，按顺序做基本能一次通。
> 文中所有路径都用 `<尖括号>` 占位，你按自己的实际情况替换即可。
> 深度排障记录见 [docs/troubleshooting.md](docs/troubleshooting.md)。

**目录**

- [0. 先填这张表](#0-先填这张表)
- [1. 一分钟速通（命令清单）](#1-一分钟速通命令清单)
- [2. 第一步：确认你的 QQ](#2-第一步确认你的-qq)
- [3. 第二步：装 DSH 插件 dsh-onebot-qq](#3-第二步装-dsh-插件-dsh-onebot-qq)
- [4. 第三步：装 NapCat](#4-第三步装-napcat)
- [5. 第四步：写三份配置](#5-第四步写三份配置)
- [6. 第五步：打补丁（重点，先跑一次再决定）](#6-第五步打补丁重点先跑一次再决定)
- [7. 第六步：启动 NapCat 并登录](#7-第六步启动-napcat-并登录)
- [8. 第七步：验证](#8-第七步验证)
- [9. 常见问题速查](#9-常见问题速查)
- [10. 文件位置速查](#10-文件位置速查)
- [11. 安全与收尾](#11-安全与收尾)

---

## 0. 先填这张表

动手前把下面这些值想好、记下来，后面每一步都会用到：

| 占位符 | 你要填的东西 | 举例 |
|---|---|---|
| `<机器人QQ号>` | 用来当机器人的号，**建议小号** | 1234567890 |
| `<你自己的QQ号>` | 你本人用来跟机器人聊天的号 | 0987654321 |
| `<NapCat目录>` | 解压 NapCat 的目录，**不能有中文和空格** | `D:\napcat` |
| `<QQ安装目录>` | 官方 QQ 安装在哪 | `C:\Program Files\Tencent\QQNT` |
| `<profile>` | DSH 的 profile 名，桌面版默认 | `desktop` |
| `<accessToken>` | 自己编一个，两边一致就行 | `my-secret-2026` |
| `<WebUI密码>` | NapCat 管理面板的 token | `napcat8888` |

> **只给自己用的话**，`<accessToken>` 和 `<WebUI密码>` 随便编，别用本文里的示例值。**不要**把真实 token 提交到 GitHub。

---

## 1. 一分钟速通（命令清单）

懒人版，细节在下面各章。**不要**直接照抄，先把占位符改掉。

```powershell
# ① 装插件（也可以直接在 DSH 界面「插件 → ＋添加插件」里搜 dsh-onebot-qq）
dsh plugin --profile <profile> add dsh-onebot-qq

# ② 解压 NapCat 到 <NapCat目录>，然后写三份配置（见第 5 章）

# ③ 打补丁（见第 6 章，先跑一次再决定要不要打）

# ④ 用计划任务起 NapCat（关键！别在终端里跑）
$action = New-ScheduledTaskAction -Execute 'cmd.exe' `
        -Argument '/c cd /d <NapCat目录> && launcher-user.bat -q <机器人QQ号>'
$princ  = New-ScheduledTaskPrincipal -UserId "$env:USERDOMAIN\$env:USERNAME" `
        -LogonType Interactive -RunLevel Limited
Register-ScheduledTask -TaskName NapCatQQ -Action $action -Principal $princ
Start-ScheduledTask -TaskName NapCatQQ

# ⑤ 浏览器打开 NapCat 面板，快速登录机器人号
#    http://127.0.0.1:6099/webui?token=<WebUI密码>

# ⑥ 重启 DSH，然后在 QQ 里给机器人发消息
```

---

## 2. 第一步：确认你的 QQ

NapCat 是靠**注入官方 QQ 客户端**工作的，所以：

- 必须是**官方 QQ NT**，不是 TIM、不是精简版、不是绿色版。
- 必须**至少登录过一次**（NapCat 要读登录态做快速登录）。
- 版本太新太旧都可能注入失败，遇到问题优先怀疑版本。

查 QQ 装在哪：

```powershell
Get-ItemProperty 'HKLM:\SOFTWARE\WOW6432Node\Microsoft\Windows\CurrentVersion\Uninstall\QQ' |
  Select-Object UninstallString
# 输出类似 C:\Program Files\Tencent\QQNT\Uninstall.exe
# 那么 QQ 主程序就是 C:\Program Files\Tencent\QQNT\QQ.exe
```

> 别自己手填路径给 NapCat，它的 `launcher` 会自己从注册表找。你只要确认 QQ 是官方原版就行。

---

## 3. 第二步：装 DSH 插件 `dsh-onebot-qq`

### 图形界面（推荐）

DSH → 左侧 **插件** → 右上 **＋添加插件** → 输入 `dsh-onebot-qq` → **安装**。

### 命令行

```powershell
dsh plugin --profile <profile> add dsh-onebot-qq
```

装完提示「已安装，请重启 DSH 加载它」就对了。

### ⚠️ 可能装不到最新版

DSH 的插件安装器有防供应链攻击策略（`minimumReleaseAge`）：**刚发布没几天的版本不让装**，会自动退到上一个版本。

你会看到类似提示：

> 根据 pnpm 供应链安全策略（minimumReleaseAge），发布时间未满规定时长的版本不会被自动安装……

想强制用某个版本，先卸载再显式指定：

```
dsh-onebot-qq@0.9.10
```

按我的经验，**0.9.8 是能跑通的**（需要打第 6 章的补丁），0.9.10 我没实测过。

---

## 4. 第三步：装 NapCat

1. 去 [NapCatQQ Releases](https://github.com/NapNeko/NapCatQQ/releases) 下载 `NapCat.Shell.zip`（带 `Shell` 字样的那个，不要 Node 版）。
2. 解压到 `<NapCat目录>`——**路径不能有中文、不能有空格**，否则启动必失败。
3. 解压后应该看到：

```
<NapCat目录>\
├─ launcher-user.bat        ← 你要用的就是这个（不提权）
├─ launcher.bat             ← 会弹 UAC，别用
├─ NapCatWinBootMain.exe
├─ NapCatWinBootHook.dll
├─ napcat.mjs
├─ qqnt.json
├─ config\                  ← 所有配置都在这
└─ cache\                   ← 二维码会存这
```

---

## 5. 第四步：写三份配置

全部在 `<NapCat目录>\config\` 下。**改完要重启 NapCat 才生效。**

### 5.1 `webui.json` —— 管理面板

```json
{
  "host": "127.0.0.1",
  "port": 6099,
  "token": "<WebUI密码>",
  "loginRate": 20,
  "autoLoginAccount": "",
  "disableWebUI": false,
  "accessControlMode": "none",
  "ipWhitelist": [],
  "ipBlacklist": []
}
```

访问地址后面会用到：

```
http://127.0.0.1:6099/webui?token=<WebUI密码>
```

> ⚠️ token 一定要带在 URL 里。只开 `/webui` 会跳到登录页让你手输 token，那个输入框经常点不进去。

### 5.2 `onebot11_<机器人QQ号>.json` —— 连 DSH 的通道

文件名**必须**是 `onebot11_` 加上机器人 QQ 号，NapCat 只认这个命名。

```json
{
  "network": {
    "httpServers": [],
    "httpClients": [],
    "websocketServers": [],
    "websocketClients": [
      {
        "name": "dsh-onebot",
        "enable": true,
        "url": "ws://127.0.0.1:8643/ws",
        "messagePostFormat": "array",
        "reportSelfMessage": false,
        "reconnectInterval": 3000,
        "token": "<accessToken>",
        "heartInterval": 30000
      }
    ]
  },
  "enableLocalFile2Url": true,
  "parseMultMsg": false,
  "musicSignUrl": ""
}
```

四个不能错的点：

- `url` 结尾必须是 `/ws`，端口 `8643` 是 DSH 插件的监听端口（不是 NapCat 的端口）。
- `token` 必须和下面插件配置里的 `accessToken` **一模一样**。
- `messagePostFormat` 必须是 `"array"`。
- `reportSelfMessage` 保持 `false`（不然自己发的消息会被当成用户消息）。

### 5.3 `napcat.json` —— 运行参数

```json
{
  "fileLog": true,
  "consoleLog": true,
  "packetBackend": "auto",
  "o3HookMode": 1
}
```

> `fileLog` 默认是 `false`，所以 `logs\` 永远是空的。**排障时一定要先改成 `true`**，跑通后再改回 `false` 省磁盘。

### 5.4 DSH 插件配置

插件配置**不能**写在 `cordis.yml`（那个文件只是个空入口），要写到：

```
%USERPROFILE%\.dsh\profiles\<profile>\cordis.patch.yml
```

在文件**末尾**追加（注意缩进，`config` 和 `id` 同级）：

```yaml
- id: dsh-onebot
  config:
    mode: reverse
    host: 127.0.0.1
    port: 8643
    accessToken: '<accessToken>'
    botQQ: '<机器人QQ号>'
    adminUsers:
      - '<机器人QQ号>'
      - '<你自己的QQ号>'
    dmPolicy: open
    groupPolicy: disabled
    traceEnabled: true
    traceLevel: debug
    recordInbound: true
```

几个容易搞错的项：

| 键 | 说明 |
|---|---|
| `mode` | `reverse` = NapCat 连 DSH（推荐）；`forward` = DSH 连 NapCat |
| `accessToken` | reverse 模式**必填**，留空插件直接拒绝启动 |
| `adminUsers` | **至少写一个**，否则谁都用不了（连私聊都不回） |
| `dmPolicy: open` | 只有 `adminUsers` 里的人能私聊机器人 |
| `groupPolicy: disabled` | 群里完全不理人。想让它进群就改 `open`（还要 @ 它） |
| `traceEnabled` | 排障必开，日志在 `%USERPROFILE%\.dsh\media\onebot\qq-trace.jsonl` |

改完**必须重启 DSH**。

---

## 6. 第五步：打补丁（重点，先跑一次再决定）

### 先别急着打

装好、配置好、重启一次 DSH，然后在 QQ 里发一条消息试试。

- **正常回复** → DSH 是新版 API，不用打补丁，跳到第 7 章。
- **报错** `⚠️ 运行出错：Cannot read properties of undefined (reading 'current')` → 回来打补丁。

### 这个错是什么原因

插件是按 DSH **新版（0.2.2）** 的接口写的，里面有一句：

```js
installModelSelection(agentCtx);        // 只传了 1 个参数
```

而你这版 DSH 的函数要 2 个参数：

```js
// dsh-agent/lib/types/model-selection.js
export function installModelSelection(agentCtx, selection) {
    ... const selected = selection.current;   // ← selection 没传，就炸在这
```

结果就是**每条消息都崩、8 毫秒内必崩**，跟消息内容、模型、工作区全都无关。

### 怎么改

文件在这里（`<用户名>` 换成你自己的）：

```
C:\Users\<用户名>\.dsh\profiles\<profile>\node_modules\dsh-onebot-qq\lib\registry.js
```

大约在 **第 107 行**，找到这一句：

```js
installModelSelection(agentCtx);
```

替换成：

```js
// PATCH: 本版 DSH 的 installModelSelection 需要 (agentCtx, selection)
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

改完重启 DSH，问题消失。

> 更省事的做法：用仓库里的 `patches/dsh-onebot-qq-0.9.8-installModelSelection.patch`，在 `%USERPROFILE%\.dsh\profiles\<profile>` 目录下执行
> `patch -p1 < dsh-onebot-qq-0.9.8-installModelSelection.patch`

> ⚠️ 补丁在 `node_modules` 里，**升级插件会被覆盖**。升级前先确认新版修没修这个问题。

---

## 7. 第六步：启动 NapCat 并登录

### 7.1 用计划任务启动（强烈推荐）

**不要**在自己的终端里直接跑 `launcher-user.bat`——那个进程属于你这个终端的进程树，终端一关它就被连带杀掉（我在这上面浪费了很久）。

```powershell
# 先确保没有普通 QQ 在跑
Get-Process QQ -ErrorAction SilentlyContinue | Stop-Process -Force

# 注册计划任务
$action = New-ScheduledTaskAction -Execute 'cmd.exe' `
        -Argument '/c cd /d <NapCat目录> && launcher-user.bat -q <机器人QQ号>'
$princ  = New-ScheduledTaskPrincipal -UserId "$env:USERDOMAIN\$env:USERNAME" `
        -LogonType Interactive -RunLevel Limited
Register-ScheduledTask -TaskName NapCatQQ -Action $action -Principal $princ

# 启动
Start-ScheduledTask -TaskName NapCatQQ
```

参数含义：

- `-RunLevel Limited` = 不提权（提权反而容易失败）
- `-LogonType Interactive` = 必须跑在已登录的桌面会话里，否则注入不了 QQ
- `-q <机器人QQ号>` = 快速登录，免扫码

### 7.2 验证 NapCat 起来了

```powershell
netstat -ano | findstr :6099
# 应该看到 127.0.0.1:6099 ... LISTENING
```

没起来？看 7.5 的排障。

### 7.3 快速登录机器人号

浏览器打开：

```
http://127.0.0.1:6099/webui?token=<WebUI密码>
```

选 **快速登录** → 下拉里选你的机器人号 → **登录**。

如果跳到了 `/webui/qq_login` 让你输 token，直接用带参数的地址重进：

```
http://127.0.0.1:6099/webui/qq_login?token=<WebUI密码>
```

> 尽量别扫码，扫码经常各种玄学失败。快速登录用的就是本机已有的登录态。

### 7.4 登录后可能要把连接开关"关一下再开"

NapCat 登录成功后**会重写自己的配置**，可能把 `enable` 改成 `false`，还会给你混进一些不认识的 WebSocket 客户端。

去 NapCat 面板 → **网络配置** → 找到 `dsh-onebot` 这个客户端 → 把开关**关掉再打开**，提示「更新配置成功」即可。

然后验证 DSH 已连上：

```powershell
Get-NetTCPConnection -LocalPort 8643 | Select-Object State,RemotePort,OwningProcess
# 期望：一条 ESTABLISHED（客户端端口是随机的）
```

### 7.5 排障

| 现象 | 原因 |
|---|---|
| 6099 根本不监听 | 你可能开着普通 QQ，两者互斥 → 见下文判断注入是否成功 |
| NapCat 启动后闪退 | 用管理员权限跑了 / 路径有中文 / QQ 版本不兼容 |
| 6099 起来了但 8643 没连接 | `onebot11_<QQ号>.json` 的 `enable` 被改成 false，或 token 不一致 |

判断 QQ 是否被正确注入：

```powershell
# 命令行里应该出现 napcat.mjs
Get-Process QQ | ForEach-Object { (Get-CimInstance Win32_Process -Filter "ProcessId=$($_.Id)").CommandLine }

# 模块里应该有注入的 DLL
Get-Process QQ | ForEach-Object { $_.Modules | Where-Object { $_.ModuleName -like '*NapCat*' } }
```

看计划任务真实状态（诊断进程为什么退出）：

```powershell
Get-ScheduledTaskInfo -TaskName NapCatQQ
# 0x41301  (267009)     = 正在运行，正常
# 0xC000013A (3221225786) = 被 Ctrl+C / 控制台关闭杀掉
```

---

## 8. 第七步：验证

按顺序打勾：

- [ ] `netstat | findstr :6099` 有 LISTENING（NapCat 面板活着）
- [ ] `netstat | findstr :8643` 有 LISTENING（DSH 插件在等连接）
- [ ] 8643 有一条 ESTABLISHED（NapCat 连上了）
- [ ] QQ 里给机器人发一条消息，能收到回复
- [ ] `%USERPROFILE%\.dsh\media\onebot\qq-trace.jsonl` 里出现完整链路

最后一条最关键，一次成功回复的日志长这样：

```json
{"stage":"inbound","ok":true, ...}
{"stage":"dispatch","ok":true, ...}
{"stage":"agent","ok":true, ...}
{"stage":"outbound","ok":true,"reason":"回复已发送","data":{"segments":1}}
```

阶段含义：`inbound` 收到 → `dispatch` 投给 agent → `agent` 模型回合 → `outbound` 发出去。
另有 `command`（斜杠命令）和 `queue`（断线时排队等重连补发）。

---

## 9. 常见问题速查

| 症状 | 该看哪 |
|---|---|
| 发消息没反应 | 检查 `adminUsers` 里有没有你自己；`dmPolicy` 是不是 `open` |
| 回复 `Cannot read properties of undefined (reading 'current')` | 第 6 章的补丁 |
| 回复 `⚠️ 连接断开，N 秒后重试` | DSH 正在重启，等几秒自动重连，正常现象 |
| 机器人不回群消息 | `groupPolicy: disabled` 下它就是不回群，这是故意的 |
| 重复回复 / 顺序乱 | NapCat 重连重发，靠 `dedupWindowSeconds`（默认 300 秒）兜底 |
| 长回复变成图片且渲染失败 | 见下方"中文长回复" |
| 想让它说话短一点 | 降 `textImageThreshold` 会更容易转图片；想让回复走纯文本就别超阈值 |

**中文长回复渲染成图片失败**：日志里会出现

```
文字图卡片渲染失败，已降级为文本: t2i: no usable CJK font family (set t2i fontFiles/fontFamilies)
```

不影响使用（自动降级成纯文本），想修就在插件配置里给 `t2i` 指一个中文字体文件，比如 `C:\Windows\Fonts\msyh.ttc`。

---

## 10. 文件位置速查

| 用途 | 路径 |
|---|---|
| NapCat 本体 | `<NapCat目录>\` |
| NapCat 启动脚本 | `<NapCat目录>\launcher-user.bat` |
| NapCat 面板配置 | `<NapCat目录>\config\webui.json` |
| NapCat 通道配置 | `<NapCat目录>\config\onebot11_<机器人QQ号>.json` |
| NapCat 运行参数 | `<NapCat目录>\config\napcat.json` |
| NapCat 日志 | `<NapCat目录>\logs\`（需 `fileLog: true`） |
| 登录二维码 | `<NapCat目录>\cache\qrcode.png` |
| DSH 插件配置 | `%USERPROFILE%\.dsh\profiles\<profile>\cordis.patch.yml` |
| DSH 插件本体 | `%USERPROFILE%\.dsh\profiles\<profile>\node_modules\dsh-onebot-qq\` |
| 要打补丁的文件 | 上面目录下 `lib\registry.js`（约第 107 行） |
| 消息链路日志 | `%USERPROFILE%\.dsh\media\onebot\qq-trace.jsonl` |
| 聊天↔会话映射 | `%USERPROFILE%\.dsh\media\onebot\chat-sessions.json` |

### 这个仓库里都有什么

```
setqq/
├─ README.md                 ← 你正在看的这份
├─ docs/
│   └─ troubleshooting.md    ← 深度排障：root cause、asar 解包、13 个坑
├─ patches/
│   └─ dsh-onebot-qq-0.9.8-installModelSelection.patch
└─ examples/
    ├─ cordis.patch.yml               ← DSH 插件配置（脱敏）
    ├─ webui.json                     ← NapCat 面板配置（脱敏）
    └─ onebot11_1234567890.json       ← NapCat 通道配置（脱敏）
```

`examples/` 里全是占位符，复制出去替换后即可用。

---

## 11. 安全与收尾

- 机器人号**用小号**，NapCat 是注入官方客户端，账号有风控风险。
- `accessToken`、`<WebUI密码>` **不要**写进任何要公开的文件。
- NapCat 面板别改成 `0.0.0.0`，除非你明确知道自己在干嘛。
- 只想给自己用：`dmPolicy: open` + `groupPolicy: disabled` + 只把自己和机器人号写进 `adminUsers`，这是最小暴露面。

### 想卸载/回滚

```powershell
# 停 NapCat
Stop-ScheduledTask -TaskName NapCatQQ
Unregister-ScheduledTask -TaskName NapCatQQ -Confirm:$false

# 卸插件
dsh plugin --profile <profile> remove dsh-onebot-qq

# 删掉 cordis.patch.yml 里的 dsh-onebot 段，然后重启 DSH
```

---

*照着做完还卡住的话，把 `qq-trace.jsonl` 最后 5 行和 `Get-ScheduledTaskInfo -TaskName NapCatQQ` 的输出发给我，基本就能定位。*
