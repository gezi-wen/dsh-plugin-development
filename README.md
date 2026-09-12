# dsh-plugin-development

一个 DSH 插件，内容只有一份技能：安装时把包内的 `skills/` 注册为一个技能根，于是这份技能随插件进入 agent 的技能目录。

```sh
dsh plugin --profile web add dsh-plugin-development
```

## 技能覆盖什么

- **双半包骨架**：`package.json` 的必要字段（`main` / `exports["./client"]` / `dsh.bundle.patch` / `dsh.client`）、`cordis.patch.yml`、手写的 ModuleLoader client 半
- **对着 fork 开发**：把 fork link 进一个丢弃式试验 profile，三级验证（配置组合 → 模块解析 → 真执行），以及 `dsh plugin` 与 profile 的关系
- **发布与上线**：推 GitHub / 发 npm，把新版本部署到正在运行的 profile 而不碰它的安装状态
- **坑清单**：包括宿主插件跑 PowerShell 子进程才会遇到的三个（AMSI 拦胖脚本、stdout 编码、编译缓存失效），以及「插件装了但界面不出现」的排查链

## 布局

```
package.json        # dsh.bundle.patch → cordis.patch.yml
cordis.patch.yml    # 插入一行 @deepseek-ai/dsh-skill-filesystem，customSkillDirs 指向 skills/
skills/
  dsh-plugin-development/
    SKILL.md
    REFERENCE.md
```

`cordis.patch.yml` 用的机制和 DSH 自带 agent preset 装载自己的技能是同一套：`!!js` 里的 `baseUrl` 解析到本包，所以路径跟着安装位置走。

## License

MIT
