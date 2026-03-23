# ACPX Harness 演進史

本文檔詳細記錄了 OpenClaw ACP（Agent Communication Protocol）外部執行後端橋接層（ACPX）的架構演進、痛點解決及發展歷程。

---

## ACPX Harness 是什麼

ACPX（ACP eXternal）Harness 是 OpenClaw ACP 的**外部執行後端橋接層**。它讓 OpenClaw gateway 能夠將 agent 任務**委派給外部 AI CLI 工具**執行，而非使用 OpenClaw 自身的內建 LLM runtime。

---

## 它解決了什麼痛點

### 問題背景

| 痛點 | 說明 |
|------|------|
| **工具生態割裂** | Claude Code、Codex CLI、Gemini CLI 各有獨立的工具呼叫能力、MCP 整合、permission model，無法在 OpenClaw 內直接使用 |
| **認證管理複雜** | 各 CLI 有自己的 OAuth / API key 管理，重複在 OpenClaw 實作成本高且易出錯 |
| **session 狀態不一致** | 外部 CLI 維護自己的對話歷史與 context window，強行整合會造成狀態不同步 |
| **能力上限不同** | Claude Code 有 computer use、file edit 等原生能力，OpenClaw 內建 runtime 無法完整複製 |

### ACPX 的解法

> 不重造輪子，直接橋接外部 CLI。

ACPX 透過標準化的 JSON-RPC over stdin/stdout 協議，讓 OpenClaw 像「指揮官」一樣派發任務給外部 CLI，由外部 CLI 負責實際執行與 AI 呼叫，結果再回傳至 OpenClaw session。

---

## 為何不可或缺

1. **多模型協作**：同一個 OpenClaw workflow 可同時調度 Claude、Codex、Gemini 等不同模型，各司其職。
2. **保留原生能力**：外部 CLI 的 tool use、MCP、permission model 完整保留。
3. **認證解耦**：OpenClaw 不需持有各 AI 廠商的 API key，由外部 CLI 自行管理。
4. **靜默 fallback 風險**：若 acpx 缺失，OpenClaw 會靜默降級至內建 runtime——這是 3.22 打包災難最嚴重的影響。

---

## 運作原理

```text
  使用者發送訊息
        │
        ▼
┌───────────────────────────────────────────────────────────────┐
│                    OpenClaw Gateway                           │
│                                                               │
│   session 設定 runtime: "acp", agent: "claude"               │
│                                                               │
│   ┌─────────────────────────────────────────────────────┐    │
│   │                  ACPX Harness                        │    │
│   │                                                      │    │
│   │  1. spawn  claude --experimental-acp                 │    │
│   │  2. 透過 stdin 送出 JSON-RPC 請求                    │    │
│   │  3. 從 stdout 讀取 JSON-RPC 回應                     │    │
│   │  4. 串流 token 回傳至 OpenClaw session               │    │
│   └──────────────────────┬───────────────────────────────┘    │
│                          │ stdin/stdout pipe                   │
└──────────────────────────│────────────────────────────────────┘
                           │
                           ▼
                    ┌─────────────────┐
                    │  外部 CLI 進程  │
                    │                 │
                    │  e.g. claude    │
                    │       codex     │
                    │       gemini    │
                    │       kiro      │
                    └────────┬────────┘
                             │
                             ▼
                      實際 AI 模型 API
