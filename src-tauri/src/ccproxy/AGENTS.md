# CCProxy — AI Proxy Engine

**Parent:** ../AGENTS.md

## OVERVIEW

Multi-protocol AI proxy supporting OpenAI, Claude, Gemini, and Ollama. Tri-layer adapter architecture for seamless protocol conversion.

## STRUCTURE

```
ccproxy/
├── mod.rs              # Module hub, public re-exports
├── router.rs           # HTTP routing (CRITICAL: order matters)
├── auth.rs             # API key validation
├── errors.rs           # CCProxyError enum
├── adapter/            # Protocol conversion
│   ├── input/          # Client → Unified format
│   ├── backend/        # Unified → Provider API
│   └── output/         # Unified → Client format
├── handler/            # Request handlers
│   ├── chat_handler.rs
│   ├── embedding_handler.rs
│   └── list_models_handler.rs
├── helper/             # Shared utilities
│   ├── stream_processor.rs
│   ├── tool_use_xml.rs     # XML parsing for compat mode
│   └── proxy_rotator.rs    # Key rotation
└── types/              # Protocol-specific types
    ├── openai.rs
    ├── claude.rs
    ├── gemini.rs
    └── ollama.rs
```

## WHERE TO LOOK

| Task | Location |
|------|----------|
| Add new provider | `adapter/backend/` + `types/` |
| Fix routing | `router.rs` (NEVER change order) |
| Tool compat mode | `helper/tool_use_xml.rs` |
| Stream handling | `helper/stream_processor.rs` |
| Add endpoint | `router.rs` + `handler/` |

## DATA FLOW

```
Client Request
    ↓
Input Adapter (client protocol → UnifiedRequest)
    ↓
Backend Adapter (UnifiedRequest → provider API)
    ↓
AI Provider (OpenAI/Claude/Gemini/Ollama)
    ↓
Output Adapter (provider response → UnifiedResponse)
    ↓
Client Response
```

## CONVENTIONS

- **Protocol types**: Each provider has dedicated types file (e.g., `gemini.rs`)
- **Gemini**: All fields `camelCase` (`#[serde(rename_all = "camelCase")]`)
- **Index fields**: Use `i32` (some providers return `-1`)
- **Header filtering**: Always use `filter_proxy_headers` for responses

## ANTI-PATTERNS

| Pattern | Reason |
|---------|--------|
| Modify route order | Causes route shadowing, breaks clients |
| Forward raw headers | Must filter `Content-Length`, `Transfer-Encoding` |
| Use `u32` for index | Some providers return `-1` |

## KEY FEATURES

- **Direct forwarding**: Same protocol + no compat mode = pass-through
- **Tool Compat Mode**: Injects prompts + XML parsing for non-tool models
- **Key rotation**: Round-robin across provider keys
- **Group routing**: `/switch/` prefix for dynamic group selection
