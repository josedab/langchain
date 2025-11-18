# LangChain Dependency Graph

> **Analysis based on commit:** [`990e346`](https://github.com/langchain-ai/langchain/tree/990e346c46251246966b562c8d6839903cb205ad)

## Package Dependency Hierarchy

```
                                    ┌──────────────┐
                                    │   User App   │
                                    └──────┬───────┘
                                           │
              ┌────────────────────────────┼────────────────────────────┐
              │                            │                            │
              ▼                            ▼                            ▼
    ┌─────────────────┐      ┌─────────────────┐        ┌─────────────────────┐
    │ langchain-openai │      │langchain-anthro │        │   langchain-cli     │
    │   (Provider)     │      │   (Provider)    │        │   (Tooling)         │
    └────────┬────────┘      └────────┬────────┘        └──────────┬──────────┘
             │                        │                            │
             │                        │                            │
             └────────────┬───────────┘                            │
                          │                                        │
                          ▼                                        │
             ┌────────────────────────┐                            │
             │   langchain-classic    │◄───────────────────────────┘
             │   (Legacy Chains)      │
             └────────────┬───────────┘
                          │
                          ▼
             ┌────────────────────────┐
             │    langchain-core      │
             │    (Foundation)        │
             └────────────┬───────────┘
                          │
        ┌─────────────────┼─────────────────┐
        │                 │                 │
        ▼                 ▼                 ▼
   ┌─────────┐      ┌─────────┐      ┌─────────┐
   │pydantic │      │langsmith│      │tenacity │
   │ (2.7+)  │      │ (0.3+)  │      │ (8.1+)  │
   └─────────┘      └─────────┘      └─────────┘
```

## Core Package Dependencies

### langchain-core (7 dependencies)

```
langchain-core
├── pydantic >=2.7.4,<3.0.0
│   └── (Data validation, BaseModel)
├── langsmith >=0.3.45,<1.0.0
│   └── (Tracing and observability)
├── tenacity !=8.4.0,>=8.1.0,<10.0.0
│   └── (Retry logic for API calls)
├── jsonpatch >=1.33.0,<2.0.0
│   └── (JSON patching for serialization)
├── PyYAML >=5.3.0,<7.0.0
│   └── (YAML configuration parsing)
├── typing-extensions >=4.7.0,<5.0.0
│   └── (Extended type hints)
└── packaging >=23.2.0,<26.0.0
    └── (Version parsing)
```

## Partner Package Dependencies

### langchain-openai

```
langchain-openai
├── langchain-core >=1.0.2,<2.0.0
├── openai >=1.109.1,<3.0.0
│   └── (OpenAI Python SDK)
└── tiktoken >=0.7.0,<1.0.0
    └── (Token counting)
```

### langchain-anthropic

```
langchain-anthropic
├── langchain-core >=1.0.5,<2.0.0
├── anthropic >=0.73.0,<1.0.0
│   └── (Anthropic Python SDK)
└── pydantic >=2.7.4,<3.0.0
```

### langchain-chroma

```
langchain-chroma
├── langchain-core >=1.0.0,<2.0.0
├── chromadb >=1.0.20,<2.0.0
│   └── (Vector database)
└── numpy (version varies by Python)
    └── (Numerical operations)
```

### langchain-qdrant

```
langchain-qdrant
├── langchain-core >=1.0.0,<2.0.0
├── qdrant-client >=1.15.1,<2.0.0
│   └── (Qdrant vector search)
└── pydantic >=2.7.4,<3.0.0

Optional:
└── fastembed >=0.3.3,<1.0.0
    └── (Embedding models)
```

### langchain-mistralai

```
langchain-mistralai
├── langchain-core >=1.0.0,<2.0.0
├── tokenizers >=0.15.1,<1.0.0
│   └── (HuggingFace tokenizer)
├── httpx >=0.25.2,<1.0.0
│   └── (Async HTTP client)
├── httpx-sse >=0.3.1,<1.0.0
│   └── (Server-Sent Events)
└── pydantic >=2.0.0,<3.0.0
```

### langchain-ollama

```
langchain-ollama
├── langchain-core >=1.0.0,<2.0.0
└── ollama >=0.6.0,<1.0.0
    └── (Local LLM inference)
```

### langchain-groq

```
langchain-groq
├── langchain-core >=1.0.2,<2.0.0
└── groq >=0.30.0,<1.0.0
    └── (Groq inference API)
```

### langchain-fireworks

```
langchain-fireworks
├── langchain-core >=1.0.0,<2.0.0
├── fireworks-ai >=0.13.0,<1.0.0
├── openai >=2.0.0,<3.0.0
├── requests >=2.0.0,<3.0.0
└── aiohttp >=3.9.1,<4.0.0
```

## Development Dependencies

### Testing Stack

```
Test Dependencies
├── pytest >=8.0.0,<9.0.0
│   └── (Test framework)
├── pytest-asyncio
│   └── (Async test support)
├── pytest-socket
│   └── (Network isolation)
├── pytest-xdist
│   └── (Parallel execution)
├── pytest-mock
│   └── (Mocking utilities)
├── freezegun
│   └── (Time mocking)
├── syrupy
│   └── (Snapshot testing)
├── responses
│   └── (HTTP mocking)
├── vcrpy
│   └── (HTTP recording)
└── blockbuster
    └── (Blocking I/O detection)
```

### Code Quality Stack

```
Quality Dependencies
├── ruff >=0.13.1,<0.14.0
│   └── (Linting & formatting)
├── mypy >=1.18.1,<1.19.0
│   └── (Type checking)
├── types-pyyaml
│   └── (PyYAML type stubs)
└── types-requests
    └── (Requests type stubs)
```

## Dependency Characteristics

### By Weight

| Category | Package | Weight | Notes |
|----------|---------|--------|-------|
| Heavy | numpy | ~30MB | Vector operations |
| Heavy | chromadb | ~50MB | Vector DB |
| Medium | transformers | ~100MB | Only in tests |
| Light | pydantic | ~5MB | Core validation |
| Light | tenacity | <1MB | Retry logic |

### By Update Frequency

| Package | Updates | Stability |
|---------|---------|-----------|
| pydantic | Frequent | Breaking changes in v1→v2 |
| openai | Frequent | API changes |
| anthropic | Frequent | New features |
| tenacity | Rare | Very stable |
| PyYAML | Rare | Very stable |

## Version Pinning Strategy

### Pattern Used

```toml
# Core pattern: Allow minor, restrict major
dependency = ">=X.Y.Z,<(X+1).0.0"

# Examples:
pydantic = ">=2.7.4,<3.0.0"       # Allow 2.7.4 - 2.x.x
langsmith = ">=0.3.45,<1.0.0"     # Allow 0.3.45 - 0.x.x
tenacity = "!=8.4.0,>=8.1.0,<10"  # Exclude vulnerable version
```

### Security Considerations

| Dependency | Exclusion | Reason |
|------------|-----------|--------|
| tenacity | 8.4.0 | Known infinite loop issue |

## Internal Package Dependencies

### Dependency Matrix

| Package | Depends On |
|---------|------------|
| langchain-classic | langchain-core |
| langchain-v1 | langchain-classic |
| langchain-cli | langchain-classic |
| text-splitters | langchain-core |
| model-profiles | langchain-core |
| standard-tests | langchain-core |
| All partners | langchain-core |

### Local Development Configuration

```toml
# In partner package pyproject.toml
[tool.uv.sources]
langchain-core = { path = "../../core", editable = true }
langchain-tests = { path = "../../standard-tests", editable = true }
```

## Third-Party Integration Points

### LLM Providers

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   OpenAI API    │     │  Anthropic API  │     │   Ollama API    │
└────────┬────────┘     └────────┬────────┘     └────────┬────────┘
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│  openai (SDK)   │     │ anthropic (SDK) │     │  ollama (SDK)   │
└────────┬────────┘     └────────┬────────┘     └────────┬────────┘
         │                       │                       │
         └───────────────┬───────┴───────────────────────┘
                         │
                         ▼
              ┌─────────────────────┐
              │   BaseChatModel     │
              │  (langchain-core)   │
              └─────────────────────┘
```

### Vector Databases

```
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│   Chroma DB     │     │    Qdrant       │     │    Pinecone     │
└────────┬────────┘     └────────┬────────┘     └────────┬────────┘
         │                       │                       │
         ▼                       ▼                       ▼
┌─────────────────┐     ┌─────────────────┐     ┌─────────────────┐
│chromadb (client)│     │qdrant-client    │     │pinecone (client)│
└────────┬────────┘     └────────┬────────┘     └────────┬────────┘
         │                       │                       │
         └───────────────┬───────┴───────────────────────┘
                         │
                         ▼
              ┌─────────────────────┐
              │     VectorStore     │
              │  (langchain-core)   │
              └─────────────────────┘
```

## Recommendations

### Monitoring

1. **Watch for breaking changes:**
   - Pydantic v3 (when released)
   - OpenAI SDK major updates
   - Anthropic SDK updates

2. **Check for vulnerabilities:**
   - Run `pip-audit` regularly
   - Monitor tenacity for new exclusions

### Optimization

1. **Reduce bundle size:**
   - Use only needed partner packages
   - Consider lighter alternatives for numpy-heavy operations

2. **Pin more strictly in production:**
   ```toml
   # Development: allow ranges
   pydantic = ">=2.7.4,<3.0.0"

   # Production: pin exactly
   pydantic = "==2.9.2"
   ```
