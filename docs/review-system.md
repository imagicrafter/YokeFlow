# Review System - Developer Guide

**Last Updated:** December 23, 2025
**Status:** Production Ready (Phases 1-3 Complete)

---

## Overview

The review system provides automated quality analysis for agent sessions with three integrated phases:

1. **Phase 1:** Quick checks (zero-cost, every session)
2. **Phase 2:** Deep reviews (AI-powered, ~$0.10 each, automated triggers)
3. **Phase 3:** Web UI dashboard (visual quality monitoring)

This system helps identify issues early, suggests prompt improvements, and tracks quality trends over time.

**Note:** Deep reviews include suggestions for manually improving coding prompts. These can be downloaded from the Quality Dashboard and reviewed by developers to enhance the `prompts/coding_prompt.md` and `prompts/coding_prompt_docker.md` files.

---

## Architecture

```
┌─────────────────┐
│  Agent Session  │
│   Completes     │
└────────┬────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  Phase 1: Quick Quality Check       │
│  (quality_integration.py)           │
│  - Parse JSONL logs                 │
│  - Extract metrics                  │
│  - Run quality checks               │
│  - Calculate rating (1-10)          │
│  - Store in database                │
│  Cost: $0 (zero API calls)          │
└────────┬────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  Phase 2: Trigger Check             │
│  (review_client.py)                 │
│  Should trigger if:                 │
│  - session_number % 5 == 0          │
│  - quality < 7/10                   │
│  - 5+ sessions since last review    │
│  - Project completion (final)       │
└────────┬────────────────────────────┘
         │
         ▼ (if triggered)
┌─────────────────────────────────────┐
│  Phase 2: Deep Review               │
│  (review_client.py)                 │
│  - Load session logs                │
│  - Send to Claude via SDK           │
│  - Store full review in database    │
│  Cost: ~$0.10 per review            │
└────────┬────────────────────────────┘
         │
         ▼
┌─────────────────────────────────────┐
│  Phase 3: Web UI Display            │
│  (QualityDashboard.tsx)             │
│  - Fetch quality data via API       │
│  - Render charts and badges         │
│  - Display full review text         │
│  - Download reviews as markdown     │
│  - Show quality trends              │
└─────────────────────────────────────┘
```

---

## How the Phases Work Together

The three phases form an integrated feedback loop:

**Real-Time Monitoring (Phase 1 → Phase 3):**
```
Session completes → Quick check runs → Store metrics → Display in dashboard
                                     ↓
                         If quality < 7 or session % 5 == 0
                                     ↓
                              Trigger deep review
                                     ↓
                         Store full review text (markdown)
                                     ↓
                         Display in Web UI with download option
```

**Manual Prompt Improvement:**
```
Multiple sessions run → Deep reviews accumulate
                                     ↓
                    Developer downloads reviews from UI
                                     ↓
                    Read suggestions in markdown format
                                     ↓
                    Manually apply improvements to prompts
                                     ↓
                    New sessions run → Quality improves
```

**Data Flow:**
1. **Phase 1** stores metrics in `session_quality_checks` table
2. **Phase 2** adds full `review_text` (markdown) to same row with prompt improvement suggestions
3. **Phase 3** displays complete review text with collapsible sections and download buttons

**Key Insight:** Each phase builds on the previous one. Phase 1 provides instant feedback, Phase 2 adds detailed analysis with actionable recommendations, and Phase 3 makes everything visible and downloadable for developers.

---

## Phase 1: Quick Quality Checks

**File:** [`review/review_metrics.py`](../review/review_metrics.py)
**Cost:** $0 (zero API calls)
**When:** After every session completes

### Metrics Extracted

From JSONL session logs:
- **Tool use count:** Total API calls made
- **Error count:** Failed tool calls or exceptions
- **Error rate:** Percentage of failed calls
- **Playwright usage:** Browser automation calls
- **Screenshot count:** Visual verification attempts

### Quality Checks

```python
def quick_quality_check(metrics: dict, is_initializer: bool = False) -> tuple[list, int]:
    """
    Returns: (issues, rating)
    - issues: List of warning/critical issue dicts
    - rating: 1-10 score
    """
```

**Checks performed:**
1. **Browser Verification** (CRITICAL for coding sessions)
   - Warns if `playwright_count == 0`
   - Initializer sessions are exempt

2. **High Error Rate** (WARNING if > 10%)
   - Indicates prompts may be unclear
   - Or environment issues

3. **No Tool Calls** (CRITICAL)
   - Session did nothing
   - Likely a prompt or API issue

### Database Storage

Stores in `session_quality_checks` table:
```sql
INSERT INTO session_quality_checks (
    session_id,
    check_type,        -- 'quick'
    overall_rating,    -- 1-10
    playwright_count,
    error_count,
    error_rate,
    critical_issues,   -- JSONB array
    warnings           -- JSONB array
)
```

---

## Phase 2: Deep Reviews

**File:** [`review/review_client.py`](../review/review_client.py)
**Cost:** ~$0.10 per review (~$0.40 per 20-session project)
**When:** Automated triggers (see below)

### Trigger Conditions

Deep reviews are triggered when:

1. **Every 5th session** (session_number % 5 == 0)
   - Sessions 5, 10, 15, 20, ...
   - Trend analysis across multiple sessions

2. **Quality drops below 7/10**
   - Immediate review when issues detected
   - Catch problems early

3. **5 sessions since last deep review**
   - Even if not at 5-session interval
   - Ensures regular coverage

4. **Project completion** (final session)
   - Always runs on last session
   - Ensures projects with <5 sessions get reviewed

**Implementation:**
```python
async def should_trigger_deep_review(
    project_id: UUID,
    session_number: int,
    last_session_quality: Optional[int] = None
) -> bool:
    # Check interval (skip session 1)
    if session_number > 1 and session_number % 5 == 0:
        return True

    # Check quality threshold
    if last_session_quality is not None and last_session_quality < 7:
        return True

    # Check gap since last review
    last_review = await db.get_last_deep_review(project_id)
    if last_review:
        gap = session_number - last_review['session_number']
        if gap >= 5:
            return True
    elif session_number >= 5:
        # First deep review
        return True

    return False
```

### Review Process

1. **Load Session Data**
   - Human-readable log (TXT file)
   - Machine-readable events (JSONL file)
   - Quick check metrics from Phase 1

2. **Send to Claude via SDK**
   ```python
   from claude_agent_sdk import ClaudeSDKClient

   client = create_review_client(model="claude-sonnet-4-5-20250929")
   async with client:
       await client.query(review_prompt)
       review_text = await collect_response(client)
   ```

3. **Extract Rating from Review**
   - Parse overall rating (1-10) from review text
   - Store complete review as markdown
   - Full review includes prompt improvement suggestions

4. **Store in Database**
   ```python
   await db.store_deep_review(
       session_id=session_id,
       review_text=review_text,      # Complete markdown review
       overall_rating=rating,
   )
   ```

### Review Prompt

**File:** [`prompts/review_prompt.md`](../prompts/review_prompt.md)

Instructs Claude to analyze:
- Browser verification usage
- Error patterns and recovery
- Task completion effectiveness
- Prompt clarity and specificity
- Agent behavior patterns

Returns structured feedback with:
- Overall rating (1-10)
- Specific prompt improvement suggestions
- Pattern observations
- Action items for improvement

### Non-Blocking Execution

Regular deep reviews run in background (don't block sessions):
```python
# For regular reviews (every 5th session, low quality)
asyncio.create_task(
    self._run_deep_review_background(session_id, project_path)
)
```

Final reviews (project completion) run synchronously (ensure completion):
```python
# For final reviews (project completion with force_final_review=True)
await self._run_deep_review_background(session_id, project_path)
```

This ensures:
- Regular sessions complete immediately
- Reviews run in parallel
- Final reviews complete before project exit
- Database updates happen asynchronously

---

## Phase 3: Web UI Dashboard

**Files:**
- [`web-ui/src/components/QualityDashboard.tsx`](../web-ui/src/components/QualityDashboard.tsx)
- [`web-ui/src/components/SessionQualityBadge.tsx`](../web-ui/src/components/SessionQualityBadge.tsx)

### Features

**1. Summary Cards** (3-column layout)
- Average quality rating across all sessions
- Sessions checked (coverage percentage)
- Browser verification compliance

**2. Quality Trend Chart**
- Visual bars for last 10 sessions
- Color-coded by rating:
  - Green (9-10): Excellent
  - Blue (7-8): Good
  - Yellow (5-6): Fair
  - Red (1-4): Poor
- Shows browser verification usage ("nn Browser Checks")
- Session labels ("Session nn")

**3. Deep Review Reports**
- Collapsible sections for each deep review
- Displays **full review text** as markdown
- Download button for each review (markdown file format)
- Expand/collapse individual reviews
- Shows session number and quality badge per review
- **Includes prompt improvement suggestions** that can be manually applied

### API Endpoints

All endpoints implemented in [`api/main.py`](../api/main.py):

```python
# Overall quality summary
GET /api/projects/{id}/quality

# Session-specific quality
GET /api/projects/{id}/sessions/{session_id}/quality

# Recent issues
GET /api/projects/{id}/quality/issues?limit=5

# Browser verification stats
GET /api/projects/{id}/quality/browser-verification
```

### Usage

1. Navigate to project detail page
2. Click "Quality" tab (📊 icon)
3. View dashboard with:
   - Summary statistics
   - Quality trend over time
   - Deep review reports with download option
4. Download reviews as markdown files
5. Read prompt improvement suggestions
6. Manually apply improvements to `prompts/coding_prompt.md` or `prompts/coding_prompt_docker.md`

---

## Database Schema

**Table:** `session_quality_checks`

```sql
CREATE TABLE session_quality_checks (
    id UUID PRIMARY KEY DEFAULT uuid_generate_v4(),
    session_id UUID NOT NULL REFERENCES sessions(id),

    -- Check metadata
    check_type VARCHAR(10) NOT NULL,  -- 'quick' | 'deep'
    check_version VARCHAR(20),         -- For tracking prompt versions
    created_at TIMESTAMPTZ DEFAULT NOW(),

    -- Quality metrics (from Phase 1)
    overall_rating INTEGER,            -- 1-10
    playwright_count INTEGER DEFAULT 0,
    playwright_screenshot_count INTEGER DEFAULT 0,
    total_tool_uses INTEGER DEFAULT 0,
    error_count INTEGER DEFAULT 0,
    error_rate DECIMAL(5,2) DEFAULT 0,

    -- Issues (JSONB arrays)
    critical_issues JSONB DEFAULT '[]',
    warnings JSONB DEFAULT '[]',

    -- Deep review fields (Phase 2)
    review_text TEXT,                  -- Full markdown review from Claude (includes suggestions)

    -- Indexes
    CONSTRAINT check_type_valid CHECK (check_type IN ('quick', 'deep'))
);

CREATE INDEX idx_quality_session ON session_quality_checks(session_id);
CREATE INDEX idx_quality_type ON session_quality_checks(check_type);
CREATE INDEX idx_quality_rating ON session_quality_checks(overall_rating);
```

**Database Views:**

```sql
-- Project-wide quality summary
CREATE VIEW v_project_quality AS
SELECT
    p.id as project_id,
    COUNT(DISTINCT s.id) as total_sessions,
    COUNT(DISTINCT q.session_id) as checked_sessions,
    AVG(q.overall_rating) as avg_quality_rating,
    COUNT(DISTINCT CASE WHEN q.playwright_count = 0 THEN s.id END)
        as sessions_without_browser_verification,
    AVG(q.error_rate) as avg_error_rate_percent,
    AVG(q.playwright_count) as avg_playwright_calls_per_session
FROM projects p
LEFT JOIN sessions s ON p.id = s.project_id
LEFT JOIN session_quality_checks q ON s.id = q.session_id
WHERE q.check_type = 'quick'
GROUP BY p.id;

-- Recent quality issues
CREATE VIEW v_recent_quality_issues AS
SELECT
    s.id as session_id,
    s.session_number,
    s.project_id,
    q.overall_rating,
    q.critical_issues,
    q.warnings,
    q.created_at
FROM session_quality_checks q
JOIN sessions s ON q.session_id = s.id
WHERE q.overall_rating < 7
   OR jsonb_array_length(q.critical_issues) > 0
ORDER BY q.created_at DESC;
```

---

## Key Design Decisions

### Why Three Phases?

1. **Phase 1 (Quick):** Catches obvious issues immediately at zero cost
2. **Phase 2 (Deep):** Provides AI-powered analysis with actionable suggestions
3. **Phase 3 (UI):** Makes data visible, downloadable, and actionable for users

### Why Automated Triggers?

- **Predictable costs:** Max ~$0.40 per 20-session project
- **Early detection:** Quality drops trigger immediate review
- **Regular coverage:** Every 5 sessions ensures trends are caught
- **Final review:** Project completion always gets reviewed (even if <5 sessions)
- **No manual work:** Runs automatically in background

### Why claude_agent_sdk?

- **Consistency:** Same SDK as core agent (agent.py, client.py)
- **Better error handling:** Retry logic and graceful degradation
- **Unified auth:** Uses CLAUDE_CODE_OAUTH_TOKEN (no separate API keys)
- **Future-proof:** Easy to add MCP tools for advanced analysis

### Why session_number vs total_sessions?

**Bug fixed December 18, 2025:**
- Old code used `total_sessions` (count of all sessions)
- Problem: Includes initializer session, offset by 1
- New code uses `session_number` directly
- Result: Correct triggers at 5, 10, 15, 20, etc.

### Why Manual Prompt Improvements?

Deep reviews include specific suggestions for improving the coding prompts. Developers can:
1. Download reviews as markdown files from the Web UI
2. Read the prompt improvement suggestions
3. Manually apply the most relevant suggestions to prompt files
4. Monitor quality improvements in subsequent sessions

This manual approach ensures:
- Human judgment in applying changes to global prompts
- Careful review before modifying core agent behavior
- Flexibility to adapt suggestions to specific use cases
- Full control over prompt evolution

---

## Testing

**Test file:** [`tests/test_review_phase2.py`](../tests/test_review_phase2.py)

Run tests:
```bash
python tests/test_review_phase2.py
```

Tests cover:
- ✅ Trigger logic (interval, quality, gap, project completion)
- ✅ Metrics extraction from logs
- ✅ Review client functionality
- ✅ Database storage and retrieval
- ✅ Edge cases (no sessions, first review, etc.)

---

## Cost Analysis

**Per Project (20 sessions):**
- Phase 1 (quick checks): $0 (20 sessions × $0)
- Phase 2 (deep reviews): ~$0.40 (4 reviews × $0.10)
- Phase 3 (Web UI): $0 (data visualization only)
- **Total: ~$0.40 per project**

**Triggers:**
- Session 5: Deep review ($0.10)
- Session 10: Deep review ($0.10)
- Session 15: Deep review ($0.10)
- Session 20: Deep review ($0.10)
- Project completion: Deep review if not done recently

If quality drops below 7/10, additional deep reviews may trigger.

---

## Usage Example

### Viewing Quality in Web UI

1. Start project via Web UI
2. Let sessions run (auto-continues)
3. Open Web UI: http://localhost:3000
4. Click project name
5. Click "Quality" tab (📊 icon)
6. View dashboard:
   - Check average quality (should be 7+)
   - Review quality trend chart
   - Expand deep review reports to read full analysis
   - Download reviews as markdown files
   - Read prompt improvement suggestions
   - Manually apply relevant suggestions to prompt files

### Manual Deep Review (CLI)

```bash
# Review specific session
python review/review_agent.py generations/my_project 5

# Output:
# Session 5 Deep Review:
# Rating: 8/10
#
# Full Review (Markdown):
# [Complete markdown review text from Claude including:]
# - Session quality rating with justification
# - Browser verification analysis
# - Error pattern analysis
# - Prompt adherence evaluation
# - Concrete prompt improvement recommendations
```

### Checking Trigger Status

```python
from review.review_client import should_trigger_deep_review
from uuid import UUID

project_id = UUID("...")
session_number = 15
quality = 8

should_trigger = await should_trigger_deep_review(
    project_id,
    session_number,
    quality
)
# Returns: True (session 15 is at 5-session interval)
```

---

## Troubleshooting

### Deep Reviews Not Triggering

**Check:**
1. Is `CLAUDE_CODE_OAUTH_TOKEN` set?
   ```bash
   echo $CLAUDE_CODE_OAUTH_TOKEN
   ```

2. Are sessions completing successfully?
   ```bash
   python scripts/task_status.py generations/my_project
   ```

3. Check database for reviews:
   ```sql
   SELECT s.session_number, q.check_type, q.overall_rating
   FROM session_quality_checks q
   JOIN sessions s ON q.session_id = s.id
   WHERE q.check_type = 'deep'
   ORDER BY s.session_number;
   ```

4. Check API logs for trigger messages:
   ```bash
   # Look for "Triggering deep review" messages in API logs
   ```

### Reviews Stored But Not Showing in UI

**Check:**
1. Is Web UI connected to correct database?
2. Refresh browser (hard refresh: Cmd+Shift+R)
3. Check browser console for API errors
4. Verify API endpoint:
   ```bash
   curl http://localhost:8000/api/projects/{id}/quality
   ```

### Quality Rating Always 10/10

**Possible causes:**
1. Agent is performing exceptionally well (good problem!)
2. Sessions are very short (not enough data)
3. Check if browser verification is being used:
   ```bash
   grep "playwright" generations/my_project/logs/*.jsonl
   ```

---

## Future Enhancements

**Phases 1-3:**
- Manual review trigger button in UI
- Quality filters and search
- PDF report export
- Email/Slack alerts for critical issues
- Cross-project comparative analysis
- Automated prompt improvement analysis system

See [TODO-FUTURE.md](../TODO-FUTURE.md) for complete roadmap.

---

## Related Documentation

**System Overview:**
- [CLAUDE.md](../CLAUDE.md) - Main project documentation
- [README.md](../README.md) - User guide and quick start
- [TODO.md](../TODO.md) - Development roadmap

**Technical Guides:**
- [docs/developer-guide.md](./developer-guide.md) - Technical deep-dive
- [docs/mcp-usage.md](./mcp-usage.md) - MCP integration details

**Review System:**
- [prompts/review_prompt.md](../prompts/review_prompt.md) - Deep review instructions for Phase 2

**Code Reference:**
- [review/review_metrics.py](../review/review_metrics.py) - Phase 1 implementation
- [review/review_client.py](../review/review_client.py) - Phase 2 implementation
- [web-ui/src/components/QualityDashboard.tsx](../web-ui/src/components/QualityDashboard.tsx) - Phase 3 UI

---

## Summary

The review system is production-ready and provides a complete feedback loop for continuous improvement:

**Phase 1 - Quick Checks:**
- ✅ Zero-cost analysis on every session
- ✅ Instant quality ratings (1-10)
- ✅ Critical issue detection

**Phase 2 - Deep Reviews:**
- ✅ AI-powered analysis at critical points
- ✅ Automated triggers (every 5 sessions, quality < 7, project completion)
- ✅ Structured prompt improvement recommendations in full review text

**Phase 3 - Quality Dashboard:**
- ✅ Beautiful web UI for monitoring trends
- ✅ Visual quality charts and badges
- ✅ Deep review display with download option
- ✅ Manual prompt improvement workflow

**Total Cost:** ~$0.40 per 20-session project

**The Complete Loop:**
1. Sessions run → Quick checks identify issues
2. Deep reviews analyze patterns → Store full recommendations in review text
3. Dashboard shows quality trends → Download reviews
4. Developer reads suggestions → Manually apply to prompts
5. New sessions run → Monitor quality improvements

Use this system to identify issues early, improve your prompts systematically, and track quality trends over time!
