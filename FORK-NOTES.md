# 关于本仓库（维护分支说明）

这是 [`deepseek-ai/chat-demo`](https://github.com/deepseek-ai/chat-demo) 的个人维护 fork，作为 **Android 壳工程的前端主线**使用。

配套的安卓壳工程：[`HLJ1997/chat-demo-android-shell`](https://github.com/HLJ1997/chat-demo-android-shell)

## 与上游的差异

当前 `main` 相对上游 `deepseek-ai/chat-demo@5d5dd75` 有 5 个提交，分两类：

### 一、兼容性修复（建议回馈上游）

| commit | 内容 |
| --- | --- |
| [`f9c1b38`](../../commit/f9c1b38) | 用 `[...arr].reverse()` / `[...arr].sort()` 替换 `toReversed()` / `toSorted()` |

原因：这两个 ES2023 方法需要 Chrome 110+，但 `rspack.config.ts` 的构建目标是 `chrome >= 105`，存在 105–109 的版本缺口。

### 二、自有功能（本 fork 独有）

| commit | 内容 |
| --- | --- |
| [`1234750`](../../commit/1234750) | 设置中支持切换对话模型（`deepseek-flash` / `deepseek-v4-pro`） |
| [`37bb008`](../../commit/37bb008) | 回复下方显示生成该回复所用的模型（Flash / V4 Pro） |
| [`a34b07e`](../../commit/a34b07e) | 会话级模型记忆：每个会话记住自己的模型，切换只影响当前会话 |
| [`9cac56e`](../../commit/9cac56e) | 旧会话首次发送时固化模型，不再跟随默认值漂移 |

改动点（`1234750` 模型切换）：

- `services/settings.ts`：新增 `model` 设置，localStorage 持久化，默认 `deepseek-flash`，非法值回退默认
- `services/chat/store.ts`：用 `currentModel()` 读取当前设置替代写死的 `MODEL` 常量；新会话、重新生成、请求三处生效，已完成的回复保留其生成时的模型
- `components/settings/SettingsDialog.tsx`：「通用设置」新增模型选择行
- `i18n/{zh,en}.ts`：中英文案

改动点（`37bb008` 回复标记）：

- `components/messages/AssistantMessage.tsx`：底部操作行（分支切换器之后）显示模型短名，带 tooltip
- `components/messages/AssistantMessage.module.css`：`.modelTag` 用 `--dsw-alias-label-secondary`，随明暗主题变化
- `i18n/{zh,en}.ts`：短名 `Flash` / `V4 Pro` 与 tooltip 文案

改动点（`a34b07e` 会话级模型记忆）：

- `services/chat/types.ts`：`Session` 增加可选 `model`
- `services/chat/store.ts`：`sessionModel(session)` 取会话模型、缺省回退设置默认；新建会话写入默认模型；`createReply(model)` 由调用方传入（发送/重新生成/编辑三处）；新增 `setSessionModel`，生成中或删除中拒绝切换
- `components/chatInput/ChatInput.tsx`：工具条新增会话模型选择器；`sessionId` 为 null（新对话）时改的是设置默认模型
- 设置里的模型项改为「默认模型」，说明为「新对话默认使用该模型；已有对话可在输入框下方单独切换」

### 已知局限

1. 导入的历史对话**不保留模型字段**（归档格式本身不含，`toLocal.ts` 会置为 `null`），因此这类回复不显示模型标记。这是导入功能的既有设计，不是本 fork 引入的。
2. 未记录 `model` 的会话（旧会话、导入会话）在**首次发送时固化**：发送前它跟随默认模型，发送时把当时的模型写回会话，之后不再随默认值变化。若在下一次发送前修改了默认模型，该会话会用修改后的值。
3. 窄屏（约 393px）下工具条可用宽度 212px，三个控件带文字需要 255px，因此「深度思考 / 智能搜索」会按项目既有机制折叠为图标（tooltip 与 `aria-label` 保留，模型选择器仍显示文字）。

真机验证：模型切换后请求体 `model` 由 `deepseek-flash` 变为 `deepseek-v4-pro`，服务端正常返回；刷新后保持，切回亦恢复默认。回复标记在 `deepseek-flash` 回复上显示「Flash」、`deepseek-v4-pro` 上显示「V4 Pro」，`model: null` 的导入会话无标记。

会话级记忆验证：A 会话切到 V4 Pro 后，全局默认仍为 flash、B/C 会话仍为 Flash、重开 A 仍为 V4 Pro；刷新后 `session.model` 已落库（IndexedDB），请求体为 `deepseek-v4-pro`；新对话跟随默认模型。

### 兼容性修复的来龙去脉

在 Android WebView **105–109** 上（实测设备：Xiaomi M2007J1SC / Android 13 / WebView Chrome 108）：

- 发送任意文本消息 → 界面显示「上传失败」
- 导入历史对话 ZIP → 失败

根因不是网络也不是 CORS，而是 `store.ts` 的 `prepareFiles()` 调用了 `.toReversed()`：

```ts
let preparing = true                        // store.ts:980
const prepared = await prepareFiles(...)    // 983
preparing = false                           // 984
} catch (error) {
  if (preparing) {                          // 1041
    kind: 'upload', localMessage: describeUploadError(error)  // → 「上传失败」
```

`TypeError` 被归类成 upload 错误，所以纯文本消息也显示「上传失败」——提示会把人误导到附件问题上。

完整分析与证据见壳仓库的 [`UPSTREAM-BUG.md`](https://github.com/HLJ1997/chat-demo-android-shell/blob/main/UPSTREAM-BUG.md)。

### 为什么没提给上游

上游仓库关闭了 Pull Requests 与 Issues（`has_pull_requests: false`、`has_issues: false`），任何人都无法提交 PR。因此改为在本 fork 上维护：

- 补丁文件：[`0001-fix-es2023-array-methods.patch`](https://github.com/HLJ1997/chat-demo-android-shell/blob/main/0001-fix-es2023-array-methods.patch)
- 兼容性修复的对比：[`5d5dd75...f9c1b38`](https://github.com/deepseek-ai/chat-demo/compare/main...HLJ1997:f9c1b38)

若上游将来开启 PR，可只提交兼容性修复那一个提交（`fix/es2023-array-methods` 分支仍保留），自有功能不必上游化。

## 同步上游更新

```sh
git remote add upstream https://github.com/deepseek-ai/chat-demo.git   # 首次
git fetch upstream
git checkout main
git merge upstream/main        # 冲突通常只出现在兼容性修复那两行
pnpm install && pnpm type && pnpm lint
pnpm build                     # 验证构建
git push origin main
```

合并后核对两件事：

1. 兼容性修复是否仍需要——若上游已自行修复（提高 target、加 polyfill 或同样替换），把 `f9c1b38` 的改动丢掉即可；
2. 自有功能是否被上游的接口变化打破——模型切换依赖 `CreateMessageRequest.model` 与 `ChatModel` 类型，若上游改动这些字段需要同步调整。

## 何时可以废弃这个 fork

分两部分看：

| 部分 | 废弃条件 |
| --- | --- |
| 兼容性修复 | 上游把 `targets` 提到 `chrome >= 110`、或加了 polyfill、或自行做了同样的替换 |
| 自有功能（模型切换） | 上游自己支持了模型选择，且交互符合需要 |

两者都满足时，本 fork 可回归纯上游；否则继续以 fork 为主。上游目前关闭了 PR/Issue，短期内无法回馈，因此兼容性修复会一直保留在本 fork。

## 构建

与上游一致（注意 Windows 下 `pnpm build` 的脚本依赖 Unix shell，见壳仓库 README 的说明）：

```sh
pnpm install
pnpm dev      # http://localhost:8090
pnpm build    # 产物在 dist/
```
