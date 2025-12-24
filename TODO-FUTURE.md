# TODO-FUTURE - Post-Release Enhancements

This document tracks enhancements and features planned for **after** the initial YokeFlow release.

**Last Updated:** December 23, 2025

---

## 🎯 POST-RELEASE PRIORITIES

### 1. Prompt Improvement System
**Status:** Under development in a non-published branch, based on refactoring an earlier experimental version.

**Why It Matters:**
- Automated prompt optimization improves agent performance
- Reduces need for manual prompt engineering
- Provides data-driven insights into agent behavior

**Priority:** Will be merged after successful testing has been completed. 

### 2. Multiple Spec Files Upload - Advanced Features
**Status:** ✅ Core feature implemented in v1.0 - Advanced features deferred

**Implemented in v1.0:**
- ✅ Multiple file upload via Web UI
- ✅ Auto-detect primary file (heuristic-based: main.md, spec.md, largest .md/.txt)
- ✅ Automatic `spec/` directory creation
- ✅ Smart `app_spec.txt` generation with file listings
- ✅ Lazy-loading instructions in prompts
- ✅ Backward compatible with single file

**Advanced Features for Post-v1.0:**

**1. Database Metadata Storage (1-2 hours)**
- Add `spec_files JSONB` column to projects table
- Track which file is primary, file sizes, upload timestamps
- Enable UI to show spec file list on project detail page
- Query: "Which projects use multi-file specs?"

```sql
ALTER TABLE projects ADD COLUMN spec_files JSONB DEFAULT '[]';

-- Example data:
[
  {"filename": "main.md", "size": 4096, "is_primary": true, "uploaded_at": "..."},
  {"filename": "api-design.md", "size": 2048, "is_primary": false, "uploaded_at": "..."}
]
```

**2. LLM-Based Primary File Detection (2-3 hours)**
- Use Claude Haiku to analyze first 500 chars of each file
- Detect primary spec based on content analysis, not just filename
- More accurate than heuristics for edge cases
- Fallback to heuristic if LLM call fails

```python
async def detect_primary_with_llm(spec_files: List[Path]) -> Path:
    """Use Claude Haiku to analyze which file is the main specification."""

    # Read first 500 chars of each file
    file_previews = {
        f.name: f.read_text()[:500]
        for f in spec_files
        if f.suffix in ['.md', '.txt']
    }

    # Ask Claude Haiku (cheap, fast)
    prompt = f"""Which file is the main project specification?

Files:
{json.dumps(file_previews, indent=2)}

Return ONLY the filename of the primary specification file."""

    response = await call_claude_haiku(prompt)
    return Path(response.strip())
```

**Cost:** ~$0.001 per project creation (negligible)

**3. Advanced Manifest Parsing (3-4 hours)**
- Parse YAML/TOML frontmatter in spec files
- Structured metadata: dependencies, file relationships, read order
- Auto-generate smarter `app_spec.txt` based on manifest
- Support for conditional includes (e.g., "read if building frontend")

```markdown
---
type: primary_specification
title: My SaaS Application
references:
  api_design:
    file: api-design.md
    read_when: implementing_backend
  database:
    file: database-schema.sql
    read_when: setting_up_database
  wireframes:
    file: wireframes/
    read_when: building_frontend
---

# My SaaS Application

Main content here...
```

**4. Spec File Versioning (4-6 hours)**
- Track changes to spec files across sessions
- Show diff when spec updated mid-project
- "What changed since initialization?" view
- Git-like history for spec evolution

**5. Template Bundles (6-8 hours)**
- Pre-packaged spec file sets for common project types
- "SaaS starter", "E-commerce", "API service", etc.
- One-click import of example specs
- Community-contributed templates

---

**Priority:** MEDIUM (implement based on user feedback after v1.0)

**Estimated Total Effort for All Advanced Features:** 16-23 hours

---

### 3. Spec File Generator - Companion Tool
**Status:** Concept phase - Standalone project

**Vision:** Interactive wizard that helps users create comprehensive specification files optimized for YokeFlow.

**Inspiration Sources:**
- **B-MAD** - Structured requirements gathering
- **SpecKit** - Template-based spec generation

**Core Concept:**
Instead of users starting with a blank file, guide them through a structured interview process that generates a complete, well-formatted spec file proven to work with YokeFlow.

---

## 📋 Proposed Features

### **Interview-Driven Spec Generation**

**Question Flow:**
```
1. Project Type
   ├─ SaaS Application
   ├─ E-commerce Site
   ├─ Mobile App Backends
   ├─ Data Dashboard
   └─ Custom...

2. Tech Stack Preferences
   ├─ Frontend: React/Vue/Angular/Next.js/None
   ├─ Backend: Node/Python/Go/Ruby/None
   ├─ Database: PostgreSQL/MySQL/MongoDB/None
   └─ Additional: Redis/WebSocket/etc.

3. Core Features (multi-select)
   ├─ User Authentication
   ├─ Payment Processing
   ├─ Real-time Updates
   ├─ File Uploads
   ├─ Admin Dashboard
   └─ API Integration...

4. User Roles & Permissions
   └─ Define: Admin, User, Guest, etc.

5. Data Models
   └─ Describe: Users, Products, Orders, etc.

6. UI/UX Requirements
   ├─ Style: Modern/Minimal/Corporate
   ├─ Responsive: Yes/No
   └─ Accessibility: WCAG 2.1 AA?

7. Non-Functional Requirements
   ├─ Performance targets
   ├─ Security requirements
   ├─ Scalability needs
   └─ Compliance (GDPR, HIPAA, etc.)
```

### **Template System**

**Built-in Templates:**
```
templates/
├── saas-starter/
│   ├── main.md
│   ├── api-design.md
│   └── database-schema.sql
├── ecommerce/
│   ├── main.md
│   ├── product-catalog.md
│   └── checkout-flow.md
├── dashboard/
│   ├── main.md
│   └── data-sources.md
└── api-service/
    ├── main.md
    └── endpoints.md
```

**Template Variables:**
- `{{project_name}}`
- `{{tech_stack}}`
- `{{features}}`
- `{{user_roles}}`
- `{{data_models}}`

### **Smart Generation Features**

**1. Progressive Disclosure**
- Start simple (project type, basic features)
- Expand with follow-up questions based on selections
- Example: If user selects "Payment Processing" → ask about payment providers

**2. Best Practices Injection**
- Auto-add security requirements for auth flows
- Suggest testing strategies based on tech stack
- Recommend accessibility features
- Include deployment considerations

**3. Example Code Snippets**
- Generate example API requests
- Create sample data models
- Provide UI component examples
- Include authentication flows

**4. Multi-File Output**
- Generate main.md + supporting files
- Organize by feature area
- Ready to upload to YokeFlow

**5. Spec Validation**
- Check for completeness
- Warn about missing critical details
- Suggest improvements
- Estimate project complexity

---

## 🎨 UI/UX Mockup

**Step-by-Step Wizard:**
```
┌─────────────────────────────────────────┐
│  YokeFlow Spec Generator                │
├─────────────────────────────────────────┤
│                                         │
│  Step 2 of 7: Choose Your Tech Stack   │
│                                         │
│  Frontend Framework:                    │
│  ○ React + TypeScript                   │
│  ⦿ Next.js (recommended for SaaS)       │
│  ○ Vue.js                               │
│  ○ Angular                              │
│  ○ None (Backend only)                  │
│                                         │
│  Backend:                               │
│  ⦿ Node.js + Express                    │
│  ○ Python + FastAPI                     │
│  ○ Go                                   │
│                                         │
│  [Back]              [Next: Features →] │
└─────────────────────────────────────────┘
```

**Live Preview:**
```
┌────────────────────┬────────────────────┐
│ Questions          │ Generated Spec     │
├────────────────────┼────────────────────┤
│ [Interview Form]   │ # My SaaS App      │
│                    │                    │
│ ✅ Project Type    │ ## Tech Stack      │
│ ✅ Tech Stack      │ - Next.js          │
│ → Features         │ - Node.js/Express  │
│   Data Models      │ - PostgreSQL       │
│   UI/UX            │                    │
│   Requirements     │ ## Features        │
│   Review           │ - User Auth        │
│                    │ - Payments         │
│                    │ ...                │
└────────────────────┴────────────────────┘
```

---

## 🛠️ Implementation Options

### **Option 1: Web-Based Tool (Standalone)**
- Separate Next.js application
- Hosted at specs.yokeflow.com
- No YokeFlow account needed
- Export .zip with all spec files
- "Upload to YokeFlow" button (API integration)

**Pros:**
- Accessible to everyone
- Marketing tool for YokeFlow
- Can be used independently
- Easier to maintain

**Cons:**
- Separate deployment
- Need to keep templates in sync

### **Option 2: Integrated in YokeFlow UI**
- New tab in YokeFlow: "Create → From Template"
- Part of project creation flow
- Auto-uploads generated specs

**Pros:**
- Seamless experience
- Single codebase
- Immediate project creation

**Cons:**
- Increases YokeFlow complexity
- Must be logged in to use

### **Option 3: CLI Tool**
- `npx @yokeflow/spec-generator`
- Interactive terminal wizard
- Generates files locally
- Upload manually to YokeFlow

**Pros:**
- Developer-friendly
- Lightweight
- Works offline

**Cons:**
- Limited to technical users
- No visual preview

**Recommendation:** Start with **Option 1** (standalone web tool), then integrate into YokeFlow UI in v2.0

---

## 📊 Estimated Effort

| Component | Effort | Notes |
|-----------|--------|-------|
| UI/UX Design | 8-12 hours | Wizard flow, question design |
| Template System | 12-16 hours | 5-7 starter templates |
| Question Engine | 16-24 hours | Logic, branching, validation |
| Preview/Export | 8-12 hours | Live preview, multi-file export |
| Integration | 4-8 hours | API to upload to YokeFlow |
| Documentation | 4-6 hours | User guide, template docs |
| **Total** | **52-78 hours** | **6-10 days of work** |

---

## 🎯 Success Metrics

**User Adoption:**
- % of new YokeFlow projects using generated specs
- Avg time to create spec (target: <10 minutes)
- Completion rate of wizard

**Quality:**
- % of generated projects that initialize successfully
- % of generated projects that complete without spec changes
- User satisfaction ratings

**Template Effectiveness:**
- Most popular templates
- Template completion rates
- Generated spec average quality score

---

## 🚀 MVP Feature Set

**Phase 1 (4-6 weeks):**
1. Basic wizard with 5 questions
2. 3 starter templates (SaaS, E-commerce, Dashboard)
3. Single-file output (main.md)
4. Download as .txt/.md
5. Copy to clipboard

**Phase 2 (2-3 weeks):**
1. Multi-file output (main + supporting files)
2. Live preview
3. Template variables system
4. 5 additional templates

**Phase 3 (2-3 weeks):**
1. YokeFlow API integration
2. "Upload to YokeFlow" button
3. Save/load draft specs
4. Community template sharing

---

## 💡 Advanced Features (Future)

**AI-Powered Enhancements:**
- Use Claude to analyze rough ideas → generate questions
- Auto-suggest features based on project type
- Validate spec completeness with AI
- Generate test scenarios automatically

**Collaboration:**
- Team spec creation (multiple stakeholders)
- Comment/review workflow
- Version comparison
- Approval process

**Learning:**
- Track which spec patterns lead to successful projects
- Auto-improve templates based on outcomes
- Suggest spec improvements based on YokeFlow session data

---

## 📚 Reference Projects to Study

**B-MAD (Behavioral Modeling and Design):**
- Structured requirements gathering
- Use case driven
- Behavior-focused specs

**SpecKit:**
- Template marketplace
- Component library
- Export to multiple formats

**PRP (Product Requirement Prompts):**
- Templates

---

**Priority:** MEDIUM-HIGH (v1.2 or standalone launch)

**Why Build This:**
- Lowers barrier to entry for YokeFlow
- Many users struggle with "blank page syndrome"
- Improves spec quality → better YokeFlow outcomes
- Marketing tool (can be free/public)
- Differentiator from competitors

**Why Wait:**
- Need real-world YokeFlow usage data first
- Learn which spec patterns work best
- Gather user feedback on pain points
- v1.0 must be stable first

---

**Recommendation:** Build as standalone tool after YokeFlow v1.0 release, once we have data on successful spec patterns.

### 4. Enhanced Session Logs Viewer
**Status:** Basic viewer complete - Could add polish

**Current Features (Working):**
- ✅ View TXT and JSONL logs
- ✅ Download capability
- ✅ Human/Events/Errors tab filtering

**Potential Enhancements:**
- Syntax highlighting for JSONL tool calls
- Search/filter within logs
- Side-by-side TXT + JSONL view
- Keyboard shortcuts
- Export to other formats (PDF, HTML)

**Priority:** LOW (current viewer meets needs)

### 4. Screenshots Gallery Enhancements
**Status:** ✅ Enhanced gallery with smart sorting - Additional polish on wishlist


**Potential Enhancements:**
- Filter/search screenshots by task ID or filename
- Bulk download (download all screenshots for a task as ZIP)
- Side-by-side comparison view for before/after screenshots
- Timeline view showing screenshots chronologically across all tasks
- User annotations - allow adding notes/comments to screenshots
- Screenshot diffing - highlight visual differences between screenshots
- Integration with task detail modal - show related screenshots when viewing a task
- Video recording support (if agents start using video capture)
- Screenshot metadata tagging (success/failure/error states)

**Priority:** LOW (basic gallery meets current needs)

### 5. Advanced History Tab Metrics
**Status:** Core metrics complete - Advanced analytics on wishlist

**Completed:**
- ✅ Token usage breakdown (input, output, cache)
- ✅ Cost calculation
- ✅ Tool usage and error counts
- ✅ Model information per session

**Future Possibilities:**
- Performance metrics (avg tool execution time)
- Visual timeline/activity graph
- Session comparison tool
- Trend analysis across projects
- Cost optimization suggestions
- Resource usage heatmaps

**Priority:** MEDIUM (nice-to-have for production deployment)

### 6. Modify to work on Brownfield or non-UI Codebases

**Issues to Consider**
- Current code focuses heavily on Browser testing
- How to import exisiting codebase - GitHub support?

---

## 🔮 FUTURE ENHANCEMENTS

### Multi-User Support & Authentication
**Status:** Planning phase

**Current State:**
- Single-user JWT authentication in place
- Development mode for easy local testing

**Future Goals:**
- Multi-user support with user accounts
- Project permissions and sharing
- Team collaboration features
- API key management per user
- Role-based access control (admin/developer/viewer)
- Activity logs per user

**Priority:** HIGH (for production SaaS deployment)

### GitHub Integration Enhancements
**Status:** Ideas stage

**Potential Features:**
- Auto-create GitHub repositories for generated projects
- Push code directly to GitHub
- Create pull requests for review
- Integration with GitHub Issues
- Sync tasks with GitHub Projects
- CI/CD pipeline generation

**Priority:** MEDIUM

### Deployment Automation
**Status:** Ideas stage

**Potential Features:**
- One-click deployment to Vercel, Netlify, or Railway
- Digital Ocean integration for full-stack apps
- Docker image generation for deployments
- CI/CD pipeline creation
- Environment variable management across environments
- Automated testing before deployment

**Priority:** MEDIUM

### Docker Container Cleanup Automation ✅ CORE FEATURES COMPLETE
**Status:** Core implementation complete (Dec 24, 2025) - Advanced features deferred

**Completed Features:**
- ✅ Container reuse between sessions (implemented)
- ✅ Manual cleanup utility (`scripts/cleanup_containers.py`)
- ✅ **Auto-cleanup on project deletion** (implemented Dec 23, 2025)
  - Containers automatically removed when project deleted via Web UI
  - Enhanced error handling and logging in `core/orchestrator.py` and `core/sandbox_manager.py`
  - Best-effort deletion (doesn't block project deletion if container cleanup fails)
- ✅ **Auto-stop on project completion** (implemented Dec 24, 2025)
  - Containers automatically stopped when all tasks complete
  - Frees up ports for other projects
  - Containers preserved (not deleted) so they can be restarted
- ✅ **Dedicated /containers page** (implemented Dec 24, 2025)
  - Centralized UI for managing all Docker containers
  - Real-time status display (running/stopped/exited)
  - Manual start/stop/delete controls
  - Port mappings display
  - Statistics dashboard
  - Docker Desktop alternative when Desktop isn't available
- ✅ **API endpoints** for container management (GET status, POST start/stop, DELETE)

**Remaining Advanced Features (Future):**
- Container retention policies (30 min - 1 hour)
  - Option to keep stopped containers for X days
  - User preference: auto-delete vs. manual
- Periodic cleanup task (1 hour)
  - Scheduled job to clean up old stopped containers
  - Configurable retention period
- Container health monitoring
  - Track CPU/memory usage
  - Alert on resource issues
- Disk space alerts
  - Warn when Docker disk usage high
  - Suggest cleanup when needed

**Priority:** MEDIUM for advanced features (core features complete and solve the main port conflict problem)

---

## 🧪 TESTING & QUALITY ASSURANCE

### Comprehensive Test Suite Creation
**Status:** Needed for production confidence

**Current State:**
- ✅ `test_security.py` - 64 tests passing (blocklist validation)
- ⚠️ Other test suites removed as obsolete after major refactoring

**Future Goals:**
- **Unit Tests:**
  - Core modules (orchestrator, agent, database, quality_integration)
  - API endpoints (CRUD operations, WebSocket)
  - MCP tools (task management, bash_docker)
  - Review system (metrics, deep reviews)
  - Sandbox manager (Docker operations)

- **Integration Tests:**
  - End-to-end project workflow (create → initialize → code → complete)
  - Database operations with real PostgreSQL
  - Docker container lifecycle
  - Quality review triggers and execution
  - Session state management

- **UI Tests:**
  - Web UI component tests (Jest/React Testing Library)
  - E2E tests with Playwright
  - API integration tests from UI
  - WebSocket connection handling

- **Performance Tests:**
  - Session execution time benchmarks
  - Database query performance
  - API endpoint response times
  - Memory usage monitoring

- **Test Infrastructure:**
  - CI/CD integration (GitHub Actions)
  - Test coverage reporting
  - Automated test runs on PR
  - Performance regression detection

**Priority:** HIGH (before v1.1 release)

**Estimated Effort:** 40-60 hours

---

## 💡 RESEARCH & EXPLORATION

### Ideas Worth Investigating

1. **Agent Collaboration**
   - Multiple agents working on same project
   - Specialized agents for frontend/backend/testing
   - Agent-to-agent communication
   - Task delegation and coordination

2. **Advanced Testing**
   - AI-generated unit tests
   - Visual regression testing
   - Performance testing automation
   - Test coverage analysis
   - Automated bug detection

3. **Code Review Integration**
   - AI code review before commits
   - Style guide enforcement
   - Security vulnerability scanning
   - Best practices suggestions
   - Automated refactoring suggestions

4. **Custom Agent Templates**
   - User-defined agent behaviors
   - Project-specific prompts
   - Industry-specific templates (e-commerce, SaaS, etc.)
   - Template marketplace/sharing

5. **Incremental Development**
   - Modify existing codebases (not just greenfield)
   - Feature additions to generated projects
   - Bug fix sessions
   - Refactoring support
   - Legacy code modernization

6. **AI Model Selection & Optimization**
   - Automatic model selection based on task complexity
   - Cost optimization strategies
   - Hybrid approaches (multiple models per session)
   - Fine-tuned models for specific project types

7. **Real-time Collaboration**
   - Live session viewing for multiple users
   - Chat/comments during sessions
   - Manual intervention/steering during sessions
   - Collaborative spec editing

8. **Analytics & Insights**
   - Project success prediction
   - Time/cost estimation improvements
   - Pattern recognition across projects
   - Common failure modes analysis
   - Best practices recommendations

---

## 🐛 KNOWN ISSUES (To Address Post-Release)

### 1. Docker Desktop Stability on macOS
**Status:** Mitigated with watchdog

- Docker Desktop can crash during long-running sessions
- Workaround: `docker-watchdog.sh` auto-restarts Docker
- See [docs/PREVENTING_MAC_SLEEP.md](docs/PREVENTING_MAC_SLEEP.md)

**Future Solution:**
- Investigate alternative container runtimes (Podman, Colima)
- Improve Docker health monitoring
- Better error recovery mechanisms

**Priority:** LOW (workaround effective)

### 2. Database Connection Pool Exhaustion
**Status:** Monitoring

- Occasional connection pool exhaustion on long sessions
- May need to tune pool size or timeout settings

**Future Solution:**
- Dynamic connection pool sizing
- Better connection lifecycle management
- Connection leak detection

**Priority:** MEDIUM

---

## 📚 DOCUMENTATION NEEDS (Post-Release)

### High Priority
- [ ] Create deployment guide for production
- [ ] Create API documentation (OpenAPI/Swagger)
- [ ] Video tutorials for Web UI
- [ ] Troubleshooting guide expansion

### Medium Priority
- [ ] Create user guide for non-technical users
- [ ] Contributing guide for open source
- [ ] Migration guide for users of original autonomous-coding
- [ ] Best practices guide

### Low Priority
- [ ] Architecture diagrams
- [ ] Database schema visualization
- [ ] MCP protocol documentation
- [ ] Performance tuning guide

---

## 🚀 DEPLOYMENT ROADMAP

### Stage 3: Hosted Service (Future)
- [ ] Deploy to Digital Ocean or AWS
- [ ] Multi-user authentication
- [ ] Payment integration (if SaaS)
- [ ] Monitoring and alerting
- [ ] Production database backups
- [ ] CDN for static assets
- [ ] Load balancing
- [ ] Auto-scaling

---

**Note:** This is a living document. Priorities may change based on user feedback and production needs.
Items in this file are **not** blocking the initial release - they are enhancements for future versions.
