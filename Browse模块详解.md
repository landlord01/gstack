# Gstack Browse 模块详解

## 1. 模块概述

**Browse 模块** 是 gstack 的核心，提供一个持久化的 headless Chromium 浏览器，可被 Claude Code 通过 CLI 命令控制。

- **代码量**：4,098 行 TypeScript
- **编译大小**：~58MB
- **性能**：首次调用 ~3s，后续调用 100-200ms
- **架构**：CLI 客户端 + HTTP 服务器 + Chromium daemon

---

## 2. 生命周期管理

### 2.1 初始化流程

```
第一次命令调用
    ↓
检查 .gstack/browse.json（状态文件）
    ↓ 文件不存在或 PID 已死亡
生成随机端口（10000-60000）
生成 UUID token
生成状态文件（权限 0o600）
启动 Bun HTTP 服务器
启动 headless Chromium
    ↓ ~3 秒
准备就绪，接收命令
```

### 2.2 后续命令流程

```
CLI 读取状态文件
    ↓
验证服务器还活着（检查 PID）
    ↓
构建 HTTP POST
发送命令 + 参数
验证 Bearer token
执行命令（Chromium + Playwright）
    ↓ ~100-200ms
返回纯文本结果
```

### 2.3 自动关闭

- **空闲超时**：30 分钟无命令后自动关闭
- **清理**：关闭 Chromium、删除状态文件
- **下一次调用**：自动重启

### 2.4 版本管理

**自动二进制重启机制：**

```
构建时写入版本号：git rev-parse HEAD → browse/dist/.version
每次 CLI 调用时比较
如果 .version 与服务器 binaryVersion 不匹配
    ↓
   杀死旧服务器
   启动新二进制
```

---

## 3. 核心架构组件

### 3.1 CLI 客户端 (`cli.ts`)

**职责**：
- 解析命令行参数
- 读取状态文件（`.gstack/browse.json`）
- 构建 HTTP 请求（包含 Bearer token）
- 发送到本地服务器
- 输出结果到 stdout

**关键特性**：
- 启动时间：~1ms（已编译二进制）
- 状态文件缺失或 PID 已死 → 启动新服务器
- 重试机制用于临时网络错误

### 3.2 HTTP 服务器 (`server.ts`)

**职责**：
- 运行 `Bun.serve()` HTTP 服务器
- 路由命令到 `READ_COMMANDS`、`WRITE_COMMANDS`、`META_COMMANDS`
- 调用 Playwright API
- 返回纯文本结果

**服务器路由：**

```
POST /command
├─ 验证 Bearer token
├─ 解析命令参数
├─ 分发到处理程序
│  ├── handleReadCommand()     // 无副作用的命令
│  ├── handleWriteCommand()    // 有副作用的命令
│  └── handleMetaCommand()     // 服务器管理命令
└─ 返回结果（plain/text）

GET /health
└─ 检查服务器活跃度

GET /cookie-picker
└─ Interactive Cookie Picker UI

POST /cookie-picker-routes
└─ Cookie 选择逻辑
```

**生命周期事件处理：**
- `browser.on('disconnected')`：Chromium 崩溃 → 服务器立即退出
- 无自我修复 - 让失败暴露

### 3.3 浏览器管理器 (`browser-manager.ts`)

**职责**：
- 启动/停止 Chromium
- 管理标签页
- 维护 ref 到 Locator 的映射
- 处理导航事件
- Ref 过期检测

**关键数据结构：**

```typescript
class BrowserManager {
  browser: Browser              // Playwright 浏览器实例
  pages: Map<string, Page>      // 标签页 ID → 页面对象
  refMap: Map<string, RefEntry> // @e1/@c1 → Locator + 元数据
  
  // RefEntry 结构
  interface RefEntry {
    ref: string           // "@e1" / "@c1"
    locator: Locator      // Playwright Locator
    role: string          // "button" / "link"
    name: string          // 元素文本
    selector: string      // CSS 选择器（仅用于 @c）
  }
}
```

---

## 4. Ref 系统（@e 和 @c）

### 4.1 @e 引用 - ARIA 元素

**生成流程：**

```
1. snapshot 命令调用
2. page.locator(scope).ariaSnapshot()
3. 返回 YAML 风格的可访问性树
4. Parser 遍历树
5. 按顺序分配 @e1, @e2, @e3...
6. 为每个元素构建 Locator
   getByRole('button', { name: 'Submit' }).nth(0) 
   → 存储为 refMap('@e1')
```

**关键优势：**
- ✅ 跨 CSP 工作（无 DOM 修改）
- ✅ 框架不可知（React/Vue/Angular）
- ✅ 支持 Shadow DOM
- ✅ 快速过期检测

**过期检测示例：**

```typescript
async function resolveRef(ref: string) {
  const entry = refMap.get(ref)
  if (!entry) throw `Ref ${ref} 不存在`
  
  // 关键：检查元素是否仍存在
  const count = await entry.locator.count()
  if (count === 0) {
    throw `Ref @e1 已过期 - 元素不再存在。运行 'snapshot' 获取新 refs。`
  }
  
  return entry  // ~5ms 开销
}
```

### 4.2 @c 引用 - 光标可交互元素

**用途**：捕获 ARIA 树中缺失但用户可点击的元素

**例子：**
- `<div cursor="pointer" onclick="...">` 
- `<div tabindex="0">`
- 自定义组件（框架渲染为 `<div>`）

**生成方式：**
- `-C` 标志启用光标交互检测
- `page.evaluate()` 扫描 DOM
- 用确定性 CSS 选择器存储（nth-child）
- 分配 @c1, @c2... 命名空间

### 4.3 Ref 生命周期

```
snapshot 命令
    ↓
分配 @e1-@eN, @c1-@cM
    ↓ 页面导航
framenavigated 事件
    ↓
清空 refMap（refs 已过期）
    ↓
需要新的 snapshot
```

**SPA 处理**：
- 路由变化（React Router）无法触发 `framenavigated`
- 相反：每条命令执行 `resolveRef()` 时检查 `locator.count()`
- 如果计数 = 0，立即失败并告诉用户重新运行 snapshot

---

## 5. 快照系统 (`snapshot.ts`)

### 5.1 基本快照

```bash
$B snapshot
```

**输出示例：**
```
┌─ @e1 heading 'Welcome'
│  └─ @e2 link 'Get Started'
├─ @e3 button 'Sign Up'
│  └─ @e4 button (nested)
└─ @e5 text 'Footer'
```

### 5.2 快照标志

| 标志 | 缩写 | 功能 |
|------|------|------|
| `--interactive` | `-i` | 仅显示可交互元素（可点击、可聚焦、可选择） |
| `--compact` | `-c` | 紧凑格式（无树结构） |
| `--depth N` | `-d` | 限制树深度 |
| `--selector` | `-s` | 从特定选择器开始 |
| `--diff` | `-D` | 与上一个快照对比 |
| `--annotate` | `-a` | 在屏幕上用标签覆盖 refs |
| `--output` | `-o` | 输出路径（用于注释） |
| `--cursor-interactive` | `-C` | 包含非 ARIA 的光标可交互元素 |

### 5.3 Diff 模式

```bash
# 第一次
$B snapshot -D

# 执行一些操作
$B click @e3

# 第二次
$B snapshot -D
```

**输出：**显示两个快照间的统一 diff

**用途**：验证操作确实起了作用

### 5.4 注释模式

```bash
$B snapshot -a -o screenshot.png
```

**流程：**
1. 获取每个 ref 的边界框
2. 在页面上注入临时 div
3. 用 @ref 标签标注它们
4. 截图
5. 移除注入的 divs
6. 保存到 `-o` 路径

---

## 6. 命令分类

### 6.1 READ 命令（无副作用）

| 命令 | 参数 | 功能 |
|------|------|------|
| `text [sel\|@ref]` | selector | 提取元素文本内容 |
| `html [sel\|@ref]` | selector | 提取 HTML |
| `links` | - | 列出所有链接及 href |
| `forms` | - | 列出表单字段 |
| `accessibility` | - | 完整可访问性树 |
| `attrs [sel\|@ref] [attr...]` | selector, attrs | 提取属性值 |
| `is [sel\|@ref] [state]` | selector, state | 检查元素状态 |
| `css [sel\|@ref]` | selector | 获取计算样式 |
| `js <code>` | JavaScript | 执行 JS 返回结果 |
| `eval <code>` | JavaScript | 评估表达式 |
| `console` | - | 读取 console 消息 |
| `network` | - | 读取网络请求 |
| `dialog` | - | 检查打开的对话 |
| `cookies` | - | 列出所有 cookies |
| `storage [key]` | key | 读取 localStorage/sessionStorage |
| `perf` | - | 性能指标 |

### 6.2 WRITE 命令（有副作用）

| 命令 | 参数 | 功能 |
|------|------|------|
| `goto <url>` | URL | 导航到 URL |
| `click <sel\|@ref>` | selector | 点击元素 |
| `fill <sel\|@ref> <text>` | selector, text | 填充输入框 |
| `select <sel\|@ref> <value>` | selector, value | 选择下拉选项 |
| `type <text>` | text | 键入文本（逐字符） |
| `press <key>` | key | 按键盘按键 |
| `scroll [sel\|@ref]` | selector | 滚动到元素 |
| `hover <sel\|@ref>` | selector | 鼠标悬停 |
| `wait <ms>` | milliseconds | 等待 |
| `viewport <width> <height>` | W, H | 设置视口大小 |
| `upload <sel\|@ref> <file>` | selector, path | 上传文件 |
| `back` | - | 返回 |
| `forward` | - | 前进 |
| `reload` | - | 重新加载 |
| `dialog-accept [text]` | text | 接受 alert/confirm |
| `dialog-dismiss` | - | 拒绝对话 |

### 6.3 META 命令

| 命令 | 功能 |
|------|------|
| `snapshot` | ARIA 树 + @refs |
| `screenshot [--clip x,y,w,h] [sel\|@ref] [path]` | 截图 |
| `pdf [path]` | PDF 导出 |
| `responsive` | 响应式设计检查 |
| `diff <url1> <url2>` | 对比两个 URL |
| `tabs` | 列出打开的标签页 |
| `tab <n>` | 切换到标签页 |
| `newtab [url]` | 打开新标签页 |
| `closetab [n]` | 关闭标签页 |
| `chain` | 从 stdin 读取 JSON 命令数组 |
| `help` | 显示所有命令 |

---

## 7. Cookie 管理

### 7.1 From File (`cookie-import`)

```bash
$B cookie-import <file.json>
```

**JSON 格式**：
```json
[
  {
    "name": "session_id",
    "value": "abc123",
    "domain": ".example.com",
    "path": "/",
    "secure": true,
    "httpOnly": true,
    "sameSite": "Strict"
  }
]
```

### 7.2 From Browser (`cookie-import-browser`)

```bash
$B cookie-import-browser
```

**工作流程：**
1. 枚举已安装的浏览器（Chrome、Arc、Brave、Edge、Comet）
2. 定位 Chromium Cookie 数据库
3. 请求 macOS Keychain 访问权限（用户确认必需）
4. 使用 PBKDF2 + AES-128-CBC 解密 cookies
5. 加载到 Playwright 上下文

**安全特性：**
- ✅ Keychain 访问需用户批准
- ✅ 解密在进程内存中发生
- ✅ 不修改原始浏览器 DB（读复制）
- ✅ 仅会话期间缓存密钥
- ✅ Cookie 值不出现在日志中

---

## 8. 日志系统

### 8.1 三个环形缓冲区

```typescript
class CircularBuffer<T> {
  buffer: T[] = new Array(50000)
  head: number = 0
  
  push(item: T): void {
    buffer[head] = item
    head = (head + 1) % 50000  // O(1)
  }
}
```

**三个缓冲区：**

| 缓冲区 | 内容 | 文件 |
|--------|------|------|
| Console | 日志消息、警告、错误 | `.gstack/console.log` |
| Network | 网络请求（URL、状态、计时） | `.gstack/network.log` |
| Dialog | alert/confirm 事件 | `.gstack/dialog.log` |

### 8.2 异步刷新

```
每 1 秒：
  ↓
仅追加新条目到磁盘（自上次刷新后）
  ↓
HTTP 响应处理不被阻塞
```

**可靠性**：
- ✅ 日志能存活 1 秒数据丢失
- ✅ 服务器崩溃时部分日志丢失
- ✅ 内存有界（50K × 3 = 150K 条目）

---

## 9. 屏幕截图模式

### 9.1 完整页面

```bash
$B screenshot
```

### 9.2 特定区域（裁剪）

```bash
$B screenshot --clip 100,50,400,300 output.png
```

### 9.3 特定元素

```bash
$B screenshot @e3 output.png
```

### 9.4 响应式检查

```bash
$B responsive
```

**输出**：
- 移动设备（375×667）
- 平板设备（768×1024）
- 桌面设备（1920×1080）

---

## 10. 错误处理

### 10.1 错误理念

**错误是给 AI agents 的，不是给人类的。** 每条错误消息必须可操作：

**✗ 坏：**
```
Element not found
```

**✓ 好：**
```
Element not found or not interactable. 
Run `snapshot -i` to see available elements.
```

### 10.2 常见错误

| 错误 | 原因 | 解决方案 |
|------|------|--------|
| `Ref @e3 is stale` | SPA 更新了 DOM | 运行 `snapshot` 获取新 refs |
| `Element not found` | 选择器无效 | 使用 `@refs` 代替原始选择器 |
| `Multiple elements matched` | 选择器不够具体 | 使用 `@refs` 或更精确的选择器 |
| `Navigation timeout` | 页面加载缓慢 | 增加超时或检查 URL |
| `Chromium crashed` | 浏览器进程崩溃 | 下一条命令自动重启服务器 |

---

## 11. 性能特性

### 11.1 基准线

| 操作 | 时间 |
|------|------|
| 首次启动 Chromium | ~3 秒 |
| HTTP 往返（无页面变化） | ~50-100ms |
| screenshot | ~100-200ms |
| 导航到新 URL | ~1-2 秒（取决于页面） |
| snapshot | ~50-100ms |
| 点击 + 等待导航 | ~1-2 秒 |

### 11.2 优化

- **CLI 启动**：已编译二进制 (~1ms vs ~100ms Node.js)
- **持久服务器**：避免 3s 启动开销
- **异步日志**：不阻塞 HTTP 响应
- **Ref 缓存**：避免重复的 ARIA 树遍历

---

## 12. 安全模型

### 12.1 Bearer Token

每个服务器会话生成一个 UUID token，存储在状态文件中（权限 0o600）。

```
请求头：Authorization: Bearer <uuid>
验证：token 与状态文件中的 token 匹配
```

### 12.2 Localhost 只

服务器仅在 `localhost` 上监听，不暴露于网络。

### 12.3 Cookie 安全

**解密流程：**
1. 从 Chrome 数据库复制（避免锁冲突）
2. 请求 Keychain 访问
3. 使用 PBKDF2 导出密钥
4. AES-128-CBC 解密
5. 加载到 Playwright 上下文
6. 服务器关闭时清空内存缓存

---

## 13. 链式命令

### 13.1 单行执行多个命令

```bash
$B chain <<'EOF'
[
  {"command": "goto", "url": "https://example.com"},
  {"command": "snapshot", "flags": ["-i"]},
  {"command": "click", "selector": "@e1"},
  {"command": "screenshot", "path": "result.png"}
]
EOF
```

**用途**：
- 减少网络往返
- 原子操作
- 提高性能

---

## 14. 开发和调试

### 14.1 开发模式

```bash
# 运行源 TS（热重载）
bun run dev

# 启动服务器
bun run server

# 测试命令
$B snapshot
$B goto https://example.com
```

### 14.2 日志检查

```bash
# 查看 console 日志
cat ~/.gstack/console.log

# 查看网络请求
cat ~/.gstack/network.log

# 查看对话事件
cat ~/.gstack/dialog.log
```

### 14.3 调试 Refs

```bash
# 查看所有 ARIA 元素
$B snapshot

# 查看包括光标可交互的元素
$B snapshot -C

# 仅限交互元素
$B snapshot -i

# 注释截图
$B snapshot -a -o debug.png
```

---

## 15. 限制和 Workarounds

| 限制 | 原因 | Workaround |
|------|------|-----------|
| 无 iframe 支持 | Ref 系统不跨边界 | 使用 JS eval 与 iframe 通信 |
| 无多进程 | 单一 Chromium 实例 | 使用 Conductor 获得多个工作区 |
| 仅 macOS Cookie 解密 | Keychain API 仅在 macOS | 手动导入 cookies JSON 或从 Linux/Windows 导出 |
| 30 分钟空闲超时 | 资源管理 | 保持活动或手动重启 |

---

## 总结

Browse 模块是 gstack 的心脏，提供：

1. **持久状态**：cookies、localStorage、登录会话在命令间保留
2. **亚秒级响应**：100-200ms 后续命令（vs 2-3s 如果每次启动浏览器）
3. **安全的 ref 系统**：无 CSS 选择器，无 CSP 冲突，快速过期检测
4. **完整的浏览器 API**：50+ 命令覆盖导航、交互、检查、可视化
5. **生产级安全**：localhost、token 认证、keychain 集成、加密 cookies

这使得 Claude Code 不仅仅是代码生成器，而是一个能看到、点击和验证网页的 QA 工程师。
