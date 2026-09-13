# DSH Web UI 插件 — 格式细节与模板

参照实现：一个纯 client 半的最小插件（会话头部时钟）、一个用 typert remote 的实例、
一个 host 半为主、跑 PowerShell 子进程的样本（`sage-guikit`，公开仓库）。
改完任何 client 代码后跑一遍对应包的 `scripts/smoke-client.mjs`。

## §1 package.json 逐字段

```jsonc
{
  "name": "sage-xxx",
  "version": "0.1.0",
  "type": "module",
  "main": "lib/index.js",                    // host 半入口（Cordis 插件模块）
  "exports": {
    ".": "./lib/index.js",
    "./client": "./lib/client.js",           // client 半入口（必须叫 ./client）
    "./typert": "./lib/typert.host.js",      // 可选：typert strict manifest
    "./package.json": "./package.json"
  },
  "dsh": {
    "bundle": { "patch": "./cordis.patch.yml" },   // bundle 自带 patch 文件
    "client": {
      "platform": "web",
      "inject": [                                  // client 模块依赖的宿主模块
        "@deepseek-ai/dsh-client-locale",
        "@deepseek-ai/dsh-client-store"           // 0.1.2 起取代已删除的 dsh-client-runtime
      ]
    }
  },
  "peerDependencies": {
    "@deepseek-ai/cordis": "^4.0.1",
    "@deepseek-ai/dsh-typert-protocol": "^0.1.0-rc.6",   // 用 typert 时才需要
    "@deepseek-ai/dsh-client-locale": "^0.1.0-rc.6",
    "@deepseek-ai/dsh-client-store": "^0.1.5-rc.1",
    "react": "^18.0.0"
  },
  "files": ["lib/index.js", "lib/client.js", "cordis.patch.yml"]
}
```

## §2 cordis.patch.yml 与 profile 接入

包内 `cordis.patch.yml`（bundle patch）：

```yaml
- insert:
    - id: sage-xxx
      name: sage-xxx
```

profile 侧（`$DSH_HOME/profiles/<profile>/package.json`）：

```jsonc
"dependencies": { "sage-xxx": "link:E:/workspace/sage-xxx" },  // 本地包用 link:
"dsh.profile.bundles": [ "...", "sage-xxx" ]                   // 追加到数组尾
```

然后 **在 profile 目录 `pnpm install`**（生成 junction）。验证：
`node lib/bin.js --profile web --dump-config`（在 dsh 安装目录跑）应出现插件行。

## §3 client 半：手写 ModuleLoader bundle

静态 client 不是自由 ESM，是构建产物同构的手写 bundle：

```js
window.__ModuleLoader__.load({
	id: "sage-xxx",
	factory: (require) => {
		var module = { exports: {} };
		var exports = module.exports;
		let react = require("react");
		const h = react.createElement;
		const appCtxBox = { current: null };

		function Panel(props) { /* ...react.createElement 树... */ }

		async function apply(ctx) {
			appCtxBox.current = ctx;
			ctx.slots.inject("shell.overlay", () => {
				ctx.slots.register(
					{ name: "shell.overlay", id: "sage-xxx-overlay", order: 50, label: "面板" },
					() => h(Overlay),
				);
				ctx.slots.register(
					{ name: "shell.overlay", id: "sage-xxx-edge-tab", order: 40, label: "入口" },
					() => h(EdgeTab),
				);
			});
		}
		exports.apply = apply;
		exports.inject = ["slots"];       // 声明用到的服务；remote 另见 §4
		return module.exports;
	},
});
```

要点：
- `exports` 必须带 `apply` 和 `inject`；
- **client 看不到宿主文件系统**：浏览器读不了 `C:\...` 这类路径。要在界面上显示 host 上的图片/文件，
  必须由 host 半转成 **data URL / base64**（或提供可访问的 URL）再回传——只回路径的结果是破图；
- 全局 `setInterval`/`clearInterval` 在 client 环境可用（会话头部时钟那个插件就是这么做的）；
- CSS 用「幂等注入」约定：`document.head` 加 `style[data-plugin-css="<pkg>/overlay.css"]`，
  写前先 querySelector 判重；类名加独立前缀（`smap-` / `slr-`），多插件共存不打架；
- 右缘浮标样式（多入口堆叠：第二个起 top 改 `calc(15% + 74px * n)`）：

```css
.smap-fab{position:fixed;right:0;top:15%;transform:translateY(-50%);z-index:900;
  width:36px;height:64px;border-radius:12px 0 0 12px;border-right:none;
  background:rgba(13,18,38,.78);color:#ffd27d;font-size:17px;opacity:.5;cursor:pointer;}
.smap-fab:hover{opacity:1;width:46px;background:rgba(20,28,56,.95);}
```

## §4 typert remote 三件套（静态 client→host 数据通道）

动态插件的 `harness.handle`/`host.call` 是 Package-private 动态通道；**静态包走 typert**。

**host 半**：`class Gateway extends TypertRemoteService`（构造传 `(ctx, "<ns>")`），
方法上用手写 decorator marker 注册（Node 不解析装饰器语法）：

```js
function markRemote(cls, method, exportName) {
	const instance = Object.create(cls.prototype);
	Remote(exportName)(undefined, {
		kind: "method", name: method, private: false, static: false,
		addInitializer(fn) { fn.call(instance); },
	});
}
markRemote(Gateway, "listStars", "listStars");
export default Gateway;
```

**manifest**（`lib/typert.host.js`，经 `exports["./typert"]` 被 typert-loader 读取）：
导出 `TYPERT = { package, face: "host", schemas: [], invocations: [...], model }`。
每条 invocation：`{ id: "<pkg>#<ns>/<method>", service, namespace, method,
invocation: { kind: "direct" }, parameters: [...], result: { mode: "strict",
typeSymbol, schema } }`。**schema 必须是 zod v4 实例**（loader 校验 `"_zod" in schema`）。
抄 `sage-starmap/lib/typert.host.js` 改字段最快。

**client 半**：浏览器无 zod，schema 手写 `{ parse(v){...} }`（typert 只要求有 parse）。
`inject` 加 `"remote"`，apply 里先挂载再使用：

```js
await ctx.remote.$mount(TYPERT_REMOTE);        // 失败 catch 住，别拖死 UI
const remote = ctx.get("remote.<ns>");         // 每次用时取
remote.listStars().then((r) => { const d = r && r.ok ? r.value : r; ... });
```

返回是 `{ok, value}` 信封，记得 unwrap。给拉数据加超时守卫（20s 后提示 Host 端未激活），
否则 host 半没起来时 UI 永远转圈。

## §5 冒烟脚本骨架

```js
import { createRequire } from 'node:module'
const req = createRequire('E:/workspace/<pkg>/lib/client.js')
const react = req('react')
let loaded = null
globalThis.window = { __ModuleLoader__: { load(def) {
	loaded = def.factory((n) => n === 'react' ? react : (() => { throw new Error('unexpected ' + n) })())
}}}
await import('file:///E:/workspace/<pkg>/lib/client.js?smoke=' + Date.now())
// 断言 loaded.apply 是函数 → mock ctx（get/slots.inject/slots.register/remote.$mount）
// → await loaded.apply(fakeCtx) → 打印注册到的 slot 清单
```

注意：mock 测不出布局死锁这类真实 DOM 时序 bug——冒烟全绿不代表 UI 能出，最终以刷新页面为准。

## §6 坑清单

| 坑 | 说明 |
|---|---|
| rev 不自动生效 | client.js 改动后服务端指纹（rev）会变，但浏览器要手动刷新才拉新 |
| 页面从未刷新 | 崩溃/换实例后旧页面一直开着 → 新插件永远不可见，与真故障难区分 |
| F5 打错窗口 | GUI 自动化刷新前先确认前台窗口句柄；最小化窗口 activate 不回前台 |
| Windows file URL | 动态 import 本地路径必须三斜杠 `file:///E:/...` |
| exports["./client"] | 必须存在且为字符串或单 default 条件形式，否则扫描报错 |
| 漏 pnpm install | 启动 import 失败崩在 readiness 前；dump-config 验不出 |
| 入口绑 Run 卡片 | `tool.view.cordis` 只适合附着运行结果；常驻入口用 `shell.overlay` |
| seq/记录损坏 | 会话日志问题见 memory `reference_session-repair.md`（另一套工具链） |
| 改名留残链 | 插件改名后 `pnpm install` **不一定**清掉旧的 `node_modules/<旧名>` junction——它会悬空。
手动摘：`cmd /c rmdir <junction 路径>`（只删链接、不碰目标；`Remove-Item -Recurse` 有删到目标内容的风险） |
| 同名两份 | 同一 profile 里一个包名只能有一份；两个插件注册相同工具名/槽 id 会冲突。要并存就换 profile |

## §7 用 link 开发调试（fork → 试验 profile → 三级验证）

一条实践约定：「**以后就按这个流程开发插件**」。全链实测通过。

| 想做的事 | 命令 |
|---|---|
| 从内置模板克隆试验 profile | `dsh --profile web-test --from-default-profile web --dump-config`（**加 `--dump-config` 只创建 + 打印，不起服务**） |
| 把 fork 挂进去 | profile 的 `package.json`：`"<包名>": "link:E:/workspace/<插件名>"` + `bundles` 加包名 → 该目录 `pnpm install` |
| 起隔离实例 | `dsh --profile web-test --port 0 --no-open`（port 0 = 系统挑空闲端口；`--no-open` = 不抢浏览器） |

- **`dsh plugin ...` 就是 pnpm 转发**（`@deepseek-ai/dsh/bin.js:105` 注释原话
  "forwarding the remaining arguments to pnpm in the profile directory"），所以加/删/换插件
  都是在 profile 目录里跑 pnpm；`dsh plugin --profile <名> add <包>@link:<路径>` 也行
- 非模板 profile 名（`web-test` 这种）**DSH 不会改写它的 bundle 列表**，完全归你
- **`-test` 加在 profile 名上，不要加在插件名上**：fork 自带的 `cordis.patch.yml` 里 `name:`
  指向原名，会去导入正式那份；就算改了名，两个插件注册的工具名/槽 id 相同 → 冲突

### 模块解析的三个锚点（权威：`dsh-app-boot/lib/index.js:300-308` 注释）

> bundle 名先**从 DSH 安装体**解析、再**从 profile 目录**解析；profile 的 `node_modules` 优先；
> `$DSH_HOME/profiles/node_modules` 靠 **Node 常规向上查找**供货。

- **`$DSH_HOME/profiles/node_modules` 现在由 DSH 自愈**：`healProfilesModuleFallback()` 每次启动
  遍历安装体的依赖 + peer 闭包重建它——老「手搓 junction 桥」的**目标端**不用再管
- `healProfileModuleFallback()` 另外把「只有某些 bundle 携带的包」投影进单个 profile 的
  `node_modules`（profile 里看到的 `react` / `loose-envify` 就是它干的）
- **插件目录在 `$DSH_HOME` 树外面**（自研插件通常放自己选的目录，不放进 `$DSH_HOME`）⇒ Node 从真实路径
  向上走永远到不了闭包 ⇒ 插件目录必须自己有一份：`node_modules/@deepseek-ai/<pkg>` 做成
  junction 指向 `$DSH_HOME/profiles/node_modules/@deepseek-ai/<pkg>`。**两个直接依赖
  （cordis + dsh-tools）就够**——junction 的*目标*在闭包里，传递依赖在那里解析。
  这就是「新插件装完必 pnpm install」背后的机制；peers 一律写 `*`

### 三级验证阶梯（每级验的东西不同，别跳级）

1. `dsh --profile <名> --dump-config` → 验**组合**（插件行在不在树里）。
   ⚠️ **假阳性常客**：只验配置组合，不验模块 import
2. 在 profile 目录 `node -e "import('<包名>').then(m => console.log(m.name))"` → 验**模块解析**
   （能 import、`realpath` 指向 fork 而不是生产路径）
3. 真启动 + 临时在 `apply()` 里塞一句 `console.log('[pkg-marker] apply() 被调用')` → 验**执行**。
   启动日志里出现它，才叫"DSH 真的加载了这份代码"；验完记得 `git checkout -- <文件>` 还原

### 测试卫生

- 两个实例可以同时跑，互不干扰（实测：测试 50215/65045，生产 58225）。**别用固定端口**去迁就
- ⚠️ **`job_kill` 只杀 pwsh 外壳，node 子进程会留下**——收工按端口清：
  `Get-NetTCPConnection -LocalPort <p> -State Listen` 取 `OwningProcess` 再 `Stop-Process`

### 发布之后的部署动作

1. fork 里改 → 试验 profile 按上面三级验
2. 提交 / 打 tag / 推 GitHub / `npm publish`
3. 生产 profile 的活体目录：`git checkout <tag> -- <运行时文件>`
   （**不碰它的 `pnpm-workspace.yaml` / `pnpm-lock.yaml`**，那两个是它的安装状态，
   覆盖会冲掉 junction 桥）→ 重启 dsh web

## §8 宿主插件跑 PowerShell 子进程的三个坑

宿主半需要调 Windows 能力时（截屏、注入输入、读窗口），本仓的做法是**每次调用起一个
`pwsh -Command <脚本>` 子进程 + 内联 C#**（`sage-guikit` 插件是完整样本）。三个坑都真踩过：

**① Defender 会经 AMSI 拦「胖脚本」。** 把「枚举窗口标题 + GetWindowRect + PrintWindow + 画网格」
全塞进一个工具的脚本里，会被判为恶意脚本**直接拒绝执行**，报
「此脚本包含恶意内容，已被防病毒软件阻止」——**症状看起来像语法错**（ParserError），极易误判成
自己的 bug。它是按整段脚本的窥屏特征**打分**的，不是某一行的问题（加法二分不自洽：单加一条被拦、
五条全加反而过）。**解法：一个工具一段脚本、把它压瘦**；改完跑冒烟挂具验。

**② stdout 编码必须自己钉死。** 子进程没有控制台时（`windowsHide: true` 就是这种），.NET 的
`[Console]::OutputEncoding` 回退到系统 ANSI 代码页（中文 Windows = gb2312），PowerShell 于是把
**GBK 字节**写进 stdout，宿主按 UTF-8 解码 → 回传的中文界面文本（控件名、窗口标题）全变 U+FFFD，
而 ASCII 部分正常，**很隐蔽**。修法：脚本第一句
`[Console]::OutputEncoding = New-Object System.Text.UTF8Encoding $false`。
诊断靠「原始 stdout 字节 hex + GBK/UTF-8 双解」，别靠肉眼看终端。

**③ 编译缓存要按内容失效。** 内联 C# 编译成 DLL 缓存能省每次几百毫秒，但文件名若用固定版本号
（`U32.v2.dll`），加了新方法后会加载到**缺方法的旧 DLL**，报「U32 不包含名为 X 的方法」。
用 C# 源码的内容哈希当文件名（`createHash('sha1').update(CS).digest('hex').slice(0,10)`）即可自失效。

## §9 携带技能的插件（skill-shipping plugin）

有一类插件本体**没有 host / client 代码，内容就是一份技能**：`cordis.patch.yml` 往组合里插一行
`@deepseek-ai/dsh-skill-filesystem`，把包内 `skills/` 注册成一个额外技能根，于是「装插件 = 装技能」。
本仓自己就是这个形制，`dsh-repair` / `dsh-skill-authoring` 也一样。

```yaml
- insert:
    - id: my-skill-plugin-skills
      name: '@deepseek-ai/dsh-skill-filesystem'
      config:
        providerName: my-skill-plugin     # ← 必须写，否则 DSH 起不来
        includeDefaultRoots: false        # ← 建议写，见下
        customSkillDirs:
          - !!js "process.getBuiltinModule('node:url').fileURLToPath(new URL('skills/', baseUrl))"
```

**`providerName` 不写会让 DSH 整个起不来。** 技能注册表按 provider 名建档、**同名只准一个**，而这个
插件的 `providerName` **默认值是 `filesystem`**（`dsh-skill-filesystem/lib/index.js:32`：
`providerName: z.string().min(1).default("filesystem")`）。harness 自己那行已经占了 `filesystem`，
第二个注册者于是直接报 `a skill provider named "filesystem" is already registered`，
**插件树加载失败、DSH 崩在 readiness 之前**——而且**只有同时装两个这类插件才会炸**，只装一个一切正常。
本仓 0.1.1 就带着这个 bug 发出去过，0.1.2 才修掉（2026-09-13）。

- 每个包用**自己的包名**当 `providerName`，互不冲突
- `includeDefaultRoots: false` 是第二件该做的事：项目 / 用户技能根由 harness 自带那个 provider 扫，
  这个 provider 只管包内 `skills/`；不写的话 N 个插件会把默认根扫 N 遍
- `!!js` 里的 `baseUrl` 指向**本包**，所以路径跟着安装位置走，不用写绝对路径

**验证别只看 `--dump-config`**：它只验配置组合，这个 bug 在组合层完全看不出来。要么真启动一个
隔离实例，要么**把两个同类插件同时挂进一个试验 profile**——只挂一个永远发现不了。
