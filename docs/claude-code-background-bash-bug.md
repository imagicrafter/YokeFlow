# Background Bash Timeout Error Not Surfaced to Agent

## Summary

When a background bash process exceeds its timeout, Claude Code's internal process manager aborts it but fails to return a `tool_result` error event to the agent. This causes the session to continue without knowing the background process failed, and generates repeated internal `AbortError` messages.

## Environment

- **Claude Code Version**: Latest (December 2025)
- **Platform**: macOS (Darwin 24.6.0)
- **Model**: claude-sonnet-4-5-20250929
- **Session Type**: Long-running autonomous coding session

## Steps to Reproduce

1. Start a coding session
2. Execute a background bash command with timeout:
   ```json
   {
     "tool": "Bash",
     "input": {
       "command": "PORT=3001 npm run dev",
       "description": "Start development server on port 3001",
       "timeout": 10000,
       "run_in_background": true
     }
   }
   ```
3. Wait for timeout to expire (10 seconds)
4. Observe session log for `tool_result` event
5. Check Claude Code server logs

## Expected Behavior

When background bash process times out:
1. ✅ Process should be aborted (working as expected)
2. ✅ Agent should receive `tool_result` event with `is_error: true` (NOT WORKING)
3. ✅ Error message should explain timeout occurred (NOT WORKING)
4. ✅ Session should have opportunity to handle the error (NOT WORKING)
5. ✅ Server should log error once and clean up (NOT WORKING - repeated errors)

## Actual Behavior

When background bash process times out:
1. ✅ Process is aborted internally
2. ❌ No `tool_result` event is written to session JSONL log
3. ❌ Agent continues without knowing tool failed
4. ❌ Claude Code server logs `AbortError` repeatedly (20+ times):
   ```
   AbortError:
         at H (/$bunfs/root/claude:4503:326)
         at abort (unknown:1:1)
         at B (/$bunfs/root/claude:1599:2956)
   ```
5. ✅ Session can still be terminated with Ctrl+C (server not fully hung)

## Evidence

### Session JSONL Log

Tool execution recorded:
```json
{"event": "tool_use", "timestamp": "2025-12-23T01:22:27.346329", "tool_number": 160, "tool_name": "Bash", "tool_id": "toolu_01C6yTGkP2Ycob42wtPWsTJf", "input": {"command": "PORT=3001 npm run dev", "description": "Start development server on port 3001", "timeout": 10000, "run_in_background": true}}
```

**No corresponding `tool_result` event exists** (next event is tool #155's result from earlier in session).

Session ended ~4.5 hours later:
```json
{"event": "session_end", "timestamp": "2025-12-23T05:56:08.921916", "session_number": 27, "session_type": "coding", "model": "claude-sonnet-4-5-20250929", "status": "interrupted", "duration_seconds": 17792.22100186348, "message_count": 88, "tool_use_count": 160, "tool_errors": 3, ...}
```

### Server Logs

Claude Code server output (repeated 20+ times):
```
AbortError:
      at H (/$bunfs/root/claude:4503:326)
      at abort (unknown:1:1)
      at B (/$bunfs/root/claude:1599:2956)

Error in hook callback hook_1: 4498 | `);if(D===-1)return JSON.parse(B.trim()).type==="summary";let H=B.substring(0,D);return JSON.parse(H).type==="summary"}catch{return!1}}async function C01(R){if(K7())return;let T=DY7(BA()),A=HY7(T);for(let B of A)try{if(WY7(B))break;if(!XO(EPR.basename(B,".jsonl")))continue;let{messages:G,summaries:$}=await nCR(B),W=$Y7(G);for(let _ of W){if($.has(_.uuid))continue;let C=GY7(_,G);if(C.length===0)continue;try{let J=await BY7(C,R);if(J)await MA1(_.uuid,J)}catch(J){t(J instanceof Error?J:Error(String(J)))}}catch(D){t(D instanceof Error?D:Error(String(D)))}}var EPR,RY7,_01=50000;var J01=Q(()=>{qW();l9();C4();ST();HA();A9();US();Z0();TN();n0();UA();EPR=require("path"),RY7=`
```

## Impact

**Severity**: Medium-High

- Agent loses awareness of background process failures
- Subsequent tasks may depend on failed background process
- No opportunity for error recovery or retry logic
- Internal error loop suggests resource leak or cleanup failure
- Session continues in degraded state without notification

**Workaround**: Avoid using background bash for long-running processes. Start servers before session begins via initialization scripts.

## Additional Context

This was discovered during a multi-session autonomous coding project using the YokeFlow platform (autonomous agent orchestrator built on Claude Code). The session was working on task #114 (creating API endpoints) when it attempted to start a dev server for testing.

The agent would have benefited from:
1. Knowing the dev server failed to start (could retry or adjust timeout)
2. Understanding why subsequent tests might fail (no server running)
3. Logging the failure to session notes for debugging

Instead, the session continued for hours without awareness of the failure, and the user only discovered the issue when checking server logs.

## Proposed Fix

### Option 1: Return Error Tool Result (Preferred)

When background bash process times out or is aborted:
```json
{
  "event": "tool_result",
  "timestamp": "2025-12-23T01:22:37.346329",
  "tool_id": "toolu_01C6yTGkP2Ycob42wtPWsTJf",
  "is_error": true,
  "content": "Error: Background process timed out after 10 seconds and was aborted.\nCommand: PORT=3001 npm run dev\nConsider increasing timeout or starting long-running servers before the session."
}
```

### Option 2: Emit Warning Event (Alternative)

If tool_result can't be emitted after long delay, emit a new event type:
```json
{
  "event": "background_process_error",
  "timestamp": "2025-12-23T01:22:37.346329",
  "tool_id": "toolu_01C6yTGkP2Ycob42wtPWsTJf",
  "error": "Process timed out and was aborted",
  "timeout_ms": 10000
}
```

### Option 3: Document Behavior (Minimum)

If current behavior is intentional, document it clearly:
- Background bash timeouts are silent (no error returned)
- Use short timeouts only for quick background tasks
- Long-running servers should not use background bash

## Related Issues

- Background bash processes are useful for build scripts, cleanup tasks, etc.
- Current timeout behavior makes them risky for any process that might exceed timeout
- Would benefit from better error visibility and cleanup

---

**Please let me know if you need:**
- Full session JSONL log (large file, can provide separately)
- Claude Code server logs (contains stack traces)
- Reproduction steps with minimal example
