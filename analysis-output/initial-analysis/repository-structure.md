# LangChain Repository Structure

> **Analysis based on commit:** [`990e346`](https://github.com/langchain-ai/langchain/tree/990e346c46251246966b562c8d6839903cb205ad)

## Directory Tree with Descriptions

```
langchain/
├── .github/                          # CI/CD and GitHub configuration
│   ├── workflows/                    # GitHub Actions (test, lint, release)
│   ├── scripts/                      # Automation scripts
│   ├── actions/                      # Custom GitHub Actions
│   └── ISSUE_TEMPLATE/               # Issue templates
│
├── libs/                             # Main source packages
│   │
│   ├── core/                         # [langchain-core] Foundation package
│   │   ├── langchain_core/           # Main module
│   │   │   ├── runnables/            # Runnable protocol (6,100+ LOC)
│   │   │   ├── language_models/      # LLM/Chat model base classes
│   │   │   ├── messages/             # Message types (AI, Human, Tool)
│   │   │   ├── prompts/              # Prompt templates
│   │   │   ├── tools/                # Tool definitions
│   │   │   ├── callbacks/            # Event handling
│   │   │   ├── output_parsers/       # Output parsing
│   │   │   ├── embeddings/           # Embedding base class
│   │   │   ├── vectorstores/         # Vector store base class
│   │   │   ├── retrievers/           # Retriever base class
│   │   │   ├── tracers/              # Tracing/logging
│   │   │   └── load/                 # Serialization
│   │   ├── tests/                    # 152+ test files
│   │   ├── pyproject.toml
│   │   └── Makefile
│   │
│   ├── langchain/                    # [langchain-classic] Legacy package
│   │   ├── langchain_classic/
│   │   │   ├── agents/               # Agent implementations
│   │   │   ├── chains/               # Chain compositions
│   │   │   ├── memory/               # Conversation memory
│   │   │   ├── tools/                # 66+ tool implementations
│   │   │   └── ...                   # Various utilities
│   │   └── tests/
│   │
│   ├── cli/                          # [langchain-cli] Command-line tool
│   │   ├── langchain_cli/
│   │   │   ├── cli.py                # Main CLI (Typer-based)
│   │   │   ├── namespaces/           # Command groups
│   │   │   └── *_template/           # Project templates
│   │   └── tests/
│   │
│   ├── text-splitters/               # [langchain-text-splitters]
│   │   └── langchain_text_splitters/ # 14 splitter implementations
│   │
│   ├── standard-tests/               # [langchain-tests] Compliance tests
│   │   └── langchain_tests/          # Reusable test definitions
│   │
│   ├── model-profiles/               # [langchain-model-profiles]
│   │   └── langchain_model_profiles/ # Model metadata/configs
│   │
│   ├── langchain_v1/                 # [langchain-v1] Compatibility layer
│   │
│   └── partners/                     # Provider integrations
│       ├── openai/                   # [langchain-openai]
│       ├── anthropic/                # [langchain-anthropic]
│       ├── ollama/                   # [langchain-ollama]
│       ├── groq/                     # [langchain-groq]
│       ├── mistralai/                # [langchain-mistralai]
│       ├── deepseek/                 # [langchain-deepseek]
│       ├── xai/                      # [langchain-xai]
│       ├── perplexity/               # [langchain-perplexity]
│       ├── huggingface/              # [langchain-huggingface]
│       ├── fireworks/                # [langchain-fireworks]
│       ├── chroma/                   # [langchain-chroma]
│       ├── qdrant/                   # [langchain-qdrant]
│       ├── exa/                      # [langchain-exa]
│       ├── nomic/                    # [langchain-nomic]
│       └── prompty/                  # [langchain-prompty]
│
├── pyproject.toml                    # Root monorepo manifest
├── Makefile                          # Root build commands
├── README.md                         # Project documentation
├── CLAUDE.md                         # Development guidelines
├── AGENTS.md                         # Agents documentation
├── SECURITY.md                       # Security policy
└── .pre-commit-config.yaml           # Pre-commit hooks
```

## Package Details

### Core Package (langchain-core)

**Location:** `/home/user/langchain/libs/core/`
**Version:** 1.0.5
**Purpose:** Foundation abstractions with zero provider dependencies

#### Key Modules

| Module | Files | LOC | Purpose |
|--------|-------|-----|---------|
| `runnables/` | 15 | 6,100+ | Universal invocation protocol |
| `language_models/` | 5 | 2,000+ | LLM/Chat model bases |
| `callbacks/` | 8 | 4,000+ | Event handling system |
| `messages/` | 10 | 1,500+ | Chat message types |
| `tools/` | 6 | 2,000+ | Tool definition system |
| `prompts/` | 12 | 2,500+ | Prompt templates |

### Partner Packages

Each partner follows a consistent structure:

```
partners/{provider}/
├── langchain_{provider}/
│   ├── chat_models.py      # ChatModel implementation
│   ├── llms.py             # LLM implementation (if different)
│   ├── embeddings/         # Embedding implementation
│   └── middleware/         # Hooks/middleware
├── pyproject.toml
├── README.md
└── tests/
    ├── unit_tests/
    └── integration_tests/
```

### Package Dependencies Flow

```
                    langchain-core
                          │
              ┌───────────┼───────────┐
              │           │           │
              ▼           ▼           ▼
    text-splitters  model-profiles  standard-tests
              │           │           │
              └─────┬─────┘           │
                    │                 │
                    ▼                 │
           langchain-classic         │
                    │                 │
                    ▼                 ▼
              langchain-v1    partner packages
                               (openai, anthropic, etc.)
```

## Key Files

### Configuration Files

| File | Purpose |
|------|---------|
| `pyproject.toml` (root) | Monorepo workspace definition |
| `libs/core/pyproject.toml` | Core package dependencies |
| `.pre-commit-config.yaml` | Git hooks for linting/formatting |
| `Makefile` | Build, test, lint commands |

### Documentation Files

| File | Purpose |
|------|---------|
| `README.md` | Project overview and installation |
| `CLAUDE.md` | Development guidelines and standards |
| `AGENTS.md` | Agent implementation documentation |
| `SECURITY.md` | Security policy and reporting |

### Entry Points

| Package | Entry Point | Usage |
|---------|-------------|-------|
| langchain-cli | `langchain = "langchain_cli.cli:app"` | `langchain --help` |
| All packages | `from langchain_{pkg} import *` | Python imports |

## File Statistics

### By Package

| Package | Production Files | Test Files | Total LOC |
|---------|------------------|------------|-----------|
| langchain-core | ~150 | 152 | ~55,000 |
| langchain-classic | ~200 | ~100 | ~40,000 |
| langchain-openai | ~10 | ~30 | ~15,000 |
| langchain-anthropic | ~10 | ~30 | ~15,000 |
| text-splitters | 14 | ~20 | ~3,000 |
| standard-tests | ~20 | - | ~7,000 |

### Largest Files (Core)

| File | LOC | Purpose |
|------|-----|---------|
| `runnables/base.py` | 6,100 | Runnable protocol implementation |
| `callbacks/manager.py` | 2,684 | Callback management |
| `tools/base.py` | 1,500+ | Tool definitions |
| `prompts/chat.py` | 1,200+ | Chat prompt templates |

## Navigation Tips

### Finding Key Abstractions

```bash
# Base classes
libs/core/langchain_core/language_models/base.py
libs/core/langchain_core/embeddings/__init__.py
libs/core/langchain_core/vectorstores/__init__.py
libs/core/langchain_core/retrievers.py

# Core protocols
libs/core/langchain_core/runnables/base.py
libs/core/langchain_core/callbacks/base.py

# Message types
libs/core/langchain_core/messages/
```

### Finding Provider Implementations

```bash
# OpenAI
libs/partners/openai/langchain_openai/chat_models.py

# Anthropic
libs/partners/anthropic/langchain_anthropic/chat_models.py

# Pattern: libs/partners/{provider}/langchain_{provider}/chat_models.py
```

### Finding Tests

```bash
# Unit tests (no network)
libs/core/tests/unit_tests/

# Integration tests (with network)
libs/core/tests/integration_tests/

# Partner tests
libs/partners/{provider}/tests/
```

## Development Workflow

### Working on Core

```bash
cd libs/core
uv sync
make test          # Run unit tests
make lint format   # Code quality
```

### Working on a Partner

```bash
cd libs/partners/openai
uv sync
make test
```

### Running Specific Tests

```bash
# Single file
uv run --group test pytest tests/unit_tests/test_specific.py

# With coverage
uv run --group test pytest --cov=langchain_core tests/
```
