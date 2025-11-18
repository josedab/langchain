# Extending and Integrating LangChain

> **Part 4 of 6** | Analysis based on commit [`990e346`](https://github.com/langchain-ai/langchain/tree/990e346c46251246966b562c8d6839903cb205ad)

## What You'll Learn

- Creating custom Runnables
- Building provider integrations
- Developing tools
- Using standard tests for compliance
- Real-world integration examples

## Introduction

LangChain's power comes from its extensibility. Whether you need a custom processing step, want to integrate a new LLM provider, or build specialized tools, the framework provides clear patterns to follow.

## Creating Custom Runnables

### Simple Custom Runnable

The simplest way is to use `RunnableLambda`:

```python
from langchain_core.runnables import RunnableLambda

def custom_process(input: dict) -> str:
    return input["text"].upper()

my_runnable = RunnableLambda(custom_process)
result = my_runnable.invoke({"text": "hello"})  # "HELLO"
```

### Custom Runnable Class

For more control, subclass `Runnable`:

```python
from langchain_core.runnables import Runnable, RunnableConfig
from typing import Any

class TextTransformer(Runnable[str, str]):
    """Transforms text with configurable operations."""

    prefix: str = ""
    suffix: str = ""
    uppercase: bool = False

    def invoke(
        self,
        input: str,
        config: RunnableConfig | None = None,
        **kwargs
    ) -> str:
        result = input
        if self.uppercase:
            result = result.upper()
        return f"{self.prefix}{result}{self.suffix}"

# Usage
transformer = TextTransformer(prefix=">>> ", uppercase=True)
result = transformer.invoke("hello")  # ">>> HELLO"

# Use in chains
chain = prompt | model | transformer
```

### Serializable Custom Runnable

For persistence and sharing:

```python
from langchain_core.runnables import RunnableSerializable

class TextTransformer(RunnableSerializable[str, str]):
    prefix: str = ""
    suffix: str = ""

    @classmethod
    def is_lc_serializable(cls) -> bool:
        return True

    @classmethod
    def get_lc_namespace(cls) -> list[str]:
        return ["my_package", "transformers"]

    def invoke(self, input: str, config=None, **kwargs) -> str:
        return f"{self.prefix}{input}{self.suffix}"

# Now serializable
transformer = TextTransformer(prefix="[", suffix="]")
json_repr = transformer.to_json()
# Can be saved and loaded later
```

### Custom Runnable with Streaming

```python
from typing import Iterator, Any
from langchain_core.runnables import Runnable

class StreamingTransformer(Runnable[str, str]):
    def invoke(self, input: str, config=None, **kwargs) -> str:
        return "".join(list(self.stream(input, config)))

    def stream(
        self,
        input: str,
        config=None,
        **kwargs
    ) -> Iterator[str]:
        for char in input:
            yield char.upper()

# Usage
for chunk in StreamingTransformer().stream("hello"):
    print(chunk, end="")  # H E L L O
```

## Building Provider Integrations

### Chat Model Integration

Here's how to integrate a new LLM provider:

```python
from langchain_core.language_models import BaseChatModel
from langchain_core.messages import (
    BaseMessage, AIMessage, HumanMessage, SystemMessage
)
from langchain_core.outputs import ChatResult, ChatGeneration
from typing import Any, Optional
from pydantic import SecretStr

class ChatMyProvider(BaseChatModel):
    """Chat model for MyProvider API."""

    api_key: SecretStr
    model_name: str = "default-model"
    temperature: float = 0.7

    @property
    def _llm_type(self) -> str:
        return "my-provider-chat"

    def _generate(
        self,
        messages: list[BaseMessage],
        stop: Optional[list[str]] = None,
        run_manager=None,
        **kwargs
    ) -> ChatResult:
        # Convert messages to provider format
        formatted = self._format_messages(messages)

        # Call API
        response = self._call_api(formatted, stop, **kwargs)

        # Convert response
        message = AIMessage(content=response["text"])
        generation = ChatGeneration(message=message)

        return ChatResult(generations=[generation])

    def _format_messages(self, messages: list[BaseMessage]) -> list[dict]:
        result = []
        for msg in messages:
            if isinstance(msg, HumanMessage):
                result.append({"role": "user", "content": msg.content})
            elif isinstance(msg, AIMessage):
                result.append({"role": "assistant", "content": msg.content})
            elif isinstance(msg, SystemMessage):
                result.append({"role": "system", "content": msg.content})
        return result

    def _call_api(self, messages, stop, **kwargs) -> dict:
        # Your API call here
        import requests
        response = requests.post(
            "https://api.myprovider.com/chat",
            json={
                "messages": messages,
                "model": self.model_name,
                "temperature": self.temperature,
                "stop": stop,
            },
            headers={"Authorization": f"Bearer {self.api_key.get_secret_value()}"}
        )
        return response.json()

    @property
    def _identifying_params(self) -> dict:
        return {
            "model_name": self.model_name,
            "temperature": self.temperature
        }
```

### Embeddings Integration

```python
from langchain_core.embeddings import Embeddings
from pydantic import SecretStr

class MyProviderEmbeddings(Embeddings):
    """Embeddings from MyProvider."""

    api_key: SecretStr
    model_name: str = "embed-model"

    def embed_documents(self, texts: list[str]) -> list[list[float]]:
        """Embed multiple documents."""
        response = self._call_api(texts)
        return [item["embedding"] for item in response["data"]]

    def embed_query(self, text: str) -> list[float]:
        """Embed a single query."""
        return self.embed_documents([text])[0]

    def _call_api(self, texts: list[str]) -> dict:
        import requests
        return requests.post(
            "https://api.myprovider.com/embed",
            json={"texts": texts, "model": self.model_name},
            headers={"Authorization": f"Bearer {self.api_key.get_secret_value()}"}
        ).json()
```

### Vector Store Integration

```python
from langchain_core.vectorstores import VectorStore
from langchain_core.documents import Document
from langchain_core.embeddings import Embeddings
from typing import Optional

class MyVectorStore(VectorStore):
    """Vector store using MyDatabase."""

    def __init__(self, embedding: Embeddings, connection_string: str):
        self.embedding = embedding
        self.connection_string = connection_string
        self._client = self._connect()

    def add_documents(
        self,
        documents: list[Document],
        **kwargs
    ) -> list[str]:
        texts = [doc.page_content for doc in documents]
        embeddings = self.embedding.embed_documents(texts)

        ids = []
        for doc, embedding in zip(documents, embeddings):
            id = self._client.insert({
                "text": doc.page_content,
                "metadata": doc.metadata,
                "embedding": embedding
            })
            ids.append(id)

        return ids

    def similarity_search(
        self,
        query: str,
        k: int = 4,
        **kwargs
    ) -> list[Document]:
        query_embedding = self.embedding.embed_query(query)
        results = self._client.search(query_embedding, limit=k)

        return [
            Document(
                page_content=r["text"],
                metadata=r["metadata"]
            )
            for r in results
        ]

    @classmethod
    def from_texts(
        cls,
        texts: list[str],
        embedding: Embeddings,
        metadatas: Optional[list[dict]] = None,
        **kwargs
    ) -> "MyVectorStore":
        store = cls(embedding=embedding, **kwargs)
        documents = [
            Document(page_content=t, metadata=m or {})
            for t, m in zip(texts, metadatas or [{}] * len(texts))
        ]
        store.add_documents(documents)
        return store
```

## Developing Tools

### Using the @tool Decorator

```python
from langchain_core.tools import tool

@tool
def search_database(query: str, limit: int = 10) -> str:
    """Search the company database for information.

    Args:
        query: The search query.
        limit: Maximum number of results.
    """
    # Your implementation
    results = db.search(query, limit)
    return "\n".join(results)

# Automatically creates:
# - name: "search_database"
# - description: from docstring
# - args_schema: from type hints
```

### Using BaseTool

For more control:

```python
from langchain_core.tools import BaseTool
from pydantic import BaseModel, Field
from typing import Optional, Type

class SearchInput(BaseModel):
    query: str = Field(description="Search query")
    limit: int = Field(default=10, description="Max results")

class DatabaseSearch(BaseTool):
    name: str = "database_search"
    description: str = "Search the company database for information"
    args_schema: Type[BaseModel] = SearchInput

    db_connection: str

    def _run(
        self,
        query: str,
        limit: int = 10,
        run_manager=None
    ) -> str:
        # Sync implementation
        results = self._search(query, limit)
        return "\n".join(results)

    async def _arun(
        self,
        query: str,
        limit: int = 10,
        run_manager=None
    ) -> str:
        # Async implementation
        results = await self._async_search(query, limit)
        return "\n".join(results)

    def _search(self, query: str, limit: int) -> list[str]:
        # Your search logic
        ...
```

### Tools with Artifacts

Return rich results:

```python
from langchain_core.tools import tool

@tool(response_format="content_and_artifact")
def generate_chart(data: list[dict]) -> tuple[str, bytes]:
    """Generate a chart from data.

    Args:
        data: Data points for the chart.

    Returns:
        Tuple of (description, chart_image_bytes)
    """
    # Generate chart
    import matplotlib.pyplot as plt
    # ... chart generation ...

    # Return both text and artifact
    return (
        "Chart generated successfully with 10 data points",
        chart_bytes  # Full image data
    )
```

### Tool Collections

```python
from langchain_core.tools import BaseToolkit

class DatabaseToolkit(BaseToolkit):
    """Tools for database operations."""

    db_connection: str

    def get_tools(self) -> list[BaseTool]:
        return [
            DatabaseSearch(db_connection=self.db_connection),
            DatabaseInsert(db_connection=self.db_connection),
            DatabaseUpdate(db_connection=self.db_connection),
        ]

# Usage
toolkit = DatabaseToolkit(db_connection="postgresql://...")
tools = toolkit.get_tools()
```

## Using Standard Tests

LangChain provides compliance tests for integrations:

### Chat Model Tests

```python
# In your tests/
from langchain_tests.unit_tests import ChatModelUnitTests

class TestMyChatModel(ChatModelUnitTests):
    @property
    def chat_model_class(self):
        return ChatMyProvider

    @property
    def chat_model_params(self):
        return {
            "api_key": "test-key",
            "model_name": "test-model"
        }

# Run with pytest
# All standard tests are inherited!
```

### Embeddings Tests

```python
from langchain_tests.unit_tests import EmbeddingsUnitTests

class TestMyEmbeddings(EmbeddingsUnitTests):
    @property
    def embeddings_class(self):
        return MyProviderEmbeddings

    @property
    def embedding_model_params(self):
        return {"api_key": "test-key"}
```

### Integration Tests

```python
from langchain_tests.integration_tests import ChatModelIntegrationTests

class TestMyChatModelIntegration(ChatModelIntegrationTests):
    @property
    def chat_model_class(self):
        return ChatMyProvider

    @property
    def chat_model_params(self):
        return {
            "api_key": os.environ["MY_PROVIDER_API_KEY"],
        }
```

## Real-World Integration Examples

### RAG Pipeline

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_core.runnables import RunnablePassthrough, RunnableParallel
from langchain_core.output_parsers import StrOutputParser
from langchain_openai import ChatOpenAI, OpenAIEmbeddings
from langchain_chroma import Chroma

# Setup
embeddings = OpenAIEmbeddings()
vectorstore = Chroma(embedding_function=embeddings)
retriever = vectorstore.as_retriever(search_kwargs={"k": 3})

prompt = ChatPromptTemplate.from_template("""
Answer based on the context below.

Context: {context}

Question: {question}
""")

model = ChatOpenAI(model="gpt-4")

# Build chain
chain = (
    RunnableParallel(
        context=lambda x: retriever.invoke(x["question"]),
        question=lambda x: x["question"]
    )
    | prompt
    | model
    | StrOutputParser()
)

# Use
result = chain.invoke({"question": "What is LangChain?"})
```

### Agent with Tools

```python
from langchain_core.prompts import ChatPromptTemplate
from langchain_openai import ChatOpenAI
from langchain_core.tools import tool

@tool
def calculator(expression: str) -> str:
    """Evaluate a mathematical expression."""
    return str(eval(expression))

@tool
def search(query: str) -> str:
    """Search for information."""
    # Implementation
    return f"Results for: {query}"

# Setup model with tools
model = ChatOpenAI(model="gpt-4")
tools = [calculator, search]
model_with_tools = model.bind_tools(tools)

# Agent loop
def run_agent(question: str) -> str:
    messages = [HumanMessage(question)]

    while True:
        response = model_with_tools.invoke(messages)
        messages.append(response)

        if not response.tool_calls:
            return response.content

        for tool_call in response.tool_calls:
            tool_fn = {"calculator": calculator, "search": search}[tool_call["name"]]
            result = tool_fn.invoke(tool_call["args"])
            messages.append(ToolMessage(
                content=result,
                tool_call_id=tool_call["id"]
            ))

result = run_agent("What is 25 * 4?")
```

### Custom Middleware

```python
from langchain_core.runnables import RunnableLambda

def logging_middleware(func):
    """Add logging to any runnable."""
    def wrapper(input, config=None, **kwargs):
        print(f"Input: {input}")
        result = func(input, config, **kwargs)
        print(f"Output: {result}")
        return result
    return wrapper

# Usage
logged_model = RunnableLambda(
    logging_middleware(model.invoke)
)

# Or as a chain step
chain = prompt | logged_model | parser
```

## Package Structure for Integrations

```
langchain-myprovider/
├── langchain_myprovider/
│   ├── __init__.py
│   ├── chat_models.py
│   ├── embeddings.py
│   └── _version.py
├── tests/
│   ├── unit_tests/
│   │   ├── test_chat_models.py
│   │   └── test_embeddings.py
│   └── integration_tests/
│       └── test_chat_models.py
├── pyproject.toml
└── README.md
```

### pyproject.toml

```toml
[project]
name = "langchain-myprovider"
version = "0.1.0"
description = "LangChain integration for MyProvider"
dependencies = [
    "langchain-core>=1.0.0,<2.0.0",
    "myprovider-sdk>=1.0.0",
]

[project.optional-dependencies]
test = [
    "pytest>=8.0.0",
    "langchain-tests>=0.1.0",
]

[tool.uv.sources]
langchain-core = { path = "../core", editable = true }
```

## Key Takeaways

1. **Start with RunnableLambda** for simple custom logic.

2. **Use BaseChatModel/Embeddings/VectorStore** for provider integrations.

3. **Leverage standard tests** for compliance verification.

4. **Follow package structure** conventions for discoverability.

5. **Use SecretStr** for API keys.

6. **Implement both sync and async** where possible.

## What's Next

In [Part 5](./05-performance-analysis.md), we'll analyze LangChain's performance characteristics and learn optimization strategies for production workloads.

---

**Code references**: All examples reference [`990e346`](https://github.com/langchain-ai/langchain/tree/990e346c46251246966b562c8d6839903cb205ad).
