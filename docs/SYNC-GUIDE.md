# 从 fork 同步上游最新代码（SYNC-GUIDE）

> 用途：规则化「把 fork（`chenpeng110701/dsh-purge`）上游最新代码拉到本地」，同时不弄丢本地分支和范本。
> 配合 `docs/UPGRADE-PLAYBOOK.md`：**先同步到最新宿主/代码，再按 playbook 适配新宿主版本。**

---

## 1. 分支与 remote 布局（先认清）

| 名字 | 类型 | 用途 |
| --- | --- | --- |
| `fork` | remote | 你的 fork：`https://github.com/chenpeng110701/dsh-purge.git` |
| `origin` | remote | 原仓库：`https://github.com/JamesWang0722/dsh-purge.git`（历史遗留，一般不用它推送） |
| `sync/fork-master` | 本地分支 | **镜像分支**，跟踪 `fork/master`，只用来接上游更新，**不要在这上面写代码** |
| `work-1.7.18` | 本地分支 | 日常开发/适配分支 |
| `docs/upgrade-playbook` | 本地分支 | 放升级范本等文档，和同步互不干扰 |

**核心事实**：
- `git pull fork master` 只更新 `sync/fork-master` 的内容，**不会删除任何本地分支，也不会碰其他分支**。
- 已推到 GitHub 的分支（`docs/upgrade-playbook`、`sync/fork-master`）**只有你手动删才会消失**；上游更新 master 不影响它们。

---

## 2. 一键同步（标准流程）

```sh
# 1) 切到镜像分支
git checkout sync/fork-master

# 2) 拉上游最新（只更新这一个分支）
git pull fork master

# 3) 看这次带来了什么
git log --oneline -10
```

`Already up to date.` = 已经是最新，无需操作。

---

## 3. 同步后怎么继续干活

同步完 `sync/fork-master` 已经是最新代码。基于它开新分支做适配/开发，**不要直接改 sync 分支**：

```sh
# 基于最新代码开一个新的工作分支
git checkout -b work-<新版本号> sync/fork-master
```

如果是继续在 `work-1.7.18` 上做，把它 rebase 到最新镜像上（保持历史线性）：

```sh
git checkout work-1.7.18
git rebase sync/fork-master
```

> rebase 前先确认工作区干净（`git status`）。有未提交改动先 `git stash` 或提交。

---

## 4. 常见操作对照

| 想做的事 | 命令 |
| --- | --- |
| 拉上游最新 | `git checkout sync/fork-master && git pull fork master` |
| 看本地和上游差多少 | `git rev-list --left-right --count work-1.7.18...fork/master` |
| 把某分支同步到最新（重置，丢弃该分支本地提交） | `git checkout sync/fork-master && git reset --hard fork/master` |
| 把本地范本分支推到 fork | `git push fork docs/upgrade-playbook` |
| 上游有新 tag/分支 | `git fetch fork --prune` |

---

## 5. 红线

1. **不要**在 `sync/fork-master` 上改代码——它是镜像，改了会和上游冲突。
2. **不要**用 `origin`（原仓库）推送——你没有它的写权限（会 403）。推送用 `fork`。
3. `git pull` 冲突时不要盲目 `reset --hard` 丢掉本地工作；先看清楚是哪边改动，必要时 `git stash`。
4. 本文件与 `docs/UPGRADE-PLAYBOOK.md` 同属文档沉淀，不随上游代码逻辑变化而失效；上游若改了分支模型，回来更新本文件。

---

## 6. 一句话记忆

> **同步 = `checkout sync/fork-master` + `pull fork master`；干活 = 从 sync 分支开新分支；推送 = `push fork`。**
