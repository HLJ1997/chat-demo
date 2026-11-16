# 关于本仓库（维护分支说明）

这是 [`deepseek-ai/chat-demo`](https://github.com/deepseek-ai/chat-demo) 的个人维护 fork，作为 **Android 壳工程的前端主线**使用。

配套的安卓壳工程：[`HLJ1997/chat-demo-android-shell`](https://github.com/HLJ1997/chat-demo-android-shell)

## 与上游的差异

当前 `main` 相对上游 `deepseek-ai/chat-demo@5d5dd75` 只有一个提交：

| commit | 内容 | 原因 |
| --- | --- | --- |
| [`f9c1b38`](../../commit/f9c1b38) | 用 `[...arr].reverse()` / `[...arr].sort()` 替换 `toReversed()` / `toSorted()` | 这两个 ES2023 方法需要 Chrome 110+，但构建目标是 `chrome >= 105` |

### 为什么要改

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
- 对比链接：[`main...f9c1b38`](https://github.com/deepseek-ai/chat-demo/compare/main...HLJ1997:f9c1b38)

若上游将来开启 PR，可直接用 [此分支比较](https://github.com/deepseek-ai/chat-demo/compare/main...HLJ1997:fix/es2023-array-methods) 提交。

## 同步上游更新

```sh
git remote add upstream https://github.com/deepseek-ai/chat-demo.git   # 首次
git fetch upstream
git checkout main
git merge upstream/main        # 冲突通常只在被改动的两行
pnpm install && pnpm build     # 验证构建
git push origin main
```

合并后记得核对补丁是否仍然存在——如果上游自己修了这两处，本 fork 的差异就可以丢弃（见下）。

## 何时可以废弃这个 fork

上游满足以下任一条件时，本 fork 不再需要：

1. 把 `rspack.config.ts` 的 `targets` 提高到 `chrome >= 110`；
2. 或加了相应 polyfill；
3. 或自行做了相同的两行替换。

## 构建

与上游一致（注意 Windows 下 `pnpm build` 的脚本依赖 Unix shell，见壳仓库 README 的说明）：

```sh
pnpm install
pnpm dev      # http://localhost:8090
pnpm build    # 产物在 dist/
```
