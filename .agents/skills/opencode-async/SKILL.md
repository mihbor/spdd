---
name: opencode-async
description: Interact with opencode's headless server API asynchronously using PowerShell Invoke-RestMethod.
---

# Interacting with Opencode Asynchronously

Use this skill when you need to interact with opencode's headless server API asynchronously — sending prompts and collecting responses without blocking.

## TRIGGER

- The user asks to interact with opencode asynchronously, delegate a task to it, or use its headless server API.
- The user mentions `opencode serve`, `prompt_async`, or polling opencode for results.
- A session needs to offload work to opencode and collect the answer later.

## DO NOT TRIGGER

- When the user wants to use opencode interactively (GUI or CLI) — just let them do that directly.
- When the task is about configuring opencode models or credentials — that's a separate concern.

## The one rule

**Always use PowerShell's `Invoke-RestMethod`. Never use `curl.exe`.**

That is the single requirement. Every error, crash, and confusing log message we encountered was caused by using `curl.exe`. Switching to `Invoke-RestMethod` made everything work immediately.

## The correct workflow

### 1. Start the server

```powershell
opencode serve --port 3333
```

Port is configurable — use any free port. Check first:

```powershell
netstat -ano | Select-String ":3333"
```

If occupied, either `Stop-Process -Id <PID>` or pick another port (e.g. `--port 3580`). Update all subsequent URIs to match.

### 2. Create a session

```powershell
$session = Invoke-RestMethod -Uri "http://127.0.0.1:3333/session" -Method Post -Body '{}' -ContentType "application/json"
$sessionID = $session.id
```

### 3. Send async message

```powershell
$msgBody = @{ parts = @( @{ type = "text"; text = "Your question here" } ) } | ConvertTo-Json -Compress
Invoke-RestMethod -Uri "http://127.0.0.1:3333/session/$sessionID/prompt_async" -Method Post -Body $msgBody -ContentType "application/json" | Out-Null
```

Returns 204 No Content immediately. The server processes in the background.

### 4. Wait & poll for response

```powershell
Start-Sleep -Seconds 10
$messages = Invoke-RestMethod -Uri "http://127.0.0.1:3333/session/$sessionID/message" -Method Get
# Result is an array of messages. Iterate to find assistant text parts:
foreach ($msg in $messages) {
    if ($msg.info.role -eq "assistant") {
        foreach ($part in $msg.parts) {
            if ($part.type -eq "text") {
                Write-Host $part.text
            }
        }
    }
}
```

Adjust sleep time based on prompt complexity. For longer tasks, poll in a loop checking for new messages.

## Why not curl.exe?

`curl.exe` on Windows encodes request bodies with CLIXML metadata that opencode's server cannot parse. The result is `UnknownError` responses and `SyntaxError: JSON Parse error` in the server logs.

**The critical lesson:** these errors look like server bugs, config problems, or credential issues. They are not. They are curl's encoding. The moment we switched to `Invoke-RestMethod`, the same server, same config, same credentials — everything worked.

Do not waste time diagnosing the JSON parse error, checking credentials, or blaming config settings. The fix is to use `Invoke-RestMethod`.

## What we got wrong (and why)

During extensive experimentation, we chased several red herrings because the symptoms (JSON parse errors) pointed away from the real cause (curl):

- **`small_model` config** — we thought it was being ignored, but the API worked fine with `Invoke-RestMethod` without any config changes.
- **Insufficient account funds** — the logs showed this error for `gpt-5.4-nano` (the session titling agent). It was a real error but not the cause of JSON parse failures — the main model requests still completed while funds were available.
- **JSON parse bug in opencode** — the parse error was curl's CLIXML encoding, not a bug in opencode's parser.
- **Headless server not respecting `opencode.jsonc`** — we assumed config wasn't applied, but the server resolved models correctly once curl was removed.

**The only problem was curl.** Everything else was a misdiagnosis caused by the wrong tool producing confusing symptoms.

## API reference

| Action | Method | Endpoint | Notes |
|--------|--------|----------|-------|
| Create session | POST | `/session` | Body: `{}` |
| Send async | POST | `/session/{id}/prompt_async` | Body: `{ "parts": [{ "type": "text", "text": "..." }] }` |
| Get messages | GET | `/session/{id}/message` | Returns array: `[user, assistant, ...]` |
| List sessions | GET | `/session` | Includes model config per session |
| Switch model | POST | `/api/session/{id}/model` | Body: `{ "model": { "id": "...", "providerID": "..." } }`, returns 204 |
| SSE events | GET | `/session/{id}/event` | **Broken** in v1.14.48+ — known regression |

**Note:** The model switch endpoint is the only one with an `/api/` prefix. The model `id` field is the model name only (e.g., `"muse-spark-1.3-contributor"`), not prefixed with the provider ID.

## Permission prompts will kill your session

When opencode needs to access paths outside the workspace (e.g., `C:\Users\MBor\AppData\Local\Temp\*`), it pauses and asks for permission in the **GUI**. If you're using the API headlessly, **the prompt never reaches you** — the session sits idle until someone approves it in the GUI.

If the server restarts while the session is paused, the session is lost.

**Fix: Start the server with auto-approve for all actions:**

```powershell
opencode serve --port 3333 --permission=*
```

Or set it globally in `~/.config/opencode/opencode.json`:

```json
{
  "permission": "*"
}
```

**Important:** `--permission=*` is required when using the API headlessly. Without a GUI, permission prompts have nowhere to appear — the session will silently pause and eventually die when the server restarts.

**If you notice a session stuck** (output tokens not growing after several minutes), check the server logs for `permission=` entries — the session is likely waiting for a GUI prompt you can't see.

## Troubleshooting

### Any error at all

1. **Did you use `Invoke-RestMethod`?** If you used `curl.exe`, switch to `Invoke-RestMethod`. This fixes everything.
2. If still using `Invoke-RestMethod` and it fails, then check server logs at `~\.local\share\opencode\log\opencode.log`.
3. **Session stuck?** Check logs for `permission=` prompts — the session may be waiting for GUI approval you never receive.

### Server won't start / port in use

```powershell
netstat -ano | Select-String ":3333"
Stop-Process -Id <PID>
```

Or use a different port: `opencode serve --port 3580` (remember to update URIs).

## Reusing existing sessions

When you have an existing session with large context already loaded (e.g., 284k+ input tokens), reuse it rather than creating a new one. The model retains all previously read files and conversation history.

**Find an active session:**

```powershell
$sessions = Invoke-RestMethod -Uri "http://127.0.0.1:3333/session" -Method Get
foreach ($s in $sessions) {
    Write-Host "$($s.id) | $($s.slug) | model=$($s.model.id) | tokens=$($s.tokens.input)/$($s.tokens.output)"
}
```

**Switch the model mid-session** (if needed):

```powershell
$sessionID = "ses_f0..."
$body = @{ model = @{ id = "muse-spark-1.3-contributor"; providerID = "opencode-go" } } | ConvertTo-Json -Compress
Invoke-RestMethod -Uri "http://127.0.0.1:3333/api/session/$sessionID/model" -Method Post -Body $body -ContentType "application/json" | Out-Null
```

Then send new prompts to the existing session — it will use the new model for subsequent messages while retaining all prior context.

## Key takeaways

1. **Use `Invoke-RestMethod`, never `curl.exe`** — this is the only thing that matters.
2. **JSON parse errors in logs mean curl** — do not diagnose them as config, credential, or server bugs.
3. **SSE streaming is broken** in recent versions — async + polling is the reliable approach.
4. **Reuse sessions with loaded context** — a session with 284k+ input tokens is worth keeping; switch models via API if needed.
5. **Permission prompts are invisible without a GUI** — start the server with `--permission=*` or sessions will silently pause and die on server restart.
