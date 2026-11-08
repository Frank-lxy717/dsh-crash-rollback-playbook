# DSH 桌面端 · 崩溃回退体系参考手册

> 版本 1.1 ｜ 整理于 2026-11-08 ｜ 面向"接手排障的 AI 或工程师"

这是一份**可执行**的手册，不是概念介绍。目标读者是另一次对话里的 AI：它没有这段历史，只拿到这份文档和一台起不来的机器，需要能在十分钟内判断"坏在哪一层、用哪把刀、会不会二次伤害"。

---

## 0. 怎么用这份文档

### 0.1 三条前提

1. **DSH 桌面端（DeepSeek Harness）的崩溃，九成不是"程序坏了"，而是"profile 里某一层配置/某一个插件把启动链拖死了"。** 所以救援的主战场是 `%USERPROFILE%\.dsh\profiles\<name>\` 下的几个文本文件，不是重装客户端。
2. **回退工具本身也会坏。** 本机历史上出现过"双击回退什么都没发生"的事故，根因全在脚本自身（行尾、编码、异常处理、选择算法）。因此文档第 8 节专门记录这些坑，读它比读功能列表更省时间。
3. **工具只能证明"配置能起来了"，不能证明"重启后每一轮对话都正常"。** 后者必须真的发一次请求。任何工具报出的"回退成功"都要按这个上限打折理解。

### 0.2 五分钟快速路径

| 你面对的情况 | 先做这个 | 再看 |
|---|---|---|
| 完全不知道哪坏了，但 DSH 还能开 | 双击 `一键体检-DSH异常时双击我.bat`（只读） | §4.1 |
| DSH 起不来 / 每轮对话都报错 | 双击 `一键回退-DSH起不来就双击我.bat` | §5 |
| 回退也救不回来，要"先让它能开机" | 手动收缩 `bundles` 到官方核心两项 | §6.2 |
| 改配置前想留后路 | 双击 `一键打快照-DSH正常时双击我.bat` | §7 |
| 想弄清某个报错什么意思 | 查症状表 | §3 |

### 0.3 路径占位符

文档里统一用占位符，落到你机器上时按实际替换：

| 占位符 | 含义 | 本机实际值（示例） |
|---|---|---|
| `<DSH_HOME>` | DSH 数据根目录 | `%USERPROFILE%\.dsh` |
| `<PROFILE>` | 当前生效的 profile 目录 | `<DSH_HOME>\profiles\desktop` |
| `<TOOLKIT>` | 崩溃回退工具箱目录 | `%USERPROFILE%\Desktop\崩溃回退` |
| `<PLUGIN_SRC>` | 本地插件源码根目录 | `D:\dsh-plugins-local` |
| `<ARCHIVE>` | 备份存档目录 | `<TOOLKIT>\_备份存档` |

> 早期版本 profile 名叫 `web`，桌面端换版后改叫 `desktop`。**"回退执行了但不生效"最常见的单一原因就是瞄准了错的 profile。** 任何动作前先跑 §4.3 的 `status`。

---

## 1. 体系总览

### 1.1 四层防线

```
┌─ 第 0 层：预防 ─────────────────────────────────────────┐
│  改动前备份（§7）· 健康快照 · 插件代码版本快照            │
└────────────────────────────────────────────────────────┘
┌─ 第 1 层：检测 ─────────────────────────────────────────┐
│  一键体检 · check-plugins.mjs（真加载+官方 schema 校验）   │
│  rollback-full status/precheck · diagnose-dsh.bat         │
└────────────────────────────────────────────────────────┘
┌─ 第 2 层：最小破坏修复 ─────────────────────────────────┐
│  只把出问题的插件写进 disabledBundles / 单行 disabled      │
│  （可逆、不碰其它配置、重启即生效）                        │
└────────────────────────────────────────────────────────┘
┌─ 第 3 层：回退 ─────────────────────────────────────────┐
│  智能选快照 → 原子恢复 → 双层验证 → 失败自动还原           │
└────────────────────────────────────────────────────────┘
┌─ 第 4 层：应急 ─────────────────────────────────────────┐
│  安全模式（屏蔽所有插件启动）· 干净 profile · 第二桌面端    │
└────────────────────────────────────────────────────────┘
```

设计原则只有一条：**能小改就不大改，能可逆就不覆盖。** 每往下一层，破坏面更大、代价更高，所以上一层没失败就不许往下走。

### 1.2 组件清单

| 组件 | 类型 | 作用 | 何时用 | 现状 |
|---|---|---|---|---|
| `rollback-oneclick.ps1` | 主力脚本 | 三模式：`diagnose` / `rollback` / `snapshot`；v2 核心 | 一切救援的入口 | 在用（v2） |
| `check-plugins.mjs` | 体检器 | 真加载插件 + `assertObjectJsonSchema()` 校验工具参数 | 定位"配置合法但每轮报错" | 在用 |
| `rollback-full.ps1` | 全量工具 | `status` / `snapshot` / `list` / `restore` / `precheck`；原子恢复 + 安全网 | 需要精确操作或看历史快照 | 在用（v1 全量层） |
| `rollback-dsh.bat` | 单插件刀 | 把一个插件从 `dependencies` + `bundles` 摘掉，塞进 `disabledBundles` | 已锁定单一嫌疑插件 | 在用（旧式） |
| `rollback-dsh-restore.bat` | 恢复刀 | 反向操作，并补 `node_modules` junction | 插件修好后 | 在用（旧式） |
| `diagnose-dsh.bat` | 诊断 | JSON 有效性、源码存在性、`dump-config`、扫 `ctx.effect` 反模式 | 手工自查 | 在用（旧式） |
| `snapshot.mjs` | 源码快照 | 给单个插件目录打版本快照，保留 20 份 | 每次改插件代码后 | 在用 |
| 三个 `.bat` 启动器 | 入口 | `一键回退` / `一键体检` / `一键打快照` | 双击即用 | 在用 |
| `回退DSH到rc.2-一键.bat` | 救援菜单 | 交互式菜单（含旧安装 host 回退） | 需要按序号选操作 | 在用 |
| `_备份存档\` | 仓库 | 所有配置备份 + 全量快照 + 压缩归档 | 恢复现场 | 在用 |

> **v1/v2 并存是有意的**：`rollback-oneclick.ps1`（v2，智能）负责"判断"，`rollback-full.ps1`（v1，全量）负责"精确恢复和看历史"。两者共用同一个 `_备份存档`，快照格式兼容。

### 1.3 一个反直觉的事实

**体检器比回退器更重要。** 本机最惨的一次事故（2026-10-24）里，回退工具全程"正常工作"——它备份了、回滚了、报了成功——但每次重启照崩，因为它回滚到了一份同样坏的快照，而且它的预检根本看不见真正的故障点。真正定位问题的是那个 347 行的 `check-plugins.mjs`。所以 §4.2 值得细读。

---

## 2. 运行时结构：为什么这些工具能生效

### 2.1 profile 是什么

一个 profile 就是一个目录，核心是三个文件：

```
<PROFILE>\
├── package.json          # 依赖清单 + profile manifest（bundles / disabledBundles）
├── cordis.patch.yml      # 用户补丁层：按 id 覆盖配置、禁用 entry、插入新 entry
├── cordis.yml            # 根 entry 列表，默认是空数组 []，正常不要动
├── pnpm-workspace.yaml   # workspace / overrides（本地插件锁定用）
├── pnpm-lock.yaml
└── node_modules\         # 实际安装的插件（pnpm）
```

`package.json` 里最关键的两段：

```jsonc
{
  "dependencies": { "some-plugin": "1.2.3", "local-plugin": "link:D:/path/to/plugin" },
  "dsh": {
    "profile": {
      "bundles": ["@deepseek-ai/dsh-base", "@deepseek-ai/dsh-web-app", "some-plugin"],
      "disabledBundles": []
    }
  }
}
```

- **`bundles` 是"要加载谁"的有序列表。** 不在这个数组里 = 不加载，哪怕 `dependencies` 里装着、`node_modules` 里有文件。
- **`disabledBundles` 是"装了但不加载"的名单。** 保留依赖、保留文件、只是不挂载——这正是"最小破坏修复"的落点。

### 2.2 分层组合顺序

DSH 的配置树不是"读一个文件"，而是从空根开始**逐层叠加**：

```
空根
 ├─ 每个 bundle 自带的 patch 层            ← dsh.profile.bundles 顺序
 ├─ profile 自己的 cordis.patch.yml        ← 用户覆盖层（最常改的就是这个）
 ├─ <DSH_HOME>\cordis.patch.yml            ← home 级覆盖
 └─ --patch <file> 指定的 overlay           ← 一次性覆盖（不写盘）
```

后层覆盖前层。这带来两个非常有用的性质：

1. **可以不改插件、不改 bundles，只在 `cordis.patch.yml` 里加一行就把某个 entry 关掉**（§6.4）。
2. **可以用 `--patch` 做"一次性安全启动"，完全不碰磁盘上的配置**（§6.5）。

### 2.3 三种"关掉插件"的粒度

| 粒度 | 写法 | 影响面 | 可逆性 | 典型用途 |
|---|---|---|---|---|
| **bundle 级** | `package.json` → `dsh.profile.disabledBundles` 追加包名 | 整个插件包不加载 | 删掉数组项即恢复 | 确认是某插件整体的问题 |
| **entry 级** | `cordis.patch.yml` → `- id: <rowId>` + `disabled: true` | 只关掉那一行 entry，同包其它 entry 照常 | 删掉那两行即恢复 | 一个聚合包（如全家桶）里只有某功能坏 |
| **层收缩** | 把 `bundles` 数组改成只剩官方核心 | 所有第三方插件不加载 | 把数组改回来 | 应急：先让它能开机（§6.2） |

entry 级禁用是**最被低估的工具**。它不需要卸载、不需要 `pnpm install`、不影响同包其它功能，而且和插件自愈写入的是同一行合并机制（见 §2.5）。

### 2.4 插件的四种来源

崩溃排查经常卡在"这个包到底从哪装的"，因为不同来源的修复方式完全不同：

| 来源写法 | 含义 | 特征 | 修复注意 |
|---|---|---|---|
| `"1.2.3"` / `"^1.2.3"` | npm registry | 最常见 | 可 `dsh plugin update` |
| `"file:<绝对路径>"` | 本地目录 | 改源码即刻生效（重装后） | **不要删源码**，回退只摘引用 |
| `"link:<绝对路径>"` | pnpm link | 与 `file:` 类似但走软链 | 链接缺失会导致加载失败 |
| `"file:...\plugin-center\operations\<uuid>\artifact.tgz"` | 插件中心打包安装 | 路径里有 `operations/<uuid>/artifact.tgz` | **不能手工改版本号**；回退时整条依赖要一起处理 |

最后一种最容易出事：`rollback-dsh.bat` 只改 `package.json`，而插件中心的写入机制会同时动 `pnpm-lock.yaml` / `node_modules`，两者打架就会产生"半覆盖"——`package.json` 说没有了，`node_modules` 还留着，或者反过来。**这就是后来必须做 `rollback-full.ps1`（整份配置 + 依赖清单一起快照/恢复）的直接原因。**

### 2.5 启动失败时 DSH 自己的行为

需要知道宿主在失败时的默认姿态，否则会误判：

- **`failOnStartupError: false`** 是常见配置。意味着某个插件启动失败不一定拖死整个进程——它会降级为"这个插件没起来"，界面还是能开。
- 所以**"DSH 能打开但某个功能没了"和"DSH 打不开"是两类完全不同的问题**：前者通常只是该插件 `fiberPhase: failed`，后者才是启动链被拖死。
- 判断运行态比判断配置更准：插件管理能力给出的 `enabled` / `installed` / `fiberPhase`（`active` = 已加载，`failed` = 加载失败）比 `dump-config` 更贴近真相。

---

## 3. 崩溃分类与判据

同一句"崩了"背后至少有七种机制，处理方式完全不同。先分类再动手。

| 现象 | 最可能的原因 | 首查 | 处置 |
|---|---|---|---|
| 双击没反应，进程起来几秒就消失 | 某插件 `apply()` 抛错（如 **Invalid effect**） | `check-plugins.mjs --limit 5` | 禁用该插件（§5） |
| 界面被半透明遮罩、点不动 | 旧版 client 插件的兼容层（boot shield）在新 UI 上不消失 | 逐个禁用 client 插件 | 禁用该插件，等作者适配 |
| 界面能开，但**每一轮对话都报错** | 插件注册了**非法工具 schema** | `check-plugins.mjs`（唯一有效工具） | 禁用该插件 |
| 页面空白 + 控制台 JS 错误 | client bundle 契约变化（`__ModuleLoader__` 格式） | 看插件 `exports["./client"]` | 升级或禁用 |
| 提示某包找不到 | 依赖没装进 profile 的 `node_modules` | `status` / `precheck` | `dsh plugin --profile <n> install` |
| 插件"已启用"但没界面 | 跨版本 peer 不兼容（`incompatible-version`，rows 为空） | 插件管理列表 | 升级到匹配版本，或禁用 |
| 随机崩溃 / 内存报错 / `0xC0000374` | 某插件在轮询里反复抛错，最终堆损坏 | 看崩溃日志字节数是否重复 | 定位到具体插件后禁用 |
| 配置合法的前提下**回退没效果** | 回退瞄错了 profile，或回退了坏快照 | `rollback-full.ps1 status` | §5.1 |

> 关于最后一行：`0xC0000374`（堆损坏）本机真实发生过——某任务看板插件在 `session/list` 失败时反复抛 `TypeError`，两次崩溃的日志字节数完全一致（66204），堆栈都落在同一个 `pollSessions`。这种"稳定复现的随机崩溃"要按日志指纹找，不要按时间猜。

---

## 4. 检测层

### 4.1 一键体检（只读）

```powershell
# 双击也行：一键体检-DSH异常时双击我.bat
powershell -NoProfile -ExecutionPolicy Bypass -File "<TOOLKIT>\rollback-oneclick.ps1" -Mode diagnose
```

它做四件事，**全部只读**：

1. 定位生效 profile（`desktop` 优先，其次 `web`），打印 `bundles` / `disabledBundles` 计数
2. 配置层检查（§4.3 的判据）
3. 插件层检查（最近改动的 6 个，§4.2）
4. 列出快照，并标注每份快照"比当前少几个插件"

输出末尾会直接给建议：插件有问题→"双击一键回退"；配置有问题→"回退到可用快照"；都没问题→提示可能不是插件导致的。

### 4.2 `check-plugins.mjs`：唯一能看见"隐形崩溃"的工具

```powershell
# 整个 profile，只看最近 5 个
node "<TOOLKIT>\check-plugins.mjs" --profile "<PROFILE>" --limit 5

# 全量
node "<TOOLKIT>\check-plugins.mjs" --profile "<PROFILE>" --all

# 单插件排查
node "<TOOLKIT>\check-plugins.mjs" --dir "<PLUGIN_SRC>\some-plugin"
```

**它和 `dump-config` 的根本区别：它真的把插件加载起来。** 用 mock ctx 跑一遍每个插件的 `apply()`，把 `ctx.tools.register()` 收到的定义交给官方的 `assertObjectJsonSchema()` 校验。

输出是 JSON：`{ ok, checked, notSelected, skipped, problems, validator }`。

实现上有四个必须知道的细节（都是踩出来的）：

| 细节 | 为什么 |
|---|---|
| `registerHooks` 把 `@deepseek-ai/*` 的解析基准改到 profile 目录 | 本地插件是 `file:` 挂载的，realpath 在工作区外，那里没有 `node_modules`，直接 import 会 `ERR_MODULE_NOT_FOUND`。不改基准就会把"解析不到"误判成"插件坏了" |
| 临时把 `console.*` 和 `process.stdout.write` 改道到 stderr | 不少插件在 `apply()` 里直接 `console.log`。这些字节混进 stdout 会让调用方的 `ConvertFrom-Json` 解析失败——**体检器恰好在最需要它的时候坏掉** |
| 读 `package.json` 先剥 BOM | 有些插件包是带 BOM 存下来的，`JSON.parse` 会抛 |
| 导出里没有 `apply()` 不算坏 | 合法形态：bundle 只通过 `cordis.patch.yml` 挂配置/技能（如纯配置型聚合包）。只有"既没 `apply` 又没 patch"才算真坏 |

`validator` 字段会告诉你用的是 `official`（拿到官方校验器）还是 `fallback`（自写的等价检查，只查最致命的形状：根节点必须 `type: 'object'`、属性上不能残留布尔 `required`）。

### 4.3 配置层预检与"软硬之分"

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File "<TOOLKIT>\rollback-full.ps1" status     # 先看地址
powershell -NoProfile -ExecutionPolicy Bypass -File "<TOOLKIT>\rollback-full.ps1" precheck   # 内建校验
```

`precheck` 的内建校验（不依赖 CLI）：

- `package.json` 能否解析
- 同一插件是否同时出现在 `bundles` 和 `disabledBundles`（**组合冲突，直接判失败**）
- 非 `@deepseek-ai/*` 的 bundle 在 `node_modules` 里是否存在（缺失只警告，不阻断）

然后才是可选的 `dsh --dump-config`。**这里必须区分两类"错误"，否则会自己制造事故：**

| 类别 | 例子 | 判据 | 处理 |
|---|---|---|---|
| **A. 真损坏** | JSON 语法坏、依赖装不上、结构错误 | 除了软问题之外还有 `error:` | 阻断，必须修 |
| **B. 环境固有差异** | `cannot resolve profile bundle "@deepseek-ai/dsh-experimental-plugin-market-bundle"` | 桌面端自带、CLI 没有 | **降级为提示** |
| **B. 配置不干净** | `patch: entry "xxx" not found` | override 指向已移除的 entry | **降级为提示** |

> 本机实测（2026-11-08）：`dsh --profile desktop --dump-config` 退出码 **0**，但 stderr 会同时吐出上面两类 B 级提示。如果脚本不区分，就会把"配置明明是好的"判成失败，然后自动还原再报"回退未成功"——这是 v1 那个"点了没反应"的直接原因之一。

还有一条匹配细节：判错时用 `\berror:` 而不是 `error:`。因为 dump-config 的输出里本来就有 `failOnStartupError: false` 这种**正常配置项**，宽松匹配会把配置内容当成错误。

### 4.4 `diagnose-dsh.bat`（旧式自查）

五步：JSON 有效性 → 每个 bundle 的源码是否存在 → `dump-config` → 扫 `ctx.effect(() => xxx.register(...))` 这类反模式 → 给结论。

它有用，但三个局限要知道：**写死了 profile 路径**、`dump-config` 用的 CLI 可能与桌面端版本不匹配、**看不见 schema 问题**。日常优先用 §4.1。

---

## 5. 救援层：智能回退 v2

### 5.1 为什么 v1 会"点了没反应"

2026-10-24 那次事故，回退工具失效不是单一 bug，是**四个独立缺陷叠加**，任何一个都足以让它归零：

| # | 缺陷 | 后果 |
|---|---|---|
| 1 | 所有 `.bat` 是 Unix 行尾（LF） | cmd 只认 CRLF，双击后整个文件被切碎，`echo` 变成 `cho`，中文提示被当命令执行。**一条命令都没跑成** |
| 2 | `$ErrorActionPreference='Stop'` + `& dsh ... 2>&1` | PowerShell 5.1 会把原生命令的 stderr 当终止错误。`dsh` 只要写一行警告，预检就跳异常分支 → 判定失败 → 自动还原 → 报"回退未成功" |
| 3 | 永远选"最近的快照" | 崩溃时最近那份往往就是出事时/出事后打的。滚过去 = 滚回同一个坏状态 |
| 4 | 预检只跑 `dsh --dump-config` | 它只验证配置能被解析。而那次坏的是**插件注册了非法 tool schema**——配置完全合法，预检一路绿灯，脚本报"回退完成"，重启照崩 |

第 4 条最恶劣：**它不是失败，是虚假的成功。**

### 5.2 v2 的三阶段流程

```
阶段 0  前置
        ├─ 定位生效 profile
        ├─ 检测 DSH 是否在运行（提示先退出）
        └─ 备份当前配置 → _备份存档\一键回退前置备份-<stamp>\

阶段 1  体检 + 最小破坏修复
        ├─ check-plugins.mjs（最近 6 个）
        ├─ 有问题 → 只把嫌疑插件加进 disabledBundles，别的什么都不动
        └─ 验证：配置层 + 插件层都过 → 完成，结束

阶段 2  智能选快照（仅在阶段 1 没解决时）
        ├─ 给每份快照打分：比当前"少几个插件"
        ├─ 只在"确实比当前少"的候选里，挑缺得最少的那个
        └─ 恢复 → 双层验证 → 不过就还原现场、试更早的（最多试 4 份）

结果
        ├─ 成功 → 同步依赖（dsh plugin install）→ 清理旧前置备份（保留 5 份）
        └─ 失败 → 还原到动手前的状态 → 明确说"没成功"+ 给下一步选项
```

### 5.3 选快照的算法（v2 最核心的一处改动）

不要按时间选，按**"它是不是发生在新插件装进来之前"**选：

```text
对每份快照 S：
    added = 当前 bundles 中、S 的 bundles 里没有的包   // S 相对当前"少了哪些"
选择：
    候选 = added 数量 > 0 的快照
    最优 = 候选中 added 数量最小的（稳定排序 → 同分取时间更近的）
    若候选为空（所有快照都含当前全部插件）：
        警告"没有更早的干净快照"，只能取最近一份
```

直觉：`added = 0` 的快照**和当前一模一样**，它凭什么是"好"的？很可能就是出事时打的那份。而 `added > 0` 说明它早于这批新插件，那才是"装之前"的状态。选 `added` 最小的，等于"只回退掉最近装的那几个"。

阶段 2 恢复前会明确提示：`它会移除这些插件的配置项：xxx（插件文件不会被删，只是不再加载）`。

### 5.4 双层验证门控

`Test-Health` = 配置层 **且** 插件层，两者都过才算通过：

1. `Test-Config`：内建校验 + `dump-config`（软硬之分见 §4.3）
2. `Invoke-PluginCheck`：`check-plugins.mjs` 必须 `problems.Count == 0`

任何一层不过，就不算修好。这条规则挡住了 v1 那种"配置合法所以报告成功"的虚假通关。

### 5.5 失败回路（关键：绝不留半成品）

- 阶段 1 修了但验证不过 → 保留已做的禁用（不撤销，因为可逆），进入阶段 2
- 阶段 2 恢复后验证不过 → **立即把现场还原成回退前的样子**，再试下一份
- 试满 4 份仍不过 → 还原到动手前状态，报"回退未成功"，给三条下一步建议
- 任何情况下退出前，配置要么是"验证通过的"，要么是"动手前的"，不存在中间态

### 5.6 演练模式（改逻辑后必跑）

```powershell
powershell -NoProfile -ExecutionPolicy Bypass -File "<TOOLKIT>\rollback-oneclick.ps1" -Mode rollback -DryRun
```

`-DryRun` 走完全部诊断与决策逻辑，**但不写任何文件**（备份也不写，只打印"[演练]"）。本机规则：每次改回退逻辑后都拿它复演一遍再上真机。

---

## 6. 应急层：屏蔽所有插件启动

### 6.1 先说清楚一件事

**DSH 本体并没有一个叫"安全模式"的开关。** 本机核查（2026-11-08 扫描桌面端 `app.asar`）确认：没有 `safeMode` / `--safe` / "安全模式"之类的内置入口。

所以"紧急情况下屏蔽所有插件启动"是**用平台已有的 profile 分层机制组合出来的一套操作规程**，不是点一个按钮。它有五种实现，破坏面从小到大：

| 方案 | 破坏面 | 保留什么 | 适合 |
|---|---|---|---|
| A. `bundles` 收缩 | 中（改 package.json 一处数组） | 全部依赖、全部文件、全部配置 | **首选** |
| B. 全量 `disabledBundles` | 中 | 同上，但改动更长 | 想逐条看清禁了谁 |
| C. entry 级 disabled | 小（只加两行 YAML） | 同包其它功能 | 只想关掉某几个功能 |
| D. `--patch` overlay | **零**（不写盘） | 磁盘配置完全不动 | 只想试一次能不能起来 |
| E. 干净 profile | 大（新建目录） | 原 profile 完全不动 | 怀疑 profile 本身已不可信 |

### 6.2 方案 A：`bundles` 收缩到官方核心（首选）

**这是本机实际用过的应急手段**，在 DSH 本体升级出问题时救过场。

原理：`bundles` 只留平台必须的两个，其余第三方全部不加载；`dependencies` 一条都不删。等于"所有插件暂时不启动，但配置和依赖原封不动"。

```jsonc
{
  "dependencies": {
    // ... 全部保留，一条都不动
  },
  "dsh": {
    "profile": {
      "bundles": [
        "@deepseek-ai/dsh-base",
        "@deepseek-ai/dsh-web-app"
      ],
      "disabledBundles": []
    }
  }
}
```

操作步骤：

1. **先备份**（§7.2 的硬规则），复制 `<PROFILE>\package.json` 到 `<ARCHIVE>\safemode-before-<原因>-<stamp>\`
2. 编辑 `bundles`，只留上面两项
3. **保存为 UTF-8 无 BOM**
4. 完全退出 DSH（含托盘图标），重启
5. 确认能起来后，按 §6.8 逐批放回

> 本机留有一份实证：`_备份存档\safemode-before-restore-20261106-133120\package.json`，其中 `bundles` 恰好只剩这两项，而 `dependencies` 保留了全部 30 个包。同目录另有 `pre-upgrade-rc5-20261106-131400`（升级前，18 个 bundle 完整）——两者的差异就是"安全模式"的精确形态。

**注意**：只改 `bundles` 意味着 `node_modules` 里的东西都还在，恢复时不需要重新 `pnpm install`，改回数组重启即可。这是它比"卸载插件"安全得多的原因。

### 6.3 方案 B：全量写进 `disabledBundles`

想"看得见禁了谁"就用这个。把现在的 `bundles` 原样复制进 `disabledBundles`，然后把 `bundles` 收缩到核心两项。

效果与方案 A 相同，但配置文件里留下了完整名单，恢复时照单反向操作即可，不依赖记忆。缺点是改得多、容易漏。

### 6.4 方案 C：entry 级禁用（只关功能，不关插件）

不改 `package.json`，只在 `<PROFILE>\cordis.patch.yml` 里加两行：

```yaml
- id: web-ui-task-board
  disabled: true
```

- `id` 是 entry 的 rowId，不是包名
- 加在文件**末尾**（后层覆盖前层，末尾最优先）
- 删掉这两行即恢复
- 同包的其它 entry 不受影响

**这是本机最常用的一把刀。** 某聚合包（全家桶）里只有"任务看板"会崩时，不需要停用整个包——只关那一行，皮肤中心、桌宠、用量、插件市场、会话归档全都还在。文件里保留注释说明"为什么关、怎么恢复"，比事后猜强。

### 6.5 方案 D：`--patch` overlay（零写入，最适合"试一次"）

CLI 支持在 profile 层之后追加一次性覆盖层：

```powershell
dsh --profile <name> --patch <your-overlay.yml>
```

`--patch` 指定的文件**不落进 profile 配置**，只影响这一次启动。可以写一份"把可疑 entry 全部 disabled"的 overlay，启动看看能不能活：

- 活了 → 说明问题就在被覆盖的东西里，再把结论固化进配置（方案 C）
- 没活 → 说明问题不在配置层，别浪费时间改配置文件

局限：桌面端（Electron）启动是否透传 `--patch`，取决于启动方式；这条路更适合 CLI / 命令行排查。**用之前先确认你的启动路径真的会带上这个参数。**

### 6.6 方案 E：干净 profile（怀疑 profile 本身不可信时）

```powershell
dsh <name> --from-default-profile web
```

从随附模板初始化一个全新 profile 并启动。它不共享原 profile 的依赖和补丁层，所以**几乎一定起得来**——这是"判断到底是 DSH 坏了还是我的 profile 坏了"的判据：

- 干净 profile 能起 → DSH 本体没事，问题在原 profile 的配置/插件
- 干净 profile 也起不来 → 问题在 DSH 本体或机器环境（运行时、权限、磁盘），换工具也没用

注意：`desktop` 这个名字保留给 Electron 持有，CLI 会拒绝针对它的某些请求。要新建就用其它名字。

### 6.7 选哪个：决策树

```
DSH 还能开吗？
├─ 能 → 先体检（§4.1）。
│        ├─ 定位到具体插件 → 方案 C（能只关一行就只关一行）
│        │                      否则方案 B
│        └─ 定位不到 → 一键回退 v2（§5）
└─ 不能开
   ├─ 想先确认"是不是我配置的问题" → 方案 D 或 E
   ├─ 只想让它起来，别的以后再说 → 方案 A（首选）
   └─ 以上都不行 → DSH 本体/环境问题，别在 profile 里继续折腾
```

### 6.8 恢复：逐批放回，不要一次全放

**屏蔽是一瞬间的事，恢复必须慢。** 一次全放回，等于把当初的崩溃现场原样复现，而且失去了"是哪一批坏的"这个信息。

推荐节奏：

1. **第一批：平台核心**（`dsh-base`、`dsh-web-app`）——确认 UI 正常
2. **第二批：官方聚合包**（如 `@linxin666/dsh-web-all`）——这是最容易出兼容问题的，单独一批
3. **第三批起：每次 2~3 个第三方插件**，每批之后**真的发一轮对话**，确认正常再打快照
4. 有插件报 `incompatible-version` 或界面异常 → 单独回退它，其余继续

每批之间打一次快照（§7.2），出问题就回到上一批，而不是回到安全模式。

---

## 7. 备份机制：每次改动都留后路

### 7.1 三层备份，各管一段

| 层 | 备份对象 | 工具 | 位置 |
|---|---|---|---|
| **配置层** | profile 的配置文件 | 手工 / 脚本复制 | `<ARCHIVE>\<标签>-<时间戳>\` |
| **全量层** | 配置 + 依赖来源清单 + artifact 映射 | `rollback-full.ps1 snapshot` | `<ARCHIVE>\snapshots\<name>\` |
| **源码层** | 单个插件的核心文件 | `snapshot.mjs` | `<PLUGIN_SRC>\.snapshots\<plugin>\` |

三层缺一不可：配置层救"改坏了配置"，全量层救"配置和依赖一起改坏了"，源码层救"改坏了插件代码本身"。

### 7.2 硬规则：工作区外的一切写操作，先备份

这是本机长期执行的门禁，不是建议：

1. **凡改动发生在工作区之外**（如 `<DSH_HOME>` 下的 profile、`<PLUGIN_SRC>`、桌面工具箱），**执行前必须先备份**
2. **"新增"同样算改动**——新建插件目录、加依赖、改配置都改变系统状态，回退时同样需要改动前的对照快照。**不许以"纯新增、没碰既有文件"为由跳过备份**
3. **备份是操作清单的第一项**，不是收尾步骤。上下文紧张、会话过长时此项优先级最高
4. 脚本自身也要遵守：所有会写盘的脚本，第一步都是备份

备份命名规范：

```
<ARCHIVE>\<用途标签>-<YYYYMMDD-HHMMSS>\
    例：pre-upgrade-rc5-20261106-131400
        safemode-before-restore-20261106-133120
        profile-before-daily-root-20261107-182654
```

单文件形式：`<原文件名>.bak-<原因>-<时间戳>`
例：`package.json.bak-rollback-20261024-021626`

标签要写清"为什么备份"（`pre-upgrade` / `before-restore` / `before-install`），三个月后只有标签能告诉你当时在干什么。

### 7.3 各脚本自己的备份行为

| 脚本 | 备份什么 | 命名 | 保留 |
|---|---|---|---|
| `rollback-oneclick.ps1 -Mode rollback` | 动手前的完整配置集 | `一键回退前置备份-<stamp>` | 最近 **5** 份，自动清理更旧 |
| `rollback-oneclick.ps1 -Mode snapshot` | 当前配置 + `_health.json` | `一键快照-<stamp>` | 不自动清理 |
| `rollback-full.ps1 restore` | ①安全网快照 ②每个文件 `.bak-restore-<stamp>` | `auto-guard-<stamp>` | 不自动清理 |
| `rollback-full.ps1 snapshot` | 配置 + `deps-manifest.json` + `artifact-map.json` | `snap-<stamp>` | 不自动清理 |
| `rollback-dsh.bat` | `package.json` + `cordis.patch.yml` | `.bak-rollback-<stamp>` | 同时同步一份到 `<ARCHIVE>` |
| `rollback-dsh-restore.bat` | `package.json` | `.bak-restore-<stamp>` | 同上 |
| `snapshot.mjs` | 插件核心文件 + `manifest.jsonl` | `<version>-<stamp>-<label>` | 每个插件最近 **20** 份 |

几点值得注意：

- **打快照前会先体检**（`-Mode snapshot`）。不健康的配置也会存，但会被标成"不健康"——那两份坏快照就是这么产生的，加体检是为了让下一次不打同样的坏快照
- **`_health.json`** 记录 `createdAt` / `profile` / `bundles` / `disabled` / `configOk` / `pluginsOk` / `problems`，回退时可以据此判断哪份快照可信
- **`deps-manifest.json`** 是 `rollback-full` 的价值所在：它记录每个依赖的 `spec`（原始写法）、`kind`（registry / local-file / git / plugin-center）、`installed`（`node_modules` 里的实际版本）。恢复时不靠猜"这个包原来从哪装的"
- **`artifact-map.json`** 记录 plugin-center 的 `operationId ↔ 包名` 映射，专治 §2.4 第四类来源

### 7.4 原子写：别制造半文件

**配置写到一半断电/崩溃，会得到一个"JSON 语法错误"的 profile，比原来的问题更难救。**

`rollback-full.ps1` 的做法：

1. 把快照文件先复制到 `%TEMP%\dsh-restore-stage-<guid>\`
2. 在临时目录里 `ConvertFrom-Json` 验证一遍
3. 全部通过后，才逐个 `Move-Item` 到 profile（`Move` 在同一卷上是原子的）
4. 无论成败，`finally` 里清掉临时目录

手工改配置时同理：**先写临时文件、验证、再替换**。宁可多两步，不要留半文件。

### 7.5 编码规则（这几条能省掉一半的"玄学问题"）

| 文件类型 | 编码要求 | 不遵守的后果 |
|---|---|---|
| `.bat` | **CRLF 行尾** | cmd 把文件切碎，中文提示被当命令执行（§5.1 第 1 条） |
| `.ps1` | **UTF-8 with BOM** | PowerShell 5.1 没 BOM 会把中文读成乱码 |
| `package.json` | **UTF-8 无 BOM** | 带 BOM 时 `ConvertFrom-Json` / node 解析可能失败 |
| `.md` / `.yml` | UTF-8 | — |

写 JSON 时优先用：

```powershell
[System.IO.File]::WriteAllText($path, $json, (New-Object System.Text.UTF8Encoding($false)))
```

`Set-Content -Encoding UTF8` 在 PowerShell 5.1 下会**带 BOM**，这是老坑。

> 另有相关坑：文件名乱码。当生成文件名时用 GBK 去读 UTF-8 内容，会产出类似"鎭㈠鎸囧畾鎻掍欢"的乱码名字（本意是"恢复指定插件"）。看到这种名字，说明上游有一个编码环节错了，不是磁盘坏了。

### 7.6 从备份恢复

按"最大确定性"排序：

1. **点快照恢复**（最稳）：`rollback-full.ps1 restore -Snapshot <name>`。它会先打安全网、再恢复、恢复后预检，失败自动回退到安全网
2. **复制回配置文件**：从 `<ARCHIVE>\<标签>-<stamp>\` 把 `package.json` / `cordis.patch.yml` 复制回 `<PROFILE>`，注意 UTF-8 无 BOM
3. **回滚插件源码**：从 `<PLUGIN_SRC>\.snapshots\<plugin>\<version>-<stamp>-<label>\` 复制回插件目录
4. **兜底**：`<ARCHIVE>\snapshots\<auto-guard-*>` 是每次 restore 前自动留的现场，永远可以退回到"上一次恢复之前"

---

## 8. 踩坑实录

按"踩到的顺序"而非严重程度排列。每条都是真实发生过的，不是理论风险。

### 8.1 脚本层

**① `.bat` 必须是 CRLF**
症状：双击后满屏"不是内部或外部命令"，`echo` 变 `cho`。
根因：文件是 LF 行尾，cmd 只认 CRLF，整个文件被当一行切碎。
规则：**每次改 `.bat` 后校验行尾。**

**② PowerShell 5.1 会把原生命令的 stderr 当终止错误**
症状：配置明明是好的，脚本报"预检异常"。
根因：`$ErrorActionPreference='Stop'` 下 `& dsh ... 2>&1`，`dsh` 写一行 stderr 就抛进 catch。
规则：**调原生 CLI 一律走 `cmd /c "... 2>&1"` 承接。**

**③ `\berror:` 不能写成 `error:`**
症状：`-DryRun` 演练时把正常配置项判成错误。
根因：dump-config 输出里有 `failOnStartupError: false`，宽松匹配命中。
规则：**匹配错误用词边界。**

**④ 预检太浅 = 虚假成功**
症状：脚本报"回退完成"，重启照崩。
根因：只验证"配置能被解析"，没验证"插件注册的工具 schema 合法"。
规则：**验证必须覆盖到真实故障点，否则"通过"没有意义。**

**⑤ 永远选"最近的快照"**
症状：回退执行了，重启后还是同一个崩法。
根因：崩溃时最近那份快照往往就是出事时打的。
规则：**按"是不是早于这批新插件"选，不按时间选。**

**⑥ 校验脚本产物时被插件自己的 `console.log` 污染**
症状：体检器 JSON 解析失败，报"体检异常"。
根因：插件在 `apply()` 里直接打印，字节混进 stdout。
规则：**自己会输出结构化数据的工具，必须把第三方输出改道到 stderr。**

**⑦ UTF-8 BOM**
症状：`ConvertFrom-Json` 报"意外的字符"。
根因：`Set-Content -Encoding UTF8`（PS 5.1）默认写 BOM。
规则：**写 JSON 用 `UTF8Encoding($false)`。**

### 8.2 结构层

**⑧ 瞄错 profile**
症状："回退执行了但不生效"。
根因：桌面端升级后 profile 从 `web` 换成 `desktop`，工具还写着 `web`。
规则：**任何动作前先 `status` 看生效地址。** 现在已改成自动探测（优先读运行中桌面端命令行里的 profile 路径，探测不到按 `desktop > web` 回退）。

**⑨ profile 迁移本身会留下一地配置**
症状：某些设置"突然回到默认"、飞书通道不工作。
根因：新版把 `settings.yaml` 归档了，其中某些 section 没落到任何 entry 里，通道就以"未配置"状态起来。
规则：**迁移后逐项检查依赖旧 `settings.yaml` 的插件，用 `cordis.patch.yml` 的 id 覆盖把配置补回去**（补在配置文件里，不是补在代码里）。

**⑩ 手工改 `package.json` 与插件中心打架**
症状：回退后依赖被清空，或 `package.json` 说没有但 `node_modules` 还留着。
根因：插件中心（`file:...artifact.tgz`）有自己的写入机制，与脚本的单点修改冲突。
规则：**这类 profile 的回退必须整份配置 + 依赖清单一起快照/恢复**（`rollback-full.ps1`），不要只用改单个字段的脚本。

**⑪ 孤儿 override**
症状：每次 `dump-config` 都报 `patch: entry "xxx" not found`。
根因：插件停用后，`cordis.patch.yml` 里对应的 override 没删，指向一个已不存在的 entry。
性质：**只是那几行静默失效，不会崩**。降级为提示即可，但建议清理时保留原值和恢复说明（注释掉而不是删除）。

**⑫ CLI 与桌面端的 bundle 集合不同**
症状：`cannot resolve profile bundle "@deepseek-ai/dsh-xxx"`。
根因：`dsh` CLI 是 npm 全局安装，桌面端是打包版，两者内置的官方 bundle 不一样。
规则：**这类差异是 B 类软问题，不能当作"配置损坏"阻断流程。**

**⑬ `ctx.effect(() => xxx.register(...))` 是启动杀手**
症状：启动即退出，日志里 **Invalid effect**。
根因：`ctx.effect` 要求回调返回 disposer，而 `register()` 返回的是 scope，Cordis 判定非法，拖垮整个启动链。
规则：**注册类调用直接调用，不要包在 `effect` 里。**

**⑭ 聚合包的兼容层会遮罩整个界面**
症状：界面被半透明层盖住，点不动。
根因：旧版 client 插件的 boot shield 在新版 UI 上无法自行消失。
规则：**这类问题主端完全无法自救**，只能从外部改配置，或直接进安全模式把它排除掉。

**⑮ 非法工具 schema：最阴的一种崩**
症状：**每一轮对话都报错**，看起来像账号/网络/宿主问题。
根因：插件用 `ctx.tools.register()` 裸注册，`parameters` 里塞的是"参数方言"（`{ name: { type, required } }`）而不是 JSON Schema。宿主编译这层是 `defineTool()` 负责的，绕过它就没人编译 → 根节点没有 `type: 'object'` → 接口拒绝**整个请求**。
为什么阴：
- `dump-config` 完全看不见（配置是对的）
- 启动不报错，插件状态甚至是 `active`
- 症状在"每轮对话"，让人往网络/账号方向查
规则：**只要"能开机但每轮报错"，先跑 `check-plugins.mjs`。**

**⑯ 稳定复现的"随机崩溃"要按日志指纹找**
症状：偶发崩溃 / 内存报错 / `0xC0000374`。
根因：某插件在轮询失败时反复抛错，最终把宿主进程打到堆损坏退出。
关键线索：**两次崩溃的日志字节数完全一致**，堆栈落在同一个函数。
规则：不要按时间猜，比日志长度、比堆栈。

**⑰ 正常配置项被当成错误**
见 8.1③。补充：`failOnStartupError: false` 是**好事**，说明插件失败不会拖死启动。

### 8.3 流程层

**⑱ 拿坏状态打快照**
症状：快照越打越多，但没一份能用。
根因：装完插件立刻打快照，没验证过它能正常跑。
规则：**装完新插件 → 确认能正常回一轮 → 再打快照。** v2 已强制在打快照前体检并标记健康度。

**⑲ 依赖同步被忘掉**
症状：配置改好了，但插件报找不到包。
根因：`package.json` 变了，`node_modules` 没跟上。
规则：**改完配置后跑 `dsh plugin --profile <n> install`**（v2 成功后会做，失败只警告不阻断，因为配置修复本身已生效）。

**⑳ 回退工具会与"正在运行的 DSH"抢文件**
症状：改完配置但行为没变，或文件被还原。
根因：宿主持有文件/单实例锁。
规则：**先完全退出 DSH**（含托盘），再回退。

**㉑ 备份文件名用了错误的编码**
症状：桌面上出现"鎭㈠鎸囧畾鎻掍欢"这类乱码目录。
根因：GBK 去解 UTF-8。
规则：**文件名一律 UTF-8 生成，出现乱码就说明上游有编码错误，停下来查。**

**㉒ 把"重启后能开"当成"修好了"**
这是一个认知坑，不是技术坑。重启能开只说明**启动链通了**，不说明每一轮请求的 schema、每个插件的运行时行为都正常。本机历史上多次出现"说修好了，其实只是这一次操作成功了"。
规则：**报"修好了"之前，必须有一轮真实的、完整的对话成功。**

---

## 9. 边界：这些工具做不到什么

明确写出来，避免下一个人误信：

| 做不到 | 原因 | 替代 |
|---|---|---|
| 验证"重启后每轮对话正常" | 需要真的发一次模型请求 | 手工发一轮，或看 `fiberPhase` |
| 修复 DSH 本体损坏 | 工具只动 profile | 重装 / 回退 host 版本 |
| 修复机器级问题 | Node 运行时加载不了、磁盘满、权限错 | 系统层排查 |
| 在 GUI 完全不可用时自救 | 没有可操作区域 | 直接改配置 / 安全模式 / 干净 profile |
| 判断插件的**业务**逻辑对不对 | 只验 schema 和加载 | 看日志、看功能 |
| 无副作用地"试一下" | 除 `-DryRun` 外都会写盘 | 先用 `-Mode diagnose`（只读） |
| 覆盖所有加载路径 | 用绝对路径直接调真实 `dsh` 会绕过 launcher | 走标准启动入口 |

需要人工介入的信号：安全网快照也验证不过、配置 JSON 无法解析、`node_modules` 结构性损坏、磁盘/权限错误。

---

## 10. 速查

```powershell
# ── 只读体检（先跑这个）───────────────────────────────
powershell -NoProfile -ExecutionPolicy Bypass -File "<TOOLKIT>\rollback-oneclick.ps1" -Mode diagnose

# ── 插件体检（定位 schema 问题）───────────────────────
node "<TOOLKIT>\check-plugins.mjs" --profile "<PROFILE>" --limit 5
node "<TOOLKIT>\check-plugins.mjs" --dir "<PLUGIN_SRC>\some-plugin"

# ── 看生效地址 ───────────────────────────────────────
powershell -NoProfile -ExecutionPolicy Bypass -File "<TOOLKIT>\rollback-full.ps1" status

# ── 打快照（体检 + 记录健康指纹）─────────────────────
powershell -NoProfile -ExecutionPolicy Bypass -File "<TOOLKIT>\rollback-oneclick.ps1" -Mode snapshot
powershell -NoProfile -ExecutionPolicy Bypass -File "<TOOLKIT>\rollback-full.ps1" snapshot

# ── 演练（不改任何文件）──────────────────────────────
powershell -NoProfile -ExecutionPolicy Bypass -File "<TOOLKIT>\rollback-oneclick.ps1" -Mode rollback -DryRun

# ── 真回退 ───────────────────────────────────────────
powershell -NoProfile -ExecutionPolicy Bypass -File "<TOOLKIT>\rollback-oneclick.ps1" -Mode rollback

# ── 精确恢复某份快照 ─────────────────────────────────
powershell -NoProfile -ExecutionPolicy Bypass -File "<TOOLKIT>\rollback-full.ps1" list
powershell -NoProfile -ExecutionPolicy Bypass -File "<TOOLKIT>\rollback-full.ps1" restore -Snapshot <name>

# ── 插件源码快照 ─────────────────────────────────────
node "<PLUGIN_SRC>\snapshot.mjs" "<PLUGIN_SRC>\<plugin>" <label>

# ── 依赖同步 ─────────────────────────────────────────
dsh plugin --profile <name> install
```

---

## 附录 A · 现场验证记录（2026-11-08）

写这份文档时实际跑过的检查，可作为"工具当前可用"的证据：

| 检查 | 结果 |
|---|---|
| `check-plugins.mjs --profile <PROFILE> --limit 3` | `ok: true`，`validator: "official"`，`totalBundles: 19`，3 个插件加载无问题 |
| `dsh --profile <PROFILE> --dump-config` | 退出码 **0**；stderr 含 1 条 bundle 解析不到 + 2 条 `patch: entry not found`（均为 B 类软问题，与 §4.3 判据一致） |
| 扫描桌面端 `app.asar` | 无 `safeMode` / `--safe` / "安全模式" 等内置开关（确认 §6.1） |
| `_备份存档` 结构 | 全量快照、前置备份、标签备份三类并存，`safemode-before-*` 与 `pre-upgrade-*` 可对照出安全模式的精确形态 |

---

## 附录 B · 演进史（只留关键节点）

| 时间 | 事件 | 留下的东西 |
|---|---|---|
| 2026-08-20 | 首次做崩溃回退工具（三个 `.bat`） | 单插件禁用 / 恢复 / 诊断 |
| 2026-09-03 | 手工改配置与插件中心冲突 | `rollback-full.ps1`（全量快照 + 原子恢复 + 安全网） |
| 2026-09-26 | 桌面端迁移到新安装目录、profile 改名 | 自动探测生效 profile、`status` 模式 |
| 2026-10-24 | 四缺陷叠加导致回退完全失效 | v2 重写：体检器、最小破坏、智能选快照、双层验证、`-DryRun` |
| 2026-11-06 | DSH 本体升级，用安全模式过渡 | `pre-upgrade-*` / `safemode-before-restore-*` 备份；确认"bundles 收缩"这条路可行 |
| 2026-11-08 | 本文档 | 把上面全部沉淀成可交接的规程 |

---

## 附录 C · 如果这份文档的读者是 AI

几条写给同类的提醒：

1. **先只读，后写入。** `-Mode diagnose`、`check-plugins.mjs`、`status` 都不改文件。把"看清楚"和"动手"分成两步。
2. **别信"回退成功"四个字。** 验证门控（§5.4）是这条流程的全部价值所在，跳过它等于退回 v1。
3. **改动前先备份，新增也算改动**（§7.2）。这不是洁癖——本机每一次"忘了备份"最后都要花十倍时间重建现场。
4. **一次只动一个变量。** 同时禁三个插件，就永远不知道是哪个坏的。
5. **报结论时区分"已验证"和"推断"。** 本机吃过"说修好了其实只是这次操作成功"的亏（§8.3㉒）。
6. **遇到文档没写的现象，先怀疑文档过时，再怀疑机器坏了。** DSH 版本迭代快，路径和 bundle 名会变；`status` 的实测输出永远比文档权威。

---

*本手册由多次真实事故复盘整理而成。所有命令均在 Windows + PowerShell 5.1 环境验证。*
