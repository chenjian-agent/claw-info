# 同一 Provider 的 Auth Profiles 輪換（Rotation / Failover）

本文件說明：當你在 OpenClaw 針對**同一個 model provider**（例如 `openai-codex`、`anthropic`、`google`）配置了**多個 auth profiles** 時，OpenClaw 在執行時如何選擇、固定、以及在失敗時如何輪換到下一個 profile。

> TL;DR
>
> - OpenClaw 會先在**同一 provider 內輪換 profiles**，都失敗才會做 **model fallback**。
> - 同一個 session 具有**黏性（stickiness）**：不會每個 request 都輪替。
> - 你可以用 `auth.order[provider]` 或（若 UI 支援）`/model …@<profileId>` 來「釘住」特定 profile。

---

## 名詞速覽

- **Provider**：模型供應商/驅動，例如 `openai-codex`、`anthropic`。
- **Auth profile**：某 provider 的一組憑證（OAuth 或 API key）。
- **Profile ID**：OpenClaw 用來識別 profile 的字串，例如 `openai-codex:default` 或 `openai-codex:you@example.com`（取決於 provider / OAuth 是否能取得 email）。
- **Rotation**：同一 provider 內，當某個 profile 失敗後，嘗試下一個 profile。
- **Fallback**：當某 provider 的 profiles 都不可用時，轉向 `agents.defaults.model.fallbacks` 的下一個模型。

---

## Profiles 存放在哪裡？

- **密鑰/OAuth token**：存放於（按 agent）
  - `~/.openclaw/agents/<agentId>/agent/auth-profiles.json`

> 注意：多 agent 可能有多份 `auth-profiles.json`。主 agent 的憑證不會自動共享給其他 agent。

---

## Profile 選擇順序（Order）

當同一 provider 有多個 profiles 時，OpenClaw 的選擇邏輯（由高到低優先）可概括為：

1. **顯式指定**：`auth.order[provider]`（如果你有設定）
2. **已配置的 profiles**：`auth.profiles` 中屬於該 provider 的 profiles
3. **已存儲的 profiles**：`auth-profiles.json` 中該 provider 的條目

若未指定順序，OpenClaw 會使用一種「輪詢 + 健康狀態」的策略（大意）：

- OAuth profiles 通常優先於 API key
- 優先選擇「較久沒用」的 profile（分散使用量）
- **冷卻/禁用**的 profile 會被放到後面

---

## Session 黏性（Stickiness）：為何不會一直自動切換？

OpenClaw 為了提高快取命中與避免不必要的抖動，會對每個 session **固定使用某一個 profile**；通常不會在每一次呼叫時輪換。

它會在以下情況才可能改選下一個 profile：

- 你開始新 session（例如 `/new`、`/reset`）
- session 壓縮完成（compression 計數增加）
- 當前 profile 進入冷卻（cooldown）或被禁用（disabled）
- 當前 profile 發生錯誤（例如 rate limit / timeout / auth error）而觸發 failover

---

## 什麼錯誤會觸發輪換？

通常包含：

- **Rate limit**（速率限制）
- **Timeout / 類似速率限制的超時**
- **Authentication errors**（憑證失效/過期）

當某個 profile 被判定失敗，OpenClaw 會將其標記為冷卻一段時間，並嘗試下一個 profile。

---

## 冷卻（Cooldown）與禁用（Disabled）

- **Cooldown（短期退避）**：常見於暫時性的錯誤（rate limit/timeout）。通常採用指數退避（例如 1m → 5m → 25m → 1h）。
- **Disabled（較長退避）**：常見於「額度/計費」類錯誤（例如 credits 不足）。因為可能不是暫時性，會給更長的禁用時間（例如數小時到 24h）。

這些狀態通常會記錄回 `auth-profiles.json` 的 `usageStats` 區塊。

---

## 如何新增同一 Provider 的另一個 Profile（用 CLI）

最直接的方法是**再跑一次該 provider 的登入/授權流程**，OpenClaw 會把新的憑證寫入同一份 `auth-profiles.json`，並以新的 `profileId`（常見形式：`provider:default` 或 `provider:<email>`）保存。

### OAuth 類（例如 `openai-codex`）

```bash
# 重新跑一次 OAuth 流程以新增另一個帳號（會新增一個新的 profile）
openclaw models auth login --provider openai-codex

# 或走 onboarding 向導
openclaw onboard --auth-choice openai-codex
```

提示：如果 provider 能取得 email，通常會產生 `openai-codex:you@example.com` 這類 ID；否則可能仍是 `openai-codex:default`。

### API key 類（視 provider 支援）

有些 provider 的 `models auth login` 會引導你貼上 API key；也可使用：

```bash
openclaw models auth add
```

> 部分 provider/流程支援用 `--profile-id <provider:xxx>` 明確指定要建立的 profile ID。若你想固定命名（例如 `anthropic:work` / `anthropic:personal`），建議查看該 provider 文件或 `openclaw models auth login --help`。

新增完成後，建議用 `openclaw models status` 檢查 profiles 是否就位。

---

## 如何強制使用特定 Profile？（避免自動輪換）

### 方式 A：設定 `auth.order[provider]`

如果你希望某 provider 永遠先用某個 profile（甚至只用它），請在 gateway config 設定：

```jsonc
{
  "auth": {
    "order": {
      "openai-codex": ["openai-codex:default"]
    }
  }
}
```

### 方式 B：每個 session 指定 profile（若你的介面支援）

某些聊天介面支援以指令形式覆蓋模型與 profile，例如：

- `/model openai-codex/gpt-5.2@openai-codex:default`

這會把該 session 釘在指定 profile 上，直到你開始新的 session。

---

## 排查建議

- 查看目前使用哪個模型/哪個 profile：
  - `openclaw models status`

- 若遇到「看起來 OAuth profile 消失/沒被用到」：
  - 檢查是否因為 session 黏性導致你仍在沿用舊 profile
  - 檢查是否有設定 `auth.order` 把它排在後面
  - 檢查 `auth-profiles.json` 中該 profile 是否在 cooldown/disabled

---

## 參考

- OpenClaw docs（概念）：Model failover / auth profiles rotation（上游文件）

