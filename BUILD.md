# 构建与排错手册

> 记录 note-gen 项目构建过程中遇到的所有坑、修法、和关键决策。后续开发/打包前先扫一遍本文档。

---

## 目录

1. [完整打包流程](#1-完整打包流程)
2. [常见错误与修法](#2-常见错误与修法)
3. [编辑器相关修复](#3-编辑器相关修复)
4. [macOS 签名 / 公证](#4-macos-签名--公证)

---

## 1. 完整打包流程

### 前置依赖

```bash
node --version    # ≥ 20
pnpm --version    # 任意现代版本
rustc --version   # ≥ 1.77
xcode-select -p   # macOS: 必须返回路径
```

### 标准流程

```bash
# 1. 装依赖
pnpm install

# 2. 同步版本号(改了 version 才需要)
pnpm sync-version

# 3. 完整构建
pnpm tauri build
```

这一步内部会按顺序执行:

| 阶段 | 命令 | 产物 |
|---|---|---|
| 前端构建 | `pnpm build`(在 `beforeBuildCommand` 触发) | `out/` |
| Rust 编译 | `cargo build --release` | `src-tauri/target/release/note-gen` |
| 打包 | `tauri bundle` | `.app` / `.dmg` |

### 产物位置

```
src-tauri/target/release/bundle/macos/NoteGen.app
src-tauri/target/release/bundle/dmg/NoteGen_<version>_<arch>.dmg
src-tauri/target/release/bundle/macos/NoteGen.app.tar.gz   # updater 用
```

### 平台变体

```bash
# 默认(当前架构)
pnpm tauri build

# 显式指定
pnpm tauri build --target aarch64-apple-darwin
pnpm tauri build --target x86_64-apple-darwin
pnpm tauri build --target universal-apple-darwin
```

### 仅构建前端(快速验证)

```bash
pnpm build                  # 输出到 out/
cd src-tauri && cargo build # Rust 单独编译(假设 out/ 已就绪)
```

### Debug 构建(更快,验证用)

```bash
pnpm tauri build --debug
# 或
cd src-tauri && cargo build
```

---

## 2. 常见错误与修法

### 2.1 `EADDRINUSE: address already in use 0.0.0.0:3456`

dev server 端口被占。

```bash
lsof -i :3456
# 看 PID 后 kill,或者一次性清掉所有残留
pkill -9 -f "next dev"
pkill -9 -f "tauri"
pkill -9 -f "transform.js"
```

### 2.2 `pnpm build` 卡在 ESLint 报错

Next.js 15 默认会在 `next build` 阶段跑 ESLint,任何 error 都会中断 build。**不要为了过 build 而删除"未使用"的代码** — 这些代码可能是留给未来功能用的接口,直接删会导致用户用不到。

正确做法是加 `eslint-disable` 注释保留代码:

```tsx
// 单行禁用
// eslint-disable-next-line @typescript-eslint/no-unused-vars
const myHelper = (x: string) => x

// 整段 import 禁用
/* eslint-disable @typescript-eslint/no-unused-vars */
import { GitCommit, Calendar, Layers, ... } from 'lucide-react'
/* eslint-enable @typescript-eslint/no-unused-vars */
```

> 📌 项目里 `slash-command/suggestion.tsx` 和 `mermaid-extension.tsx` 中的 `getLabel`、`createMermaidCommand`、7 个 lucide-react icon 都是这类"留作未来接口"的代码,已经加了 disable 注释。

如果想临时跳过 lint 跑 build:

```bash
NEXT_DISABLE_ESLINT=1 pnpm build
```

### 2.3 打包后打开 .app 显示 `Internal Server Error`

**根因**:`tauri.conf.json` 的 `bundle.macOS.signingIdentity: null` 会让产物是 ad-hoc 签名但**没有 entitlements / sealed resources**。macOS(尤其 Apple Silicon)的严格模式下,webview 加载 `tauri://localhost/index.html` 被 sandbox 拒绝,webview 报这个错。

**症状**:
- .app 进程在跑,WebKit 子进程也起来了
- `codesign -dv .app` 显示 `Sealed Resources=none`
- `spctl -a .app` 报 `code has no resources but signature indicates they must be present`

**修法**(已应用):修改 `src-tauri/tauri.conf.json`:

```json
"macOS": {
  "signingIdentity": "-",            // 原来是 null
  "entitlements": "entitlements.plist", // 原来是 null
  "minimumSystemVersion": "10.13",
  "exceptionDomain": null
}
```

并在 `src-tauri/entitlements.plist` 写入:

```xml
<?xml version="1.0" encoding="UTF-8"?>
<plist version="1.0">
<dict>
    <key>com.apple.security.cs.allow-jit</key>
    <true/>
    <key>com.apple.security.cs.allow-unsigned-executable-memory</key>
    <true/>
    <key>com.apple.security.cs.disable-library-validation</key>
    <true/>
    <key>com.apple.security.network.client</key>
    <true/>
    <key>com.apple.security.network.server</key>
    <true/>
    <key>com.apple.security.files.user-selected.read-write</key>
    <true/>
</dict>
</plist>
```

**临时手动修法**(不想重 build):

```bash
APP="src-tauri/target/release/bundle/macos/NoteGen.app"
codesign --force --sign - --entitlements src-tauri/entitlements.plist \
  --options runtime --timestamp=none --deep "$APP"
```

### 2.4 `beforeBuildCommand pnpm build failed with exit code 1`

`pnpm tauri build` 在跑 `beforeBuildCommand` 时就失败了,根本进不到 cargo 阶段。看 Next.js 报错的具体文件,通常是 ESLint 错(见 2.2)。

### 2.5 Rust 编译期 `A public key has been found, but no private key`

Tauri updater 签名问题,**不影响 .app 本体使用**,只在用 `tauri update` 自动更新时报错。

如果是开发/本地使用,忽略即可。

如果要发版,设置:

```bash
export TAURI_SIGNING_PRIVATE_KEY="<your-private-key>"
export TAURI_SIGNING_PRIVATE_KEY_PASSWORD="<password>"
```

密钥生成:`tauri signer generate -w ~/.tauri/myapp.key`

### 2.6 cargo 编译中途想取消,但留下 `src-tauri/target/` 损坏

```bash
cd src-tauri && cargo clean   # 全清,下次 build 从头编
```

不推荐 `rm -rf src-tauri/target/`,cargo 自己的清理逻辑能保留必要的 cache。

---

## 3. 编辑器相关修复

> 这些是代码层面的修复,在 commit `ef2b6e51` 中,不在 build 阶段。

### 3.1 首次进入笔记页面显示空白,需切换 tab 才显示内容

**症状**:打开 .app 或刷新后,编辑区域是白的(看不到任何 markdown),切换到其他笔记再切回来就正常。

**根因**(三处连锁 race):

1. `EditorLayout` 调 `initOpenTabs()`,`FileSidebar` 调 `initCollapsibleList()`,两个 useEffect **并行触发**。
2. 当 `initCollapsibleList` 先完成时,store 里的 `currentArticle` 已经设好,但 `MdEditor` 自己 state 的 `initialContent` 还是 `null`。
3. `MdEditor` 的渲染门槛 `showContent = (currentArticle && ...) || initialContent !== null` 在 `currentArticle` 已就位时**过早放行** TipTapEditor。
4. `TipTapEditor` 第一次 render 拿到的是 `initialContent || ''` = `''`,`useEditor({ content: '' })` 创建空文档;setContent useEffect 看到空内容,`isInitializedRef = true` 但**没调 setContent**。
5. 后续 `initialContent` 真的有内容传过来,setContent useEffect 重跑但被 `!isInitializedRef.current` 拦掉,**新内容被吞**。

**修法**(已应用,见 `md-editor-wrapper.tsx` + `tiptap-editor.tsx`):

- **Fix 1**:`md-editor-wrapper.tsx:181-196` — `currentArticle` 分支同时写 `tabContentsRef.current[filePath]`,切 tab 再回来能命中缓存。
- **Fix 2**:`md-editor-wrapper.tsx:391-395` — 渲染门槛从 `(currentArticle && ...) || initialContent !== null` 改成 `initialContent !== null`,强制等 MdEditor 自己的 initialContent 落地。
- **Fix 3**:`tiptap-editor.tsx:2911-2934` — setContent useEffect 里 `isInitializedRef = true` 移到 `if (initialContent)` 内部,空内容首发时不锁死,等真有内容再初始化。

### 3.2 其他编辑器问题(已修,供参考)

- `tabContentsRef` 在 `md-editor-wrapper.tsx` 跨 tab 共享,卸载时只清 `loadedPathsRef`,`tabContentsRef` 保留(正确)。
- `loadedPathsRef.current.add(filePath)` 在 `loadContent()` 之前同步加,防止 `currentArticle` 变化时 effect 重跑重复加载(正确,但前提是 initialContent 正确流转)。

---

## 4. macOS 签名 / 公证

### 当前状态:ad-hoc 签名

仓库配置的是 `"signingIdentity": "-"`,这会让 Tauri 用 `codesign -s -` 做 ad-hoc 签名。**仅限本机运行**。

### 正式发布要做的

1. 加入 Apple Developer Program($99/年)
2. 申请 Developer ID Application 证书
3. 在 `tauri.conf.json` 配:

```json
"macOS": {
  "signingIdentity": "Developer ID Application: Your Name (TEAMID)",
  "entitlements": "entitlements.plist"
}
```

4. 设置环境变量:

```bash
export APPLE_CERTIFICATE="<base64-p12>"
export APPLE_CERTIFICATE_PASSWORD="<password>"
export APPLE_ID="your@email.com"
export APPLE_PASSWORD="<app-specific-password>"
```

5. 跑 `pnpm tauri build`,会自动签名 + 上传到 Apple 公证。

### 不签名直接分发(不推荐)

只发给熟人测试可以,但 macOS 用户首次打开要右键 → 打开,绕过 Gatekeeper。每次重装都要重新授权。

---

## 附:常用诊断命令

```bash
# 看 .app 签名状态
codesign -dv --entitlements - path/to/NoteGen.app

# Gatekeeper 是否会拦截
spctl -a -t exec -vvv path/to/NoteGen.app

# 跑 .app 时的 webview 错误
log show --last 30s --predicate 'process == "note-gen" OR process == "com.apple.WebKit.WebContent"' --info

# 看 .app 加载的动态库
lsof -p $(pgrep -f NoteGen.app/Contents/MacOS/note-gen) | head -30

# 静态起一个文件服务验证 out/ 完整性
cd out && python3 -m http.server 8765
curl -I http://localhost:8765/index.html   # 应返回 200
```
