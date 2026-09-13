---
name: dsh-plugin-development
description: Build and ship DSH plugins (host half + client half), develop and debug them safely by forking the plugin and linking it into a throwaway test profile, publish to GitHub/npm, and fix "the plugin loaded but its UI or tool never appears". Use when writing any DSH plugin (client UI or host-side tools), creating sage-* style packages, wiring a plugin into a profile bundle, running PowerShell subprocesses from a host plugin, promoting a dynamic cordis_define plugin into a permanent bundle, or diagnosing a plugin that silently does nothing.
---

# DSH Web UI 插件开发与固化

给 DSH web profile 写带 UI 的静态插件（host 半 + client 半），把动态插件（cordis_define 的产物）固化成永久包，以及排查「插件装了但 UI 不出现」。

## 双半包最小骨架

静态 UI 插件 = 一个 npm 包（`type: module`）+ 一个 bundle patch 声明：

```
sage-xxx/
├── package.json        # 双半声明，见 REFERENCE §1
├── cordis.patch.yml    # - insert: [- id: sage-xxx, name: sage-xxx]
├── lib/index.js        # host 半：apply(ctx)，数据/文件/SSH 在这层
├── lib/client.js       # client 半：window.__ModuleLoader__.load({id, factory})，见 REFERENCE §3
└── scripts/smoke-client.mjs  # 冒烟：Node 侧模拟 ModuleLoader 跑 factory+apply
```

`package.json` 四个关键声明缺一不可：`main`（host 入口）、`exports["./client"]`（client 入口）、
`dsh.bundle.patch: "./cordis.patch.yml"`、`dsh.client: { platform: "web", inject: [...] }`。
完整模板见 [REFERENCE.md](REFERENCE.md)。

## 固化三步铁律（动态 → 永久）

动态插件（`cordis_define`/`cordis_run` 的产物）**只活在当前进程**，重启即消失。
固化成静态包的完整路径，三步缺一不可：

1. **建包**：把动态 `code.host` / `code.client` 移植进 `lib/index.js` / `lib/client.js`。
   注意通道不同：动态用 `harness.handle` + `host.call`；静态 client→host 用
   typert remote（`ctx.remote.$mount` + `ctx.get("remote.<ns>")`），见 REFERENCE.md 的「typert remote 三件套」一节。
   （别写成 `见 REFERENCE §4`——`check-skill.js` 会把正文里的 `§N` 当**本文件**的章节引用，
   而 SKILL.md 没有编号章节，于是报「交叉引用找不到对应章节标题」。指章节要么写全名，
   要么用文件内标题。）
2. **接入 profile**：`profiles/web/package.json` 的 `dependencies` 加
   `"sage-xxx": "link:E:/workspace/sage-xxx"`，`dsh.profile.bundles` 数组加包名。
3. **`pnpm install`**：在 profile 目录跑。漏这步 = DSH 启动 import 失败崩在
   readiness 之前（tavily 插件的教训）。`--dump-config` 只验组合、不验 import，不能替代。

完成后 `--dump-config` 验组合 → 刷新页面验证 UI。**重启后自动加载，这就是「永久」。**

## 开发流程（2026-09-12 定的标准姿势）

改一个**已经在跑**的插件，不要在生产的 profile 里改。四步：

1. **fork 到工作区**：`<workspace>/<插件名>`（clone 仓库本身；别从 `node_modules` 里 fork，
   那里只有发布出去的 `files`）。fork 只是**开发副本**——正式/活体的插件源码放自己的插件目录
   `<plugin-dir>\<名>\`，profile 的 `link:` 最终要指向那里；工作区是"制作中"区，
   别把生产 `link:` 长期指向它。
2. **建试验 profile**（一次性）：`dsh --profile web-test --from-default-profile web --dump-config`
   —— 加 `--dump-config` 只创建 + 打印配置树，**不起服务**。然后把 fork 挂进去：
   `"<包名>": "link:<workspace>/<插件名>"` + `bundles` 加包名 + 在该目录 `pnpm install`。
   **`-test` 加在 profile 名上，绝对不要加在插件名上**——同一 profile 一个包名只能有一份，
   给 fork 改名会连环撞（patch 里的 `name` 仍指向原名；两个插件注册的工具名/槽 id 会冲突）。
3. **三级验证，别跳级**（每级验的东西不一样）：
   ① `--dump-config` 验**组合** → ② 在 profile 目录 `node -e "import('<包名>')"` 验**模块解析**
   （realpath 要指向 fork）→ ③ 真启动 `dsh --profile web-test --port 0 --no-open` 验**执行**
   （临时塞一句 `console.log` 进 `apply()`，看启动日志里有没有它）。
4. **发布之后才动生产**：打 tag / 推 GitHub / `npm publish` → 生产 profile 的活体目录
   `git checkout <tag> -- <运行时文件>`（**别碰它的 `pnpm-workspace.yaml` / `pnpm-lock.yaml`**，
   那两个是它的安装状态，覆盖会冲掉 junction 桥）→ 重启 dsh web。

为什么要这么绕：插件目录里 `link:` 的是**生产路径**，直接改等于改线上；而 `--port 0 --no-open`
起的隔离实例与生产互不干扰（实测两个实例可同时跑，端口各自独立）。
机制与踩坑见 REFERENCE.md 的「用 link 开发调试」与「宿主插件跑 PowerShell 的坑」两节。

**手滑红线**：每条 dsh 命令都显式写 `--profile`。`dsh plugin` 的 `--profile` 是**必填**
（漏了直接报错，不会默默改生产），但 **`dsh web` 是 `--profile web` 的别名**——
想开试验实例却顺手敲了 `dsh web`，起的就是生产 profile。习惯上把 `--profile <名>` 当第一个参数写。

## 「UI 不出现」排查链（按序执行，每步都有硬证据）

1. `dsh --profile web --dump-config` — 组合里有没有插件行？
2. `profiles/web/node_modules/<pkg>` junction 在不在？
3. `curl` 首页看有没有客户端插件注入行 —— ⚠️ **必须带上 `dsh web` 启动时打印的 `?token=`**，
   否则被打回 `dsh web authentication required`（0.1.5 实测）。不带 token 的 curl 会拿到空响应，
   看起来像「插件没注入」，其实连门都没进——这一个假象够骗人半小时。
   最省事的判据其实是肉眼：会话头部那个时钟就是某个 client 插件的注入效果，
   它在那儿＝客户端插件链路正常。（bundle 直连形如 `/plugins/<pkg>/client.js?rev=...`，同样要过鉴权。）
   （`<port>` 每次重启都变，动态发现方法见坑三）
4. `curl` 那个 URL — bundle 是否 200、内容是不是最新（搜你新加的类名/注释）？
5. `node --check lib/client.js` — 语法对不对。改完 client **立刻**跑这一条，最便宜也最快。
   exit 0 不证明逻辑对，但能挡掉「改完整个插件白屏」这类低级事故。
6. `node scripts/smoke-client.mjs` — factory/apply 在 Node 侧能否跑通？
   ⚠️ 它可能报 `react not resolvable from plugin dir`，而插件目录里其实**有** react——
   那是脚本自身的解析问题，不是插件坏了，别被它带偏。
7. **刷新页面** — rev 变了浏览器也不会自动拉新；页面可能从崩溃起就没刷新过。
   「装好但没刷新」和「真坏了」的症状一模一样。刷新后**用 GUI 截图验证真实渲染**，
   再**点一次入口确认开合**——冒烟测试过了不等于界面能用，注册成功 ≠ DOM 渲染成功。

## 三个致命坑（都真实发生过）

**布局死锁**：`layout` 的 effect 等 `canvasEl`，而 canvas 要等 `layout` 算完才被
渲染分支放进 DOM → ref 永远 null，永远停在「整理星图…」。修法：**布局 effect 只依赖
数据**（`[data]`），canvas 的 effect 才依赖 `[data, layout, canvasEl]`。
冒烟测试的 mock ctx 测不出它——mock 环境里 slot 注册成功≠真实 DOM 渲染成功。

**入口绑错位置**：UI 注册进 `tool.view.cordis`（Run 卡片）会跟着某次运行记录走，
换会话/换窗口就「消失」。常驻入口有两个正解，先想清楚这个入口属于哪一类：

| 入口性质 | 放哪 | 说明 |
|---|---|---|
| **日常入口**（会话里随手要用） | `conversation.session.header.actions` | 会话头部那一排、标题旁边。位置由 `order` 决定——想紧挨某个既有入口右侧，就取它 order + 1 |
| **全局入口 / 面板本体** | `shell.overlay` | 帧级浮层，不依赖会话。全屏面板（veil + modal）放这里；确实要在任何页面都能看到的浮标也放这里 |

不要侧栏。旧的右缘浮标写法（`fixed; right:0; top:15%`，多个按 74px 垂直堆叠）仍可用，
但**日常入口优先走 `header.actions`**——省地方、符合直觉，也不挡内容。

**端口易变，绝不写死**：dsh web 的监听端口
**每次重启都会变**——启动命令甚至可能是 `--port 0`（随机空闲端口）。任何文档、
脚本、测试里看到的 `127.0.0.1:51971` 之类都只是某一次启动的快照。需要访问 Web UI
时动态发现（已实测）：

```powershell
# 按进程命令行过滤出 dsh 实例，再查它监听的 127.0.0.1 端口
$pids = (Get-CimInstance Win32_Process |
  Where-Object { $_.CommandLine -like '*@deepseek-ai\dsh*bin.js*' }).ProcessId
Get-NetTCPConnection -State Listen -ErrorAction SilentlyContinue |
  Where-Object { $_.OwningProcess -in $pids -and $_.LocalAddress -eq '127.0.0.1' } |
  Select-Object LocalPort, OwningProcess
```

同理推广：给 DSH 配套的辅助服务（MCP http server、sidecar 脚本、测试断言）
一律**不要绑死端口**去迁就它——stdio 或「随机端口 + 动态回报」才是稳的。
多个 dsh 实例并存时（如 `--port 0` 与 `--profile web` 各一个），先看命令行再认端口。

## 详细格式与模板

package.json 逐字段、ModuleLoader bundle 模板、typert remote 三件套
（host manifest / client codec / $mount）、冒烟脚本全文、坑清单、
**用 link 开发调试（试验 profile / 模块解析三锚点 / 三级验证）**、
**宿主插件跑 PowerShell 子进程的三个坑（AMSI 拦胖脚本 / stdout 编码 / 编译缓存失效）**、
**携带技能的插件（`providerName` 不写会让整个 DSH 起不来）**：
见 [REFERENCE.md](REFERENCE.md)。
