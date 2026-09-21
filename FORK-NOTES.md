# FORK NOTES — dsh-better-reasoning-effort

本文件记录**这个 fork 的存在理由与维护规则**，供后续维护者（包括 AI Agent）阅读。
它属于 fork 私有维护信息，**不要合并进上游**。

## 1. 这是什么

| 项 | 值 |
|---|---|
| fork 自 | `HaoyueQin/dsh-better-reasoning-effort` |
| 本 fork | `MUMU-DADA/dsh-better-reasoning-effort` |
| 本地克隆 | `C:\Users\mumu\source\dsh-better-reasoning-effort` |
| fork 的**唯一理由** | 上游 **PR #11**：让模态探测识别 `supports_images` / `supportsImages` |
| 补丁所在 | `master` 分支（也即 PR #11 的 head） |

补丁要解决的问题：CodeBuddy 系网关把逐模型的图片能力公布为 `supports_images`，
而插件原本只认 `supports_vision` / `supportsVision` / `architecture.input_modalities` 等写法。
于是这类端点被判为"沉默"，模态判断落回知识库，而命中的目录条目是纯文本的
（`deepseek-v4` 条目为 `input: ["text"]`），导致**实际支持图片的模型被声明为纯文本**，
图片附件被端到端拒绝。

## 2. ⚠️ `master` 必须保持干净

`master` 是 PR #11 的 head 分支。**不要把 fork 专属文件（包括本文件）提交到 `master`** ——
它们会出现在 PR 的 diff 中，把本机路径等信息暴露给上游。

本文件之所以位于 `fork-notes` 分支，正是为了避免这一点。

## 3. 更新插件时的正确顺序

**不要把这个 fork 当作普通 npm 包来升级。** 顺序如下：

### 3.1 先同步上游，再把补丁 rebase 上去

```powershell
cd C:\Users\mumu\source\dsh-better-reasoning-effort
git fetch upstream
git rebase upstream/master
```

fork 上正常情况下只有 PR #11 这一个提交，不会有冲突。
**若出现难以解决的冲突，停下来问用户，不要强推、不要 `--skip` 糊过去。**

### 3.2 必须重新构建

DSH 加载的是 `lib/`，而 `lib/` 被 `.gitignore` 排除、由 `prepare` 钩子（`npm run build`）生成：

```powershell
npm install                       # 仅当依赖有变化
npm run typecheck; npm test; npm run build
```

> **只改 `src/` 不跑 `npm run build` = 改动完全不生效**，DSH 读到的还是旧的 `lib/`。

### 3.3 两种安装方式对构建的要求不同

| 安装源 | 构建 |
|---|---|
| `github:MUMU-DADA/dsh-better-reasoning-effort#master` | 安装时由 `prepare` 自动构建；构建脚本受 pnpm `allowBuilds` 闸门约束，按安装器打印的键批准后重跑 `add` |
| `link:C:/Users/mumu/source/dsh-better-reasoning-effort` | **不自动构建**，改完源码必须手动执行 3.2 |

## 4. 上游合入 PR #11 之后：废弃本 fork

先查状态：

```powershell
gh pr view 11 --repo HaoyueQin/dsh-better-reasoning-effort --json state,mergedAt
```

**一旦 `state == "MERGED"`，本 fork 即失去存在价值，按此处理：**

1. 切回上游 npm 版插件：

   ```powershell
   dsh plugin --profile web rm dsh-better-reasoning-effort
   dsh plugin --profile web add dsh-better-reasoning-effort
   ```

2. 归档或删除本 fork；本地克隆目录 `C:\Users\mumu\source\dsh-better-reasoning-effort` 亦可删除。
3. 本文件随之作废——fork 只为这一个补丁而存在。

**若上游用别的方式实现了同样能力**（改了字段名、或换了实现思路），同样按废弃处理。

## 5. 补丁范围（便于 rebase 时判断是否已被上游吸收）

| 文件 | 改动 |
|---|---|
| `src/detection.ts` | 模态布尔标志链追加 `supports_images` / `supportsImages` 两个分支 |
| `tests/detection.spec.ts` | 两种拼写的读写、显式拒绝落 text 底线、列表形态优先于布尔标志 |
| `tests/knowledge.spec.ts` | 端到端：仅 `supports_images: true` 的列表融合为 `["text","image"]` 且 `inputSource === "endpoint"` |
| `README.md` / `README.zh.md` | 模态披露清单补上新字段 |

上游基线为 20 文件 / 390 测试；打完补丁后为 **394**。
若 rebase 后测试数不再是 394（或上游已自行处理该字段），说明补丁可能已被吸收或需要调整。
