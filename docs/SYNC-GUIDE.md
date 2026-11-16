# 同步上游 + 更新运行插件（SYNC-GUIDE）

> 用途：把上游最新代码同步到本地，并**真正更新到 DSH 里运行的插件**。
> 配合 `docs/UPGRADE-PLAYBOOK.md`：宿主（DSH）升级后，先按本文件同步，再按 playbook 适配。

---

## 0. 四层关系（先认清，别搞混）

| 层 | 位置 | 作用 |
| --- | --- | --- |
| 上游源码 | `kingcharleslzy-ai/dsh-purge`（`upstream` remote） | 官方最新 |
| 你的 GitHub fork | `chenpeng110701/dsh-purge`（`fork` remote，master 是镜像） | 备份 / 可开 PR |
| 本地 git 仓库 | `E:\pojia\evans-repo\dsh-purge` | 拉代码、研究、改文档的地方（**不参与运行**） |
| **真正运行的插件** | `E:\plugins\dsh-plugins\dsh-purge-community-<版本>` | **DSH 实际加载的**（通过插件管理器 link 进 profile） |

**核心认知**：本地 git 仓库更新了，**不等于**运行中的插件更新了。要真正生效，必须「同步到本地仓库 → 导出到 `E:\plugins\dsh-plugins\` → 插件管理器重装 → 重启 DSH」。

> ⚠️ 旧 fork 关系提醒：最早那个 `origin`（`JamesWang0722/dsh-purge`）是历史遗留，1.7.18 的实际上游是 **`kingcharleslzy-ai/dsh-purge`**。同步只认 `upstream` 和 `fork`，不碰 `origin`。

---

## 1. remote / 分支布局

| 名字 | 类型 | 用途 |
| --- | --- | --- |
| `upstream` | remote | 官方上游：`https://github.com/kingcharleslzy-ai/dsh-purge.git` |
| `fork` | remote | 你的 fork：`https://github.com/chenpeng110701/dsh-purge.git`（推送用它） |
| `origin` | remote | 历史遗留（`JamesWang0722`），一般不用 |
| `sync/fork-master` | 本地分支 | **镜像分支**，接上游更新用，**不要在上面写代码** |
| `work-1.7.18` | 本地分支 | 日常适配/开发 |
| `docs/upgrade-playbook` | 本地分支 | 放范本/文档，和同步互不干扰 |

一次性配置（已配过可跳过）：

```sh
git remote add upstream https://github.com/kingcharleslzy-ai/dsh-purge.git
# 若想 GitHub 上 fork master 也保持最新，可在 fork 页面点 "Sync fork"，或 gh repo sync chenpeng110701/dsh-purge
```

---

## 2. 同步上游到本地仓库

```sh
# 1) 切到镜像分支，直接拉上游最新
git checkout sync/fork-master
git pull upstream master

# 2) 看这次带来了什么（尤其版本号变化）
git log --oneline -10
```

想同时让 GitHub 上的 fork master 也最新（可选，仅备份用途）：

```sh
# 二选一：
#  a) GitHub fork 页面点 "Sync fork → Update branch"
#  b) gh repo sync chenpeng110701/dsh-purge -b master
```

---

## 3. 更新真正运行的插件（关键，别漏）

同步完本地仓库后，把新版导出成运行目录，再让 DSH 插件管理器重装：

```sh
# 记下新版本号（看 package.json）
#   git show sync/fork-master:package.json | findstr version

# 1) 导出干净副本到 E 盘插件区（排除 .git / 旧 node_modules）
#    $NEW = 新版本号，例如 1.7.19
robocopy "E:\pojia\evans-repo\dsh-purge" "E:\plugins\dsh-plugins\dsh-purge-community-$NEW" /E /XD ".git" "node_modules" ".node_modules-host"

# 2) 装依赖（只需要 yaml；cordis 由 DSH 宿主注入，别自己装）
cd "E:\plugins\dsh-plugins\dsh-purge-community-$NEW"
npm install --omit=dev --no-audit --no-fund
```

然后在 **DSH 界面**（我作为 agent 没法代点）：
1. 插件页 → 已安装的 `dsh-purge-community` → **卸载/移除**（清旧 link 与宿主依赖）。
2. 右上角 **「+ 添加插件」** → 输入新目录路径 `E:\plugins\dsh-plugins\dsh-purge-community-$NEW` → 安装。
3. 打开发插件设置，点 **「应用」**（解开/写入宿主补丁，会自动备份）。
4. **完全退出 DSH 再打开**（补丁只有重启后才加载）。
5. 打开面板核对：`已应用 / 78 条已写入宿主` 且顶部显示「已从解包目录启动（换包完成）」= 真正生效。

---

## 4. 验证生效（不只看插件装上，要看补丁真加载）

- 面板「Purge 状态」：`已应用` 接近 76/78，`本端不适用` 只剩下布局本来没有的（约 10 条），顶部「已从解包目录启动（换包完成）」。
- `purge-community probe new` → 发一句对话，模型**原样回显 nonce** = 注入链路通。
- 之前踩过的坑：**只装插件不点「应用」+ 不重启**，面板会显示「本端不适用 84 条 / 已应用 2」——那是补丁没换包加载，不是装错。

---

## 5. 常见操作对照

| 想做的事 | 命令 |
| --- | --- |
| 拉上游最新到本地 | `git checkout sync/fork-master && git pull upstream master` |
| 同步 GitHub fork master | fork 页 "Sync fork"，或 `gh repo sync chenpeng110701/dsh-purge -b master` |
| 看本地落后上游多少 | `git rev-list --left-right --count work-1.7.18...upstream/master` |
| 从最新代码开工作分支 | `git checkout -b work-<新版本> sync/fork-master` |
| 更新运行插件 | 第 3 节：导出 → 装依赖 → 插件管理器重装 → 应用 → 重启 |
| 推送文档分支到 fork | `git push fork docs/upgrade-playbook` |

---

## 6. 红线

1. **不要**在 `sync/fork-master` 上改代码——它是镜像。
2. **不要把本地改动/文档合入 `master`**——master 是上游镜像，合入会让同步分叉。文档放独立分支，不开 PR 合 master。
3. **不要**向 `origin`（JamesWang0722）推送——没写权限（403）。推送用 `fork`。
4. **cordis 等宿主依赖不要自己装进插件目录**——由 DSH 插件管理器注入（`.node_modules-host`）；插件目录只 `npm install --omit=dev` 装 `yaml`。
5. `git pull` 冲突时别盲目 `reset --hard`；先看清哪边改动，必要时 `git stash`。

---

## 7. 一句话记忆

> **同步 = `checkout sync/fork-master` + `pull upstream master`；
> 生效 = 导出到 `E:\plugins\dsh-plugins\` + 插件管理器重装 + 点「应用」+ 重启；
> 推送 = `push fork`；`master` 保持纯净不合文档。**
