# Workflow 工具设计草稿

## 项目愿景
构建一个通用的自主智能体（Autonomous Agent），类似 OpenClaw 或 Claude Code 的核心能力：
- 支持多模型接入（通过 CCProxy）
- 多场景自动化（编程、PPT、文案、网页浏览、信息收集）
- 可扩展的工具系统

## 当前聚焦：信息收集场景

**决策**：先从信息收集场景切入，原因：
- WebSearch、WebFetch 工具已存在，基础已就绪
- 风险低、价值明确
- 可快速迭代验证工作流架构

**关键阻碍**：WebFetch 对知乎、CSDN 等内容型网站抓取成功率低

---

## WebFetch 深度技术分析

### 当前架构

```
WebFetch Tool
    ↓
engine::run() - src-tauri/src/scraper/engine.rs
    ↓
ScraperPool::scrape() - src-tauri/src/scraper/pool.rs
    ↓
WebviewScraper::scrape() - src-tauri/src/scraper/webview_wrapper.rs
    ↓
WebView 加载页面 → 注入 JS → 提取内容
```

### 技术细节

**1. WebView 池化管理** (`pool.rs`)
- 最大并发：10 个 WebView
- 复用策略：空闲 WebView 保留 5 分钟
- 清理机制：超过容量一半时关闭旧实例

**2. 页面加载流程** (`webview_wrapper.rs`)
```rust
1. 导航到目标 URL
2. 等待 page_loaded 事件 (最多 15s)
3. 等待 DOMContentLoaded 事件 (最多 30s)
4. 特殊处理：bing.com 额外等待 3s
5. 注入脚本 (最多重试 3 次)
6. 等待 scrape_result 事件 (最多 30s)
```

**3. 脚本注入** (`webview_wrapper.rs:302-342`)
```javascript
// 注入顺序：
1. utility.min.js      - 工具函数
2. turndown.min.js     - HTML → Markdown
3. readability.min.js  - 文章内容提取
4. scrape_logic.min.js - 核心提取逻辑
// 调用 window.performScrape(config, rule)
```

**4. 配置驱动** (`engine.rs`, `config_loader.rs`)
- 支持按 host 加载专用配置
- 通用内容规则：`format`, `keep_link`, `keep_image`

### 知乎/CSDN 抓取失败根因

| 问题 | 具体表现 | 技术原因 |
|------|----------|----------|
| **SPA 渲染延迟** | 内容未完全加载 | DOMContentLoaded 后仍有异步数据请求 |
| **登录墙** | 返回登录提示而非正文 | 需要有效 Cookie/Session |
| **反爬虫检测** | 返回验证码或封禁页面 | UA 检测、行为分析、IP 限制 |
| **动态内容** | 无限滚动、懒加载 | 初始 DOM 不包含全部内容 |
| **超时过短** | 复杂页面 15s 不够 | 大量 JS/CSS 资源加载 |

### 关键代码问题

**1. 超时设置过短** (`webview_wrapper.rs:207`)
```rust
let page_load_timeout = std::cmp::min(page_timeout, Duration::from_secs(15));
// 15s 对于知乎等内容密集型网站经常不够
```

**2. 图片阻塞未启用** (`webview_wrapper.rs:349`)
```rust
pub fn create_webview(&self, url: &str, visible: bool, _block_images: bool)
// block_images 参数被忽略，未实际减少资源加载
```

**3. 缺乏动态等待机制**
- 只有固定 3s 等待（针对 Bing）
- 没有针对具体元素的就绪检测
- 没有网络空闲检测

**4. 无 Cookie/Session 管理**
- WebView 每次新建，无状态保持
- 无法维持登录态

**5. User-Agent 不可配置**
- 使用系统默认 WebView UA
- 容易被识别为自动化工具

---

## WebFetch 改进计划

### Wave 1: 稳定性提升（2-3 天）

**1.1 自适应超时机制**
- [ ] 增加 per-host 超时配置
- [ ] 知乎/CSDN 默认 60s
- [ ] 普通页面保持 30s
- [ ] 可配置最大超时 120s

**1.2 启用图片阻塞**
- [ ] 修复 `_block_images` 参数
- [ ] 通过 MutationObserver 拦截图片加载
- [ ] 显著减少带宽和加载时间

**1.3 增强等待策略**
- [ ] 实现 `wait_for` 配置项（CSS Selector）
- [ ] 增加 `wait_for_network_idle` 选项
- [ ] 针对知乎：等待 `.RichContent` 或 `.Post-content`
- [ ] 针对 CSDN：等待 `#article_content`

**1.4 重试机制优化**
- [ ] 指数退避重试（1s → 2s → 4s）
- [ ] 区分临时错误（超时）和永久错误（404）
- [ ] 记录详细失败原因

### Wave 2: 反检测机制（2-3 天）

**2.1 User-Agent 管理**
- [ ] 支持 per-host UA 配置
- [ ] 提供常见浏览器 UA 列表
- [ ] 随机轮换机制（可选）

**2.2 Cookie 持久化**
- [ ] 在 MainStore 中存储 Cookie
- [ ] 支持手动导入 Cookie
- [ ] 针对知乎/CSDN 的登录态保持

**2.3 请求指纹混淆**
- [ ] 禁用 `navigator.webdriver` 标志
- [ ] 随机化部分浏览器指纹
- [ ] 模拟真实用户滚动行为（针对无限滚动）

### Wave 3: 智能降级（2-3 天）

**3.1 静态 HTML 回退**
- [ ] WebView 失败时尝试 HTTP 直接请求
- [ ] 使用 reqwest + html2text
- [ ] 适用于非 JS 依赖的内容页

**3.2 内容质量评分**
- [ ] 基于字数、结构完整性评分
- [ ] 低质量自动触发重试
- [ ] 返回质量指标给调用方

**3.3 诊断模式**
- [ ] 截图保存失败页面
- [ ] 输出 DOM 快照用于调试
- [ ] 详细的抓取过程日志

---

## 实施优先级

| 优先级 | 任务 | 影响 | 工作量 |
|--------|------|------|--------|
| P0 | 增加 per-host 超时配置 | 解决 60% 的抓取失败 | 0.5 天 |
| P0 | 启用图片阻塞 | 减少 50% 加载时间 | 0.5 天 |
| P1 | 实现 wait_for 机制 | 解决 SPA 渲染问题 | 1 天 |
| P1 | Cookie 持久化 | 解决登录墙问题 | 1 天 |
| P2 | UA 管理 | 降低反爬虫检测 | 0.5 天 |
| P2 | 静态 HTML 回退 | 提高整体成功率 | 1 天 |
| P3 | 诊断模式 | 便于问题排查 | 0.5 天 |

---

## 测试验证方案

**测试目标页面**：
1. 知乎专栏文章：`https://zhuanlan.zhihu.com/p/xxx`
2. 知乎问答：`https://www.zhihu.com/question/xxx/answer/xxx`
3. CSDN 博客：`https://blog.csdn.net/xxx/article/details/xxx`

**验证指标**：
- 成功率 > 90%
- 内容完整性（字数、格式）
- 抓取时间 < 30s（优化后）

**测试流程**：
1. 收集 20+ 个测试 URL（不同作者、不同内容类型）
2. 记录当前失败率和错误类型
3. 实施改进后对比
4. 持续监控生产环境成功率

---

## 后续扩展

WebFetch 稳定后，信息收集场景的完整工具链：

| 工具 | 状态 | 说明 |
|------|------|------|
| WebSearch | ✅ 已有 | 搜索聚合 |
| WebFetch | 🔄 优化中 | 内容提取 |
| **DeepFetch** | 📋 待规划 | 深度链接抓取（递归） |
| **ContentAnalyze** | 📋 待规划 | 内容摘要/关键信息提取 |
| **SearchMonitor** | 📋 待规划 | 定期搜索监控变化 |

---

## 下一步行动

1. **确认 Wave 1 优先级**：是否先专注超时和图片阻塞？
2. **提供测试 URL**：能否提供 5-10 个你常用的知乎/CSDN 链接用于测试？
3. **确认 Cookie 策略**：是否需要支持手动登录后导出 Cookie？
