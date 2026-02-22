# Tauri IPC Commands

**Parent:** ../AGENTS.md

## OVERVIEW

IPC command handlers for frontend-backend communication. All commands registered in `lib.rs` invoke_handler.

## STRUCTURE

```
commands/
├── mod.rs           # Module declarations
├── chat.rs          # AI chat completion
├── mcp.rs           # MCP server management
├── setting.rs       # App configuration
├── window.rs        # Window lifecycle (emit events!)
├── workflow.rs      # Workflow/agent operations
├── note.rs          # Notes CRUD
├── model.rs         # AI model management
├── clipboard.rs     # System clipboard
├── env.rs           # Environment info
├── updater.rs       # App updates
└── types/           # Command-specific types
```

## WHERE TO LOOK

| Task | Location |
|------|----------|
| Add new command | Create file → add to `mod.rs` → register in `lib.rs` |
| Chat-related | `chat.rs` |
| MCP operations | `mcp.rs` |
| Settings/config | `setting.rs` |
| Window management | `window.rs` (use events!) |

## CONVENTIONS

### Command Signature
```rust
#[tauri::command]
async fn my_command(
    window: tauri::Window,
    main_store: State<'_, Arc<RwLock<MainStore>>>,
    param: String,
) -> Result<ReturnType, AppError> {
    // ...
}
```

### Error Handling
- Return `Result<T, AppError>` (defined in `src/error.rs`)
- Use `map_err(|e| AppError::Db(e))?` for database errors
- User messages via `t!` macro for i18n

### State Injection
- `State<'_, Arc<RwLock<MainStore>>>` — Database/config
- `State<'_, Arc<ChatState>>` — Chat/tool state
- `Window` — Current window reference
- `AppHandle` — App-wide operations

## ANTI-PATTERNS

| Pattern | Reason |
|---------|--------|
| Create window in command | Deadlock on Windows; emit event instead |
| Block in async command | Use `tokio::spawn` for long tasks |
| Return `String` errors | Use `AppError` for rich error context |

## EVENT-BASED WINDOW CREATION

Instead of creating windows directly:
```rust
// BAD: Direct creation
window.create(...) // Can deadlock on Windows

// GOOD: Emit event
window.emit("create-setting-window", ())?;
```
Handled in `src/window.rs` setup handlers.

## TESTING

```bash
cargo test --manifest-path src-tauri/Cargo.toml
```

Tests embedded in modules via `#[cfg(test)] mod tests { ... }`
