# YokeFlow Tests

Test suite for the YokeFlow autonomous development platform.

## Current Status

⚠️ **Unit Tests Needed** - This is a planned enhancement for post-release development.

### What's Currently Tested

**Integration Tests** ✅
- `test_security.py` - Security validation tests (64 test cases)
  - Command blocklist validation
  - Path traversal prevention
  - Shell injection protection
  - Environment variable sanitization
- `test_mcp.py` - MCP task manager integration tests
- `test_database_abstraction.py` - Database layer tests
- `test_orchestrator.py` - Orchestrator component tests

### What's Missing

**Unit Tests** ⏳ (Planned for future release)
- Core modules (agent.py, orchestrator.py, database.py)
- API endpoints (api/main.py)
- Review system (review/review_client.py, review/review_metrics.py)
- Sandbox management (core/sandbox_manager.py)
- Utility modules (core/prompts.py, core/config.py)

**End-to-End Tests** ⏳ (Planned for future release)
- Complete project lifecycle (initialization → coding → completion)
- Multi-session workflows
- WebSocket real-time updates
- Browser-based UI testing (Playwright/Cypress)

## Running Current Tests

### Prerequisites

```bash
# Start PostgreSQL
docker-compose up -d

# Install dependencies
pip install -r requirements.txt
```

### Run Security Tests

```bash
python tests/test_security.py
```

**Expected output:** 64 tests should pass

### Run MCP Tests

```bash
# Build MCP server first
cd mcp-task-manager && npm run build && cd ..

# Run tests
python tests/test_mcp.py
```

### Run Database Tests

```bash
python tests/test_database_abstraction.py
```

### Run Orchestrator Tests

```bash
python tests/test_orchestrator.py
```

## Test Coverage Goals (Post-Release)

### Phase 1: Core Unit Tests
- Agent workflow components
- Database operations
- MCP tool interactions
- Configuration loading
- Prompt rendering

### Phase 2: API Tests
- Endpoint behavior
- Request validation
- Error handling
- WebSocket events
- Authentication/authorization

### Phase 3: Integration Tests
- Database migrations
- Docker sandbox lifecycle
- Session state management
- Quality review workflow
- Git operations

### Phase 4: E2E Tests
- Full project lifecycle
- UI interactions
- Multi-session continuity
- Error recovery
- Performance benchmarks

## Contributing Tests

When the unit test framework is implemented, tests should follow these patterns:

### Test Structure

```python
import pytest
from core.agent import Agent

class TestAgent:
    """Tests for core agent functionality."""

    @pytest.fixture
    def agent(self):
        """Create test agent instance."""
        return Agent(project_id="test-project")

    def test_initialization(self, agent):
        """Test agent initializes correctly."""
        assert agent.project_id == "test-project"
        assert agent.session_number == 0

    def test_get_next_task(self, agent):
        """Test task retrieval from database."""
        task = agent.get_next_task()
        assert task is not None
        assert 'id' in task
        assert 'description' in task
```

### Testing Guidelines

1. **Isolation:** Each test should be independent
2. **Fixtures:** Use pytest fixtures for setup/teardown
3. **Mocking:** Mock external dependencies (database, API calls)
4. **Coverage:** Aim for 80%+ code coverage
5. **Documentation:** Clear docstrings explaining what's being tested

## Test Data

Test fixtures and sample data will be provided in:
- `tests/fixtures/` - Sample spec files, project configs
- `tests/data/` - Database seeds, mock responses
- `tests/mocks/` - Mock objects for testing

## CI/CD Integration (Future)

Planned GitHub Actions workflow:

```yaml
name: Tests
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v2
      - name: Set up Python
        uses: actions/setup-python@v2
      - name: Install dependencies
        run: pip install -r requirements.txt
      - name: Run tests
        run: pytest tests/ --cov=core --cov=api --cov=review
      - name: Upload coverage
        uses: codecov/codecov-action@v2
```

## Related Documentation

- [Developer Guide](../docs/developer-guide.md) - Platform architecture
- [Security Tests](test_security.py) - Current security test suite
- [CLAUDE.md](../CLAUDE.md) - Development guidelines

## Future Enhancements

See [TODO-FUTURE.md](../TODO-FUTURE.md) for:
- Comprehensive unit test coverage
- Integration test suites
- E2E testing framework
- Performance benchmarks
- Test automation in CI/CD

---

**Last Updated:** December 24, 2025
**Status:** Integration tests only - unit tests planned for post-release
