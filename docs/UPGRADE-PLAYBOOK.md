# DSH 宿主升级适配范本（Upgrade Playbook）

> 用途：DSH 每次发新版本（`app.asar` 换新、宿主源码形状变化）后，按本范本把 `dsh-purge` 的 86 条补丁对齐到新宿主。
> 这份是从 1.7.x 适配 0.2.2-rc.x 的真实做法中提炼的**可复用流程**。开始前先读 `AGENTS.md` §7 / `compatibility.md`。

---

## 0. 心智模型（先记住三件事）

1. **补丁 = 对宿主源码的精确片段替换**。每条 `{id, name, layer, file, replacements:[{pattern, replace}], markers, optional?}`：
   - `pattern`：宿主源码里的**逐字片段**（含 tab/空格/换行）或正则；
   - `replace`：替换体，**必须含 `[dsh-purge]` 标记**并写进 `markers`；
   - `markers`：判断"已应用/幂等"的标志。
2. **宿主升级后旧 `pattern` 会漂移（match 不到）**。这不是插件坏了，是宿主源码形状变了。要逐条重新定位。
3. **三种状态要分开取证**：磁盘已应用 ≠ 当前进程已加载 ≠ 模型行为达标。别把"能写盘"当"已适配"。

---

## 1. 触发时机

DSH 官方更新后（`app.asar` 重新出现、旧补丁全失效）、插件自身升级后、模型开始推脱或工具被追问审批时。

---

## 2. 标准适配流程（7 步）

### 第 1 步：确认宿主版本 + 环境

```sh
# 桌面客户端真实版本（文件属性 FileVersion）
# Windows: (Get-Item "…\DeepSeek Harness.exe").VersionInfo.FileVersion
```
记下确切版本号（如 `0.2.2-rc.3`），后面 `dshReleases` 要按它填。

### 第 2 步：解包宿主，拿到完整源码树

不要用野路子 offset 解 asar（差 8 字节会截断文件）。用项目自带的 `lib/extract-asar.js`。
关键：asar 的 `dataOffset = 16 + headerSize`（不是 `8 + headerSize`）。
解出后核对：随便挑一个 `lib/index.js` 的**末尾 export 行是否完整闭合**，确认没截断。

### 第 3 步：跑 drift 诊断，列出所有漂移补丁

```sh
DSH_SURFACE=desktop node bin/community.js diagnose --json
```

输出把补丁分成四类：
- `drifted`：目标文件在、marker 不在、锚点全不中 → **必须处理**（真漂移）；
- `optionalDrift`：`optional`/`fileOptional` 目标的漂移，不牵连整条（按需处理）；
- `chained`：级联补丁（如 #88 依赖 #87），前置没落地就不算漂移；
- `missing`：目标文件整组不存在（可能是该布局本来没这个包，也可能路径映射错）。

每条 drifted 会附 `anchors[]`，含 `head`（旧锚点首行）+ `suggestion`（`locateAnchor` 用旧 pattern 的特征 token 在新源码里找到的**可能新落点**：行号 + 片段）。

### 第 4 步：逐条判定——真漂移 vs 结构消失（**最关键，别硬凑**）

对每个漂移补丁，先分清属于哪种：

| 情形 | 判定 | 动作 |
|---|---|---|
| 逻辑还在，只是移了位置/改了缩进/变量改名 | **真漂移** | 重新取精确片段，改 pattern |
| 逻辑迁到了新包（如身份句从 `dsh-web-app` 迁到 `dsh-harness-dev-context`） | **路径迁移** | 在 `targetFiles()` 加新 key，改补丁 `file` 字段 |
| 宿主已把目标态改到位（官方自己删了那段禁止句） | **已达成** | 加等价"已达成"marker，或不改 |
| 结构彻底重写消失（如 rewind 客户端整体压缩重写） | **结构消失** | 保持原定义不匹配、标 `fileOptional` 或 `silentWhenMissing`，**不要硬拼一个假锚点** |
| 拿不准是真没了还是换写法 | **存疑** | **保持不动 + 记录待确认**（像 0.2.2 诊断里 #67 那样），绝不为了消红而吞错误/丢输入 |

> ⚠️ 真实范例（`maintenance-022-diagnostics.md`）：面对 8 条漂移，作者逐条分析，能确认"已达成/迁址"的说明清楚，**拿不准的 #67 保持漂移并写明理由**，没有强行改造。这是标准做法。

### 第 5 步：重新取锚点（真漂移时）

方法 A（改源码，正式修）：把 0.2.2 真实源码里那段的**精确字节**（含 tab）复制进 `replacements[].pattern`。
取字节用 Node 读原文 `JSON.stringify(text.slice(start,end))`，避免手抄空格错。

方法 B（临时重锚，不改源码）：`reanchor` 把漂移锚点覆盖到用户目录：

```sh
purge-community diagnose                       # 看 anchor 文件 + 行号 + 建议片段
purge-community reanchor apply <id> --lines A-B          # dry-run
purge-community reanchor apply <id> --lines A-B --yes    # 采纳
purge-community reanchor clear <id>                      # 撤销
```
覆盖存 `$DSH_HOME/dsh-purge/anchor-overrides.json`，不下沉源码（`lib/anchors.js` 的 `applyOverrides` 在 apply 时叠加）。适合临时救急；正式发布仍应在源码里改 pattern。

### 第 6 步：验证命中 + 幂等 + 不引入语法错误

```sh
# 逐条对完整解包副本跑命中
# 判据：HIT（全命中）/ PARTIAL（部分命中，幂等分支）/ MISS（没命中）/ FILE_MISSING
# 有效 = HIT + 合理的 PARTIAL；MISS 必须逐条能解释（迁移型/结构消失/链式）
```

关键三项：
1. **每个改动的 .js 文件对比 apply 前后**：`node --check`（用 `.mjs` 后缀跑，避免 ESM 被当 CJS 误报）。**补丁引入的新语法错误必须 = 0**。
2. **markers 落位**：apply 后目标文件含各 patch 的 marker。
3. **幂等**：同一补丁二次 apply，`changed=false`。

### 第 7 步：更新文档 + 兼容矩阵 + 锚点测试

- `package.json` → `dsh.compatibility.dshReleases` 加新版本状态（`compatible` / `unknown`，**有证据才写 compatible**）。
- `AGENTS.md` §9 表格、`compatibility.md` 矩阵、`release-notes.md` 同步版本。
- `test/patch-anchors.test.mjs`：把每条补丁锚点**钉在真实源码片段**上——宿主再变形状，这个测试会先失败，等于"升级哨兵"。
- `docs/releases/<version>.md` 写发版说明（≤ 8 KiB）。
- `npm run check:docs` 校验版本一致。

---

## 3. 新增一条补丁的标准动作（`AGENTS.md` §7）

1. `targetFiles()` 登记目标文件键；
2. `replace` 里留 `[dsh-purge]` 标记并写进 `markers`；
3. 在 `test/patch-anchors.test.mjs` 钉一份真实源码片段；
4. 第三方插件目标用 `optional: true`（不在场时显示"不适用"而非失败）。

---

## 4. 判定口径速查（`compatibility.md` §4）

- `fileOptional: ["key"]`：该目标在新宿主已不承载该行为，**不参与 partial 判定**，但仍逐文件显示状态。加之前必须核对"真没这段逻辑"还是"只换写法"，**说不清就不加**。
- `partial`：只有 `fileAllRequired: true` 的多文件补丁才产生，含义"存在的一个目标文件没命中、另一个命中"。
- `drifted`：文件在、marker 不在、锚点全不中。不再软成功。
- `settleWhenAbsent` / `silentWhenMissing`：撤销型/可缺省补丁缺失时判 `skipped`，不报红。
- `chainedAfter`：级联补丁，前置未落地不算漂移。

---

## 5. 硬性红线（`AGENTS.md` §10）

1. 不自动重启 DSH；2. 不自动点「应用」；3. 不改 `prompt-inject.md`（除非用户保存）；4. 无要求不 commit/push/发版；5. 补丁号 0–99，历史版本保留原编号。

---

## 6. 本次 0.2.2-rc.3 适配结果（基线，供下次对照）

| 指标 | 值 |
|---|---|
| 补丁总数 | 86 |
| 官方 rc.3 隔离副本实测 | 74 applied / 零漂移；二次应用幂等 |
| 已知未修（rc.3） | #14/#17/#60/#62/#67/#74 部分旧授权补丁保持原定义（见 rc.1 诊断），#54/#82 说明为"已达成/官方已处理" |
| 验证 | Host Loader 预设注册通过；Windows rc.3 仅部分验证（不写"Windows 全量通过"） |

> 下次升级照第 2 节七步走：**先 diagnose 分类 → 逐条判"真漂移/迁址/已达成/结构消失/存疑" → 只对真漂移重新取锚点 → 语法零引入 + markers 落位 + 幂等 → 更新矩阵与锚点测试**。拿不准就保持原样并记录，绝不为消红硬凑。
