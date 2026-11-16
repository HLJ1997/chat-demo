# 关于本仓库（维护分支说明）

这是 [`deepseek-ai/chat-demo`](https://github.com/deepseek-ai/chat-demo) 的个人维护 fork，作为 **Android 壳工程的前端主线**使用。

配套的安卓壳工程：[`HLJ1997/chat-demo-android-shell`](https://github.com/HLJ1997/chat-demo-android-shell)

## 与上游的差异

当前 `main` 相对上游 `deepseek-ai/chat-demo@5d5dd75` 有 2 个提交，分两类：

### 一、兼容性修复（建议回馈上游）

| commit | 内容 |
| --- | --- |
| [`f9c1b38`](../../commit/f9c1b38) | 用 `[...arr].reverse()` / `[...arr].sort()` 替换 `toReversed()` / `toSorted()` |

原因：这两个 ES2023 方法需要 Chrome 110+，但 `rspack.config.ts` 的构建目标是 `chrome >= 105`，存在 105–109 的版本缺口。

### 二、自有功能（本 fork 独有）

| commit | 内容 |
| --- | --- |
| [`1234750`](../../commit/1234750) | 设置中支持切换对话模型（`deepseek-flash` / `deepseek-v4-pro`） |

改动点：

- `services/settings.ts`：新增 `model` 设置，localStorage 持久化，默认 `deepseek-flash`，非法值回退默认
- `services/chat/store.ts`：用 `currentModel()` 读取当前设置替代写死的 `MODEL` 常量；新会话、重新生成、请求三处生效，已完成的回复保留其生成时的模型
- `components/settings/SettingsDialog.tsx`：「通用设置」新增模型选择行
- `i18n/{zh,en}.ts`：中英文案

真机验证：切换后请求体 `model` 由 `deepseek-flash` 变为 `deepseek-v4-pro`，服务端正常返回；刷新后保持，切回亦恢复默认。

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
