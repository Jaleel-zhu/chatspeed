# 智能体原子能力工具集开发计划 (Agent Atomic Tools Plan)

## 1. 目标
在 Rust 后端实现一套高性能、高安全性的智能体工具集，替代传统的基于 Node.js 的文件操作和 Shell 执行。通过 Rust 的静态类型、内存安全和 Tauri v2 的权限管理，确保智能体在操作用户系统时的可控性。

## 2. 核心原则
*   **Rust 优先**: 凡涉及 IO、多线程计算、进程管理的操作，全部下沉至 Rust 端。
*   **路径锚定 (Path Anchoring)**: 所有文件操作强制限制在用户授权的“工作区”目录内，防止智能体逃逸到系统关键目录。
*   **AST 审计 (AST Auditing)**: 对 Shell 命令进行语法树解析，而非简单的字符串过滤。
*   **最小权限 (Least Privilege)**: 利用 Tauri v2 的权限配置文件，精确授权每个工具的访问范围。

## 3. 任务分解

### 第一阶段：后端基础设施与安全层
1.  **权限控制模块**: 
    *   在 `src-tauri` 中建立工作区管理逻辑，获取并验证当前项目的根路径。
    *   配置 Tauri 的 `capabilities`，限制智能体对 FS 和 Shell 的原始访问。
2.  **安全审计引擎**:
    *   引入 `tree-sitter-bash` 解析器。
    *   实现命令提取逻辑，识别复合命令（如 `&&`, `|`, `;`）中的每一个子命令及其参数。

### 第二阶段：文件系统工具 (High Performance FS)
1.  **`safe_read_file`**: 支持大文件流式读取，自动检测编码。
2.  **`safe_write_file`**: 写入前备份（可选），提供 Diff 预览给用户确认。
3.  **`fast_search`**: 利用 Rust 的 `ignore` 库实现忽略 `.gitignore` 文件的极速搜索。
4.  **`list_directory`**: 递归列出目录结构，支持过滤策略。

### 第三阶段：安全 Shell 执行器 (Secure Shell)
1.  **命令解析器**: 能够将 `cd src && rm -rf .` 解析为 `chdir(src)` 和 `remove_dir_all(.)`。
2.  **黑名单与规则过滤**:
    *   禁止操作绝对路径（如 `/`, `/etc`, `C:\Windows`）。
    *   禁止高危命令（`rm`, `mkfs`, `dd` 等）在未授权路径下运行。
3.  **执行反馈**: 实时捕获 `stdout` 和 `stderr` 并流式传回前端。

### 第四阶段：前端集成与 UI 适配
1.  **ToolRegistry 注册**: 在 `src/pkg/workflow/tools/` 下创建 TS 包装层，通过 `invoke` 调用 Rust 命令。
2.  **审批 UI 强化**:
    *   当 Shell 审计标记为“中/高风险”时，前端弹出醒目的警告弹窗。
    *   展示命令解析后的“意图预览”（例如：此操作将删除 `dist` 目录）。

## 4. 技术栈
*   **解析器**: `tree-sitter`, `tree-sitter-bash` (用于 Shell 审计)
*   **文件处理**: `tokio::fs`, `ignore`, `walkdir`
*   **路径安全**: `path-clean`, `dunce` (处理符号链接安全)

## 5. 实施步骤建议
1.  **Step 1**: 实现 Rust 端的路径验证逻辑（确保所有操作在工作区内）。
2.  **Step 2**: 实现 `safe_read_file` 和 `list_directory`，建立基础 FS 信任。
3.  **Step 3**: 攻克 `tree-sitter-bash` 集成，实现 Shell 审计原型。
4.  **Step 4**: 完成 Shell 执行器并在自动模式中启用。

---
**当前状态**: 待细化 Step 1 的具体接口定义。
