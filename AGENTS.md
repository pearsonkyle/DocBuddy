# DocBuddy - AI Assistant Context

This document provides comprehensive context for AI assistants working on the DocBuddy repository.

## Project Overview

**DocBuddy** is a Python package that enhances FastAPI's Swagger UI documentation with an AI assistant. It enables developers to interact with their API documentation using natural language, execute API calls via tool calling, and build complex workflows with LLMs.

### Key Capabilities
- 💬 Chat interface with full OpenAPI context
- 🤖 LLM Settings panel with local providers (Ollama, LM Studio, vLLM)
- 🔗 Tool-calling for API requests (GET, POST, PUT, PATCH, DELETE)
- 📋 Plan/Act mode workflow for autonomous task execution
- 🎨 Dark/light theme support with customization

### Installation
```bash
pip install docbuddy
```

### Quick Start
```python
from fastapi import FastAPI
from docbuddy import setup_docs

app = FastAPI()
setup_docs(app)  # replaces default /docs
```

---

## Architecture

### Backend (`src/docbuddy/`)

#### Main Files
| File | Purpose |
|------|---------|
| `plugin.py` | Core plugin logic - mounts custom Swagger UI with LLM panels |
| `__init__.py` | Package exports (`setup_docs`, `get_swagger_ui_html`) |

#### Key Functions
- **`setup_docs(app, ...)`** - Mounts the LLM-enhanced docs on a FastAPI app
  - Disables default `/docs` and `/redoc` routes
  - Mounts static files at `/docbuddy-static/`
  - Registers custom docs route

- **`get_swagger_ui_html(...)`** - Returns HTMLResponse with custom Swagger UI
  - Lower-level helper; most users should use `setup_docs`

#### Thread Safety
- Uses `_route_lock` (threading.Lock) for route modifications
- `_llm_apps` tracks which apps have LLM docs setup to avoid duplicates

### Frontend (`src/docbuddy/static/`)

All JavaScript uses a shared namespace pattern: `window.DocBuddy`

| File | Purpose |
|------|---------|
| `core.js` | Shared utilities, storage, theme management, OpenAPI context building |
| `chat.js` | Chat interface with streaming LLM responses |
| `agent.js` | Autonomous task execution with Plan/Act modes (5 iterations max) |
| `workflow.js` | Visual workflow builder for chaining LLM blocks |
| `settings.js` | LLM provider configuration panel |
| `plugin.js` | Swagger UI plugin that integrates all panels |

#### JavaScript Patterns

**Namespace Initialization:**
```javascript
var DocBuddy = window.DocBuddy = {};
```

**Component Factory Pattern:**
```javascript
function ChatPanelFactory(system) {
  return class ChatPanel extends React.Component { ... }
}
DocBuddy.ChatPanelFactory = ChatPanelFactory;
```

**Storage Keys:**
- `docbuddy-settings` - LLM settings (baseUrl, apiKey, modelId)
- `docbuddy-chat-history` - Chat conversation history
- `docbuddy-agent-history` - Agent task history
- `docbuddy-workflow` - Workflow definitions
- `docbuddy-theme` - Theme preferences

---

## File Structure

```
.
├── src/
│   └── docbuddy/
│       ├── __init__.py          # Package exports
│       ├── plugin.py            # Core plugin logic (Python)
│       ├── static/              # Frontend assets
│       │   ├── core.js          # Shared utilities & namespace
│       │   ├── chat.js          # Chat panel component
│       │   ├── agent.js         # Agent panel with Plan/Act modes
│       │   ├── workflow.js      # Workflow builder panel
│       │   ├── settings.js      # LLM settings panel
│       │   ├── plugin.js        # Swagger UI plugin integration
│       │   ├── system-prompt-config.json  # AI prompt presets
│       │   └── themes/          # CSS themes
│       │       ├── dark-theme.css
│       │       ├── light-theme.css
│       │       └── swagger-overrides.css
│       └── templates/
│           └── swagger_ui.html  # Jinja2 template for docs page
├── tests/
│   └── test_plugin.py           # Comprehensive pytest suite (200+ tests)
├── examples/
│   ├── demo_server.py           # Example FastAPI app with DocBuddy
│   └── *.png                    # Feature screenshots
├── pyproject.toml               # Project configuration & dependencies
└── AGENTS.md                    # This file
```

---

## Frontend Architecture

### Core Module (`core.js`)

**System Prompt Presets:**
- `api_assistant` - General API assistant with tool calling
- `agent` - Autonomous task execution (Plan/Act workflow)

**Key Functions:**
| Function | Purpose |
|----------|---------|
| `buildOpenApiContext(schema)` | Generates OpenAPI context for system prompts |
| `buildCurlCommand(...)` | Builds curl commands from tool args |
| `buildApiRequestTool(schema)` | Creates tool definition for LLM tool calling |
| `parseMarkdown(text)` | Safely renders markdown with DOMPurify |
| `loadFromStorage()` / `saveToStorage()` | Settings persistence |
| `applyLLMTheme(theme, colors)` | Dynamic theme injection |

**LLM Provider Configs:**
```javascript
{
  ollama: { name: 'Ollama', url: 'http://localhost:11434/v1' },
  lmstudio: { name: 'LM Studio', url: 'http://localhost:1234/v1' },
  vllm: { name: 'vLLM', url: 'http://localhost:8000/v1' },
  custom: { name: 'Custom', url: '' }
}
```

### Agent Module (`agent.js`)

**Plan/Act Workflow:**
1. **Clarification Phase**: Ask up to 3 targeted questions
2. **Planning Phase**: Outline step-by-step plan with available tools
3. **Execution Phase**: Execute iteratively (max 5 tool calls)
4. **Delivery Phase**: Synthesize final output

**Key Features:**
- `toggleMode()` - Switch between Plan and Act modes
- `MAX_AGENT_ITERATIONS = 5` - Prevent infinite loops
- `handleExecuteToolCall()` - Execute API requests with validation
- `sendToolResult()` - Process tool results and continue streaming

### Chat Module (`chat.js`)

**Streaming Response Handling:**
```javascript
_streamLLMResponse(apiMessages, streamMsgId, fullSchema)
```

**Tool Calling Support:**
- Parses `tool_calls` from LLM responses
- Shows editable tool call panel before execution
- Supports auto-execute in Act mode

### Workflow Module (`workflow.js`)

**Block Types:**
- `prompt` - User message or instruction
- `llm` - LLM processing with system prompt
- `tool_call` - API request via tool calling
- `tool_result` - Tool execution result

**Features:**
- Drag-and-drop block reordering
- Block output display with code blocks
- Export workflow as JSON
- Per-block execution (run single block)

---

## Development

### Setting Up

```bash
# Clone and install dev dependencies
pip install -e ".[dev]"

# Run tests
pytest tests/ -v --tb=short

# Run linters
pre-commit run --all-files
```

### Running Demo Server

```bash
uvicorn examples.demo_server:app --reload --host 0.0.0.0 --port 8000
```

Visit `http://localhost:8000/docs` to see DocBuddy in action.

---

## Testing Strategy

### Test Coverage (`tests/test_plugin.py`)
- **200+ tests** covering all functionality
- Uses `FastAPI.TestClient` for integration testing
- Tests verify HTML output, JavaScript inclusion, tab persistence

### Running Tests
```bash
pytest tests/ -v --tb=short
```

### Key Test Categories:
1. `setup_docs` parameter forwarding & route management
2. Template structural integrity (DOMPurify, marked.js, SRI)
3. Provider configuration (Ollama, LM Studio, vLLM)
4. JavaScript function existence checks
5. Thread safety tests
6. Theme CSS scoping
7. Local storage key verification
8. Agent tab functionality

---

## Common Tasks for AI Assistants

### Adding a New Feature

1. **Backend changes** (if needed):
   - Modify `src/docbuddy/plugin.py` or `__init__.py`
   - Update tests in `tests/test_plugin.py`

2. **Frontend changes**:
   - Add component to appropriate JS file (`chat.js`, `agent.js`, etc.)
   - Follow existing patterns and naming conventions
   - Ensure theme compatibility (use CSS variables)

3. **Testing**:
   - Add corresponding test cases
   - Run `pytest tests/` before committing

### Modifying System Prompts

Edit `src/docbuddy/static/system-prompt-config.json`:

```json
{
  "presets": {
    "preset_name": {
      "name": "Display Name",
      "description": "Brief description",
      "prompt": "Prompt template with {openapi_context} placeholder"
    }
  }
}
```

### Debugging

1. Enable debug mode: `setup_docs(app, debug=True)`
2. Check browser console for JavaScript errors
3. Verify static files are served at `/docbuddy-static/`
4. Check localStorage for saved settings

---

## Security Considerations

1. **XSS Protection**: DOMPurify sanitizes all markdown rendering
2. **Path Validation**: Tool calls reject paths containing `..` or non-relative URLs
3. **CORS Guidance**: LLM providers must enable CORS for browser access
4. **API Key Handling**: Keys stored in localStorage; never sent to server

---

## CI/CD Pipeline

### GitHub Actions Workflows

| Workflow | Purpose |
|----------|---------|
| `ci.yml` | Lint (pre-commit, ruff, mypy) and test on Python 3.9-3.12 |
| `publish-to-pypi.yml` | Publish to PyPI on release |
| `publish-to-testpypi.yml` | Publish to TestPyPI for testing |

### CI Requirements
- All tests must pass
- Pre-commit hooks run automatically
- Code coverage maintained

---

## LLM Provider Integration

### Supported Providers

| Provider | Default URL | Notes |
|----------|-------------|-------|
| Ollama | `http://localhost:11434/v1` | Local LLM server |
| LM Studio | `http://localhost:1234/v1` | Local LLM interface |
| vLLM | `http://localhost:8000/v1` | High-performance inference |
| Custom | User-defined | For any OpenAI-compatible API |

### Adding a New Provider

1. Add to `LLM_PROVIDERS` in `core.js`
2. Update settings panel UI if needed
3. Add test in `tests/test_plugin.py`

---

## Theme System

### Default Themes

**Dark Theme:**
- Background: `#0f172a`
- Panel BG: `#1f2937`
- Text Primary: `#f7fafc`

**Light Theme:**
- Background: `#f7fafc`
- Panel BG: `#ffffff`
- Text Primary: `#1a202c`

### Customizing Themes

Users can customize colors via the Settings panel. Colors are persisted to localStorage.

---

## Troubleshooting

### Common Issues

1. **LLM connection fails**
   - Check LLM provider settings in Settings tab
   - Ensure CORS is enabled on LLM provider
   - Verify provider URL and API key

2. **Tools not working**
   - Enable tools in Settings panel
   - Check API key has required permissions
   - Verify tool call parameters are valid JSON

3. **Changes not reflecting**
   - Clear browser cache or use incognito mode
   - Rebuild static files if developing locally
   - Restart dev server with `--reload`

---

## Contribution Guidelines

1. Follow existing code patterns and conventions
2. Add tests for new features
3. Update this document for significant changes
4. Run all tests before submitting PRs
5. Ensure pre-commit hooks pass

### Code Style

- Python: PEP 8 compliant (via ruff)
- JavaScript: Consistent with existing codebase
- Use CSS variables for theming
- Keep functions focused and testable

---

## CI/CD Pipeline

### GitHub Actions Workflows

| Workflow | Purpose |
|----------|---------|
| `ci.yml` | Lint (pre-commit, ruff, mypy) and test on Python 3.9-3.12 with coverage |
| `release.yml` | Auto-publish to PyPI on version tags with release notes |
| `publish-to-pypi.yml` | Manual dispatch to publish to PyPI (legacy) |
| `publish-to-testpypi.yml` | Manual dispatch to publish to TestPyPI (legacy) |

### CI Features
- **4 Python versions tested**: 3.9, 3.10, 3.11, 3.12
- **Test coverage reporting** with Codecov integration
- **Security scanning** using Bandit (static analysis)
- **Dependency review** on pull requests
- **Documentation verification** on every PR

### Release Process
1. Tag commit with `vX.Y.Z` (semantic versioning)
2. CI validates version and builds package
3. Tests publish to TestPyPI first (optional)
4. Package uploads to PyPI automatically
5. GitHub Release created with changelog

---

## Additional Resources

- **PyPI**: https://pypi.org/project/docbuddy/
- **GitHub**: https://github.com/pearsonkyle/docbuddy
- **License**: MIT
- **Python Versions**: 3.9, 3.10, 3.11, 3.12

---

*Last updated: Generated by AI assistant*
