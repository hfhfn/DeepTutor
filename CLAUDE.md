# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

**DeepTutor** is an AI-powered personalized learning assistant with a multi-agent architecture supporting:
- Problem solving with RAG and web search (Smart Solver)
- Question generation in custom and exam-mimicking modes
- Deep research with multi-phase pipeline (planning → researching → reporting)
- Interactive learning with guided paths and Q&A
- Collaborative writing with AI assistance and TTS
- Idea generation from learning materials

**Stack**: Python 3.10+ (FastAPI backend), Node.js 18+ (Next.js 16 frontend)

---

## Essential Commands

### Installation & Environment Setup

```bash
# One-click installation (recommended)
python scripts/install_all.py

# Or use shell script
bash scripts/install_all.sh

# Or manually
pip install -r requirements.txt
npm install --prefix web
```

### Running the Application

```bash
# Start frontend + backend together (recommended for development)
python scripts/start_web.py

# OR start them separately:

# Backend only (FastAPI)
python src/api/run_server.py
# Or: uvicorn src.api.main:app --host 0.0.0.0 --port 8001 --reload

# Frontend only (Next.js)
cd web && npm run dev -- -p 3782
```

### Linting & Code Quality

```bash
# Lint Python code (Ruff)
ruff check src tests

# Format Python code
ruff format src tests

# Run security checks
bandit -r src tests -ll

# Check dependencies for vulnerabilities
safety check

# Run pre-commit hooks on modified files
pre-commit run --all-files
```

### Testing

```bash
# Run all tests
pytest tests -v

# Run specific test file
pytest tests/test_solve.py -v

# Run with coverage
pytest tests --cov=src --cov-report=html

# Run browser automation tests
npm run audit          # Run Playwright tests in headless mode
npm run audit:ui       # Run with Playwright inspector UI
npm run audit:report   # View test report
```

### Docker

```bash
# Build and start (first run ~11 minutes)
docker compose up

# Clear cache and rebuild after pulling new code
docker compose build --no-cache

# Start in background
docker compose up -d

# View logs
docker compose logs -f

# Stop services
docker compose down
```

### Useful Utilities

```bash
# Check Python installation and dependencies
python scripts/check_install.py

# Extract numbered items (definitions, theorems, etc.) from knowledge base
python src/knowledge/extract_numbered_items.py --kb <kb_name>

# Or using shell script
./scripts/extract_numbered_items.sh <kb_name>

# Audit prompt templates for consistency
python scripts/audit_prompts.py

# Audit internationalization (i18n) for frontend
npm run i18n:audit --prefix web

# Generate star/fork roster images
python scripts/generate_roster.py
```

---

## Architecture Overview

### Backend Structure (`src/`)

```
src/
├── agents/             # Multi-agent modules
│   ├── solve/         # Problem solving (dual-loop: analysis + solve)
│   ├── question/      # Question generation (custom + exam-mimicking)
│   ├── research/      # Deep research (3-phase: planning/researching/reporting)
│   ├── guide/         # Guided learning with progressive knowledge points
│   ├── co_writer/     # Interactive IdeaGen with editing & TTS
│   ├── ideagen/       # Automated idea generation from materials
│   └── chat/          # Chat interaction layer
├── api/               # FastAPI backend
│   ├── routers/       # API route handlers for each module
│   ├── main.py        # FastAPI app with middleware & validation
│   └── run_server.py  # Server startup script
├── knowledge/         # RAG pipeline & knowledge base management
│   ├── rag.py        # RAG implementation (naive/hybrid retrieval)
│   └── kb.py         # Knowledge base initialization & document processing
├── tools/            # Tool implementations
│   ├── web_search/   # Web search integration (multiple providers)
│   ├── code_executor/ # Python code execution with safety
│   ├── rag_tools.py  # RAG-related tool wrappers
│   └── ...           # Other tools (paper search, query item, etc.)
├── core/             # Core utilities
│   ├── core.py       # Configuration loading, LLM setup
│   ├── logging/      # Logging system
│   └── setup.py      # System initialization
└── utils/            # Helper utilities
```

### Configuration System

**Three-layer hierarchy** (from highest to lowest priority):

1. **Environment Variables** (`.env` or `DeepTutor.env`)
   - LLM API keys, model names, ports
   - Override all other settings

2. **`config/agents.yaml`** ⭐
   - **SINGLE source of truth** for agent `temperature` and `max_tokens`
   - One set of parameters per module (solve, research, question, guide, ideagen, co_writer)
   - Exception: `narrator` has independent settings for TTS

3. **`config/main.yaml`**
   - Server & system settings
   - Path configurations
   - All module-specific non-LLM settings
   - Tool configurations (RAG, code execution, web search)

**Key Configuration Functions**:
```python
from src.core.core import get_agent_params, load_config_with_main

# Load LLM parameters from agents.yaml
params = get_agent_params("solve")  # Returns {"temperature": X, "max_tokens": Y}

# Load all configuration from main.yaml
config = load_config_with_main("main.yaml", project_root)
```

**⚠️ CRITICAL**: Never hardcode `temperature` or `max_tokens` in agent code. Always use `get_agent_params()`.

---

## Module Architecture

### Smart Solver (`src/agents/solve/`)

**Architecture**: Dual-loop (Analysis Loop + Solve Loop)
- **Analysis Loop**: InvestigateAgent (RAG + web search) → NoteAgent (context synthesis)
- **Solve Loop**: PlanAgent → ManagerAgent → SolveAgent → CheckAgent → Format
- Tools: RAG, web search, code execution, query item
- Output: `data/user/solve/solve_YYYYMMDD_HHMMSS/` with step-by-step reasoning

### Question Generator (`src/agents/question/`)

**Modes**:
- **Custom**: Requirement text → Question planning → Generation → Single-pass validation
- **Mimic**: PDF upload → MinerU parsing → Question extraction → Style mimicking

**Output Structure**:
```
data/user/question/
├── custom_YYYYMMDD_HHMMSS/
│   ├── background_knowledge.json
│   ├── question_plan.json
│   └── question_N_result.json
└── mimic_papers/mimic_YYYYMMDD_HHMMSS_{pdf_name}/
    ├── {pdf_name}.pdf
    ├── auto/{pdf_name}.md (MinerU parsed)
    └── {pdf_name}_*_questions.json
```

### Deep Research (`src/agents/research/`)

**Architecture**: Dynamic Topic Queue with three phases
- **Phase 1 (Planning)**: RephraseAgent + DecomposeAgent (subtopic generation)
- **Phase 2 (Researching)**: ManagerAgent (queue scheduling) + ResearchAgent (tool selection) + NoteAgent (compression)
- **Phase 3 (Reporting)**: Deduplication → Outline generation → Report writing with citations

**Citation System**: Centralized CitationManager
- Planning citations: `PLAN-XX`
- Research citations: `CIT-block-index`
- Inline format: `[N]` → converted to `[[N]](#ref-N)` markdown links

**Execution Modes**:
- `series`: Sequential topic processing
- `parallel`: Concurrent processing with `asyncio.Semaphore` (max_parallel_topics)

**Output**: Timestamped markdown reports with clickable citations in `data/user/research/reports/`

### Guided Learning (`src/agents/guide/`)

**Pipeline**: Select notebook(s) → LocateAgent (3-5 knowledge points) → InteractiveAgent (HTML pages) → ChatAgent (Q&A) → SummaryAgent

### Interactive IdeaGen (`src/agents/co_writer/`)

**Features**: Markdown editor → EditAgent (rewrite/shorten/expand) → Auto-annotation → NarratorAgent (TTS with voice selection)

### Automated IdeaGen (`src/agents/ideagen/`)

**Pipeline**: MaterialOrganizerAgent → Multi-stage filtering (loose → explore → strict → markdown)

---

## API Architecture

**FastAPI Router Structure** (`src/api/routers/`):
- Each module has a dedicated router (solve.py, question.py, research.py, etc.)
- WebSocket support for real-time streaming (solve, research)
- Standard REST endpoints with Pydantic validation
- CORS middleware configured for frontend access

**Key Validation**:
- Tool consistency check at startup (agents.yaml tools must be in main.yaml)
- Configuration drift detection prevents misalignment

**Data Paths**:
```
data/
├── knowledge_bases/    # RAG vector stores & graphs
└── user/
    ├── solve/          # Problem solving artifacts
    ├── question/       # Generated questions
    ├── research/       # Research reports & cache
    ├── guide/          # Learning sessions
    ├── co-writer/      # Documents & audio
    ├── notebook/       # User learning records
    ├── logs/           # System logs
    └── run_code_workspace/  # Code execution sandbox
```

---

## Frontend Structure (`web/`)

**Stack**: Next.js 16 + React 19 + Tailwind CSS

**Key Features**:
- `next dev` for development with fast refresh
- ESLint for code quality
- Internationalization (i18n) support
- Playwright browser automation tests
- Mermaid diagrams & LaTeX math rendering

**API Communication**:
- Environment variable: `NEXT_PUBLIC_API_BASE` (defaults to `http://localhost:8001`)
- WebSocket connections for real-time features
- Session persistence via localStorage

---

## Development Best Practices

### Code Organization

1. **Don't hardcode LLM parameters**: Use `get_agent_params()` from config
2. **Use main.yaml for all other config**: Never hardcode paths, ports, or tool settings
3. **Environment variables only for secrets**: API keys, model names go in `.env`
4. **Relative paths from project root**: For portability across environments

### Error Handling

- Pre-commit hooks validate tool configuration at startup
- Configuration drift detection prevents agent/tool misalignment
- Safety checks on code execution (limited allowed_roots)
- Validation of web search enabled state (global switch in main.yaml)

### Adding New Features

1. **New agent module**: Create `src/agents/{module_name}/` with agents and orchestration
2. **New tool**: Implement in `src/tools/`, add to `main.yaml solve.valid_tools`
3. **New API endpoint**: Add router in `src/api/routers/`, register in main.py
4. **Configuration**: Add to `agents.yaml` (if LLM params) or `main.yaml` (other settings)

### Testing

- Unit tests in `tests/` directory
- Use pytest fixtures for common setup
- Mock external API calls
- Playwright tests for frontend UI automation

---

## Common Troubleshooting

**Port already in use after Ctrl+C**:
```bash
# macOS/Linux
lsof -i :8001 && kill -9 <PID>

# Windows
netstat -ano | findstr :8001 && taskkill /PID <PID> /F
```

**Frontend can't connect to backend**:
- Verify backend is running: `http://localhost:8001/docs`
- Set `NEXT_PUBLIC_API_BASE=http://localhost:8001` in `web/.env.local`

**WebSocket connection fails**:
- Check firewall settings
- Ensure backend WebSocket routes are correctly registered
- Verify URL format: `ws://localhost:8001/api/v1/...`

**npm command not found**:
- Ensure Node.js 18+ is installed
- Activate conda environment if using conda
- Use `conda install -c conda-forge nodejs` if needed

---

## Important Notes

- **Language support**: Configured globally in `main.yaml` (system.language: "en" or "zh")
- **Knowledge base management**: CLI: `python -m src.knowledge.start_kb init <name> --docs <path>`
- **Incremental document addition**: `python -m src.knowledge.add_documents <kb_name> --docs <path>`
- **Research presets**: quick/medium/deep/auto defined in `main.yaml research.presets`
- **Session management**: Solved problems, research, and learning sessions are persisted in `data/user/`
