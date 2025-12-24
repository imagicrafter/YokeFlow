# YokeFlow v1.0.0 - Initial Public Release

**Release Date:** December 24, 2025

We're thrilled to announce the initial public release of **YokeFlow** - an autonomous AI development platform that uses Claude to build complete applications across multiple sessions.

---

## 🎉 What is YokeFlow?

YokeFlow is a production-ready platform for autonomous software development. It combines the power of Anthropic's Claude with a robust API-first architecture, modern web UI, and intelligent orchestration to create complete applications from specifications.

**Key Differentiators:**
- **Two-Phase Workflow**: Opus plans the complete roadmap (Session 0), Sonnet implements features (Sessions 1+)
- **Complete Visibility**: All tasks, tests, and progress tracked from day 1
- **Production Ready**: PostgreSQL database, async operations, real-time updates, authentication
- **Quality Focused**: Automated reviews, quality dashboard, trend tracking
- **Browser Verification**: Every feature tested automatically with Playwright

---

## ✨ Features

### Core Platform
- **API-First Architecture**: FastAPI REST API with WebSocket for real-time updates
- **Modern Web UI**: Next.js/TypeScript interface with 5 comprehensive tabs
- **PostgreSQL Database**: Production-ready async operations with connection pooling
- **Agent Orchestrator**: Decoupled session lifecycle management
- **MCP Task Management**: 15+ tools for structured task operations

### Autonomous Development
- **Hierarchical Planning**: Epics → Tasks → Tests (complete roadmap upfront)
- **Multi-Session Execution**: Automatic continuation between sessions
- **Browser Verification**: Playwright automation validates every feature
- **Git Integration**: Automatic commits with descriptive messages
- **Docker Sandbox**: Isolated execution environment for safety

### Quality & Monitoring
- **Automated Reviews**: Deep quality analysis every 5 sessions
- **Quality Dashboard**: Real-time metrics, trends, and recommendations
- **Session Logs**: Dual format (JSONL + TXT) for human and machine
- **Screenshots Gallery**: Visual verification of all features
- **Progress Tracking**: Real-time counters and completion status

### Developer Experience
- **Multiple Spec Files**: Upload main spec + supporting files (code examples, schemas, etc.)
- **Environment Management**: In-browser .env editor
- **Container Management**: Dedicated UI for Docker container control
- **Real-time Updates**: WebSocket live progress across all sessions
- **Comprehensive Logs**: Human/Events/Errors tabs with filtering

---

## 🚀 Getting Started

### Prerequisites
- Node.js 20+
- Python 3.9+
- Docker (for PostgreSQL and sandboxing)
- Claude API token

### Quick Setup

```bash
# Clone the repository
git clone https://github.com/ms4inc/yokeflow.git
cd yokeflow

# Install dependencies
pip install -r requirements.txt
cd web-ui && npm install && cd ..
cd mcp-task-manager && npm install && npm run build && cd ..

# Start PostgreSQL
docker-compose up -d

# Initialize database
python scripts/init_database.py --docker

# Configure environment
cp .env.example .env
# Edit .env with your Claude API token

# Start API server (Terminal 1)
python api/start_api.py

# Start Web UI (Terminal 2)
cd web-ui && npm run dev
```

Visit http://localhost:3000 to access the web interface!

---

## 📊 Platform Statistics

**Development Metrics:**
- **31 consecutive coding sessions** validated in production test
- **64 security tests** passing
- **100% async database** operations
- **5 UI tabs**: Overview, History, Quality, Logs, Containers
- **15+ MCP tools** for task management
- **3 quality review phases** production-ready

**Codebase:**
- **Core modules**: 15 Python files (orchestrator, agent, database, etc.)
- **Review system**: 3 production-ready modules
- **Web UI**: 25+ React components
- **API endpoints**: 30+ REST + WebSocket
- **Database schema**: PostgreSQL with 8 tables + 3 views
- **Tests**: Security, database, orchestrator, MCP integration

---

## 🎯 Use Cases

YokeFlow is perfect for:
- **Greenfield Projects**: Build new applications from scratch
- **Rapid Prototyping**: Go from idea to working app in hours
- **Learning**: Study AI-generated code patterns
- **Experimentation**: Test architectural approaches quickly
- **Internal Tools**: Create custom dashboards, APIs, utilities

**What YokeFlow Builds:**
- Full-stack web applications (React + Node/Python)
- REST APIs and microservices
- Data processing pipelines
- CLI tools and utilities
- Dashboard applications

---

## 📝 Example Workflow

1. **Create Project**: Upload `app_spec.txt` describing your application
2. **Initialize (Session 0)**: Opus creates complete roadmap (epics/tasks/tests)
3. **Review**: Examine the roadmap, adjust if needed
4. **Start Coding**: Sonnet implements tasks with browser verification
5. **Monitor Progress**: Watch real-time updates in Web UI
6. **Quality Reviews**: Automated deep reviews every 5 sessions
7. **Completion**: Celebrate when all tasks are done!

---

## 🔒 Security

YokeFlow includes comprehensive security measures:
- **Command Blocklist**: Prevents dangerous operations (rm, sudo, etc.)
- **Docker Isolation**: Sandboxed execution environment
- **JWT Authentication**: Production-ready API security
- **Environment Protection**: .env files never committed
- **Database Security**: Parameterized queries, SSL support

See [SECURITY.md](SECURITY.md) for details.

---

## 🤝 Contributing

We welcome contributions! See [CONTRIBUTING.md](CONTRIBUTING.md) for:
- Development setup
- Coding standards
- Pull request process
- Testing guidelines

---

## 📖 Documentation

- **[README.md](README.md)** - User guide and quick start
- **[CLAUDE.md](CLAUDE.md)** - Comprehensive quick reference
- **[QUICKSTART.md](QUICKSTART.md)** - Getting started guide
- **[docs/developer-guide.md](docs/developer-guide.md)** - Technical architecture
- **[docs/review-system.md](docs/review-system.md)** - Quality monitoring
- **[docs/configuration.md](docs/configuration.md)** - Configuration options

---

## 🙏 Acknowledgments

YokeFlow is built on Anthropic's Claude Agent SDK and was originally forked from their autonomous coding demonstration. Special thanks to:
- **Anthropic** for Claude and the foundational demo
- **Dynamous Community** for ongoing support and collaboration
- **All Contributors** who helped shape this release

---

## 📜 License

YokeFlow is released under the **YokeFlow Community License (YCL) v1.0** - a permissive license that allows:
- ✅ Free use, modification, and distribution
- ✅ Commercial consulting and support
- ❌ Hosting as a paid service (without permission)

See [LICENSE](LICENSE) for full details.

---

## 🔮 What's Next?

See [TODO-FUTURE.md](TODO-FUTURE.md) for post-release enhancements:
- Per-user authentication and isolation
- Advanced prompt improvement system
- E2B sandbox integration
- Enhanced testing framework
- CI/CD for generated projects
- And more!

---

## 💬 Community

- **GitHub Issues**: Report bugs, request features
- **GitHub Discussions**: Ask questions, share ideas
- **Documentation**: Comprehensive guides in `docs/`

---

## 📊 Release Summary

**Version:** 1.0.0
**Release Type:** Initial Public Release
**Release Date:** December 24, 2025
**Stability:** Production Ready
**Breaking Changes:** N/A (initial release)

**Files Changed:**
- 50+ files updated across core, API, web-ui, and docs
- 10+ new files created (LICENSE, CONTRIBUTING.md, etc.)
- 100+ commits since fork

**Key Milestones:**
- ✅ Complete rebranding from autonomous-coding to YokeFlow
- ✅ PostgreSQL migration complete
- ✅ Web UI v2.0 production-ready
- ✅ Review system (3 phases) complete
- ✅ Docker container management
- ✅ Multiple spec files support
- ✅ All high-priority bugs fixed

---

**Ready to build?** Install YokeFlow and start creating! 🚀

For questions or support, open an issue on GitHub or check our documentation.

**Happy Coding!**
The YokeFlow Team
