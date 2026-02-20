# Page: Overview

# Overview

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [.github/dependabot.yml](.github/dependabot.yml)
- [.lycheeignore](.lycheeignore)
- [README.md](README.md)
- [docs/docs/core/basics.md](docs/docs/core/basics.md)
- [docs/docs/core/flow_def.mdx](docs/docs/core/flow_def.mdx)
- [docs/docs/core/flow_methods.mdx](docs/docs/core/flow_methods.mdx)
- [docs/docs/getting_started/installation.md](docs/docs/getting_started/installation.md)
- [docs/docs/getting_started/quickstart.md](docs/docs/getting_started/quickstart.md)

</details>



## Purpose and Scope

This document provides a high-level introduction to CocoIndex, a hybrid Python-Rust data transformation framework designed for AI applications. It covers the core value proposition, architectural layers, and fundamental concepts that underpin the system. For details on specific subsystems, refer to the corresponding numbered sections: [Architecture Overview](#2) for layer interactions, [Flow Definition System](#3) for defining data pipelines, [Operation System](#4) for type marshalling, and [Incremental Processing](#9) for change detection mechanisms.

**Sources:** [README.md:1-50](), [docs/docs/core/basics.md:1-20]()

## What is CocoIndex?

CocoIndex is a data transformation framework with a Rust core engine and Python interface, optimized for AI indexing workloads. The framework enables developers to define data processing pipelines declaratively in Python, which are then executed by a high-performance Rust engine.

### Core Value Proposition

| Feature | Description |
|---------|-------------|
| **Incremental Processing** | Minimal recomputation on source or logic changes; only processes changed/new data |
| **Dataflow Programming Model** | Each transformation creates new fields based solely on input fields, without hidden state or mutation |
| **Real-time Synchronization** | Continuous monitoring of sources with automatic target updates via live update mode |
| **Production-ready Performance** | Rust engine provides high throughput with automatic batching and caching |
| **Developer Velocity** | Declarative Python API with ~100 lines to define complex transformation flows |

The framework uses PostgreSQL as internal storage for state management, enabling persistent incremental processing across runs. Data lineage is observable by default, with all intermediate transformations accessible for debugging.

**Sources:** [README.md:26-106](), [docs/docs/core/basics.md:65-96]()

## Architecture Overview

CocoIndex employs a three-layer architecture separating concerns between user interaction, type safety, and execution performance.

```mermaid
graph TB
    subgraph UserLayer["User Layer"]
        CLI["cocoindex CLI<br/>(commands: setup, update, drop, evaluate)"]
        FlowDef["@flow_def decorator<br/>FlowBuilder / DataScope"]
        WebUI["CocoInsight Web UI<br/>(visualization & debugging)"]
    end
    
    subgraph PythonLayer["Python Layer (cocoindex package)"]
        FlowBuilder["FlowBuilder<br/>add_source() / declare()"]
        DataScope["DataScope / DataSlice<br/>transform() / row() / collect()"]
        OpRegistry["Operation Registration<br/>@function / @source_connector / @target_connector"]
        TypeSystem["Type System<br/>analyze_type_info() / make_encoder() / make_decoder()"]
        Settings["Settings / init()<br/>add_auth_entry() / ref_auth_entry()"]
    end
    
    subgraph FFILayer["FFI Boundary (PyO3)"]
        EngineBindings["_engine Module<br/>Rust-Python Bindings"]
    end
    
    subgraph RustCore["Rust Core (cocoindex_engine)"]
        LibContext["LibContext<br/>Global State Management"]
        FlowContext["FlowContext<br/>Per-Flow State"]
        DataflowExec["Dataflow Executor<br/>Incremental Processing Engine"]
        Concurrency["Concurrency Control<br/>max_inflight_rows / max_inflight_bytes"]
    end
    
    subgraph External["External Systems"]
        Sources["Sources<br/>LocalFile / AmazonS3 / GoogleDrive / Postgres"]
        Targets["Targets<br/>Postgres / Qdrant / Neo4j / Kuzu"]
        LLMAPI["LLM APIs<br/>OpenAI / Gemini / Ollama"]
        InternalDB["PostgreSQL<br/>Internal State Storage"]
    end
    
    CLI --> FlowDef
    WebUI -.HTTP API.-> EngineBindings
    
    FlowDef --> FlowBuilder
    FlowDef --> DataScope
    FlowBuilder --> OpRegistry
    DataScope --> OpRegistry
    
    OpRegistry --> TypeSystem
    TypeSystem --> EngineBindings
    FlowBuilder --> EngineBindings
    Settings --> EngineBindings
    
    EngineBindings --> LibContext
    LibContext --> FlowContext
    LibContext --> InternalDB
    FlowContext --> DataflowExec
    DataflowExec --> Concurrency
    
    DataflowExec --> Sources
    DataflowExec --> Targets
    DataflowExec --> LLMAPI
```

**Architecture Layers:**

### User Layer
Provides three interaction modes:
- **CLI**: Commands like `cocoindex setup`, `cocoindex update`, `cocoindex drop` for flow lifecycle management
- **Python API**: Programmatic access via `@flow_def` decorator and `Flow` object methods
- **CocoInsight**: Web-based visualization for flow graphs and data lineage

### Python Layer
Handles flow definition and operation specification:
- `FlowBuilder`: Constructs flows via `add_source()` and `declare()` methods
- `DataScope` / `DataSlice`: Represents data containers and typed references
- Operation decorators: `@function`, `@source_connector`, `@target_connector` register operations
- Type system: `analyze_type_info()` extracts Python type annotations; `make_encoder()` / `make_decoder()` handle serialization

### FFI Boundary
The `_engine` module (PyO3) bridges Python and Rust, exposing:
- Async bridges with cancellation (`CancelOnDropPy`)
- Bidirectional value conversion using encoder/decoder schemas
- Operation registry compiled at application startup

### Rust Core Engine
Executes dataflow with:
- `LibContext`: Global state shared across flows
- `FlowContext`: Per-flow execution state and cache management
- Dataflow executor: Implements incremental processing with change detection
- Concurrency control: Enforces `max_inflight_rows` and `max_inflight_bytes` limits

**Sources:** [README.md:1-238](), [docs/docs/core/flow_def.mdx:1-100](), provided Diagram 1

## Key Concepts

### Flows and Flow Definition

A **flow** is defined by a Python function decorated with `@flow_def`, which takes `FlowBuilder` and `DataScope` as arguments. The function body declares sources, transformations, collectors, and export targets.

```mermaid
graph LR
    FlowDefFunc["@flow_def(name='FlowName')<br/>def flow_func(flow_builder, data_scope)"]
    FlowObj["Flow object<br/>setup() / update() / drop() / close()"]
    Registry["Flow Registry<br/>open_flow() / setup_all_flows() / drop_all_flows()"]
    
    FlowDefFunc -->|"returns"| FlowObj
    FlowDefFunc -->|"registers in"| Registry
```

The `Flow` object provides lifecycle methods:
- `setup()`: Creates internal state tables and target resources
- `update()`: Performs one-time incremental update
- `drop()`: Removes all persistent backends
- `close()`: Removes in-memory state (does not affect persistence)

**Sources:** [docs/docs/core/flow_def.mdx:14-65](), [docs/docs/core/flow_methods.mdx:37-109]()

### Dataflow Programming Model

CocoIndex follows the dataflow paradigm where transformations are pure functions of their inputs. Data flows through operations without mutation:

```mermaid
graph TB
    TopScope["DataScope (top-level)<br/>data_scope['documents']"]
    Source["add_source()<br/>LocalFile / AmazonS3 / GoogleDrive"]
    
    RowScope["DataSlice.row()<br/>with data_scope['documents'].row() as doc"]
    Transform1["doc['chunks'] = doc['content'].transform()<br/>SplitRecursively"]
    
    NestedRow["with doc['chunks'].row() as chunk"]
    Transform2["chunk['embedding'] = chunk['text'].transform()<br/>SentenceTransformerEmbed"]
    
    Collector["DataCollector<br/>data_scope.add_collector()"]
    Collect["collector.collect()<br/>filename=doc['filename'], embedding=chunk['embedding']"]
    
    Export["collector.export()<br/>Postgres / Qdrant / Neo4j"]
    
    TopScope --> Source
    Source --> RowScope
    RowScope --> Transform1
    Transform1 --> NestedRow
    NestedRow --> Transform2
    
    TopScope --> Collector
    Transform2 --> Collect
    Collect --> Collector
    Collector --> Export
```

Key operations:
- **add_source()**: Creates root data table (e.g., `documents` field with `filename`, `content`)
- **transform()**: Applies functions like `SplitRecursively`, `SentenceTransformerEmbed` to create derived fields
- **row()**: Iterates over table rows, creating child scopes for nested processing
- **collect()**: Gathers data from potentially nested scopes
- **export()**: Writes collected data to targets with primary keys and vector indexes

**Sources:** [docs/docs/core/flow_def.mdx:66-296](), [docs/docs/getting_started/quickstart.md:42-132]()

### Data Types and Schema

CocoIndex enforces static schemas determined at flow definition time:

| Type Category | Description | Examples |
|--------------|-------------|----------|
| **Basic Types** | Primitive values | `Int64`, `Float32`, `Str`, `Bytes` |
| **Vector** | Fixed-dimension embeddings | `Vector[Float32, 384]` |
| **Struct** | Named fields | `Struct(filename: Str, content: Str)` |
| **KTable** | Keyed table (primary key) | Document rows identified by `filename` |
| **LTable** | List table (ordered, no keys) | Chunks ordered by `location` |

The type system bridges Python annotations to Rust schemas:
- `analyze_type_info()` extracts type information from Python function signatures
- `make_engine_value_encoder()` serializes Python values to Rust engine format
- `make_engine_value_decoder()` deserializes Rust values back to Python

**Sources:** [docs/docs/core/basics.md:18-30](), [docs/docs/core/flow_def.mdx:74-216](), provided Diagram 3

### Incremental Processing

The core differentiator is incremental processing, which minimizes recomputation by tracking:

```mermaid
stateDiagram-v2
    [*] --> ReadSources
    
    ReadSources --> ContentFingerprint: "Compute fingerprint<br/>(hash of content)"
    ContentFingerprint --> CachedState: "Check internal DB"
    
    state CachedState {
        [*] --> CompareFingerprints
        CompareFingerprints --> Unchanged: "Fingerprint matches"
        CompareFingerprints --> Changed: "Fingerprint differs"
        CompareFingerprints --> New: "No cached entry"
        
        Unchanged --> [*]: "Reuse cached results"
        Changed --> [*]: "Recompute affected paths"
        New --> [*]: "Process new data"
    }
    
    CachedState --> ExecuteTransforms: "For changed/new data only"
    ExecuteTransforms --> CacheResults: "Store in PostgreSQL state tables"
    CacheResults --> ExportTargets: "Update target with deltas"
    ExportTargets --> [*]
```

**Incremental mechanisms:**
- Content fingerprinting: Hashes source data to detect changes
- Dependency tracking: Only recomputes downstream transformations when inputs change
- Result memoization: Caches transformation outputs in PostgreSQL state tables
- Delta exports: Targets receive only additions, updates, and deletions

**Live update mode** continuously monitors sources via:
- `refresh_interval` parameter in `add_source()` for periodic polling
- Source-specific change capture (e.g., PostgreSQL notifications, S3 bucket events)
- `FlowLiveUpdater` class manages background monitoring threads

**Sources:** [docs/docs/core/basics.md:65-96](), [docs/docs/core/flow_methods.mdx:102-366](), [README.md:95-106](), provided Diagram 4

## Operation System

Operations are registered at application startup, creating a bridge between Python and Rust:

```mermaid
graph TB
    PythonFunc["Python Function<br/>@function / @source_connector / @target_connector<br/>def my_operation(...): ..."]
    
    RegCall["_register_op_factory()<br/>Analyzes function signature"]
    
    AnalyzeType["analyze_type_info()<br/>Extracts Python type annotations"]
    
    CreateCodec["make_engine_value_encoder()<br/>make_engine_value_decoder()<br/>Bidirectional serialization"]
    
    RustRegistry["Rust Operation Registry<br/>OpArgs: gpu, cache, batching, behavior_version"]
    
    Runtime["Runtime Execution<br/>Python Value → Encode → Rust Engine → Decode → Python Value"]
    
    PythonFunc --> RegCall
    RegCall --> AnalyzeType
    AnalyzeType --> CreateCodec
    RegCall --> RustRegistry
    CreateCodec --> Runtime
    RustRegistry --> Runtime
```

**Operation categories:**
- **Sources**: `@source_connector` decorates classes that read external data (e.g., `LocalFile`, `AmazonS3`, `GoogleDrive`, `Postgres`)
- **Functions**: `@function` decorates transformations (e.g., `SplitRecursively`, `SentenceTransformerEmbed`, `ExtractByLlm`)
- **Targets**: `@target_connector` decorates classes that write to storage (e.g., `Postgres`, `Qdrant`, `Neo4j`, `Kuzu`)

**OpArgs configuration** controls execution:
- `gpu: bool`: Enable GPU acceleration for operations like `SentenceTransformerEmbed`
- `cache: bool`: Cache transformation results in internal storage
- `batching: BatchingOptions`: Configure batch sizes for parallel processing
- `timeout: timedelta`: Maximum execution time per operation
- `arg_relationship: ArgRelationship`: Defines dependencies between operation arguments

**Sources:** [docs/docs/core/flow_def.mdx:369-564](), provided Diagram 3

## Multi-Tenancy and Namespaces

The `COCOINDEX_APP_NAMESPACE` environment variable enables multi-tenant deployments:

```mermaid
graph TB
    AppNS["COCOINDEX_APP_NAMESPACE<br/>e.g., 'staging', 'production', 'team_a'"]
    
    FlowName["Flow Name<br/>staging.my_flow / production.my_flow"]
    
    StateTable["Internal State Tables<br/>staging__my_flow__state<br/>production__my_flow__state"]
    
    TargetDB["Target Resources<br/>staging__embeddings (Postgres table)<br/>production__embeddings (Postgres table)"]
    
    AppNS --> FlowName
    FlowName --> StateTable
    FlowName --> TargetDB
```

**Isolation guarantees:**
- Flow registry prefixes flow names with namespace
- Internal state tables use namespace-prefixed names
- Target resources can optionally use `get_app_namespace(trailing_delimiter='__')` for schema separation
- Independent lifecycle management: One namespace can be set up/dropped without affecting others

**Sources:** [docs/docs/core/flow_def.mdx:370-400](), provided Diagram 6

## Build and Deployment

CocoIndex distributes as platform-specific Python wheels containing compiled Rust extensions:

```mermaid
graph LR
    PyCode["Python Code<br/>cocoindex/*.py"]
    RustCode["Rust Code<br/>cocoindex_engine/src/**/*.rs"]
    
    Maturin["maturin build<br/>PyO3 Compilation"]
    
    Wheel["Python Wheel<br/>cocoindex-*.whl<br/>(includes compiled Rust)"]
    
    Install["pip install cocoindex"]
    
    Runtime["Runtime Environment<br/>Python 3.10-3.13<br/>PostgreSQL with pgvector<br/>Optional: LLM APIs / Vector DBs"]
    
    PyCode --> Maturin
    RustCode --> Maturin
    Maturin --> Wheel
    Wheel --> Install
    Install --> Runtime
```

**Deployment requirements:**
- Python 3.10+ (supports 3.11-3.13)
- PostgreSQL database with pgvector extension (mandatory for internal state)
- Environment variable: `COCOINDEX_DATABASE_URL`
- Optional: LLM API keys for `ExtractByLlm` and `EmbedText` functions
- Optional: Vector database credentials for targets like `Qdrant`

**Sources:** [docs/docs/getting_started/installation.md:1-54](), provided Diagram 5

## Settings and Configuration

Configuration is managed via the `Settings` dataclass and environment variables:

| Setting | Environment Variable | Purpose |
|---------|---------------------|---------|
| `database_url` | `COCOINDEX_DATABASE_URL` | PostgreSQL connection string for internal state |
| `app_namespace` | `COCOINDEX_APP_NAMESPACE` | Multi-tenant namespace prefix |
| `source_max_inflight_rows` | `COCOINDEX_SOURCE_MAX_INFLIGHT_ROWS` | Global concurrency limit (rows) |
| `source_max_inflight_bytes` | `COCOINDEX_SOURCE_MAX_INFLIGHT_BYTES` | Global concurrency limit (bytes) |

**Initialization:**
```python
cocoindex.init(
    database_url="postgresql://user:pass@localhost:5432/cocoindex",
    app_namespace="staging"
)
```

**Auth registry** manages credentials:
- `add_auth_entry(key, value)`: Creates persistent auth entry with stable key (for targets)
- `ref_auth_entry(key)`: References auth entry by key
- `add_transient_auth_entry(value)`: Creates ephemeral auth entry (for sources/functions)

Auth entries separate secret data from operation specs, enabling backend cleanup even after spec removal.

**Sources:** [docs/docs/core/flow_def.mdx:476-564]()

## Related Documentation

- **[Architecture Overview](#2)**: Detailed layer interactions, PyO3 integration, and Rust engine internals
- **[Flow Definition System](#3)**: Comprehensive guide to `@flow_def`, `FlowBuilder`, `DataScope`, and `DataSlice`
- **[Operation System and Type Marshalling](#4)**: Deep dive into operation registration, type system, encoders/decoders
- **[Built-in Data Sources](#5)**: Catalog of sources like `LocalFile`, `AmazonS3`, `GoogleDrive`, `Postgres`
- **[Built-in Processing Functions](#6)**: Text processing, embedding, LLM extraction functions
- **[Storage Backends and Export Targets](#8)**: Vector databases, graph databases, and custom targets
- **[Incremental Processing and State Management](#9)**: Change detection, fingerprinting, and state storage details
- **[Configuration and Settings](#10)**: Settings dataclass, auth registry, namespaces
- **[CLI and Development Tools](#11)**: Command-line interface, query handlers, CocoInsight UI
- **[Examples and Use Cases](#13)**: Practical implementations across different domains

**Sources:** Table of contents provided in context

---

# Page: Architecture Overview

# Architecture Overview

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [.github/workflows/CI.yml](.github/workflows/CI.yml)
- [.github/workflows/_docs_release.yml](.github/workflows/_docs_release.yml)
- [.github/workflows/_test.yml](.github/workflows/_test.yml)
- [.github/workflows/docs_release.yml](.github/workflows/docs_release.yml)
- [.github/workflows/docs_test.yml](.github/workflows/docs_test.yml)
- [.github/workflows/format.yml](.github/workflows/format.yml)
- [.github/workflows/release.yml](.github/workflows/release.yml)
- [Cargo.lock](Cargo.lock)
- [Cargo.toml](Cargo.toml)
- [pyproject.toml](pyproject.toml)
- [python/cocoindex/__init__.py](python/cocoindex/__init__.py)
- [python/cocoindex/cli.py](python/cocoindex/cli.py)
- [python/cocoindex/flow.py](python/cocoindex/flow.py)
- [python/cocoindex/lib.py](python/cocoindex/lib.py)
- [python/cocoindex/setup.py](python/cocoindex/setup.py)
- [python/cocoindex/subprocess_exec.py](python/cocoindex/subprocess_exec.py)
- [rust/cocoindex/Cargo.toml](rust/cocoindex/Cargo.toml)
- [rust/cocoindex/src/lib_context.rs](rust/cocoindex/src/lib_context.rs)
- [rust/cocoindex/src/prelude.rs](rust/cocoindex/src/prelude.rs)
- [rust/py_utils/Cargo.toml](rust/py_utils/Cargo.toml)
- [rust/py_utils/src/future.rs](rust/py_utils/src/future.rs)
- [rust/utils/Cargo.toml](rust/utils/Cargo.toml)
- [rust/utils/src/error.rs](rust/utils/src/error.rs)
- [rust/utils/src/prelude.rs](rust/utils/src/prelude.rs)
- [rust/utils/src/retryable.rs](rust/utils/src/retryable.rs)
- [uv.lock](uv.lock)

</details>



This document explains CocoIndex's three-layer hybrid Python-Rust architecture, the rationale for this design, and how the layers interact. For information about flow definition APIs and execution lifecycle, see [Flow Definition System](#3). For details on the Rust engine internals, see [Rust Engine Architecture](#12).

CocoIndex is architected as a hybrid system where Python provides the user-facing interface and Rust provides the performance-critical execution engine. This approach combines Python's ergonomics for data pipeline definition with Rust's performance for incremental processing, concurrency control, and stateful computation.

**Sources:** [python/cocoindex/__init__.py:1-128](), [Cargo.toml:1-124](), [pyproject.toml:1-143]()

---

## Three-Layer Architecture

CocoIndex consists of three distinct architectural layers that communicate across language boundaries:

```mermaid
graph TB
    subgraph "Layer 1: Python Interface"
        FlowAPI["cocoindex package<br/>flow.py, sources.py, functions.py, targets.py"]
        CLI["cocoindex.cli<br/>Command-line interface"]
        Settings["cocoindex.setting<br/>Settings, DatabaseConnectionSpec"]
        LibAPI["cocoindex.lib<br/>init(), settings(), start_server()"]
    end
    
    subgraph "Layer 2: FFI Boundary (PyO3)"
        EngineModule["cocoindex._engine<br/>(Rust-compiled Python extension)"]
        PyBridge["rust/py_utils<br/>Python↔Rust value conversion"]
        AsyncBridge["pyo3-async-runtimes<br/>Async runtime coordination"]
    end
    
    subgraph "Layer 3: Rust Core Engine"
        LibContext["rust/cocoindex/src/lib_context.rs<br/>LibContext, FlowContext"]
        Execution["rust/cocoindex/src/execution/<br/>Dataflow executor, incremental engine"]
        Setup["rust/cocoindex/src/setup/<br/>State management, schema setup"]
        Ops["rust/cocoindex/src/ops/<br/>Operation executors"]
    end
    
    FlowAPI --> EngineModule
    CLI --> EngineModule
    Settings --> EngineModule
    LibAPI --> EngineModule
    
    EngineModule --> PyBridge
    EngineModule --> AsyncBridge
    
    PyBridge --> LibContext
    AsyncBridge --> LibContext
    
    LibContext --> Execution
    LibContext --> Setup
    LibContext --> Ops
    
    style EngineModule fill:#f9f9f9
    style LibContext fill:#f9f9f9
```

**Purpose of Each Layer:**

1. **Python Interface Layer**: Provides declarative APIs for flow definition, configuration management, and CLI commands. Users interact exclusively with this layer.

2. **FFI Boundary**: Bridges Python and Rust using PyO3, handling type marshalling, async coordination, and error translation across language boundaries.

3. **Rust Core Engine**: Implements performance-critical operations including incremental processing logic, concurrent execution, state persistence, and external system integration.

**Sources:** [python/cocoindex/flow.py:1-100](), [rust/cocoindex/Cargo.toml:1-103](), [rust/py_utils/src/future.rs:1-87]()

---

## Python Interface Layer

The Python layer exposes the entire user-facing API surface through the `cocoindex` package. This layer is responsible for:

- Flow definition via decorators (`@flow_def`) and builder patterns (`FlowBuilder`, `DataScope`, `DataSlice`)
- Operation registration (`@function`, `@source_connector`, `@target_connector`)
- Configuration management (`Settings`, `DatabaseConnectionSpec`, `GlobalExecutionOptions`)
- CLI commands (`cocoindex setup`, `cocoindex update`, `cocoindex server`)

### Key Python Modules

| Module | Purpose | Key Classes/Functions |
|--------|---------|----------------------|
| `cocoindex.flow` | Flow definition and execution | `Flow`, `FlowBuilder`, `DataScope`, `DataSlice`, `@flow_def` |
| `cocoindex.lib` | Library initialization and settings | `init()`, `settings()`, `start_server()`, `stop()` |
| `cocoindex.setting` | Configuration structures | `Settings`, `DatabaseConnectionSpec`, `ServerSettings` |
| `cocoindex.cli` | Command-line interface | `cli()`, `setup()`, `update()`, `server()` |
| `cocoindex.op` | Operation registration | `@function`, `@source_connector`, `@target_connector` |

The Python layer communicates with Rust by calling functions on the `cocoindex._engine` module, which is the compiled Rust extension:

```python
# python/cocoindex/__init__.py
from . import _engine  # Rust-compiled extension module
_engine.init_pyo3_runtime()

# python/cocoindex/flow.py
from . import _engine
engine_flow_builder = _engine.FlowBuilder(full_name, execution_context.event_loop)
```

**Sources:** [python/cocoindex/__init__.py:1-128](), [python/cocoindex/flow.py:1-150](), [python/cocoindex/lib.py:1-76](), [python/cocoindex/cli.py:1-100]()

---

## FFI Boundary: PyO3 Integration

The FFI boundary uses PyO3 to create a Python extension module from Rust code. This layer handles:

- **Type Marshalling**: Converting Python values to Rust representations and vice versa
- **Async Runtime Coordination**: Bridging Python's `asyncio` event loop with Tokio runtime
- **Error Translation**: Converting Rust errors to Python exceptions
- **Memory Management**: Coordinating reference counting across language boundaries

### Build Configuration

The `_engine` module is built using maturin, configured in `pyproject.toml`:

```toml
[tool.maturin]
bindings = "pyo3"
module-name = "cocoindex._engine"
manifest-path = "rust/cocoindex/Cargo.toml"
```

The Rust crate is configured as a `cdylib` (C-compatible dynamic library):

```toml
[lib]
crate-type = ["cdylib"]
```

**Sources:** [pyproject.toml:57-68](), [rust/cocoindex/Cargo.toml:8-10]()

### PyO3 Feature Set

CocoIndex uses PyO3 with the following capabilities:

- **ABI3 Stable ABI** (`abi3-py311`): Single wheel compatible with Python 3.11+
- **Auto-initialize**: Automatic Python interpreter initialization
- **Async Runtimes** (`pyo3-async-runtimes`): Tokio-asyncio bridge
- **NumPy Integration**: Native NumPy array support

**Sources:** [Cargo.toml:12-20]()

### Async Bridge Implementation

The `rust/py_utils/src/future.rs` module implements `CancelOnDropPy`, a future wrapper that automatically cancels Python async tasks when dropped from Rust:

```rust
// Simplified structure
struct CancelOnDropPy {
    inner: BoxFuture<'static, PyResult<Py<PyAny>>>,
    task: Py<PyAny>,
    event_loop: Py<PyAny>,
    ctx: Py<PyAny>,
}
```

This ensures proper cleanup when Rust futures are dropped, preventing Python task leaks.

**Sources:** [rust/py_utils/src/future.rs:14-86]()

### Error Translation

The Rust error system distinguishes between three error categories for proper FFI handling:

```rust
// rust/utils/src/error.rs
pub enum Error {
    Context { msg: String, source: Box<SError> },
    HostLang(Box<dyn HostError>),  // Python exceptions
    Client { msg: String, bt: Backtrace },
    Internal(anyhow::Error),
}
```

`HostLang` errors preserve Python exceptions across the FFI boundary, while `Client` and `Internal` errors are created in Rust and converted to appropriate Python exception types.

**Sources:** [rust/utils/src/error.rs:18-95]()

---

## Rust Core Engine

The Rust core implements all performance-critical operations. The entry point is the `rust/cocoindex/` crate, which depends on several internal crates:

```mermaid
graph LR
    cocoindex["rust/cocoindex<br/>(Main engine crate)"]
    utils["rust/utils<br/>(Error handling, retries, HTTP)"]
    py_utils["rust/py_utils<br/>(PyO3 utilities)"]
    extra_text["rust/extra_text<br/>(Text processing)"]
    
    cocoindex --> utils
    cocoindex --> py_utils
    cocoindex --> extra_text
    py_utils --> utils
```

**Sources:** [rust/cocoindex/Cargo.toml:16-26](), [rust/utils/Cargo.toml:1-42](), [rust/py_utils/Cargo.toml:1-18]()

### Core Engine Components

The Rust engine is organized into several subsystems:

| Directory/Module | Purpose |
|-----------------|---------|
| `src/lib_context.rs` | Global state management: `LibContext`, `FlowContext`, database connection pooling |
| `src/builder/` | Flow analysis, planning, and execution context building |
| `src/execution/` | Dataflow execution engine, incremental processing, source indexing |
| `src/setup/` | Schema management, state persistence, setup change detection |
| `src/ops/` | Operation implementations (sources, functions, targets) |
| `src/service/` | HTTP server, query handlers, REST API endpoints |
| `src/settings.rs` | Configuration structures matching Python `Settings` |

**Sources:** [rust/cocoindex/src/lib_context.rs:1-50](), [rust/cocoindex/src/prelude.rs:1-42]()

### Global State Management

`LibContext` is the global singleton managing all engine state:

```rust
pub struct LibContext {
    pub db_pools: DbPools,
    pub persistence_ctx: Option<PersistenceContext>,
    pub progress_bars: MultiProgress,
    pub flow_contexts: RwLock<HashMap<String, Arc<FlowContext>>>,
}
```

`FlowContext` represents per-flow state:

```rust
pub struct FlowContext {
    pub flow: Arc<AnalyzedFlow>,
    execution_ctx: Arc<tokio::sync::RwLock<FlowExecutionContext>>,
    pub query_handlers: RwLock<HashMap<String, QueryHandlerContext>>,
}
```

**Sources:** [rust/cocoindex/src/lib_context.rs:241-290]()

### Database Connection Pooling

The `DbPools` structure maintains a connection pool cache keyed by connection URL and user:

```rust
type PoolKey = (String, Option<String>);
type PoolValue = Arc<tokio::sync::OnceCell<PgPool>>;

pub struct DbPools {
    pub pools: Mutex<HashMap<PoolKey, PoolValue>>,
}
```

Pools are lazily initialized with configuration from `DatabaseConnectionSpec` and shared across flows.

**Sources:** [rust/cocoindex/src/lib_context.rs:174-230]()

### Tokio Runtime

A global Tokio runtime is initialized at library load time:

```rust
static TOKIO_RUNTIME: LazyLock<Runtime> = LazyLock::new(|| Runtime::new().unwrap());

pub fn get_runtime() -> &'static Runtime {
    &TOKIO_RUNTIME
}
```

This runtime executes all async operations in the Rust engine, coordinated with Python's asyncio loop via PyO3.

**Sources:** [rust/cocoindex/src/lib_context.rs:164-169]()

---

## Build and Distribution System

CocoIndex uses maturin to build and distribute platform-specific wheels containing the compiled Rust extension:

```mermaid
graph TB
    subgraph "Development"
        PythonSrc["python/cocoindex/*.py"]
        RustSrc["rust/cocoindex/src/*.rs"]
        PyProject["pyproject.toml"]
    end
    
    subgraph "Build Process"
        Maturin["maturin build"]
        Cargo["cargo build"]
        Rustc["rustc (LLVM)"]
    end
    
    subgraph "Artifacts"
        SharedLib["_engine.so / _engine.pyd<br/>(Compiled Rust)"]
        Wheel["cocoindex-*.whl<br/>(Python wheel)"]
    end
    
    subgraph "CI/CD"
        GHActions[".github/workflows/release.yml"]
        Matrix["Build matrix:<br/>Linux x86_64/aarch64<br/>macOS x86_64/arm64<br/>Windows x64"]
    end
    
    PythonSrc --> Maturin
    RustSrc --> Maturin
    PyProject --> Maturin
    
    Maturin --> Cargo
    Cargo --> Rustc
    
    Rustc --> SharedLib
    SharedLib --> Wheel
    PythonSrc --> Wheel
    
    GHActions --> Matrix
    Matrix --> Maturin
    Wheel --> PyPI["PyPI Distribution"]
```

**Sources:** [pyproject.toml:1-68](), [.github/workflows/release.yml:1-164]()

### Cross-Platform Compilation

The release workflow builds wheels for multiple platforms using PyO3/maturin-action:

- **Linux**: x86_64 and aarch64 using manylinux_2_28 containers
- **macOS**: x86_64 (Intel) and aarch64 (Apple Silicon)
- **Windows**: x64

Each platform gets a pre-compiled Rust extension, enabling users to `pip install cocoindex` without requiring Rust toolchains.

**Sources:** [.github/workflows/release.yml:43-78]()

### ABI3 Stable ABI

By using PyO3's `abi3-py311` feature, a single wheel works across Python 3.11, 3.12, 3.13, and future Python versions that maintain ABI compatibility:

```toml
[workspace.dependencies]
pyo3 = { version = "0.27.1", features = ["abi3-py311", ...] }
```

This reduces the build matrix from 15 wheels (5 platforms × 3 Python versions) to 5 wheels (5 platforms), significantly simplifying distribution.

**Sources:** [Cargo.toml:12-17](), [.github/workflows/release.yml:79-95]()

---

## Data Flow Through Layers

The following diagram illustrates how a typical flow operation (e.g., `flow.update()`) traverses the three layers:

```mermaid
sequenceDiagram
    participant User
    participant PythonAPI["Python API<br/>(flow.py)"]
    participant Engine["_engine Module<br/>(PyO3 FFI)"]
    participant LibCtx["LibContext<br/>(Rust)"]
    participant FlowCtx["FlowContext<br/>(Rust)"]
    participant Executor["Execution Engine<br/>(Rust)"]
    participant DB["PostgreSQL"]
    
    User->>PythonAPI: flow.update()
    PythonAPI->>PythonAPI: execution_context.run()
    PythonAPI->>Engine: FlowLiveUpdater.create()
    
    Engine->>Engine: Deserialize Python values
    Engine->>LibCtx: get_lib_context()
    LibCtx->>FlowCtx: get_flow_context(name)
    
    FlowCtx->>FlowCtx: use_execution_ctx()
    FlowCtx->>FlowCtx: Check setup state
    
    FlowCtx->>Executor: Start incremental update
    
    loop For each source
        Executor->>DB: Read previous state
        Executor->>Executor: Detect changes
        Executor->>Executor: Execute transforms
        Executor->>DB: Write new state
    end
    
    Executor->>FlowCtx: Update stats
    FlowCtx->>Engine: Return IndexUpdateInfo
    Engine->>Engine: Serialize Rust values
    Engine->>PythonAPI: PyResult<IndexUpdateInfo>
    PythonAPI->>User: Return stats
```

### Type Marshalling

Values crossing the FFI boundary are serialized using `pythonize` (Rust serde → Python) and custom encoders/decoders:

1. **Python → Rust**: `dump_engine_object()` serializes Python values to dictionaries, which PyO3 converts to Rust structs via serde deserialization
2. **Rust → Python**: Rust structs are serialized to JSON-like values, then converted to Python objects via `pythonize`

**Sources:** [python/cocoindex/flow.py:460-476](), [python/cocoindex/engine_object.py:1-50]()

### Async Coordination

Python's `asyncio` event loop and Tokio runtime are coordinated through `pyo3-async-runtimes`:

- Python async functions are bridged to Rust futures using `pyo3_async_runtimes::into_future_with_locals()`
- Rust futures calling Python code use `pyo3_async_runtimes::tokio::into_py_future()`
- Task cancellation is handled via `CancelOnDropPy` to prevent resource leaks

**Sources:** [rust/py_utils/src/future.rs:60-86](), [python/cocoindex/runtime.py:1-50]()

---

## Rationale for Hybrid Architecture

The hybrid Python-Rust design provides several advantages over pure-Python or pure-Rust approaches:

| Aspect | Pure Python | Hybrid (CocoIndex) | Pure Rust |
|--------|-------------|-------------------|-----------|
| **User ergonomics** | ✅ Excellent | ✅ Excellent (Python API) | ❌ More verbose |
| **Execution performance** | ❌ Limited by GIL | ✅ Native speed | ✅ Native speed |
| **Ecosystem integration** | ✅ Rich ecosystem | ✅ Both ecosystems | ⚠️ Growing ecosystem |
| **Deployment simplicity** | ✅ pip install | ✅ pip install (pre-built) | ⚠️ Requires compilation |
| **Incremental processing** | ⚠️ Complex to implement | ✅ Rust implementation | ✅ Native support |
| **Concurrency** | ❌ GIL limitations | ✅ True parallelism | ✅ True parallelism |

The hybrid approach allows CocoIndex to:

1. **Provide Python ergonomics** for flow definition while executing transformations in Rust without GIL constraints
2. **Leverage Python's ecosystem** for data science libraries (NumPy, pandas) while using Rust for I/O-bound operations
3. **Distribute pre-compiled binaries** so users don't need Rust toolchains
4. **Implement complex state management** in Rust with type safety while exposing a simple Python API

**Sources:** [README.md:1-100]() (conceptual), [Cargo.toml:1-124](), [pyproject.toml:1-143]()

---

## Testing and Development Workflow

The CI/CD pipeline tests both Python and Rust components across platforms:

```mermaid
graph TB
    subgraph "Local Development"
        DevCode["Edit Python/Rust code"]
        UvRun["uv run python"]
        PreCommit["pre-commit hooks"]
    end
    
    subgraph "CI Pipeline (.github/workflows/)"
        FormatCheck["format.yml<br/>Rust fmt + Python ruff"]
        Test["_test.yml<br/>Cross-platform tests"]
        Release["release.yml<br/>Build + publish wheels"]
    end
    
    subgraph "Test Execution"
        RustTest["cargo test"]
        MaturinDev["maturin develop"]
        PythonTest["pytest"]
        MyPy["mypy"]
    end
    
    DevCode --> PreCommit
    PreCommit --> UvRun
    
    DevCode --> FormatCheck
    DevCode --> Test
    Test --> RustTest
    Test --> MaturinDev
    Test --> PythonTest
    Test --> MyPy
    
    Test --> Release
```

The test workflow runs on multiple platforms (Linux x86_64/ARM, macOS Intel/ARM, Windows) to ensure compatibility:

**Sources:** [.github/workflows/_test.yml:1-99](), [.github/workflows/CI.yml:1-32](), [.github/workflows/format.yml:1-56]()

---

## Summary

CocoIndex's three-layer architecture separates concerns effectively:

- **Python Layer**: User-facing API, flow definition, configuration management
- **FFI Boundary**: Type marshalling, async coordination, error translation via PyO3
- **Rust Core**: Performance-critical execution, state management, concurrent processing

This architecture enables CocoIndex to provide a Python-native developer experience while achieving Rust-level performance for incremental data processing. The build system produces platform-specific wheels with embedded Rust extensions, requiring no compilation from users.

For detailed information about specific subsystems, see:
- Flow definition and lifecycle: [Flow Definition System](#3)
- Type system and marshalling: [Operation System and Type Marshalling](#4)
- Rust engine internals: [Rust Engine Architecture](#12)
- Build system details: [Build System: Maturin and Cross-Platform Wheels](#12.2)

**Sources:** [python/cocoindex/__init__.py:1-128](), [rust/cocoindex/src/lib_context.rs:1-300](), [pyproject.toml:1-143](), [Cargo.toml:1-124]()

---

# Page: Flow Definition System

# Flow Definition and Python API

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [docs/docs/core/basics.md](docs/docs/core/basics.md)
- [docs/docs/core/flow_def.mdx](docs/docs/core/flow_def.mdx)
- [docs/docs/core/flow_methods.mdx](docs/docs/core/flow_methods.mdx)
- [python/cocoindex/__init__.py](python/cocoindex/__init__.py)
- [python/cocoindex/cli.py](python/cocoindex/cli.py)
- [python/cocoindex/flow.py](python/cocoindex/flow.py)
- [python/cocoindex/lib.py](python/cocoindex/lib.py)
- [python/cocoindex/setup.py](python/cocoindex/setup.py)

</details>



This document covers CocoIndex's Python API for defining data processing flows, including the core classes and decorators used to build indexing pipelines. It explains how to use `@flow_def`, `FlowBuilder`, `DataScope`, `DataSlice`, and related components to create flows that extract, transform, and export data.

For information about how flows are executed and analyzed by the engine, see [Flow Analysis and Execution Engine](#2.2). For details about type handling between Python and Rust, see [Type System and Data Marshalling](#2.3).

## Flow Definition Overview

CocoIndex flows are defined using Python functions decorated with `@flow_def` or created explicitly with `open_flow()`. These functions receive a `FlowBuilder` and root `DataScope` to construct the data processing pipeline.

### Flow Definition Architecture

```mermaid
graph TD
    User["User Code"] --> FlowDef["@flow_def decorator"]
    User --> OpenFlow["cocoindex.open_flow()"]
    
    FlowDef --> FlowObj["Flow object"]
    OpenFlow --> FlowObj
    
    FlowObj --> FlowBuilder["FlowBuilder"]
    FlowObj --> RootScope["Root DataScope"]
    
    FlowBuilder --> AddSource["add_source()"]
    FlowBuilder --> Transform["transform()"]
    FlowBuilder --> Declare["declare()"]
    
    RootScope --> Fields["Fields (DataSlice)"]
    RootScope --> Collectors["DataCollector"]
    
    Fields --> RowScope["row() → DataScope"]
    Fields --> TransformOp["transform()"]
    
    Collectors --> Collect["collect()"]
    Collectors --> Export["export()"]
```

**Flow Definition Components Mapping**

Sources: [python/cocoindex/flow.py:893-899](), [python/cocoindex/flow.py:868-876](), [python/cocoindex/flow.py:470-546](), [python/cocoindex/flow.py:287-341]()

## The @flow_def Decorator

The `@flow_def` decorator is the primary way to define flows. It wraps a function that defines the flow's data processing logic.

### Basic Usage

```python
@cocoindex.flow_def(name="MyFlow")
def my_flow(flow_builder: cocoindex.FlowBuilder, data_scope: cocoindex.DataScope):
    # Define your flow here
    pass
```

The decorator creates a `Flow` object that can be used for updates, setup, and other operations. The decorated function receives:
- `flow_builder`: A `FlowBuilder` instance for adding sources and transformations
- `data_scope`: The root `DataScope` for managing fields and collectors

### Alternative: open_flow()

For dynamic flow creation, use `open_flow()` directly:

```python
def flow_definition(flow_builder, data_scope):
    # Flow logic here
    pass

my_flow = cocoindex.open_flow("MyFlow", flow_definition)
```

Sources: [python/cocoindex/flow.py:893-899](), [python/cocoindex/flow.py:868-876]()

## FlowBuilder Class

`FlowBuilder` provides methods to add data sources, apply transformations, and make declarations. It serves as the entry point for constructing the flow's data pipeline.

### FlowBuilder Methods

```mermaid
graph TD
    FlowBuilder["FlowBuilder"] --> AddSource["add_source(spec, **options)"]
    FlowBuilder --> Transform["transform(fn_spec, *args, **kwargs)"]
    FlowBuilder --> Declare["declare(spec)"]
    
    AddSource --> SourceOptions["Options:<br/>• name<br/>• refresh_interval<br/>• max_inflight_rows<br/>• max_inflight_bytes"]
    
    Transform --> TransformArgs["Arguments:<br/>• fn_spec (FunctionSpec)<br/>• *args (positional)<br/>• **kwargs (keyword)"]
    
    Declare --> DeclareSpecs["Declaration specs:<br/>• Target declarations<br/>• Schema declarations"]
    
    AddSource --> DataSlice["Returns DataSlice"]
    Transform --> DataSlice
```

**FlowBuilder Method Details**

### add_source()

Imports data from external sources into the flow:

```python
data_scope["documents"] = flow_builder.add_source(
    cocoindex.sources.LocalFile(path="/data"),
    refresh_interval=datetime.timedelta(minutes=5),
    max_inflight_rows=100
)
```

Parameters:
- `spec`: A `SourceSpec` defining the data source
- `name`: Optional name for the source (auto-generated if not provided)
- `refresh_interval`: How often to check for source changes in live mode
- `max_inflight_rows/max_inflight_bytes`: Concurrency limits

### transform()

Applies transformations to data:

```python
result = flow_builder.transform(
    cocoindex.functions.SplitRecursively(),
    input_data,
    param1="value1"
)
```

### declare()

Makes declarations for target configurations:

```python
flow_builder.declare(
    cocoindex.targets.Neo4jDeclarations(
        connection=graph_conn,
        nodes_label="Document",
        primary_key_fields=["id"]
    )
)
```

Sources: [python/cocoindex/flow.py:470-546]()

## DataScope Class

`DataScope` represents a scope of data within the flow, containing fields and collectors. It provides the context for data operations and can create child scopes.

### DataScope Operations

```mermaid
graph TD
    DataScope["DataScope"] --> GetField["scope['field_name']"]
    DataScope --> SetField["scope['field_name'] = data_slice"]
    DataScope --> AddCollector["add_collector()"]
    DataScope --> ContextMgr["__enter__/__exit__"]
    
    GetField --> DataSlice["Returns DataSlice"]
    SetField --> Validation["Field name validation"]
    AddCollector --> DataCollector["Returns DataCollector"]
    
    DataSlice --> RowMethod["row() method"]
    RowMethod --> ChildScope["Returns child DataScope"]
    ChildScope --> ForEach["for_each() iteration"]
```

**DataScope Field Management**

Fields in a `DataScope` are accessed and set using dictionary-like syntax:

```python
# Get a field (returns DataSlice)
documents = data_scope["documents"]

# Set a field
data_scope["processed_docs"] = transformed_data
```

### Row Iteration

For table-type data, use the `row()` method to create child scopes:

```python
with data_scope["documents"].row() as doc:
    # doc is a DataScope representing each document
    doc["summary"] = doc["content"].transform(summarize_func)
```

The `row()` method accepts concurrency parameters:
- `max_inflight_rows`: Maximum concurrent rows being processed
- `max_inflight_bytes`: Maximum memory footprint for concurrent processing

Sources: [python/cocoindex/flow.py:287-341]()

## DataSlice Class

`DataSlice` represents a slice of data within a flow. It's the primary data representation that flows between operations and can be transformed or used as input to other operations.

### DataSlice Operations

```mermaid
graph TD
    DataSlice["DataSlice[T]"] --> Transform["transform(fn_spec, *args, **kwargs)"]
    DataSlice --> GetField["slice['field_name']"]
    DataSlice --> RowMethod["row(**options)"]
    DataSlice --> ForEach["for_each(f, **options)"]
    DataSlice --> Call["call(func, *args, **kwargs)"]
    
    Transform --> NewSlice["Returns new DataSlice"]
    GetField --> SubSlice["Returns field DataSlice"]
    RowMethod --> RowScope["Returns DataScope for iteration"]
    ForEach --> Iteration["Applies function to each row"]
    Call --> FuncResult["Calls function with slice as first arg"]
    
    RowScope --> ConcurrencyOpts["Concurrency options:<br/>• max_inflight_rows<br/>• max_inflight_bytes"]
```

**DataSlice Usage Patterns**

### Field Access
For struct-type data, access sub-fields:
```python
content = document["content"]  # Get content field from document
```

### Transformations
Apply functions to transform data:
```python
chunks = content.transform(
    cocoindex.functions.SplitRecursively(),
    chunk_size=1000
)
```

### Row Processing
Process each row of table data:
```python
# Using row() method
with documents.row() as doc:
    process_document(doc)

# Using for_each() helper
documents.for_each(lambda doc: process_document(doc))
```

Sources: [python/cocoindex/flow.py:197-281]()

## DataCollector Class

`DataCollector` accumulates data from various sources and exports it to targets. Collectors are created from `DataScope` instances and provide the bridge between processed data and external storage systems.

### DataCollector Workflow

```mermaid
graph TD
    DataScope["DataScope"] --> AddCollector["add_collector(name)"]
    AddCollector --> DataCollector["DataCollector"]
    
    DataCollector --> Collect["collect(**fields)"]
    DataCollector --> Export["export(name, target_spec, **options)"]
    
    Collect --> FieldTypes["Field types:<br/>• DataSlice values<br/>• GeneratedField.UUID<br/>• Mixed field types"]
    
    Export --> TargetSpec["Target specifications:<br/>• Postgres<br/>• Qdrant<br/>• Neo4j/Kuzu"]
    Export --> IndexOptions["Index options:<br/>• primary_key_fields<br/>• vector_indexes<br/>• setup_by_user"]
```

**DataCollector Usage**

### Collecting Data
```python
collector = data_scope.add_collector("my_collector")

# In a loop or scope, collect data entries
collector.collect(
    id=cocoindex.GeneratedField.UUID,  # Auto-generated UUID
    filename=doc["filename"],          # From DataSlice
    content=doc["processed_content"],  # From DataSlice
    timestamp=current_time            # Constant value
)
```

### Exporting to Targets
```python
collector.export(
    "my_target",
    cocoindex.targets.Postgres(table_name="documents"),
    primary_key_fields=["id"],
    vector_indexes=[
        cocoindex.VectorIndexDef("embedding", cocoindex.VectorSimilarityMetric.COSINE_SIMILARITY)
    ]
)
```

Sources: [python/cocoindex/flow.py:352-428]()

## Transform Flows

`TransformFlow` enables in-memory data processing separate from persistent flows. It's useful for testing transformations or processing data without setting up full indexing pipelines.

### TransformFlow Architecture

```mermaid
graph TD
    TransformFunc["Transform Function"] --> Decorator["@transform_flow"]
    TransformFunc --> DirectCreate["TransformFlow(func)"]
    
    Decorator --> TransformFlowObj["TransformFlow[T]"]
    DirectCreate --> TransformFlowObj
    
    TransformFlowObj --> TypeAnalysis["Type analysis of parameters"]
    TransformFlowObj --> FlowBuilding["Build internal flow"]
    TransformFlowObj --> Eval["eval(*args, **kwargs)"]
    
    TypeAnalysis --> ParamValidation["Parameter validation:<br/>• DataSlice[T] type hints<br/>• Argument matching"]
    
    FlowBuilding --> EngineFlow["_engine.TransientFlow"]
    EngineFlow --> InMemoryExec["In-memory execution"]
    
    Eval --> TypedResult["Returns T (typed result)"]
```

**TransformFlow Usage**

### Defining Transform Functions
```python
@cocoindex.transform_flow()
def process_text(content: cocoindex.DataSlice[str]) -> cocoindex.DataSlice[list[str]]:
    return content.transform(cocoindex.functions.SplitRecursively())

# Usage
result = process_text.eval("Some long text to process")
```

### Manual TransformFlow Creation
```python
def my_transform(data: cocoindex.DataSlice[str]) -> cocoindex.DataSlice[int]:
    return data.transform(lambda x: len(x))

transform_flow = cocoindex.TransformFlow(my_transform)
result = transform_flow.eval("hello world")  # Returns 11
```

Sources: [python/cocoindex/flow.py:995-1143]()

## Flow State Management

Flows maintain both in-memory state and persistent resources. The system provides APIs to manage these lifecycle aspects.

### Flow Lifecycle Operations

```mermaid
graph TD
    FlowDef["Flow Definition"] --> CreateFlow["Flow Creation"]
    CreateFlow --> FlowObj["Flow Object"]
    
    FlowObj --> Setup["setup() / setup_async()"]
    FlowObj --> Update["update() / update_async()"]
    FlowObj --> Drop["drop() / drop_async()"]
    FlowObj --> Close["close()"]
    
    Setup --> PersistentRes["Persistent Resources:<br/>• Database tables<br/>• Target backends<br/>• Internal storage"]
    
    Update --> DataProcessing["Data Processing:<br/>• Source ingestion<br/>• Transformations<br/>• Target updates"]
    
    Drop --> Cleanup["Resource Cleanup:<br/>• Drop tables<br/>• Clean backends"]
    
    Close --> MemoryFree["Memory Cleanup:<br/>• Remove from registry<br/>• Free resources"]
```

**Flow Management Functions**

The system also provides global flow management:

- `flow_names()`: List all registered flow names
- `flows()`: Get dictionary of all flows
- `flow_by_name(name)`: Get specific flow by name
- `setup_all_flows()`: Setup all registered flows
- `drop_all_flows()`: Drop all registered flows

Sources: [python/cocoindex/flow.py:672-831](), [python/cocoindex/flow.py:902-924](), [python/cocoindex/flow.py:1167-1178]()

## Integration with Operation Registry

The Flow API integrates closely with CocoIndex's operation registry system, which provides pluggable sources, functions, and targets.

### Operation Spec Integration

```mermaid
graph TD
    FlowBuilder --> AddSource["add_source(SourceSpec)"]
    DataSlice --> Transform["transform(FunctionSpec)"]
    DataCollector --> Export["export(name, TargetSpec)"]
    
    AddSource --> SourceRegistry["Source Factory Registry"]
    Transform --> FunctionRegistry["Function Factory Registry"]  
    Export --> TargetRegistry["Target Factory Registry"]
    
    SourceRegistry --> SourceTypes["Built-in sources:<br/>• LocalFile<br/>• AmazonS3<br/>• GoogleDrive<br/>• AzureBlob"]
    
    FunctionRegistry --> FunctionTypes["Built-in functions:<br/>• SplitRecursively<br/>• EmbedText<br/>• ExtractByLlm<br/>• ParseJson"]
    
    TargetRegistry --> TargetTypes["Built-in targets:<br/>• Postgres<br/>• Qdrant<br/>• Neo4j<br/>• Kuzu"]
```

**Operation Spec Classes**

All operation specs inherit from base classes defined in the operation registry:
- `SourceSpec`: Base for data sources
- `FunctionSpec`: Base for transformation functions  
- `TargetSpec`: Base for export targets
- `DeclarationSpec`: Base for target declarations

Sources: [python/cocoindex/flow.py:486-521](), [python/cocoindex/flow.py:523-539](), [python/cocoindex/flow.py:387-428]()

---

# Page: DataScope and DataSlice

# DataScope and DataSlice

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [docs/docs/core/basics.md](docs/docs/core/basics.md)
- [docs/docs/core/flow_def.mdx](docs/docs/core/flow_def.mdx)
- [docs/docs/core/flow_methods.mdx](docs/docs/core/flow_methods.mdx)
- [python/cocoindex/__init__.py](python/cocoindex/__init__.py)
- [python/cocoindex/cli.py](python/cocoindex/cli.py)
- [python/cocoindex/flow.py](python/cocoindex/flow.py)
- [python/cocoindex/lib.py](python/cocoindex/lib.py)
- [python/cocoindex/setup.py](python/cocoindex/setup.py)

</details>



## Purpose and Scope

This document explains **DataScope** and **DataSlice**, the two fundamental abstractions for representing and manipulating data within CocoIndex flows. These types form the core interface through which users define data transformations and organize data hierarchically.

- **DataSlice** represents a typed reference to data within a flow (e.g., a field, a transformation result)
- **DataScope** represents a container that organizes multiple DataSlices as fields and manages data collectors

For information about defining complete flows using these constructs, see [Flow Definition System](#3). For details on the operations that produce and consume DataSlices, see [Operations: Source, Transform, Collect, Export](#3.2).

---

## Overview: Data Organization Model

CocoIndex organizes data hierarchically using scopes and slices. Understanding their relationship is essential to defining flows.

**Diagram: DataScope and DataSlice Relationship**

```mermaid
graph TB
    FlowBuilder["FlowBuilder"]
    RootScope["DataScope<br/>(root scope)"]
    
    subgraph "Top-Level Data Organization"
        Field1["DataSlice&lt;Table&gt;<br/>documents"]
        Field2["DataSlice&lt;...&gt;<br/>other_field"]
        Collector1["DataCollector<br/>results"]
    end
    
    subgraph "Nested Scope (per row)"
        RowScope["DataScope<br/>(document scope)"]
        RowField1["DataSlice&lt;String&gt;<br/>content"]
        RowField2["DataSlice&lt;Table&gt;<br/>chunks"]
        RowCollector["DataCollector<br/>doc_collector"]
    end
    
    FlowBuilder -->|"add_source()"| Field1
    FlowBuilder -->|"transform()"| Field2
    
    RootScope -->|"data_scope['documents']"| Field1
    RootScope -->|"data_scope['other_field']"| Field2
    RootScope -->|"add_collector()"| Collector1
    
    Field1 -->|"row()"| RowScope
    
    RowScope -->|"doc['content']"| RowField1
    RowScope -->|"doc['chunks']"| RowField2
    RowScope -->|"add_collector()"| RowCollector
    
    RowField1 -->|"transform()"| RowField2
```

Sources: [python/cocoindex/flow.py:302-357](), [python/cocoindex/flow.py:212-296](), [docs/docs/core/flow_def.mdx:67-103]()

---

## DataSlice

A `DataSlice[T]` represents a typed, immutable reference to data within a flow. It serves as the primary data representation passed between operations.

### Type Parameterization

DataSlice is a generic type parameterized by the data type it contains:

```python
DataSlice[str]              # String data
DataSlice[Vector[Float32, 384]]  # Vector embedding
DataSlice[KTable[...]]      # Table with key columns
```

The type parameter `T` is used for type hints but does not affect runtime behavior—type information is tracked by the underlying Rust engine.

Sources: [python/cocoindex/flow.py:212-219]()

### Core Properties

| Property | Description |
|----------|-------------|
| **Immutable** | DataSlices are read-only; transformations create new DataSlices |
| **Lazy** | May not be computed until attached to a scope or accessed |
| **Typed** | Carries type information for validation and encoding |
| **Composable** | Can be transformed, iterated, or nested |

Sources: [python/cocoindex/flow.py:212-296](), [docs/docs/core/flow_def.mdx:109-140]()

### Key Methods

**Diagram: DataSlice Method Categories**

```mermaid
graph LR
    DataSlice["DataSlice[T]"]
    
    subgraph "Data Access"
        GetItem["__getitem__(field_name)<br/>→ DataSlice"]
    end
    
    subgraph "Iteration"
        Row["row()<br/>→ DataScope"]
        ForEach["for_each(f)"]
    end
    
    subgraph "Transformation"
        Transform["transform(fn_spec, *args, **kwargs)<br/>→ DataSlice"]
        Call["call(func, *args, **kwargs)<br/>→ Any"]
    end
    
    DataSlice --> GetItem
    DataSlice --> Row
    DataSlice --> ForEach
    DataSlice --> Transform
    DataSlice --> Call
```

Sources: [python/cocoindex/flow.py:226-296]()

#### Field Access: `__getitem__(field_name: str)`

For DataSlices with `Struct` type, access sub-fields by name:

```python
# Access the 'content' field from a document struct
document_content = document_slice["content"]
```

Raises `KeyError` if the field does not exist.

Sources: [python/cocoindex/flow.py:226-230]()

#### Row Iteration: `row()`

For DataSlices with table types (`KTable` or `LTable`), create a child scope representing each row:

```python
with documents_slice.row() as doc:
    # Operations within this scope apply to each document row
    doc["summary"] = doc["content"].transform(Summarize(...))
```

The `row()` method accepts optional concurrency control parameters:
- `max_inflight_rows`: Limit concurrent row processing
- `max_inflight_bytes`: Limit memory footprint of concurrent rows

Sources: [python/cocoindex/flow.py:232-251]()

#### Convenience: `for_each(f: Callable[[DataScope], None])`

Shorthand for iterating rows without explicit context manager:

```python
documents_slice.for_each(lambda doc: doc["summary"] = doc["content"].transform(...))
```

Sources: [python/cocoindex/flow.py:253-268]()

#### Transformation: `transform(fn_spec, *args, **kwargs)`

Apply a function to the DataSlice, producing a new DataSlice:

```python
# First argument is implicit (the slice being transformed)
embeddings = text_slice.transform(SentenceTransformerEmbed(...))

# Additional arguments can be positional or keyword
result = data_slice.transform(CustomFunction(...), arg1, arg2, key=kwarg)
```

Sources: [python/cocoindex/flow.py:270-289]()

#### Custom Processing: `call(func, *args, **kwargs)`

Invoke an arbitrary function with the DataSlice as the first argument:

```python
custom_result = data_slice.call(my_custom_processor, extra_arg)
# Equivalent to: my_custom_processor(data_slice, extra_arg)
```

Sources: [python/cocoindex/flow.py:291-295]()

### Internal State and Lazy Evaluation

**Diagram: DataSlice Lazy Evaluation**

```mermaid
stateDiagram-v2
    [*] --> Lazy: Create with creator function
    [*] --> Eager: Create with engine DataSlice
    
    Lazy --> Materializing: Access engine_data_slice
    Materializing --> Materialized: Creator invoked
    
    Eager --> Ready: Already has engine DataSlice
    Materialized --> Ready
    
    Ready --> [*]: Can be used in operations
```

DataSlices use `_DataSliceState` internally, which can operate in two modes:

1. **Eager Mode**: The underlying `_engine.DataSlice` is immediately available
2. **Lazy Mode**: Uses a creator function that produces `_engine.DataSlice` when needed

Lazy evaluation enables DataSlices to be created without immediately determining their attachment point in the flow graph. The attachment is resolved when the DataSlice is assigned to a scope field via `__setitem__`.

Sources: [python/cocoindex/flow.py:149-210]()

**Table: _DataSliceState Fields**

| Field | Type | Purpose |
|-------|------|---------|
| `flow_builder_state` | `_FlowBuilderState` | Reference to parent flow builder |
| `_data_slice` | `_engine.DataSlice \| None` | Materialized Rust engine data slice |
| `_data_slice_creator` | `Callable \| None` | Lazy creator function |
| `_lazy_lock` | `Lock \| None` | Synchronization for lazy materialization |

Sources: [python/cocoindex/flow.py:149-157]()

---

## DataScope

A `DataScope` represents a container that organizes data within a flow. Scopes exist hierarchically: the root scope contains all top-level data, and child scopes are created via `row()` iteration.

### Core Responsibilities

| Responsibility | Description |
|----------------|-------------|
| **Field Management** | Store and retrieve named DataSlices |
| **Collector Management** | Create DataCollectors for aggregating data |
| **Hierarchy** | Create child scopes for row-wise operations |
| **Context Management** | Lifecycle control via context manager protocol |

Sources: [python/cocoindex/flow.py:302-357](), [docs/docs/core/flow_def.mdx:67-82]()

### Key Methods

#### Field Access: `__getitem__(field_name: str)` and `__setitem__(field_name: str, value: DataSlice[T])`

Access existing fields or assign new DataSlices to fields:

```python
# Assign a DataSlice to a field
data_scope["documents"] = flow_builder.add_source(LocalFile(...))

# Retrieve the field
docs = data_scope["documents"]

# Nested field access
content = data_scope["documents"]["content"]
```

**Important**: You cannot override an existing field. Field names must be valid identifiers (validated by `validate_field_name`).

Sources: [python/cocoindex/flow.py:323-337](), [docs/docs/core/flow_def.mdx:73-103]()

#### Collector Creation: `add_collector(name: str | None = None)`

Create a `DataCollector` for aggregating data within this scope:

```python
collector = data_scope.add_collector()
# or with explicit name
collector = data_scope.add_collector("my_collector")

# Later collect data
collector.collect(id=GeneratedField.UUID, text=doc["content"])
```

If `name` is `None`, an auto-generated name is assigned (e.g., `_collector_0`, `_collector_1`).

Sources: [python/cocoindex/flow.py:345-356](), [docs/docs/core/flow_def.mdx:218-220]()

#### Context Manager: `__enter__()` and `__exit__()`

DataScopes support the context manager protocol, primarily used with `row()`:

```python
with data_slice.row() as row_scope:
    # Operations within this scope
    row_scope["field"] = ...
# row_scope is cleaned up on exit
```

The `__exit__` method deletes the internal `_engine_data_scope` reference to aid cleanup.

Sources: [python/cocoindex/flow.py:339-343]()

### Internal Structure

**Diagram: DataScope Internal Components**

```mermaid
graph TB
    DataScope["DataScope"]
    FlowBuilderState["_FlowBuilderState"]
    EngineDataScope["_engine.DataScopeRef<br/>(Rust)"]
    NameBuilder["_NameBuilder"]
    
    DataScope -->|"_flow_builder_state"| FlowBuilderState
    DataScope -->|"_engine_data_scope"| EngineDataScope
    
    FlowBuilderState -->|"field_name_builder"| NameBuilder
    FlowBuilderState -->|"engine_flow_builder"| EngineFlowBuilder["_engine.FlowBuilder<br/>(Rust)"]
    
    EngineDataScope -->|"add_collector()"| EngineCollector["_engine.DataCollector"]
    EngineFlowBuilder -->|"scope_field()"| EngineDataSlice["_engine.DataSlice"]
```

Sources: [python/cocoindex/flow.py:302-357](), [python/cocoindex/flow.py:456-477]()

**Table: DataScope Fields**

| Field | Type | Purpose |
|-------|------|---------|
| `_flow_builder_state` | `_FlowBuilderState` | Shared state for flow construction |
| `_engine_data_scope` | `_engine.DataScopeRef` | Reference to Rust engine's scope representation |

Sources: [python/cocoindex/flow.py:308-315]()

---

## Hierarchical Data Organization

DataScope and DataSlice work together to create hierarchical data structures within flows. Understanding this hierarchy is key to writing complex transformations.

**Diagram: Nested Scope Hierarchy Example**

```mermaid
graph TB
    subgraph "Root Scope"
        RootScope["DataScope<br/>(root)"]
        DocumentsSlice["DataSlice&lt;KTable&gt;<br/>documents"]
    end
    
    subgraph "Document Scope (for each document)"
        DocScope["DataScope<br/>(per document)"]
        FilenameSlice["DataSlice&lt;String&gt;<br/>filename"]
        ContentSlice["DataSlice&lt;String&gt;<br/>content"]
        ChunksSlice["DataSlice&lt;KTable&gt;<br/>chunks"]
    end
    
    subgraph "Chunk Scope (for each chunk)"
        ChunkScope["DataScope<br/>(per chunk)"]
        LocationSlice["DataSlice&lt;Int64&gt;<br/>location"]
        TextSlice["DataSlice&lt;String&gt;<br/>text"]
        EmbeddingSlice["DataSlice&lt;Vector&gt;<br/>embedding"]
    end
    
    RootScope -->|"['documents']"| DocumentsSlice
    DocumentsSlice -->|"row()"| DocScope
    
    DocScope -->|"['filename']"| FilenameSlice
    DocScope -->|"['content']"| ContentSlice
    DocScope -->|"['chunks']"| ChunksSlice
    
    ChunksSlice -->|"row()"| ChunkScope
    
    ChunkScope -->|"['location']"| LocationSlice
    ChunkScope -->|"['text']"| TextSlice
    ChunkScope -->|"['embedding']"| EmbeddingSlice
```

Sources: [docs/docs/core/flow_def.mdx:67-103](), [docs/docs/core/flow_def.mdx:194-211]()

### Typical Flow Pattern

1. **Root level**: Add sources and define top-level fields
2. **Row iteration**: Use `row()` to create child scopes for each table row
3. **Field operations**: Within each scope, access fields and apply transformations
4. **Nested iteration**: Repeat for nested table structures
5. **Collection**: Use `add_collector()` at appropriate scope levels to aggregate results

Example code structure:

```python
@flow_def(name="Example")
def example_flow(flow_builder: FlowBuilder, root: DataScope):
    # 1. Root level
    root["documents"] = flow_builder.add_source(LocalFile(...))
    
    # 2. Row iteration
    with root["documents"].row() as doc:
        # 3. Field operations
        doc["chunks"] = doc["content"].transform(SplitRecursively(...))
        
        # 4. Nested iteration
        with doc["chunks"].row() as chunk:
            chunk["embedding"] = chunk["text"].transform(Embed(...))
            
            # 5. Collection
            collector = root.add_collector()
            collector.collect(
                doc_id=doc["filename"],
                chunk_loc=chunk["location"],
                vector=chunk["embedding"]
            )
```

Sources: [docs/docs/core/flow_def.mdx:88-99](), [docs/docs/core/flow_def.mdx:199-211]()

---

## Relationship to Rust Engine

The Python `DataScope` and `DataSlice` types are thin wrappers around Rust engine types, connected via PyO3 bindings.

**Diagram: Python-Rust Type Mapping**

```mermaid
graph LR
    subgraph "Python Layer"
        PyDataScope["DataScope"]
        PyDataSlice["DataSlice[T]"]
        PyDataSliceState["_DataSliceState"]
        PyDataCollector["DataCollector"]
    end
    
    subgraph "FFI Boundary (_engine module)"
        EngineDataScopeRef["_engine.DataScopeRef"]
        EngineDataSlice["_engine.DataSlice"]
        EngineDataCollector["_engine.DataCollector"]
    end
    
    subgraph "Rust Core (cocoindex_engine)"
        RustDataScopeRef["DataScopeRef"]
        RustDataSlice["DataSlice"]
        RustDataCollector["DataCollector"]
    end
    
    PyDataScope -->|"_engine_data_scope"| EngineDataScopeRef
    PyDataSliceState -->|"_data_slice"| EngineDataSlice
    PyDataCollector -->|"_engine_data_collector"| EngineDataCollector
    
    EngineDataScopeRef -.PyO3.-> RustDataScopeRef
    EngineDataSlice -.PyO3.-> RustDataSlice
    EngineDataCollector -.PyO3.-> RustDataCollector
```

Sources: [python/cocoindex/flow.py:302-357](), [python/cocoindex/flow.py:149-210](), [python/cocoindex/flow.py:367-380]()

### Key Integration Points

| Python Type | Rust Engine Type | Connection |
|-------------|------------------|------------|
| `DataScope._engine_data_scope` | `_engine.DataScopeRef` | Reference to Rust scope |
| `_DataSliceState._data_slice` | `_engine.DataSlice` | Materialized Rust data slice |
| `DataCollector._engine_data_collector` | `_engine.DataCollector` | Reference to Rust collector |

The Rust engine maintains the actual dataflow graph and execution logic, while Python types provide an ergonomic API for flow definition.

Sources: [python/cocoindex/flow.py:308-315](), [python/cocoindex/flow.py:149-210](), [python/cocoindex/flow.py:370-379]()

---

## DataCollector

While not a DataScope or DataSlice, `DataCollector` is closely related—it's created from a DataScope and collects DataSlices into exportable collections.

### Purpose

Aggregates multiple entries of data (each composed of multiple fields) from the same or child scopes, preparing them for export to targets.

### Key Methods

#### `collect(**kwargs: Any)`

Collect a single entry with named fields:

```python
collector.collect(
    id=GeneratedField.UUID,          # Auto-generated stable UUID
    filename=doc["filename"],         # DataSlice reference
    embedding=chunk["embedding"]      # DataSlice reference
)
```

Field values can be:
- A `DataSlice` (most common)
- `GeneratedField.UUID` for auto-generated, stable UUIDs (limited to one per collect call)

Sources: [python/cocoindex/flow.py:381-400]()

#### `export(target_name, target_spec, *, primary_key_fields, ...)`

Export collected data to an external target:

```python
collector.export(
    "my_index",
    Postgres(...),
    primary_key_fields=["id"],
    vector_indexes=[VectorIndexDef("embedding", VectorSimilarityMetric.COSINE_SIMILARITY)]
)
```

Parameters include:
- `target_name`: Identifier for the export target
- `target_spec`: Target specification (e.g., `Postgres`, `Qdrant`)
- `primary_key_fields`: Required list of primary key field names
- `attachments`: Optional target-specific attachments
- `vector_indexes`, `fts_indexes`: Optional index specifications
- `setup_by_user`: Whether CocoIndex manages target setup (default: `False`)

Sources: [python/cocoindex/flow.py:402-450](), [docs/docs/core/flow_def.mdx:262-296]()

### Internal Structure

**Table: DataCollector Fields**

| Field | Type | Purpose |
|-------|------|---------|
| `_flow_builder_state` | `_FlowBuilderState` | Reference to parent flow builder |
| `_engine_data_collector` | `_engine.DataCollector` | Reference to Rust engine's collector |

Sources: [python/cocoindex/flow.py:367-379]()

---

## Concurrency Control

Both `FlowBuilder.add_source()` and `DataSlice.row()` accept concurrency control parameters to prevent system overload during parallel processing.

### Parameters

| Parameter | Type | Effect |
|-----------|------|--------|
| `max_inflight_rows` | `int \| None` | Maximum concurrent rows being processed |
| `max_inflight_bytes` | `int \| None` | Maximum memory footprint of concurrent data |

### Usage

```python
# Limit source reading
root["docs"] = flow_builder.add_source(
    LocalFile(...),
    max_inflight_rows=10,
    max_inflight_bytes=100*1024*1024  # 100 MB
)

# Limit nested iteration
with root["docs"].row(max_inflight_rows=5) as doc:
    doc["chunks"] = doc["content"].transform(...)
    
    with doc["chunks"].row(max_inflight_rows=100) as chunk:
        # Finer-grained control for chunk processing
        ...
```

Global limits can also be set via `GlobalExecutionOptions` or environment variables (`COCOINDEX_SOURCE_MAX_INFLIGHT_ROWS`, `COCOINDEX_SOURCE_MAX_INFLIGHT_BYTES`). When both global and local limits are specified, both are enforced independently.

Sources: [python/cocoindex/flow.py:232-251](), [python/cocoindex/flow.py:511-546](), [docs/docs/core/flow_def.mdx:402-452]()

---

## Field Naming and Auto-Generation

DataScopes and DataCollectors use `_NameBuilder` to manage field and collector names, automatically generating unique names when not explicitly provided.

### Auto-Generated Names

When `name=None` is passed to operations that accept names:

| Operation | Prefix | Example Generated Names |
|-----------|--------|------------------------|
| `add_source(LocalFile(...))` | `local_file_` | `local_file_0`, `local_file_1` |
| `transform(SplitRecursively(...))` | `split_recursively_` | `split_recursively_0` |
| `add_collector()` | `_collector_` | `_collector_0`, `_collector_1` |

The prefix is derived from the operation spec's class name converted to snake_case.

Sources: [python/cocoindex/flow.py:52-76](), [python/cocoindex/flow.py:78-82](), [python/cocoindex/flow.py:511-546]()

### Manual Naming

Explicit names provide better readability and stability:

```python
root["documents"] = flow_builder.add_source(LocalFile(...), name="documents")
collector = root.add_collector(name="doc_embeddings")
```

Sources: [python/cocoindex/flow.py:345-356](), [python/cocoindex/flow.py:511-520]()

---

## Summary

**DataSlice** and **DataScope** form the foundation of CocoIndex's data representation:

- **DataSlice[T]**: Typed, immutable reference to data; supports transformation, iteration, and field access
- **DataScope**: Container for organizing DataSlices as fields and managing DataCollectors
- **Hierarchy**: Scopes nest via `row()`, creating parent-child relationships for structured data processing
- **Integration**: Python wrappers delegate to Rust engine types via PyO3 for performance
- **Flexibility**: Lazy evaluation and auto-naming simplify flow definition while maintaining efficiency

These abstractions enable declarative, type-safe flow definitions while the Rust engine handles execution, caching, and incremental processing.

Sources: [python/cocoindex/flow.py:212-357](), [docs/docs/core/flow_def.mdx:67-140]()

---

# Page: Operations: Source, Transform, Collect, Export

# Operations: Source, Transform, Collect, Export

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [docs/docs/core/basics.md](docs/docs/core/basics.md)
- [docs/docs/core/data_types.mdx](docs/docs/core/data_types.mdx)
- [docs/docs/core/flow_def.mdx](docs/docs/core/flow_def.mdx)
- [docs/docs/core/flow_methods.mdx](docs/docs/core/flow_methods.mdx)
- [python/cocoindex/__init__.py](python/cocoindex/__init__.py)
- [python/cocoindex/cli.py](python/cocoindex/cli.py)
- [python/cocoindex/flow.py](python/cocoindex/flow.py)
- [python/cocoindex/lib.py](python/cocoindex/lib.py)
- [python/cocoindex/op.py](python/cocoindex/op.py)
- [python/cocoindex/setup.py](python/cocoindex/setup.py)
- [python/cocoindex/tests/test_typing.py](python/cocoindex/tests/test_typing.py)
- [python/cocoindex/typing.py](python/cocoindex/typing.py)

</details>



This page details the four fundamental operation types that compose a CocoIndex flow: `add_source`, `transform`, `collect`, and `export`. These operations form a declarative pipeline where users specify what transformations to apply rather than managing data mutations explicitly.

For information about the broader flow definition system and DataScope/DataSlice abstractions, see [Flow Definition System](#3). For details on operation registration and type marshalling, see [Operation System and Type Marshalling](#4).

---

## Overview

CocoIndex flows follow a declarative dataflow programming model where operations create new fields based on input data without hidden state or value mutation. The four operation types serve distinct purposes:

| Operation | Purpose | Returns | Where Used |
|-----------|---------|---------|------------|
| `add_source` | Import data from external sources | `DataSlice[T]` | `FlowBuilder` at top level |
| `transform` | Apply functions to data | `DataSlice[T]` | `FlowBuilder` or `DataSlice` |
| `collect` | Aggregate data into collections | `None` | `DataCollector` |
| `export` | Export collections to targets | `None` | `DataCollector` at top level |

**Operation Flow Pattern**

```mermaid
graph TB
    subgraph "FlowBuilder Operations"
        AddSource["add_source(spec)"]
        Transform["transform(fn_spec, args)"]
    end
    
    subgraph "DataSlice Operations"
        SliceTransform["slice.transform(fn_spec, args)"]
        Row["slice.row()"]
    end
    
    subgraph "DataScope Operations"
        AddCollector["scope.add_collector()"]
        GetField["scope[field_name]"]
    end
    
    subgraph "DataCollector Operations"
        Collect["collector.collect(**kwargs)"]
        Export["collector.export(name, spec, options)"]
    end
    
    AddSource --> GetField
    Transform --> GetField
    SliceTransform --> GetField
    GetField --> SliceTransform
    GetField --> Row
    Row --> AddCollector
    AddCollector --> Collect
    Collect --> Export
```

Sources: [python/cocoindex/flow.py:492-568](), [docs/docs/core/flow_def.mdx:1-526]()

---

## Source Operations

The `add_source()` method on `FlowBuilder` imports data from external sources, creating the initial `DataSlice` in a flow. Sources must be added at the top level of a flow.

**Method Signature**

```python
def add_source(
    spec: SourceSpec,
    *,
    name: str | None = None,
    refresh_interval: datetime.timedelta | None = None,
    max_inflight_rows: int | None = None,
    max_inflight_bytes: int | None = None,
) -> DataSlice[T]
```

**Parameters**

| Parameter | Type | Description |
|-----------|------|-------------|
| `spec` | `SourceSpec` | Source connector specification (e.g., `LocalFile`, `AmazonS3`, `Postgres`) |
| `name` | `str \| None` | Optional field name; auto-generated if omitted |
| `refresh_interval` | `timedelta \| None` | Interval for polling source changes in live update mode |
| `max_inflight_rows` | `int \| None` | Maximum concurrent rows being processed from this source |
| `max_inflight_bytes` | `int \| None` | Maximum memory footprint (bytes) of in-flight data |

**Implementation Flow**

```mermaid
graph LR
    FlowBuilder["FlowBuilder"]
    SourceSpec["SourceSpec<br/>(LocalFile, S3, etc.)"]
    EngineFlowBuilder["engine_flow_builder.add_source()"]
    EngineDataSlice["_engine.DataSlice"]
    DataSlice["DataSlice[T]"]
    
    FlowBuilder -->|"add_source(spec)"| SourceSpec
    SourceSpec -->|"dump_engine_object(spec)"| EngineFlowBuilder
    EngineFlowBuilder -->|creates| EngineDataSlice
    EngineDataSlice -->|wrapped in| DataSlice
```

**Example Usage**

```python
# Import files from local directory
data_scope["documents"] = flow_builder.add_source(
    cocoindex.sources.LocalFile(path="markdown_files")
)

# Import with refresh interval for live updates
data_scope["logs"] = flow_builder.add_source(
    cocoindex.sources.Postgres(
        table_name="application_logs",
        primary_key_columns=["id"]
    ),
    refresh_interval=datetime.timedelta(minutes=5),
    max_inflight_rows=100
)
```

**Source Naming**

If `name` is not provided, CocoIndex auto-generates a name using the source spec's class name in snake_case plus an index. The `_NameBuilder` class ensures unique names within a flow.

Sources: [python/cocoindex/flow.py:508-543](), [python/cocoindex/flow.py:51-75](), [docs/docs/core/flow_def.mdx:113-168]()

---

## Transform Operations

Transform operations apply functions to data slices, creating new data slices with transformed values. Transforms can be invoked from `FlowBuilder` or from an existing `DataSlice`.

**Method Signatures**

```python
# On FlowBuilder - requires at least one input argument
def transform(
    fn_spec: FunctionSpec | Callable[..., Any],
    *args: Any,
    **kwargs: Any
) -> DataSlice[Any]

# On DataSlice - implicitly uses self as first argument
def transform(
    fn_spec: FunctionSpec | Callable[..., Any],
    *args: Any,
    **kwargs: Any
) -> DataSlice[Any]
```

**Transform Resolution Process**

```mermaid
graph TB
    Input["transform(fn_spec, args, kwargs)"]
    CheckType{"Is fn_spec<br/>FunctionSpec?"}
    GetKind["kind = fn_spec.__class__.__name__"]
    GetOpKind["kind = fn_spec.__cocoindex_op_kind__"]
    EmptySpec["spec = EmptyFunctionSpec()"]
    BuildArgs["Build transform_args list"]
    EngineTransform["engine_flow_builder.transform()"]
    CreateSlice["_create_data_slice()"]
    Result["DataSlice[Any]"]
    
    Input --> CheckType
    CheckType -->|Yes| GetKind
    CheckType -->|No, callable| GetOpKind
    GetKind --> BuildArgs
    GetOpKind --> EmptySpec
    EmptySpec --> BuildArgs
    BuildArgs --> EngineTransform
    EngineTransform --> CreateSlice
    CreateSlice --> Result
```

**Transform Arguments**

Transform arguments are processed into a list of tuples `(data_slice, name | None)`:

- For `FlowBuilder.transform()`: all positional and keyword arguments are converted to data slices
- For `DataSlice.transform()`: `self` becomes the first argument, followed by additional arguments

The `_FlowBuilderState.get_data_slice()` method converts arguments to engine data slices:
- `DataSlice` instances → unwrapped to `_engine.DataSlice`
- Other values → wrapped as constants with type encoding

**Example Usage**

```python
# Transform from FlowBuilder with multiple inputs
data_scope["combined"] = flow_builder.transform(
    CombineFunction(),
    data_scope["field1"],
    data_scope["field2"],
    separator=", "
)

# Transform from DataSlice (most common pattern)
doc["chunks"] = doc["content"].transform(
    cocoindex.functions.SplitRecursively(),
    language="markdown",
    chunk_size=2000,
    chunk_overlap=500
)

# Chain transformations
chunk["embedding"] = chunk["text"].transform(
    cocoindex.functions.SentenceTransformerEmbed(
        model="sentence-transformers/all-MiniLM-L6-v2"
    )
)
```

**Built-in Functions**

CocoIndex provides numerous built-in functions for common transformations:

| Function | Purpose |
|----------|---------|
| `SplitRecursively` | Language-aware text chunking |
| `SentenceTransformerEmbed` | Embed text using SentenceTransformers |
| `EmbedText` | Embed text using LLM APIs |
| `ExtractByLlm` | Extract structured data using LLMs |
| `ParseJson` | Parse JSON from text |
| `DetectProgrammingLanguage` | Detect language from filename |

For complete function documentation, see [Built-in Processing Functions](#6).

Sources: [python/cocoindex/flow.py:545-561](), [python/cocoindex/flow.py:269-288](), [python/cocoindex/flow.py:106-142](), [docs/docs/core/flow_def.mdx:169-192]()

---

## Collect Operations

The `collect()` method on `DataCollector` aggregates data entries into a collection for export. Each collected entry consists of named fields with values from data slices or generated fields.

**Method Signature**

```python
def collect(self, **kwargs: Any) -> None
```

**Parameters**

All parameters are keyword arguments representing field names and their values:

| Value Type | Description |
|------------|-------------|
| `DataSlice` | References a data slice in the current or parent scope |
| `GeneratedField.UUID` | Auto-generated UUID that remains stable when other fields are unchanged |

**Collector Creation and Lifecycle**

```mermaid
graph TB
    DataScope["DataScope"]
    AddCollector["add_collector(name?)"]
    NameBuilder["_NameBuilder.build_name()"]
    EngineCollector["_engine.DataCollector"]
    DataCollector["DataCollector"]
    Collect["collect(**kwargs)"]
    EngineCollect["engine_flow_builder.collect()"]
    
    DataScope -->|"add_collector()"| AddCollector
    AddCollector --> NameBuilder
    NameBuilder -->|"_collector_0, _collector_1, ..."| EngineCollector
    EngineCollector -->|wrapped in| DataCollector
    DataCollector --> Collect
    Collect --> EngineCollect
```

**Implementation Details**

The `collect()` method processes arguments through these steps:

1. **Separate generated fields**: Identifies `GeneratedField.UUID` values and extracts the target field name
2. **Convert to data slices**: Uses `_FlowBuilderState.get_data_slice()` to convert all regular values to engine data slices
3. **Invoke engine collect**: Calls `engine_flow_builder.collect()` with processed arguments

**Example Usage**

```python
# Create collector in document scope
doc_embeddings = data_scope.add_collector()

# Iterate through nested structure
with data_scope["documents"].row() as doc:
    doc["chunks"] = doc["content"].transform(SplitRecursively(), ...)
    
    with doc["chunks"].row() as chunk:
        chunk["embedding"] = chunk["text"].transform(SentenceTransformerEmbed(...))
        
        # Collect from nested scope with generated UUID
        doc_embeddings.collect(
            id=cocoindex.GeneratedField.UUID,
            filename=doc["filename"],
            location=chunk["location"],
            text=chunk["text"],
            embedding=chunk["embedding"]
        )
```

**Generated UUID Behavior**

When using `GeneratedField.UUID`:
- Only one UUID field is allowed per collect call
- The UUID remains stable across updates if all other collected fields are unchanged
- This enables incremental processing - unchanged entries retain their IDs

Sources: [python/cocoindex/flow.py:366-399](), [python/cocoindex/flow.py:344-355](), [docs/docs/core/flow_def.mdx:218-260]()

---

## Export Operations

The `export()` method on `DataCollector` exports collected data to external targets like databases, vector stores, or graph databases. Export must occur at the top level of a flow, outside any `row()` iteration scopes.

**Method Signature**

```python
def export(
    target_name: str,
    target_spec: TargetSpec,
    *,
    primary_key_fields: Sequence[str],
    attachments: Sequence[TargetAttachmentSpec] = (),
    vector_indexes: Sequence[VectorIndexDef] = (),
    vector_index: Sequence[tuple[str, VectorSimilarityMetric]] = (),  # deprecated
    setup_by_user: bool = False,
) -> None
```

**Parameters**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `target_name` | `str` | Yes | Unique identifier for this export target within the flow |
| `target_spec` | `TargetSpec` | Yes | Target connector specification (e.g., `Postgres`, `Qdrant`, `Neo4j`) |
| `primary_key_fields` | `Sequence[str]` | Yes | Field names forming the primary key |
| `attachments` | `Sequence[TargetAttachmentSpec]` | No | Additional target-specific configurations (e.g., `PostgresSqlAttachment`) |
| `vector_indexes` | `Sequence[VectorIndexDef]` | No | Vector index definitions for similarity search |
| `setup_by_user` | `bool` | No | If `True`, user manages target setup/schema manually |

**Export Processing Flow**

```mermaid
graph TB
    Collector["DataCollector"]
    Export["export(name, spec, options)"]
    ValidateName["validate_target_name()"]
    ValidateSpec{"Is TargetSpec?"}
    Error["Raise ValueError"]
    BuildOptions["Build IndexOptions"]
    DumpSpec["dump_engine_object(spec)"]
    DumpAttachments["dump_engine_object(attachments)"]
    EngineExport["engine_flow_builder.export()"]
    
    Collector --> Export
    Export --> ValidateName
    ValidateName --> ValidateSpec
    ValidateSpec -->|No| Error
    ValidateSpec -->|Yes| BuildOptions
    BuildOptions --> DumpSpec
    DumpSpec --> DumpAttachments
    DumpAttachments --> EngineExport
```

**Target Name Stability**

The `target_name` parameter is critical for incremental processing:
- Must remain stable across flow updates
- CocoIndex uses it to track which backends correspond to which exports
- Changing the name causes CocoIndex to treat it as a removed target + new target
- This triggers setup/drop operations and full reindexing

**Setup Management**

By default (`setup_by_user=False`), CocoIndex manages target lifecycle:
- Creates tables/collections with compatible schema during `flow.setup()`
- Updates schema when collected fields change
- Drops resources during `flow.drop()`

When `setup_by_user=True`:
- User is responsible for creating and managing the target
- CocoIndex only writes data, does not modify schema
- Useful for shared databases or complex manual configurations

**Vector Index Configuration**

The `vector_indexes` parameter accepts a list of `VectorIndexDef` objects:

```python
VectorIndexDef(
    field_name: str,
    metric: VectorSimilarityMetric,
    method: VectorIndexMethod | None = None  # HNSW, IVFFlat, or target default
)
```

Supported metrics:
- `VectorSimilarityMetric.COSINE_SIMILARITY` - Cosine similarity (larger = more similar)
- `VectorSimilarityMetric.L2_DISTANCE` - Euclidean distance (smaller = more similar)  
- `VectorSimilarityMetric.INNER_PRODUCT` - Inner product (larger = more similar)

**Example Usage**

```python
# Export to Postgres with vector index
doc_embeddings.export(
    "doc_embeddings",
    cocoindex.targets.Postgres(),
    primary_key_fields=["filename", "location"],
    vector_indexes=[
        cocoindex.VectorIndexDef(
            field_name="embedding",
            metric=cocoindex.VectorSimilarityMetric.COSINE_SIMILARITY,
            method=cocoindex.HnswVectorIndexMethod(m=16, ef_construction=64)
        )
    ]
)

# Export to Qdrant
doc_embeddings.export(
    "doc_embeddings_qdrant",
    cocoindex.targets.Qdrant(
        collection_name="documents",
        url="http://localhost:6333"
    ),
    primary_key_fields=["id"],
    vector_indexes=[
        cocoindex.VectorIndexDef(
            field_name="embedding",
            metric=cocoindex.VectorSimilarityMetric.COSINE_SIMILARITY
        )
    ]
)

# Export to Neo4j with attachment
relationship_collector.export(
    "knowledge_graph",
    cocoindex.targets.Neo4jRelationship(
        connection=neo4j_conn,
        relationship_type="RELATES_TO",
        from_node_label="Entity",
        to_node_label="Entity"
    ),
    primary_key_fields=["from_id", "to_id", "relation_type"],
    attachments=[
        cocoindex.targets.Neo4jConstraint(
            node_label="Entity",
            constraint_type="UNIQUE",
            properties=["id"]
        )
    ]
)
```

**Built-in Targets**

CocoIndex provides built-in target connectors for common systems:

| Target | Purpose |
|--------|---------|
| `Postgres` | PostgreSQL with pgvector for vector indexes |
| `Qdrant` | Dedicated vector database |
| `Neo4j` | Graph database for knowledge graphs |
| `Kuzu` | Embedded graph database |
| `LanceDB` | Embedded columnar database with vector support |

For complete target documentation, see [Storage Backends and Export Targets](#8).

Sources: [python/cocoindex/flow.py:401-447](), [docs/docs/core/flow_def.mdx:261-301](), [python/cocoindex/index.py:1-100]()

---

## Operation Composition Pattern

CocoIndex operations compose into a declarative pipeline pattern where each operation type has a specific role. Understanding this composition is key to building effective flows.

**Typical Flow Structure**

```mermaid
graph TB
    subgraph "Top Level - FlowBuilder & DataScope"
        Source["add_source(SourceSpec)"]
        Collector["add_collector()"]
    end
    
    subgraph "Document Level - row() Scope"
        Transform1["transform(SplitRecursively)"]
    end
    
    subgraph "Chunk Level - nested row() Scope"
        Transform2["transform(Embed)"]
        Collect["collect(fields)"]
    end
    
    subgraph "Top Level - Export"
        Export["export(TargetSpec)"]
    end
    
    Source -->|"DataSlice[Table]"| Transform1
    Transform1 -->|"DataSlice[Table]"| Transform2
    Transform2 -->|"DataSlice[Vector]"| Collect
    Collector -->|aggregates| Collect
    Collect --> Export
```

**Example: Text Embedding Pipeline**

This example from the quickstart demonstrates the complete operation composition pattern:

```python
@cocoindex.flow_def(name="TextEmbedding")
def text_embedding_flow(
    flow_builder: cocoindex.FlowBuilder,
    data_scope: cocoindex.DataScope
):
    # 1. SOURCE: Import documents from local filesystem
    data_scope["documents"] = flow_builder.add_source(
        cocoindex.sources.LocalFile(path="markdown_files")
    )
    
    # 2. COLLECT: Create collector at top level
    doc_embeddings = data_scope.add_collector()
    
    # 3. TRANSFORM: Process each document
    with data_scope["documents"].row() as doc:
        # Split document into chunks
        doc["chunks"] = doc["content"].transform(
            cocoindex.functions.SplitRecursively(),
            language="markdown",
            chunk_size=2000,
            chunk_overlap=500
        )
        
        # Process each chunk
        with doc["chunks"].row() as chunk:
            # Embed chunk text
            chunk["embedding"] = chunk["text"].transform(
                cocoindex.functions.SentenceTransformerEmbed(
                    model="sentence-transformers/all-MiniLM-L6-v2"
                )
            )
            
            # 4. COLLECT: Aggregate chunk data
            doc_embeddings.collect(
                filename=doc["filename"],
                location=chunk["location"],
                text=chunk["text"],
                embedding=chunk["embedding"]
            )
    
    # 5. EXPORT: Write to target database
    doc_embeddings.export(
        "doc_embeddings",
        cocoindex.targets.Postgres(),
        primary_key_fields=["filename", "location"],
        vector_indexes=[
            cocoindex.VectorIndexDef(
                field_name="embedding",
                metric=cocoindex.VectorSimilarityMetric.COSINE_SIMILARITY
            )
        ]
    )
```

**Data Flow Through Operations**

```mermaid
graph LR
    Source["LocalFile Source<br/>→ Table[filename, content]"]
    Split["SplitRecursively<br/>→ Table[location, text]"]
    Embed["SentenceTransformerEmbed<br/>→ Vector[384]"]
    Collect["Collector<br/>→ aggregated rows"]
    Export["Postgres Export<br/>→ doc_embeddings table"]
    
    Source -->|"documents"| Split
    Split -->|"chunks"| Embed
    Embed -->|"embedding"| Collect
    Collect -->|"doc_embeddings"| Export
```

**Key Composition Rules**

1. **Sources at top level only**: `add_source()` must be called directly from `FlowBuilder`, not within `row()` scopes

2. **Transforms anywhere**: `transform()` can be called from `FlowBuilder` or any `DataSlice`, at any nesting level

3. **Collectors in parent scopes**: Create collectors in scopes that will aggregate data from child iterations

4. **Exports at top level only**: `export()` must be called outside all `row()` scopes

5. **Field assignment via DataScope**: Use `scope[field_name] = slice` to add fields to the current scope

**Multi-Collector Pattern**

A flow can have multiple collectors exporting to different targets:

```python
# Collector for embeddings
embeddings_collector = data_scope.add_collector()

# Collector for metadata
metadata_collector = data_scope.add_collector()

with data_scope["documents"].row() as doc:
    # ... transformations ...
    
    # Collect embeddings
    embeddings_collector.collect(
        id=doc["id"],
        embedding=doc["embedding"]
    )
    
    # Collect metadata
    metadata_collector.collect(
        id=doc["id"],
        title=doc["title"],
        created_at=doc["created_at"]
    )

# Export to different targets
embeddings_collector.export(
    "embeddings",
    cocoindex.targets.Qdrant(...),
    primary_key_fields=["id"],
    vector_indexes=[...]
)

metadata_collector.export(
    "metadata",
    cocoindex.targets.Postgres(),
    primary_key_fields=["id"]
)
```

Sources: [docs/docs/getting_started/quickstart.md:42-133](), [README.md:66-80](), [examples/hn_trending_topics/README.md:23-30](), [examples/pdf_elements_embedding/README.md:14-26]()

---

# Page: Flow Lifecycle and Methods

# Flow Lifecycle and Methods

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [docs/docs/core/basics.md](docs/docs/core/basics.md)
- [docs/docs/core/flow_def.mdx](docs/docs/core/flow_def.mdx)
- [docs/docs/core/flow_methods.mdx](docs/docs/core/flow_methods.mdx)
- [python/cocoindex/__init__.py](python/cocoindex/__init__.py)
- [python/cocoindex/cli.py](python/cocoindex/cli.py)
- [python/cocoindex/flow.py](python/cocoindex/flow.py)
- [python/cocoindex/lib.py](python/cocoindex/lib.py)
- [python/cocoindex/setup.py](python/cocoindex/setup.py)

</details>



## Purpose and Scope

This page documents the operational lifecycle of a `Flow` object and its methods for managing persistent backends, updating data, evaluating pipelines, and handling queries. A `Flow` represents a complete indexing pipeline after it has been defined using `@flow_def` or `open_flow()`.

For information about defining flows and building pipelines, see [Flow Definition System](#3). For details about incremental processing mechanics, see [Incremental Processing and State Management](#9). For change capture mechanisms, see [Live Updates and Change Capture](#9.1).

---

## Flow Object Overview

The `Flow` class [python/cocoindex/flow.py:700-917]() is the primary interface for operating an indexing pipeline after definition. Once created, a `Flow` object provides methods to:

- Manage persistent backends (setup/drop)
- Update target data (one-time or continuous)
- Evaluate transformations without persisting
- Register query handlers for semantic search
- Remove the flow from memory

Key properties:
- `name` - The flow's short name
- `full_name` - The complete name including app namespace

Sources: [python/cocoindex/flow.py:700-917](), [docs/docs/core/flow_methods.mdx:1-40]()

---

## Flow Lifecycle States

The following diagram shows the complete lifecycle of a Flow object from creation to teardown:

```mermaid
stateDiagram-v2
    [*] --> Defined: "@flow_def or open_flow()"
    
    Defined --> SetupReady: "setup()"
    note right of SetupReady
        Persistent backends created:
        - Internal Postgres storage
        - Target tables/collections
    end note
    
    SetupReady --> Updating: "update()"
    SetupReady --> LiveUpdating: "FlowLiveUpdater"
    SetupReady --> Evaluating: "evaluate_and_dump()"
    SetupReady --> Querying: "Query handlers"
    
    Updating --> SetupReady: "Update complete"
    LiveUpdating --> SetupReady: "abort() + wait()"
    Evaluating --> SetupReady: "Evaluation complete"
    Querying --> Querying: "Handle queries"
    
    SetupReady --> Dropped: "drop()"
    note right of Dropped
        All backends removed
        Flow object still valid
    end note
    
    Dropped --> SetupReady: "setup() again"
    Dropped --> Closed: "close()"
    SetupReady --> Closed: "close()"
    Defined --> Closed: "close()"
    
    Closed --> [*]
    note right of Closed
        Flow removed from memory
        Object no longer usable
    end note
```

Sources: [python/cocoindex/flow.py:700-917](), [docs/docs/core/flow_methods.mdx:37-109]()

---

## Backend Management

### Setup Operation

The `setup()` method creates or updates all persistent backends required by a flow, including internal storage and export targets.

**Method Signatures:**
- `setup(report_to_stdout: bool = False) -> None` [python/cocoindex/flow.py:829-833]()
- `setup_async(report_to_stdout: bool = False) -> None` [python/cocoindex/flow.py:835-840]()

**Behavior:**
- Creates internal Postgres tables for state tracking
- Creates target backends (tables, collections, indexes) based on export specs
- Updates existing backends if schema changes are detected
- Idempotent: no-op if already up-to-date
- Uses `SetupChangeBundle` internally [python/cocoindex/setup.py:10-73]()

**CLI Equivalent:**
```bash
cocoindex setup path/to/app.py
```

Sources: [python/cocoindex/flow.py:829-841](), [python/cocoindex/cli.py:327-340](), [docs/docs/core/flow_methods.mdx:37-100]()

### Drop Operation

The `drop()` method removes all persistent backends owned by the flow while keeping the in-memory `Flow` object valid.

**Method Signatures:**
- `drop(report_to_stdout: bool = False) -> None` [python/cocoindex/flow.py:842-851]()
- `drop_async(report_to_stdout: bool = False) -> None` [python/cocoindex/flow.py:853-858]()

**Behavior:**
- Drops internal storage tables for this flow
- Drops all export target backends
- Does NOT invalidate the Flow object (can call `setup()` again)
- Uses auth registry to identify backends even after code changes

**CLI Equivalent:**
```bash
cocoindex drop path/to/app.py
```

Sources: [python/cocoindex/flow.py:842-858](), [python/cocoindex/cli.py:343-388](), [docs/docs/core/flow_methods.mdx:37-108]()

### Close Operation

The `close()` method removes the flow from the current process, freeing resources.

**Method Signature:**
- `close() -> None` [python/cocoindex/flow.py:860-870]()

**Behavior:**
- Removes flow from internal registry via `_engine.remove_flow_context()` [python/cocoindex/flow.py:867]()
- Deletes flow from `_flows` dict [python/cocoindex/flow.py:869-870]()
- Does NOT touch persistent backends (use `drop()` for that)
- Flow object becomes unusable after this call

**Important Distinction:**
- `drop()` - Removes persistent backends, keeps object valid
- `close()` - Removes object from memory, keeps backends intact

Sources: [python/cocoindex/flow.py:860-870](), [docs/docs/core/flow_def.mdx:49-62]()

---

## Setup Change Bundle

The setup/drop operations internally use `SetupChangeBundle` to compute and apply backend changes:

```mermaid
graph LR
    FlowList["List of Flow objects"] --> MakeBundle["make_setup_bundle() or<br/>make_drop_bundle()"]
    MakeBundle --> Bundle["SetupChangeBundle"]
    
    Bundle --> Describe["describe() /<br/>describe_async()"]
    Bundle --> Apply["apply() /<br/>apply_async()"]
    
    Describe --> Output["(description: str,<br/>is_up_to_date: bool)"]
    Apply --> Changes["Backend mutations:<br/>CREATE/ALTER/DROP"]
    
    Changes --> Postgres[("Postgres<br/>tables")]
    Changes --> Qdrant[("Qdrant<br/>collections")]
    Changes --> Neo4j[("Neo4j<br/>nodes/rels")]
```

**Key Methods:**
- `describe()` / `describe_async()` - Returns description of pending changes and whether already up-to-date [python/cocoindex/setup.py:39-49]()
- `apply()` / `apply_async()` - Executes the changes [python/cocoindex/setup.py:27-37]()

**Helper Functions:**
- `make_setup_bundle(flows: Iterable[Flow]) -> SetupChangeBundle` - Declared in flow module, creates bundle for setup
- `make_drop_bundle(flows: Iterable[Flow]) -> SetupChangeBundle` - Creates bundle for drop

Sources: [python/cocoindex/setup.py:10-73](), [python/cocoindex/flow.py:839-858]()

---

## One-Time Data Update

The `update()` method performs a single incremental update of target data based on current source state.

**Method Signatures:**
- `update(*, reexport_targets: bool = False) -> IndexUpdateInfo` [python/cocoindex/flow.py:766-773]()
- `update_async(*, reexport_targets: bool = False) -> IndexUpdateInfo` [python/cocoindex/flow.py:775-787]()

**Behavior:**
- Processes all source changes since last update
- Applies transformations incrementally (only changed data)
- Updates targets with new/modified/deleted rows
- Returns statistics via `IndexUpdateInfo`
- Multiple concurrent calls are automatically coalesced

**Parameters:**
| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `reexport_targets` | `bool` | `False` | Force reexport even if no changes detected |

**Return Value:**
The `IndexUpdateInfo` object contains update statistics (defined in Rust engine).

**CLI Equivalent:**
```bash
cocoindex update path/to/app.py
cocoindex update path/to/app.py --reexport
```

**Implementation Note:**
`update()` internally creates a `FlowLiveUpdater` with `live_mode=False`, starts it, and immediately waits [python/cocoindex/flow.py:782-787]().

Sources: [python/cocoindex/flow.py:766-787](), [python/cocoindex/cli.py:439-485](), [docs/docs/core/flow_methods.mdx:110-203]()

---

## Live Update with FlowLiveUpdater

For continuous monitoring and updates, use `FlowLiveUpdater` which manages a long-running update process.

### FlowLiveUpdater Class

```mermaid
graph TB
    subgraph "FlowLiveUpdater Lifecycle"
        Create["FlowLiveUpdater(flow, options)"]
        Start["start() / start_async()"]
        Running["Background Processing"]
        StatusLoop["next_status_updates() loop"]
        Stats["update_stats()"]
        Abort["abort()"]
        Wait["wait() / wait_async()"]
        
        Create --> Start
        Start --> Running
        Running --> StatusLoop
        Running --> Stats
        StatusLoop --> StatusLoop
        Abort --> Running
        Running --> Wait
    end
    
    subgraph "Background Threads"
        SourceMonitor["Source change monitors"]
        Transformer["Transformation workers"]
        Exporter["Export mutators"]
        
        Running --> SourceMonitor
        Running --> Transformer
        Running --> Exporter
    end
```

**Constructor:**
```python
FlowLiveUpdater(flow: Flow, options: FlowLiveUpdaterOptions | None = None)
```
[python/cocoindex/flow.py:607-609]()

**Key Methods:**

| Method | Description | Blocking | File Reference |
|--------|-------------|----------|----------------|
| `start()` | Start background updater | No | [flow.py:627-631]() |
| `start_async()` | Async version of start | Yes (async) | [flow.py:633-639]() |
| `wait()` | Wait for completion | Yes | [flow.py:641-645]() |
| `wait_async()` | Async version of wait | Yes (async) | [flow.py:647-651]() |
| `abort()` | Stop the updater | No | [flow.py:672-676]() |
| `next_status_updates()` | Get next status change | Yes | [flow.py:653-660]() |
| `next_status_updates_async()` | Async version | Yes (async) | [flow.py:662-670]() |
| `update_stats()` | Get current statistics | No | [flow.py:678-682]() |

**Context Manager Support:**
```python
# Synchronous
with FlowLiveUpdater(flow, options) as updater:
    # Automatically starts on entry, aborts+waits on exit
    pass

# Asynchronous
async with FlowLiveUpdater(flow, options) as updater:
    pass
```
[python/cocoindex/flow.py:611-625]()

Sources: [python/cocoindex/flow.py:598-688](), [docs/docs/core/flow_methods.mdx:204-366]()

### FlowLiveUpdaterOptions

Configuration for live updater behavior:

```python
@dataclass
class FlowLiveUpdaterOptions:
    live_mode: bool = True          # Enable live monitoring
    reexport_targets: bool = False  # Force reexport
    print_stats: bool = False       # Print statistics
```
[python/cocoindex/flow.py:571-582]()

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `live_mode` | `bool` | `True` | Whether to continuously monitor sources with change capture |
| `reexport_targets` | `bool` | `False` | Force reexport on initial update even if no changes |
| `print_stats` | `bool` | `False` | Print statistics to stdout during processing |

**Important:** When `live_mode=False`, `FlowLiveUpdater` performs a one-time update and exits. This is how `Flow.update()` is implemented [python/cocoindex/flow.py:782-787]().

Sources: [python/cocoindex/flow.py:571-582](), [docs/docs/core/flow_methods.mdx:236-248]()

### FlowUpdaterStatusUpdates

Status information returned by `next_status_updates()`:

```python
@dataclass
class FlowUpdaterStatusUpdates:
    active_sources: list[str]   # Sources still processing
    updated_sources: list[str]  # Sources updated since last call
```
[python/cocoindex/flow.py:586-595]()

**Usage Pattern:**
```python
updater = FlowLiveUpdater(flow)
updater.start()

while True:
    updates = updater.next_status_updates()  # Blocks until new update
    
    for source in updates.updated_sources:
        # React to source updates
        handle_updated_source(source)
    
    if not updates.active_sources:
        break  # Updater stopped
```

**Note:** Status updates are automatically coalesced if multiple arrive between calls, preventing backlog [docs/docs/core/flow_methods.mdx:311-318]().

Sources: [python/cocoindex/flow.py:586-688](), [docs/docs/core/flow_methods.mdx:268-318]()

### CLI Usage

```bash
# Live update mode
cocoindex update path/to/app.py -L

# One-time with setup
cocoindex update path/to/app.py --setup

# Live with reexport
cocoindex update path/to/app.py -L --reexport
```

Sources: [python/cocoindex/cli.py:439-485](), [docs/docs/core/flow_methods.mdx:204-229]()

---

## Evaluation Mode

The `evaluate_and_dump()` method runs transformations and dumps outputs to files without touching target backends.

### Method Signature

```python
def evaluate_and_dump(options: EvaluateAndDumpOptions) -> IndexUpdateInfo
```
[python/cocoindex/flow.py:789-795]()

### EvaluateAndDumpOptions

```python
@dataclass
class EvaluateAndDumpOptions:
    output_dir: str              # Required: output directory
    use_cache: bool = True       # Use cached intermediate results
```
[python/cocoindex/flow.py:691-697]()

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `output_dir` | `str` | Required | Directory path for dumped outputs |
| `use_cache` | `bool` | `True` | Read from cache if available (cache not updated) |

### Use Cases

- **Testing transformations** before deploying to production
- **Debugging pipeline logic** by inspecting intermediate outputs
- **Verifying data quality** without affecting live indexes
- **Development iteration** without backend writes

### CLI Usage

```bash
# Basic evaluation
cocoindex evaluate path/to/app.py

# Specify output directory
cocoindex evaluate path/to/app.py -o ./my_eval

# Disable cache
cocoindex evaluate path/to/app.py --no-cache
```

Sources: [python/cocoindex/flow.py:789-795](), [python/cocoindex/cli.py:487-527](), [docs/docs/core/flow_methods.mdx:368-409]()

---

## Query Handlers

Query handlers enable semantic search and custom query logic over indexed data.

### Adding Query Handlers

**Method Signature:**
```python
def add_query_handler(
    name: str,
    handler: Callable[[str], Any],
    *,
    result_fields: QueryHandlerResultFields | None = None
) -> None
```
[python/cocoindex/flow.py:872-900]()

**Parameters:**
- `name` - Unique identifier for the handler
- `handler` - Function that takes query string and returns results
- `result_fields` - Optional schema description for results

**Handler Function:**
- Input: `query: str` - The search query
- Output: Must return `QueryOutput` object [python/cocoindex/query_handler.py]()

### Decorator Syntax

```python
@flow.query_handler("semantic_search")
def my_search_handler(query: str) -> QueryOutput:
    # Perform vector search
    # Return QueryOutput with results
    ...
```
[python/cocoindex/flow.py:902-917]()

The decorator calls `add_query_handler()` with the function name as default handler name.

### Query Handler Architecture

```mermaid
graph LR
    Client["HTTP Client"] --> Server["CocoIndex Server"]
    
    Server --> FlowRegistry["Flow Registry<br/>_flows dict"]
    FlowRegistry --> FlowObj["Flow object"]
    
    FlowObj --> Engine["_engine.Flow"]
    Engine --> Handler["Registered handler<br/>async function"]
    
    Handler --> VectorDB[("Vector DB<br/>(Postgres/Qdrant)")]
    Handler --> QueryOutput["QueryOutput<br/>{results, query_info}"]
    
    QueryOutput --> Server
    Server --> Client
```

**Internal Flow:**
1. Handler registered with flow via `add_query_handler()` [python/cocoindex/flow.py:872-900]()
2. Handler wrapped as async and stored in `_lazy_query_handler_args` or added directly to engine [python/cocoindex/flow.py:896-900]()
3. HTTP server routes queries to appropriate flow handler
4. Handler executes vector search or custom logic
5. Results returned as `QueryOutput` with `results` list and `query_info` metadata

Sources: [python/cocoindex/flow.py:872-917](), [python/cocoindex/query_handler.py]()

---

## Multi-Flow Operations

For managing multiple flows simultaneously, CocoIndex provides batch operations.

### Setup All Flows

```python
# Synchronous
cocoindex.setup_all_flows(report_to_stdout: bool = False)

# Available in flow module (not shown in __init__.py __all__)
```

Sets up all flows currently registered in the process.

### Drop All Flows

```python
# Synchronous
cocoindex.drop_all_flows(report_to_stdout: bool = False)

# Available in flow module (not shown in __init__.py __all__)
```

Drops all flows currently registered in the process.

### Update All Flows

```python
# Asynchronous only
async def update_all_flows_async(
    options: FlowLiveUpdaterOptions
) -> dict[str, IndexUpdateInfo]
```
[python/cocoindex/flow.py:1035-1052]()

Returns a dictionary mapping flow names to their update statistics.

**Implementation:**
- Creates `FlowLiveUpdater` for each flow
- Uses `asyncio.gather()` to run updates concurrently [python/cocoindex/flow.py:1049-1051]()
- Returns aggregated statistics

### CLI Equivalents

```bash
# Setup all flows in app
cocoindex setup path/to/app.py

# Drop all flows in app
cocoindex drop path/to/app.py

# Update all flows in app
cocoindex update path/to/app.py
```

Sources: [python/cocoindex/flow.py:1026-1052](), [python/cocoindex/cli.py:310-485]()

---

## Internal State Management

### Flow Registry

The flow module maintains a global registry of active flows:

```python
_flows_lock = Lock()
_flows: dict[str, Flow] = {}
```
[python/cocoindex/flow.py:942-943]()

**Registry Functions:**
- `open_flow(name, fl_def)` - Register new flow [python/cocoindex/flow.py:953-961]()
- `flow_by_name(name)` - Retrieve flow by name [python/cocoindex/flow.py:1003-1008]()
- `flows()` - Get all flows as dict [python/cocoindex/flow.py:995-1000]()
- `flow_names()` - Get list of flow names [python/cocoindex/flow.py:987-992]()
- `remove_flow(fl)` / `fl.close()` - Remove from registry [python/cocoindex/flow.py:860-870]()

### Lazy Initialization

Flows use lazy initialization to defer engine flow creation until first use:

```mermaid
graph TD
    Create["Flow object created"] --> FirstUse["First method call"]
    FirstUse --> CheckLazy{"_lazy_engine_flow<br/>is None?"}
    
    CheckLazy -->|Yes| BuildFlow["Execute _engine_flow_creator()"]
    CheckLazy -->|No| UseCached["Use cached engine flow"]
    
    BuildFlow --> BuildState["Build _FlowBuilderState"]
    BuildState --> CallDef["Call flow definition function"]
    CallDef --> EngineFlow["Build _engine.Flow"]
    EngineFlow --> Cache["Store in _lazy_engine_flow"]
    
    Cache --> Return["Return engine flow"]
    UseCached --> Return
```

**Benefits:**
- Flow definitions don't execute until needed
- Multiple flows can be registered without overhead
- Query handlers can be added before flow is built

**Key Code:**
- Lazy creator stored in `_engine_flow_creator` [python/cocoindex/flow.py:706]()
- First access triggers `_internal_flow()` [python/cocoindex/flow.py:813-827]()
- Result cached in `_lazy_engine_flow` [python/cocoindex/flow.py:710]()

Sources: [python/cocoindex/flow.py:700-827](), [python/cocoindex/flow.py:920-939]()

---

## Full Lifecycle Example

The following code demonstrates a complete flow lifecycle:

```python
import cocoindex

# 1. Define flow
@cocoindex.flow_def(name="MyFlow")
def my_flow(builder, scope):
    scope["docs"] = builder.add_source(...)
    # ... more operations

# 2. Setup backends
my_flow.setup(report_to_stdout=True)

# 3. Perform one-time update
stats = my_flow.update()
print(f"Processed {stats}")

# 4. Or use live updater
with cocoindex.FlowLiveUpdater(my_flow) as updater:
    while True:
        updates = updater.next_status_updates()
        if not updates.active_sources:
            break

# 5. Add query handler
@my_flow.query_handler("search")
def search(query: str):
    # Perform search logic
    return QueryOutput(...)

# 6. Evaluate without persisting
my_flow.evaluate_and_dump(
    cocoindex.EvaluateAndDumpOptions(output_dir="./eval")
)

# 7. Drop backends
my_flow.drop(report_to_stdout=True)

# 8. Remove from memory
my_flow.close()
```

Sources: [python/cocoindex/flow.py:700-917](), [docs/docs/core/flow_methods.mdx:1-409]()

---

## Related Engine Objects

### Internal Flow Object

The `Flow` class wraps an internal `_engine.Flow` object created by the Rust engine:

| Python Layer | Rust Engine Layer |
|--------------|-------------------|
| `Flow` | `_engine.Flow` |
| `FlowLiveUpdater` | `_engine.FlowLiveUpdater` |
| `SetupChangeBundle` | `_engine.SetupChangeBundle` |

The engine flow is accessed via:
- `internal_flow()` - Synchronous access [python/cocoindex/flow.py:797-803]()
- `internal_flow_async()` - Asynchronous access [python/cocoindex/flow.py:805-811]()

### Flow Full Names

Flow full names include the app namespace prefix:

```python
def get_flow_full_name(name: str) -> str:
    return f"{setting.get_app_namespace(trailing_delimiter='.')}{name}"
```
[python/cocoindex/flow.py:946-950]()

Examples:
- No namespace: `"MyFlow"`
- With namespace: `"Staging.MyFlow"` or `"Production.MyFlow"`

This enables multi-tenancy and environment separation. See [App Namespaces and Multi-tenancy](#10.2) for details.

Sources: [python/cocoindex/flow.py:700-950](), [python/cocoindex/setting.py]()

---

# Page: TransformFlow and In-Memory Evaluation

# TransformFlow and In-Memory Evaluation

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [python/cocoindex/__init__.py](python/cocoindex/__init__.py)
- [python/cocoindex/cli.py](python/cocoindex/cli.py)
- [python/cocoindex/flow.py](python/cocoindex/flow.py)
- [python/cocoindex/lib.py](python/cocoindex/lib.py)
- [python/cocoindex/setup.py](python/cocoindex/setup.py)

</details>



## Purpose and Scope

This document explains two mechanisms for working with data transformations without persistent storage:

1. **TransformFlow**: A decorator (`@transform_flow`) for creating reusable, composable transformation functions that can be evaluated in-memory with typed inputs
2. **Flow Evaluation**: Methods for evaluating flows and dumping results to files instead of persisting to database backends

For information about standard persistent flows and their lifecycle, see [Flow Lifecycle and Methods](#3.3). For details on the operation system underlying transformations, see [Operation System and Type Marshalling](#4).

## TransformFlow Overview

`TransformFlow` is a special type of flow designed for reusable, in-memory transformations. Unlike regular `Flow` objects that persist state and export to backends, `TransformFlow` operates entirely in memory and can be called like a function with typed inputs.

### Key Characteristics

| Feature | Regular Flow | TransformFlow |
|---------|-------------|---------------|
| State Persistence | PostgreSQL backend required | No persistent state |
| Setup Required | Yes (`flow.setup()`) | No |
| Input Method | Sources via `add_source()` | Function parameters |
| Output Method | Collectors and `export()` | Return value |
| Reusability | One-time definition | Callable multiple times |
| Use Case | Data indexing pipelines | Reusable transformations |

Sources: [python/cocoindex/flow.py:1095-1231]()

## Creating TransformFlows

### The @transform_flow Decorator

The `@transform_flow` decorator converts a function that returns a `DataSlice[T]` into a reusable `TransformFlow[T]` object:

```python
from cocoindex import transform_flow, DataSlice, Vector, Float32

@transform_flow()
def embed_text(text: DataSlice[str]) -> DataSlice[Vector[Float32, 384]]:
    """Embeds text using a sentence transformer."""
    from cocoindex.functions import SentenceTransformerEmbed
    
    return text.transform(
        SentenceTransformerEmbed(model_name="all-MiniLM-L6-v2")
    )

# Use the transform flow
embeddings = embed_text.evaluate(text=["hello world", "foo bar"])
```

The decorator is defined as:

```python
def transform_flow(
    name: str | None = None,
) -> Callable[[Callable[..., DataSlice[T]]], TransformFlow[T]]
```

If `name` is not provided, an auto-generated name with prefix `_transform_flow_` is used.

Sources: [python/cocoindex/flow.py:1200-1207](), [python/cocoindex/__init__.py:20]()

### Type Annotation Requirements

All parameters in a `TransformFlow` function **must** be annotated with `DataSlice[T]` where `T` is a concrete type:

```python
# ✓ Correct: Type annotation provided
@transform_flow()
def process(data: DataSlice[str]) -> DataSlice[int]:
    ...

# ✗ Incorrect: Missing type annotation
@transform_flow()
def process(data: DataSlice) -> DataSlice[int]:
    ...

# ✗ Incorrect: Not wrapped in DataSlice
@transform_flow()
def process(data: str) -> DataSlice[int]:
    ...
```

The type annotations are analyzed at initialization time to create encoders/decoders for the FFI boundary:

Sources: [python/cocoindex/flow.py:1119-1141]()

### TransformFlow Architecture

```mermaid
graph TB
    subgraph "Python Layer"
        A["@transform_flow()"]
        B["TransformFlow[T]"]
        C["FlowArgInfo[]<br/>(name, type_hint, encoder)"]
        D["_flow_fn: Callable"]
    end
    
    subgraph "Lazy Initialization"
        E["_build_flow_info_async()"]
        F["TransformFlowInfo[T]"]
        G["_engine.TransientFlow"]
        H["result_decoder"]
    end
    
    subgraph "Evaluation"
        I["evaluate(**kwargs)"]
        J["Encode Inputs"]
        K["_engine.TransientFlow.evaluate_async()"]
        L["Decode Results"]
        M["list[T]"]
    end
    
    A --> B
    B --> C
    B --> D
    B --> E
    E --> F
    F --> G
    F --> H
    I --> J
    J --> K
    K --> L
    L --> M
    
    C -.used by.-> J
    H -.used by.-> L
```

**Diagram: TransformFlow Component Architecture**

The `TransformFlow` class has three main phases:

1. **Initialization**: Analyzes function signature and creates `FlowArgInfo` for each parameter
2. **Lazy Building**: First evaluation triggers creation of `_engine.TransientFlow` and decoder
3. **Evaluation**: Encodes inputs, evaluates in engine, decodes results

Sources: [python/cocoindex/flow.py:1095-1199]()

### Internal Structure

The `TransformFlow` class contains:

| Field | Type | Purpose |
|-------|------|---------|
| `_flow_fn` | `Callable[..., DataSlice[T]]` | Original decorated function |
| `_flow_name` | `str` | Unique flow name |
| `_args_info` | `list[FlowArgInfo]` | Metadata for each parameter |
| `_lazy_lock` | `asyncio.Lock` | Ensures single initialization |
| `_lazy_flow_info` | `TransformFlowInfo[T] \| None` | Cached flow info after first use |

Each `FlowArgInfo` contains:

| Field | Type | Purpose |
|-------|------|---------|
| `name` | `str` | Parameter name |
| `type_hint` | `Any` | Python type annotation |
| `encoder` | `Callable[[Any], Any]` | Converts Python value to engine representation |

Sources: [python/cocoindex/flow.py:1089-1117]()

## Evaluating TransformFlows

### Synchronous Evaluation

The `evaluate()` method accepts keyword arguments matching the function's parameters:

```python
@transform_flow()
def chunk_text(
    text: DataSlice[str],
    chunk_size: DataSlice[int]
) -> DataSlice[str]:
    from cocoindex.functions import SplitRecursively
    
    return text.transform(SplitRecursively(max_chunk_size=chunk_size))

# Evaluate with inputs
chunks = chunk_text.evaluate(
    text=["long document..."],
    chunk_size=[500]
)
# Returns: list[str] with chunked text
```

The method signature is:

```python
def evaluate(self, **kwargs: Any) -> list[T]:
    """Evaluate the flow with the given inputs and return the results."""
    return execution_context.run(self.evaluate_async(**kwargs))
```

Sources: [python/cocoindex/flow.py:1184-1188]()

### Asynchronous Evaluation

For async contexts, use `evaluate_async()`:

```python
async def process_batch():
    chunks = await chunk_text.evaluate_async(
        text=["doc1", "doc2"],
        chunk_size=[500, 500]
    )
    return chunks
```

Sources: [python/cocoindex/flow.py:1190-1199]()

### Evaluation Flow Diagram

```mermaid
sequenceDiagram
    participant U as "User Code"
    participant TF as "TransformFlow"
    participant FI as "TransformFlowInfo"
    participant E as "_engine.TransientFlow"
    
    U->>TF: evaluate(text=["hello"])
    TF->>TF: Check _lazy_flow_info
    
    alt First Evaluation
        TF->>TF: _build_flow_info_async()
        TF->>E: Create TransientFlow
        TF->>FI: Create TransformFlowInfo
        Note over TF,FI: Includes result_decoder
    end
    
    TF->>TF: Encode inputs using FlowArgInfo.encoder
    Note over TF: text -> engine representation
    
    TF->>E: evaluate_async(encoded_inputs)
    E-->>TF: Raw results
    
    TF->>TF: Decode using result_decoder
    Note over TF: engine representation -> list[T]
    
    TF-->>U: list[T]
```

**Diagram: TransformFlow Evaluation Sequence**

Sources: [python/cocoindex/flow.py:1152-1199]()

### Calling TransformFlows in Other Flows

`TransformFlow` objects are callable and can be used within regular flow definitions:

```python
@transform_flow()
def normalize_text(text: DataSlice[str]) -> DataSlice[str]:
    # ... transformation logic
    pass

@flow_def()
def my_flow(builder: FlowBuilder, scope: DataScope):
    source = builder.add_source(LocalFile(path="data.txt"))
    
    # Call TransformFlow within a regular flow
    normalized = normalize_text(source["content"])
    
    collector = scope.add_collector()
    collector.collect(text=normalized)
```

When called this way, the `__call__` method delegates to the original function, returning a `DataSlice[T]` that integrates into the flow graph.

Sources: [python/cocoindex/flow.py:1143-1144]()

## In-Memory Flow Evaluation

### The evaluate_and_dump Method

Regular `Flow` objects support in-memory evaluation via `evaluate_and_dump()`, which processes data and writes results to files instead of database backends:

```python
from cocoindex import flow_def, EvaluateAndDumpOptions

@flow_def()
def my_flow(builder, scope):
    # ... flow definition
    pass

# Evaluate and dump to files
options = EvaluateAndDumpOptions(
    output_dir="./output",
    use_cache=True
)
my_flow.evaluate_and_dump(options)
```

The method signature:

```python
def evaluate_and_dump(
    self, options: EvaluateAndDumpOptions
) -> _engine.IndexUpdateInfo:
    """Evaluate the flow and dump flow outputs to files."""
    return self.internal_flow().evaluate_and_dump(dump_engine_object(options))
```

Sources: [python/cocoindex/flow.py:802-808]()

### EvaluateAndDumpOptions

The `EvaluateAndDumpOptions` dataclass configures evaluation behavior:

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `output_dir` | `str` | Required | Directory path for output files |
| `use_cache` | `bool` | `True` | Whether to reuse cached intermediate results |

Sources: [python/cocoindex/flow.py:695-702]()

### CLI Usage

The `cocoindex evaluate` command provides command-line access to flow evaluation:

```bash
# Evaluate flow and dump to default output directory
cocoindex evaluate path/to/app.py:MyFlow

# Specify custom output directory
cocoindex evaluate path/to/app.py:MyFlow -o ./results

# Disable cache usage
cocoindex evaluate path/to/app.py:MyFlow --no-cache
```

If no output directory is specified, a timestamped directory is created:

```python
if output_dir is None:
    output_dir = f"eval_{setting.get_app_namespace(trailing_delimiter='_')}{fl.name}_{datetime.datetime.now().strftime('%y%m%d_%H%M%S')}"
```

Example: `eval_production_my_flow_241215_143022`

Sources: [python/cocoindex/cli.py:506-546]()

### Evaluation vs. Update Comparison

```mermaid
graph LR
    subgraph "Flow.update()"
        A1["Read Sources"]
        A2["Check Cache<br/>(PostgreSQL)"]
        A3["Apply Transforms"]
        A4["Export to<br/>Targets"]
        A5["Update State<br/>(PostgreSQL)"]
        
        A1 --> A2
        A2 --> A3
        A3 --> A4
        A4 --> A5
    end
    
    subgraph "Flow.evaluate_and_dump()"
        B1["Read Sources"]
        B2["Check Cache<br/>(Optional)"]
        B3["Apply Transforms"]
        B4["Write to<br/>Files"]
        
        B1 --> B2
        B2 --> B3
        B3 --> B4
    end
    
    style A4 fill:#f9f9f9
    style A5 fill:#f9f9f9
    style B4 fill:#e8f5e9
```

**Diagram: Update vs. Evaluate-and-Dump Execution Paths**

Key differences:

| Aspect | `update()` | `evaluate_and_dump()` |
|--------|------------|----------------------|
| Backend Setup | Required | Not required |
| State Persistence | Yes (PostgreSQL) | No |
| Output Destination | Configured targets | Local files |
| Incremental Processing | Yes | Optional (via cache) |
| Use Case | Production indexing | Development/testing |

Sources: [python/cocoindex/flow.py:771-808]()

### Use Cases for Evaluation

**Development and Testing**:
```python
# Test flow without setting up backends
options = EvaluateAndDumpOptions(
    output_dir="./test_output",
    use_cache=False
)
my_flow.evaluate_and_dump(options)
# Inspect files in ./test_output
```

**Debugging**:
```python
# Evaluate with cache disabled to see fresh results
options = EvaluateAndDumpOptions(
    output_dir="./debug",
    use_cache=False
)
my_flow.evaluate_and_dump(options)
# Examine intermediate outputs
```

**Benchmarking**:
```python
import time

start = time.time()
my_flow.evaluate_and_dump(EvaluateAndDumpOptions(output_dir="./benchmark"))
elapsed = time.time() - start
print(f"Evaluation took {elapsed:.2f} seconds")
```

Sources: [python/cocoindex/cli.py:522-546]()

## Type Safety and Validation

### Parameter Validation at Initialization

`TransformFlow` validates parameters when the decorator is applied:

```python
# Constructor validation logic
sig = inspect.signature(flow_fn)
args_info = []
for param_name, param in sig.parameters.items():
    if param.kind not in (
        inspect.Parameter.POSITIONAL_OR_KEYWORD,
        inspect.Parameter.KEYWORD_ONLY,
    ):
        raise ValueError(
            f"Parameter `{param_name}` is not a parameter can be passed by name"
        )
    value_type_annotation = _get_data_slice_annotation_type(param.annotation)
    if value_type_annotation is None:
        raise ValueError(
            f"Parameter `{param_name}` for {flow_fn} has no value type annotation. "
            "Please use `cocoindex.DataSlice[T]` where T is the type of the value."
        )
```

Only `POSITIONAL_OR_KEYWORD` and `KEYWORD_ONLY` parameters are allowed.

Sources: [python/cocoindex/flow.py:1119-1136]()

### Runtime Input Validation

At evaluation time, all required parameters must be provided:

```python
encoded_inputs = {}
for arg_info in self._args_info:
    if arg_info.name not in kwargs:
        raise ValueError(f"Missing input: {arg_info.name}")
    encoded_inputs[arg_info.name] = arg_info.encoder(kwargs[arg_info.name])
```

Missing parameters raise `ValueError` immediately.

Sources: [python/cocoindex/flow.py:1193-1196]()

### Type Encoding and Decoding

The encoder/decoder system ensures type safety across the FFI boundary:

```mermaid
graph LR
    subgraph "Python Space"
        A["Python Value<br/>list[str]"]
        H["Python Result<br/>list[T]"]
    end
    
    subgraph "Encoding Layer"
        B["analyze_type_info(T)"]
        C["make_engine_value_encoder()"]
        D["Encoder Function"]
        
        F["result_decoder"]
        G["Decoder Function"]
    end
    
    subgraph "Rust Engine"
        E["Engine Value<br/>(serialized)"]
    end
    
    A --> D
    D --> E
    E --> G
    G --> H
    
    B --> C
    C --> D
    
    style E fill:#f9f9f9
```

**Diagram: Type Encoding/Decoding in TransformFlow**

The encoder is created from type analysis:

```python
encoder = make_engine_value_encoder(
    analyze_type_info(value_type_annotation)
)
```

The decoder is created when building the flow:

```python
result_decoder = make_engine_value_decoder(
    decode_value_type(result_slice_type)
)
```

Sources: [python/cocoindex/flow.py:1137-1139](), [python/cocoindex/flow.py:1177-1179]()

## Integration with Regular Flows

### Using TransformFlows as Building Blocks

`TransformFlow` objects compose naturally with regular flows:

```python
# Define reusable transformations
@transform_flow()
def clean_text(text: DataSlice[str]) -> DataSlice[str]:
    # ... cleaning logic
    pass

@transform_flow()
def extract_entities(text: DataSlice[str]) -> DataSlice[dict]:
    # ... entity extraction
    pass

# Use in a persistent flow
@flow_def()
def document_pipeline(builder, scope):
    docs = builder.add_source(LocalFile(path="docs/"))
    
    # Compose transformations
    clean = clean_text(docs["content"])
    entities = extract_entities(clean)
    
    collector = scope.add_collector()
    collector.collect(text=clean, entities=entities)
    collector.export(
        "postgres_target",
        PostgresTarget(...),
        primary_key_fields=["id"]
    )
```

This pattern enables:
- **Modularity**: Break complex pipelines into reusable pieces
- **Testability**: Test transformations independently
- **Reusability**: Share transformations across multiple flows

Sources: [python/cocoindex/flow.py:1143-1144]()

### Standalone vs. Embedded Usage

`TransformFlow` supports two usage patterns:

**Standalone** (with `evaluate()`):
```python
# Direct evaluation in memory
results = my_transform.evaluate(input=data)
```

**Embedded** (with `__call__`):
```python
# Called within flow definition
@flow_def()
def flow(builder, scope):
    source = builder.add_source(...)
    result = my_transform(source)  # Returns DataSlice
```

The `__call__` method delegates to the original function to produce a `DataSlice`:

```python
def __call__(self, *args: Any, **kwargs: Any) -> DataSlice[T]:
    return self._flow_fn(*args, **kwargs)
```

Sources: [python/cocoindex/flow.py:1143-1144]()

---

# Page: Operation System and Type Marshalling

# Operation System and Type Marshalling

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [docs/docs/core/data_types.mdx](docs/docs/core/data_types.mdx)
- [python/cocoindex/op.py](python/cocoindex/op.py)
- [python/cocoindex/tests/test_typing.py](python/cocoindex/tests/test_typing.py)
- [python/cocoindex/typing.py](python/cocoindex/typing.py)

</details>



This page provides a comprehensive overview of CocoIndex's operation registration system and how data is marshalled between Python and Rust across the FFI boundary. It explains how Python operations (functions, sources, targets) are registered with the Rust engine and how type information is analyzed, validated, and used to create bidirectional encoders/decoders for runtime value conversion.

**Scope**: This page covers the architectural patterns and data flow. For implementation details, see child pages: [Operation Registration and Decorators](#4.1) for decorator usage, [Type System and Schema Analysis](#4.2) for type inference, [Encoders and Decoders](#4.3) for value conversion, and [OpArgs and Execution Options](#4.4) for execution configuration.

For information about defining flows using operations, see [Flow Definition System](#3). For specific built-in operations, see [Built-in Data Sources](#5) and [Built-in Processing Functions](#6).

## Operation Categories

CocoIndex organizes operations into distinct categories, each serving a specific role in data processing pipelines. These categories are defined in [python/cocoindex/op.py:40-48]().

| Category | Base Class | Purpose | Registration Method |
|----------|-----------|---------|---------------------|
| `FUNCTION` | `FunctionSpec` | Data transformations and processing | `_engine.register_function_factory` |
| `SOURCE` | `SourceSpec` | Reading data from external systems | `_engine.register_source_connector` |
| `TARGET` | `TargetSpec` | Writing data to external systems | `_engine.register_target_connector` |
| `DECLARATION` | `DeclarationSpec` | Schema and metadata declarations | (Special handling) |
| `TARGET_ATTACHMENT` | `TargetAttachmentSpec` | Additional target configurations | (Special handling) |

All spec base classes use the `SpecMeta` metaclass [python/cocoindex/op.py:50-68](), which automatically converts them to dataclasses, enabling structured configuration of operations.

**Sources**: [python/cocoindex/op.py:40-89]()

## Registration Flow Architecture

The operation registration system bridges Python's dynamic typing with Rust's static execution model. Registration happens at module import time, before any flow execution.

```mermaid
flowchart TB
    subgraph Python["Python Definition Layer"]
        Decorator["@function / @executor_class<br/>@source_connector / @target_connector"]
        TypeHints["Python Type Annotations<br/>inspect.signature()"]
        SpecClass["Spec Class<br/>(FunctionSpec/SourceSpec/TargetSpec)"]
    end
    
    subgraph Registration["Registration Layer"]
        AnalyzeTypes["analyze_type_info()<br/>Extract CocoIndex types from<br/>Python annotations"]
        CreateEncoders["make_engine_value_encoder()<br/>make_engine_value_decoder()<br/>make_engine_key_encoder()<br/>make_engine_key_decoder()"]
        RegisterFactory["_register_op_factory()<br/>or register connector"]
    end
    
    subgraph RustEngine["Rust Engine (_engine module)"]
        FunctionRegistry["Function Factory Registry<br/>_engine.register_function_factory()"]
        SourceRegistry["Source Connector Registry<br/>_engine.register_source_connector()"]
        TargetRegistry["Target Connector Registry<br/>_engine.register_target_connector()"]
    end
    
    Decorator --> TypeHints
    Decorator --> SpecClass
    TypeHints --> AnalyzeTypes
    AnalyzeTypes --> CreateEncoders
    CreateEncoders --> RegisterFactory
    SpecClass --> RegisterFactory
    
    RegisterFactory --> FunctionRegistry
    RegisterFactory --> SourceRegistry
    RegisterFactory --> TargetRegistry
```

**Registration Process**:

1. **Decorator Application**: When a decorator like `@function` or `@source_connector` is applied, it immediately triggers registration [python/cocoindex/op.py:489-512]()

2. **Type Analysis**: The decorator extracts type information from Python function signatures using `inspect.signature()` and analyzes return type annotations to determine output schema [python/cocoindex/op.py:497-508]()

3. **Encoder/Decoder Creation**: Based on analyzed types, the system creates bidirectional conversion functions that will be used at runtime [python/cocoindex/op.py:323-325]()

4. **Engine Registration**: The operation factory is registered with the Rust engine, which stores it in a global registry keyed by operation kind (class name) [python/cocoindex/op.py:441-445]()

**Sources**: [python/cocoindex/op.py:176-446](), [python/cocoindex/op.py:489-512](), [python/cocoindex/op.py:746-766](), [python/cocoindex/op.py:1073-1095]()

## Type Information Flow Across FFI Boundary

Type information flows from Python annotations through analysis to Rust schema representation, enabling type-safe execution despite the language boundary.

```mermaid
flowchart LR
    subgraph Python["Python Type Space"]
        PyAnnotation["Python Type Annotations<br/>Vector[Float32, Literal[384]]<br/>dict[str, Person]<br/>list[Chunk]"]
        PyValue["Python Values<br/>np.ndarray, dict, list<br/>dataclass instances"]
    end
    
    subgraph Analysis["Type Analysis Layer"]
        DataTypeInfo["datatype.DataTypeInfo<br/>Analyzed type metadata<br/>nullable, base_type, variant"]
        TypeSchema["engine_type.EnrichedValueType<br/>Rust-compatible schema<br/>StructType, TableType, etc."]
    end
    
    subgraph Marshalling["Marshalling Layer"]
        Encoder["Encoder Function<br/>Python → Engine Value<br/>make_engine_value_encoder()"]
        Decoder["Decoder Function<br/>Engine Value → Python<br/>make_engine_value_decoder()"]
    end
    
    subgraph Rust["Rust Engine Space"]
        EngineValue["Engine Values<br/>Serialized binary format<br/>Type-tagged data"]
        EngineSchema["Type Schema Registry<br/>Validates operations<br/>Enforces type safety"]
    end
    
    PyAnnotation -->|"analyze_type_info()"| DataTypeInfo
    DataTypeInfo -->|"encode_enriched_type_info()"| TypeSchema
    TypeSchema --> EngineSchema
    
    TypeSchema -.->|"Creates based on schema"| Encoder
    TypeSchema -.->|"Creates based on schema"| Decoder
    
    PyValue -->|"Runtime"| Encoder
    Encoder --> EngineValue
    EngineValue --> Decoder
    Decoder --> PyValue
```

**Type Flow Stages**:

1. **Python Annotation → DataTypeInfo**: The `analyze_type_info()` function parses Python type annotations (including complex types like `Vector[Float32, Literal[384]]`) and extracts structural information [python/cocoindex/typing.py:1-90]()

2. **DataTypeInfo → TypeSchema**: The analyzed type is encoded into `engine_type.EnrichedValueType`, a representation compatible with Rust's type system [python/cocoindex/op.py:349-350]()

3. **Schema Registration**: During `analyze_schema()`, the type schema is sent to the Rust engine, where it's validated and stored [python/cocoindex/op.py:214-351]()

4. **Encoder/Decoder Creation**: Based on both source and destination type information, the system creates specialized encoder and decoder functions that know how to convert between Python and engine representations [python/cocoindex/op.py:236-243]()

**Example Type Mappings**:

| Python Type | CocoIndex Type | Engine Representation |
|-------------|----------------|----------------------|
| `Vector[Float32, Literal[384]]` | Vector[Float32, 384] | Serialized float array with dimension metadata |
| `dict[str, Person]` | KTable with Str key | Table with key and value columns |
| `list[Chunk]` | LTable | Ordered table without keys |
| `Person` (dataclass) | Struct | Struct with named fields |

**Sources**: [python/cocoindex/op.py:214-351](), [python/cocoindex/typing.py:17-89](), [docs/docs/core/data_types.mdx:1-333]()

## Runtime Execution and Value Marshalling

At runtime, operations execute through a wrapper layer that handles value conversion, batching, and null propagation. The execution model separates registration-time analysis from runtime value processing.

```mermaid
flowchart TB
    subgraph EngineCall["Rust Engine Invocation"]
        EngineArgs["Engine Argument Values<br/>Serialized binary format<br/>Type: OpArgSchema"]
        EngineResult["Engine Result Value<br/>Serialized binary format"]
    end
    
    subgraph WrappedExecution["_WrappedExecutor<br/>(python/cocoindex/op.py)"]
        DecodeArgs["Decode Arguments<br/>decoder(engine_value)<br/>→ Python value"]
        CheckNull["Null Propagation<br/>If non-optional arg is None<br/>→ return None immediately"]
        HandleBatching["Batching Logic<br/>Skip None values in batches<br/>Pad results with None"]
        CallPython["Call Python Function<br/>await self._acall(*args, **kwargs)"]
        EncodeResult["Encode Result<br/>encoder(python_value)<br/>→ Engine value"]
    end
    
    subgraph PythonExecution["Python Execution"]
        UserFunction["User-Defined Function<br/>Custom transformation logic"]
        ReturnValue["Python Return Value<br/>Native Python types"]
    end
    
    EngineArgs --> DecodeArgs
    DecodeArgs --> CheckNull
    CheckNull -->|"All required args present"| HandleBatching
    CheckNull -->|"Required arg is None"| EngineResult
    HandleBatching --> CallPython
    CallPython --> UserFunction
    UserFunction --> ReturnValue
    ReturnValue --> EncodeResult
    EncodeResult --> EngineResult
```

**Execution Phases**:

### 1. Preparation Phase

Before any invocations, the `prepare()` method is called once per operation instance [python/cocoindex/op.py:353-361]():

- Initializes the executor
- Converts synchronous functions to async using `to_async_call()`
- Prepares any resources needed for execution

### 2. Argument Decoding Phase

Each invocation begins by decoding engine values to Python [python/cocoindex/op.py:363-399]():

```mermaid
flowchart LR
    EngineValue["Engine Value<br/>(binary format)"]
    Decoder["Type-Specific Decoder<br/>Created during registration"]
    CheckRequired["Check if Argument<br/>is Required"]
    PyValue["Python Value<br/>(native type)"]
    NullResult["Return None<br/>(skip execution)"]
    
    EngineValue --> Decoder
    Decoder --> CheckRequired
    CheckRequired -->|"Required & value is None"| NullResult
    CheckRequired -->|"Optional or value present"| PyValue
```

The decoder function is created during registration based on the analyzed type [python/cocoindex/op.py:236-243](). It handles:
- Basic type conversions (int, float, str, etc.)
- Complex type reconstruction (dataclasses, vectors, tables)
- Nested structure decoding

### 3. Batching Optimization

For operations with `batching=True`, special logic handles batch processing [python/cocoindex/op.py:366-383]():

- **Null Skipping**: None values in batches are tracked and skipped during processing
- **Batch Execution**: Only non-None values are passed to the function
- **Result Padding**: None values are re-inserted at correct positions in output

### 4. Python Execution

The actual user-defined function is called with decoded Python values [python/cocoindex/op.py:401-402]():

```python
output = await self._acall(*decoded_args, **decoded_kwargs)
```

### 5. Result Encoding Phase

The Python return value is encoded back to engine format [python/cocoindex/op.py:404-421]():

- Uses the encoder created during registration [python/cocoindex/op.py:323-325]()
- Handles batched results by encoding each element
- Preserves None values and structure

**Sources**: [python/cocoindex/op.py:353-421]()

## Key Components Reference

### Core Classes

| Class | Location | Purpose |
|-------|----------|---------|
| `OpCategory` | [python/cocoindex/op.py:40-48]() | Enum defining operation categories |
| `SpecMeta` | [python/cocoindex/op.py:50-68]() | Metaclass for spec classes, adds dataclass behavior |
| `OpArgs` | [python/cocoindex/op.py:134-156]() | Configuration for operation execution |
| `_WrappedExecutor` | [python/cocoindex/op.py:193-421]() | Runtime wrapper for function execution |
| `_SourceConnector` | [python/cocoindex/op.py:669-744]() | Source operation connector |
| `_TargetConnector` | [python/cocoindex/op.py:819-1071]() | Target operation connector |

### Key Functions

| Function | Location | Purpose |
|----------|----------|---------|
| `_register_op_factory()` | [python/cocoindex/op.py:176-446]() | Core registration logic for operations |
| `function()` | [python/cocoindex/op.py:489-512]() | Decorator for simple function operations |
| `executor_class()` | [python/cocoindex/op.py:448-475]() | Decorator for class-based executors |
| `source_connector()` | [python/cocoindex/op.py:746-766]() | Decorator for source connectors |
| `target_connector()` | [python/cocoindex/op.py:1073-1095]() | Decorator for target connectors |

### Type System Functions

| Function | Purpose |
|----------|---------|
| `analyze_type_info()` | Extract CocoIndex type from Python annotation |
| `make_engine_value_encoder()` | Create Python → Engine encoder [python/cocoindex/op.py:323-325]() |
| `make_engine_value_decoder()` | Create Engine → Python decoder [python/cocoindex/op.py:236-243]() |
| `make_engine_key_encoder()` | Create key encoder for sources/targets [python/cocoindex/op.py:602]() |
| `make_engine_key_decoder()` | Create key decoder for sources/targets [python/cocoindex/op.py:720-722]() |

**Sources**: [python/cocoindex/op.py:1-1102]()

## Operation Lifecycle Summary

```mermaid
stateDiagram-v2
    [*] --> Definition: Apply decorator
    
    state Definition {
        [*] --> ExtractSig: Extract function signature
        ExtractSig --> AnalyzeTypes: Analyze type annotations
        AnalyzeTypes --> CreateCodecs: Create encoders/decoders
        CreateCodecs --> RegisterEngine: Register with Rust engine
        RegisterEngine --> [*]
    }
    
    Definition --> Ready: Registration complete
    
    state Ready {
        [*] --> WaitingForFlow: Operation available
    }
    
    Ready --> Execution: Flow uses operation
    
    state Execution {
        [*] --> AnalyzeSchema: analyze_schema() called once
        AnalyzeSchema --> Prepare: prepare() called once
        Prepare --> WaitingForCall: Ready to process data
        
        WaitingForCall --> DecodeArgs: __call__() invoked
        DecodeArgs --> ExecutePython: Call user function
        ExecutePython --> EncodeResult: Encode return value
        EncodeResult --> WaitingForCall: Ready for next call
    }
    
    Execution --> Ready: Flow execution complete
    Ready --> [*]: Process exit
```

**Key Lifecycle Events**:

1. **Definition/Registration** (Module import time):
   - Decorator applied to Python function/class
   - Type information extracted and analyzed
   - Encoders/decoders created
   - Operation registered with engine

2. **Schema Analysis** (Flow setup time):
   - `analyze_schema()` called with actual argument types from flow
   - Validates argument compatibility
   - Returns expected result type
   - Creates argument-specific decoders

3. **Preparation** (Before first execution):
   - `prepare()` called once per operation instance
   - Initializes resources
   - Sets up async wrappers

4. **Execution** (Runtime, many times):
   - `__call__()` invoked with engine values
   - Arguments decoded to Python
   - User function executed
   - Result encoded back to engine

**Sources**: [python/cocoindex/op.py:176-512]()

---

# Page: Operation Registration and Decorators

# Operation Registration and Decorators

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [docs/docs/core/data_types.mdx](docs/docs/core/data_types.mdx)
- [python/cocoindex/op.py](python/cocoindex/op.py)
- [python/cocoindex/tests/test_typing.py](python/cocoindex/tests/test_typing.py)
- [python/cocoindex/typing.py](python/cocoindex/typing.py)

</details>



## Purpose and Scope

This document explains how to register custom operations in CocoIndex using Python decorators. The operation registration system allows you to extend CocoIndex with custom data sources, transformation functions, and export targets. All operations—both built-in and custom—are registered through a unified interface that bridges Python code to the Rust execution engine.

This page focuses on the decorator API and registration mechanics. For details on type analysis and schema handling, see [Type System and Schema Analysis](#4.2). For information on encoders/decoders, see [Encoders and Decoders](#4.3). For execution configuration options, see [OpArgs and Execution Options](#4.4).

## Operation Categories

CocoIndex defines five operation categories through the `OpCategory` enum:

| Category | Purpose | Decorator | Spec Base Class |
|----------|---------|-----------|-----------------|
| `FUNCTION` | Transform data | `@function`, `@executor_class` | `FunctionSpec` |
| `SOURCE` | Read data from external sources | `@source_connector` | `SourceSpec` |
| `TARGET` | Write data to external targets | `@target_connector` | `TargetSpec` |
| `TARGET_ATTACHMENT` | Attach indexes to targets | (built-in only) | `TargetAttachmentSpec` |
| `DECLARATION` | Declare target schemas | (built-in only) | `DeclarationSpec` |

The first three categories are user-extensible via decorators. Target attachments and declarations are currently built-in only.

**Sources:**
- [python/cocoindex/op.py:55-63]()

## Spec Classes

Every operation is configured through a **spec** (specification) object. Spec classes inherit from base classes defined using the `SpecMeta` metaclass, which automatically converts them into dataclasses.

### Title: Spec Class Hierarchy

```mermaid
graph TB
    SpecMeta["SpecMeta<br/>Metaclass<br/>Auto-applies @dataclass"]
    
    SourceSpec["SourceSpec<br/>category=SOURCE"]
    FunctionSpec["FunctionSpec<br/>category=FUNCTION"]
    TargetSpec["TargetSpec<br/>category=TARGET"]
    TargetAttachmentSpec["TargetAttachmentSpec<br/>category=TARGET_ATTACHMENT"]
    DeclarationSpec["DeclarationSpec<br/>category=DECLARATION"]
    
    CustomSource["Custom source specs<br/>class MySourceSpec(SourceSpec):<br/>    field1: str<br/>    field2: int"]
    CustomFunction["Custom function specs<br/>class MyFunctionSpec(FunctionSpec):<br/>    param: float"]
    CustomTarget["Custom target specs<br/>class MyTargetSpec(TargetSpec):<br/>    connection_string: str"]
    
    SpecMeta --> SourceSpec
    SpecMeta --> FunctionSpec
    SpecMeta --> TargetSpec
    SpecMeta --> TargetAttachmentSpec
    SpecMeta --> DeclarationSpec
    
    SourceSpec --> CustomSource
    FunctionSpec --> CustomFunction
    TargetSpec --> CustomTarget
```

**Sources:**
- [python/cocoindex/op.py:65-104]()

### Defining a Spec Class

```python
from cocoindex.op import FunctionSpec

class MyTransformSpec(FunctionSpec):
    max_length: int
    add_prefix: str = "processed: "
```

The `SpecMeta` metaclass automatically applies `@dataclass` to the subclass, making it instantiable like:

```python
spec = MyTransformSpec(max_length=100, add_prefix="output: ")
```

**Sources:**
- [python/cocoindex/op.py:65-84]()

## The Decorator System

CocoIndex provides four primary decorators for registering operations:

### Title: Decorator Overview

```mermaid
graph TB
    function_decorator["@function<br/>Simplest: decorate a function"]
    executor_decorator["@executor_class<br/>More control: decorate a class"]
    source_decorator["@source_connector<br/>Define data source"]
    target_decorator["@target_connector<br/>Define export target"]
    
    function_impl["def my_func(x: int) -> str:<br/>    return str(x)"]
    executor_impl["class MyExecutor:<br/>    spec: MySpec<br/>    def __call__(self, x: int) -> str:<br/>        return process(x, self.spec)"]
    source_impl["class MySource:<br/>    @staticmethod<br/>    async def create(spec) -> connector<br/>    def list(self) -> Iterator[PartialSourceRow]<br/>    def get_value(self, key) -> PartialSourceRowData"]
    target_impl["class MyTarget:<br/>    @staticmethod<br/>    def get_persistent_key(spec) -> key<br/>    @staticmethod<br/>    def get_setup_state(spec) -> state<br/>    @staticmethod<br/>    async def mutate(*mutations)"]
    
    function_decorator --> function_impl
    executor_decorator --> executor_impl
    source_decorator --> source_impl
    target_decorator --> target_impl
    
    function_impl --> register_func["_engine.register_function_factory()"]
    executor_impl --> register_func
    source_impl --> register_source["_engine.register_source_connector()"]
    target_impl --> register_target["_engine.register_target_connector()"]
```

**Sources:**
- [python/cocoindex/op.py:417-481]()
- [python/cocoindex/op.py:707-728]()
- [python/cocoindex/op.py:983-1005]()

## The `@function` Decorator

The `@function` decorator is the simplest way to register a custom transformation function. It wraps a Python function and automatically handles type marshalling.

### Basic Usage

```python
import cocoindex
from cocoindex import op

@op.function(behavior_version=1)
def add_prefix(text: str) -> str:
    """Add a prefix to text."""
    return f"processed: {text}"
```

### Registration Flow

**Title: Function Registration Process**

```mermaid
graph TB
    UserFunction["User defines:<br/>@op.function(...)<br/>def my_func(arg: T) -> R"]
    
    Decorator["@function decorator"]
    ExtractSig["inspect.signature(fn)<br/>Extract parameters and return type"]
    OpKind["Generate op_kind:<br/>Convert snake_case to CamelCase<br/>my_func -> MyFunc"]
    
    RegisterFactory["_register_op_factory()<br/>category=FUNCTION"]
    
    CreateWrapper["Create _WrappedExecutor:<br/>- Analyze argument types<br/>- Build encoders/decoders<br/>- Handle nullable propagation"]
    
    EngineReg["_engine.register_function_factory()<br/>(op_kind, factory)"]
    
    Registry["Rust ExecutorFactoryRegistry<br/>Stores PyFunctionFactory"]
    
    UserFunction --> Decorator
    Decorator --> ExtractSig
    ExtractSig --> OpKind
    OpKind --> RegisterFactory
    RegisterFactory --> CreateWrapper
    CreateWrapper --> EngineReg
    EngineReg --> Registry
```

**Sources:**
- [python/cocoindex/op.py:458-481]()
- [python/cocoindex/op.py:189-415]()

### Operation Kind Naming

The decorator automatically converts the function name from `snake_case` to `CamelCase` to generate the operation kind:

| Python Function Name | Operation Kind |
|---------------------|----------------|
| `add_prefix` | `AddPrefix` |
| `split_text` | `SplitText` |
| `extract_entities` | `ExtractEntities` |

This operation kind is used when referencing the function in flows and is stored in the registry.

**Sources:**
- [python/cocoindex/op.py:466-467]()

### The `behavior_version` Parameter

When using `cache=True` in `OpArgs`, you must provide a `behavior_version` integer. This version should be incremented whenever the function's behavior changes, invalidating cached results.

```python
@op.function(cache=True, behavior_version=1)
def expensive_transform(data: str) -> str:
    # Expensive computation
    return result
```

If you change the implementation:

```python
@op.function(cache=True, behavior_version=2)  # Increment version
def expensive_transform(data: str) -> str:
    # Modified implementation
    return new_result
```

**Sources:**
- [python/cocoindex/op.py:149-171]()

## The `@executor_class` Decorator

The `@executor_class` decorator provides more control than `@function` by allowing you to define a class with a spec. This is useful when your operation needs configuration or state.

### Basic Usage

```python
from cocoindex.op import FunctionSpec, executor_class
from dataclasses import dataclass

@dataclass
class PrefixSpec(FunctionSpec):
    prefix: str
    suffix: str = ""

@executor_class(behavior_version=1)
class PrefixExecutor:
    spec: PrefixSpec
    
    def __call__(self, text: str) -> str:
        return f"{self.spec.prefix}{text}{self.spec.suffix}"
```

### Requirements

1. **spec field**: The class must have a `spec` field with a type annotation pointing to a `FunctionSpec` subclass
2. **__call__ method**: The class must implement `__call__` with the operation's logic
3. **Type annotations**: Both input arguments and return type must be annotated

### Optional Methods

| Method | Purpose | When Called |
|--------|---------|-------------|
| `prepare()` | Initialize resources (e.g., load models) | Once before any execution |
| `analyze()` | Custom output type analysis | During flow analysis |

**Example with prepare:**

```python
@executor_class(behavior_version=1)
class ModelExecutor:
    spec: ModelSpec
    model: Any = None
    
    def prepare(self) -> None:
        """Load the model once before execution."""
        self.model = load_model(self.spec.model_path)
    
    def __call__(self, text: str) -> str:
        return self.model.predict(text)
```

**Sources:**
- [python/cocoindex/op.py:417-444]()

## The `@source_connector` Decorator

The `@source_connector` decorator registers a custom data source. Sources provide data to flows through a key-value interface.

### Basic Usage

```python
from cocoindex.op import SourceSpec, source_connector, PartialSourceRow, PartialSourceRowData
from dataclasses import dataclass
from typing import Iterator

@dataclass
class MySourceSpec(SourceSpec):
    path: str
    filter: str | None = None

@source_connector(
    spec_cls=MySourceSpec,
    key_type=str,
    value_type=MyDocument
)
class MySourceConnector:
    @staticmethod
    async def create(spec: MySourceSpec) -> "MySourceConnector":
        """Create and initialize the connector."""
        return MySourceConnector(spec)
    
    def __init__(self, spec: MySourceSpec):
        self.spec = spec
    
    def list(self, options) -> Iterator[PartialSourceRow[str, MyDocument]]:
        """List all available keys and optionally their values."""
        for key, doc in self._read_data():
            yield PartialSourceRow(
                key=key,
                data=PartialSourceRowData(value=doc)
            )
    
    def get_value(self, key: str, options) -> PartialSourceRowData[MyDocument]:
        """Retrieve a specific document by key."""
        doc = self._fetch_document(key)
        if doc is None:
            return PartialSourceRowData(value=NON_EXISTENCE)
        return PartialSourceRowData(value=doc)
```

### Required Methods

| Method | Signature | Purpose |
|--------|-----------|---------|
| `create()` | `async def create(spec) -> Connector` | Factory method to create connector instance |
| `list()` | `def list(options) -> Iterator[PartialSourceRow]` | List all keys with optional values |
| `get_value()` | `def get_value(key, options) -> PartialSourceRowData` | Fetch a single value by key |

### Optional Methods

| Method | Signature | Purpose |
|--------|-----------|---------|
| `provides_ordinal()` | `def provides_ordinal() -> bool` | Whether source provides ordinal values for change detection |

### PartialSourceRow and PartialSourceRowData

The `PartialSourceRow` contains:
- `key`: The key identifying the row
- `data`: A `PartialSourceRowData` object

The `PartialSourceRowData` can include:
- `value`: The actual data (or `NON_EXISTENCE` if the row doesn't exist)
- `ordinal`: Optional ordering value for change detection
- `content_version_fp`: Optional fingerprint for change detection

**Sources:**
- [python/cocoindex/op.py:707-728]()
- [python/cocoindex/op.py:489-636]()
- [python/cocoindex/op.py:517-543]()

## The `@target_connector` Decorator

The `@target_connector` decorator registers a custom export target. Targets receive data from flows and persist it to external systems.

### Basic Usage

```python
from cocoindex.op import TargetSpec, target_connector
from dataclasses import dataclass

@dataclass
class MyTargetSpec(TargetSpec):
    connection_string: str
    table_name: str

@dataclass
class MySetupState:
    """Describes the target's current schema and configuration."""
    table_name: str
    columns: list[str]

@target_connector(
    spec_cls=MyTargetSpec,
    persistent_key_type=str,
    setup_state_cls=MySetupState
)
class MyTargetConnector:
    @staticmethod
    def get_persistent_key(spec: MyTargetSpec) -> str:
        """Return a stable identifier for this target."""
        return f"{spec.connection_string}/{spec.table_name}"
    
    @staticmethod
    def get_setup_state(
        spec: MyTargetSpec,
        key_fields_schema: list[FieldSchema],
        value_fields_schema: list[FieldSchema],
        index_options: IndexOptions
    ) -> MySetupState:
        """Determine what setup is needed for this target."""
        columns = [f.name for f in key_fields_schema + value_fields_schema]
        return MySetupState(table_name=spec.table_name, columns=columns)
    
    @staticmethod
    async def apply_setup_change(
        key: str,
        previous_state: MySetupState | None,
        current_state: MySetupState | None
    ) -> None:
        """Apply schema changes (CREATE TABLE, ALTER TABLE, DROP TABLE)."""
        if previous_state is None and current_state is not None:
            # Create table
            create_table(current_state.table_name, current_state.columns)
        elif previous_state is not None and current_state is None:
            # Drop table
            drop_table(previous_state.table_name)
        elif previous_state != current_state:
            # Alter table
            alter_table(previous_state, current_state)
    
    @staticmethod
    async def mutate(
        *mutations: tuple[MyTargetSpec, dict[str, Any]]
    ) -> None:
        """Apply data mutations (INSERT, UPDATE, DELETE)."""
        for spec, mutation_dict in mutations:
            for key, value in mutation_dict.items():
                if value is None:
                    # Delete
                    delete_row(spec.table_name, key)
                else:
                    # Upsert
                    upsert_row(spec.table_name, key, value)
```

### Required Methods

| Method | Purpose | Phase |
|--------|---------|-------|
| `get_persistent_key()` | Generate stable identifier for the target | Setup |
| `apply_setup_change()` | Execute DDL changes (CREATE/DROP/ALTER) | Setup |
| `mutate()` | Execute data changes (INSERT/UPDATE/DELETE) | Runtime |

### Optional Methods

| Method | Purpose |
|--------|---------|
| `get_setup_state()` | Custom logic for determining setup state (default: returns spec) |
| `check_state_compatibility()` | Determine if state changes are backward compatible |
| `prepare()` | Initialize connections before mutations |
| `describe()` | Human-readable description of the resource |

### Setup State Management

The setup state describes the target's schema and configuration. When the flow definition changes, CocoIndex compares the desired setup state with the existing state:

1. **Create**: `previous_state=None, current_state=Some` → Create new resource
2. **Drop**: `previous_state=Some, current_state=None` → Remove resource
3. **Update**: `previous_state=Some, current_state=Some` → Alter resource

**Sources:**
- [python/cocoindex/op.py:983-1005]()
- [python/cocoindex/op.py:735-1011]()

## OpArgs: Execution Configuration

The `OpArgs` dataclass configures how operations execute. It's passed to decorators to control behavior:

```python
@dataclasses.dataclass
class OpArgs:
    gpu: bool = False                          # Execute on GPU
    cache: bool = False                        # Cache results
    batching: bool = False                     # Batch multiple calls
    max_batch_size: int | None = None         # Maximum batch size
    behavior_version: int | None = None       # Cache invalidation version
    timeout: datetime.timedelta | None = None # Execution timeout
    arg_relationship: tuple[ArgRelationship, str] | None = None  # Input-output relationships
```

### Common Patterns

**Caching expensive operations:**

```python
@op.function(cache=True, behavior_version=1)
def expensive_embedding(text: str) -> Vector[Float32, Literal[384]]:
    return model.encode(text)
```

**GPU execution:**

```python
@executor_class(gpu=True, behavior_version=1)
class GPUModelExecutor:
    spec: ModelSpec
    
    def __call__(self, text: str) -> str:
        return gpu_model.predict(text)
```

**Batching for throughput:**

```python
@op.function(batching=True, max_batch_size=32, behavior_version=1)
def batch_embed(texts: list[str]) -> list[Vector[Float32, Literal[768]]]:
    return model.encode_batch(texts)
```

**Custom timeout:**

```python
@op.function(timeout=datetime.timedelta(seconds=60), behavior_version=1)
def slow_operation(data: str) -> str:
    return process_slowly(data)
```

### ArgRelationship

The `arg_relationship` parameter specifies semantic relationships between inputs and outputs:

| Relationship | Meaning |
|--------------|---------|
| `EMBEDDING_ORIGIN_TEXT` | Output is an embedding of the input text |
| `CHUNKS_BASE_TEXT` | Output chunks are derived from the input text |
| `RECTS_BASE_IMAGE` | Output rectangles reference positions in the input image |

**Example:**

```python
@op.function(
    arg_relationship=(ArgRelationship.CHUNKS_BASE_TEXT, "text"),
    behavior_version=1
)
def split_text(text: str, chunk_size: int) -> list[TextChunk]:
    return create_chunks(text, chunk_size)
```

This tells CocoIndex that the output chunks are derived from the `text` argument, enabling features like chunk-to-source mapping.

**Sources:**
- [python/cocoindex/op.py:149-171]()
- [python/cocoindex/op.py:141-147]()

For detailed coverage of `OpArgs`, see [OpArgs and Execution Options](#4.4).

## Registration Internals

### The _register_op_factory Function

All decorator registration paths converge at `_register_op_factory()`, which handles the common registration logic:

**Title: Registration Internal Flow**

```mermaid
graph TB
    Decorator["Decorator:<br/>@function, @executor_class, etc."]
    
    RegisterFactory["_register_op_factory(<br/>category,<br/>expected_args,<br/>expected_return,<br/>executor_factory,<br/>spec_loader,<br/>op_kind,<br/>op_args)"]
    
    CreateWrapper["Create _WrappedExecutor class:<br/>- analyze_schema(args, kwargs)<br/>- prepare()<br/>- __call__(args, kwargs)<br/>- enable_cache()<br/>- behavior_version()<br/>- batching_options()"]
    
    AnalyzePhase["analyze_schema() phase:<br/>1. Build arg encoders/decoders<br/>2. Check nullable propagation<br/>3. Build result encoder<br/>4. Return encoded type"]
    
    PreparePhase["prepare() phase:<br/>Convert executor methods to async"]
    
    ExecutePhase["__call__() phase:<br/>1. Decode Rust args to Python<br/>2. Call executor<br/>3. Encode Python result to Rust"]
    
    EngineFactory["_engine.register_function_factory(<br/>op_kind,<br/>_EngineFunctionExecutorFactory)"]
    
    RustRegistry["Rust: ExecutorFactoryRegistry<br/>Stores PyFunctionFactory"]
    
    Decorator --> RegisterFactory
    RegisterFactory --> CreateWrapper
    CreateWrapper --> AnalyzePhase
    CreateWrapper --> PreparePhase
    CreateWrapper --> ExecutePhase
    RegisterFactory --> EngineFactory
    EngineFactory --> RustRegistry
```

**Sources:**
- [python/cocoindex/op.py:189-415]()

### Type Analysis During Registration

When an operation is registered, CocoIndex analyzes its type signature:

1. **Extract parameter types** from function signature using `inspect.signature()` and type hints
2. **Build encoders** for Python → Rust conversion (see [Encoders and Decoders](#4.3))
3. **Build decoders** for Rust → Python conversion
4. **Determine output nullability** based on input nullability
5. **Encode type information** for the Rust engine

This analysis happens at registration time (when the decorator is applied), not at runtime.

**Sources:**
- [python/cocoindex/op.py:225-359]()

### The _WrappedExecutor Class

The `_WrappedExecutor` class bridges Python functions to Rust executors:

| Method | Purpose | Phase |
|--------|---------|-------|
| `analyze_schema()` | Validate argument types, compute output type | Analysis |
| `prepare()` | Initialize executor, convert methods to async | Before execution |
| `__call__()` | Execute the operation with type conversion | Runtime |
| `enable_cache()` | Return whether caching is enabled | Analysis |
| `behavior_version()` | Return cache version | Analysis |
| `batching_options()` | Return batching configuration | Analysis |
| `timeout()` | Return execution timeout | Analysis |

**Nullable Argument Handling:**

If a required (non-nullable) argument receives `None`, the executor returns `None` immediately without calling the Python function:

```python
async def __call__(self, *args: Any, **kwargs: Any) -> Any:
    decoded_args = []
    for arg_info, arg in zip(self._args_info, args):
        if arg_info.is_required and arg is None:
            return None  # Short-circuit on null
        decoded_args.append(arg_info.decoder(arg))
    # ... execute function ...
```

**Sources:**
- [python/cocoindex/op.py:206-408]()
- [python/cocoindex/op.py:370-391]()

## Engine Bridge: Python to Rust

The registration system bridges Python operations to the Rust execution engine through PyO3. Here's how decorators connect to the engine:

**Title: Python-Rust Bridge Architecture**

```mermaid
graph TB
    subgraph Python["Python Space"]
        Decorator["@function decorator"]
        WrappedExec["_WrappedExecutor<br/>Python execution wrapper"]
        EngineModule["_engine module<br/>PyO3 bindings"]
    end
    
    subgraph "Rust Space"
        PyFactory["PyFunctionFactory<br/>src/ops/py_factory.rs"]
        Registry["ExecutorFactoryRegistry<br/>HashMap<String, ExecutorFactory>"]
        Analyzer["analyzer.rs<br/>Flow analysis"]
        PyExec["PyFunctionExecutor<br/>Async Python caller"]
    end
    
    Decorator --> WrappedExec
    WrappedExec --> EngineModule
    EngineModule -->|"register_function_factory()"| PyFactory
    PyFactory --> Registry
    
    Analyzer -->|"get_function_factory()"| Registry
    Registry -->|"Returns factory"| Analyzer
    Analyzer -->|"factory.build()"| PyFactory
    PyFactory -->|"Calls Python GIL"| WrappedExec
    WrappedExec -->|"analyze_schema()"| PyFactory
    PyFactory -->|"Returns (schema, future)"| Analyzer
    
    PyFactory -->|"Creates executor"| PyExec
    PyExec -->|"Calls at runtime"| WrappedExec
```

**Sources:**
- [python/cocoindex/op.py:189-415]()
- [src/ops/py_factory.rs:120-223]()

### The PyFunctionFactory

The `PyFunctionFactory` (Rust side) wraps Python-registered operations:

1. **During build()**: Acquires Python GIL and calls Python factory's `analyze_schema()` method
2. **Returns**: Output type schema and an async executor future
3. **Executor future**: When awaited, calls Python executor's `prepare()` and wraps in `PyFunctionExecutor`
4. **Runtime**: `PyFunctionExecutor.evaluate()` calls Python's `__call__()` with type conversion

**Sources:**
- [src/ops/py_factory.rs:120-223]()

### The _engine Module

The `_engine` module provides Python bindings to Rust registration functions:

| Python Function | Rust Implementation | Purpose |
|----------------|---------------------|---------|
| `_engine.register_function_factory()` | `register_function_factory_py()` | Register a Python function operation |
| `_engine.register_source_connector()` | `register_source_connector_py()` | Register a Python source connector |
| `_engine.register_target_connector()` | `register_target_connector_py()` | Register a Python target connector |

These bindings are defined in the Rust codebase and exposed to Python via PyO3.

**Sources:**
- [src/ops/py_factory.rs:1013-1047]()

## Common Patterns and Best Practices

### Pattern 1: Stateless Functions

Use `@function` for simple transformations without configuration:

```python
@op.function(behavior_version=1)
def lowercase(text: str) -> str:
    return text.lower()

@op.function(behavior_version=1)
def extract_domain(url: str) -> str:
    from urllib.parse import urlparse
    return urlparse(url).netloc
```

### Pattern 2: Configurable Operations

Use `@executor_class` when you need configuration:

```python
@dataclass
class TruncateSpec(FunctionSpec):
    max_length: int
    suffix: str = "..."

@executor_class(behavior_version=1)
class TruncateExecutor:
    spec: TruncateSpec
    
    def __call__(self, text: str) -> str:
        if len(text) <= self.spec.max_length:
            return text
        return text[:self.spec.max_length] + self.spec.suffix
```

### Pattern 3: Resource Initialization

Use `prepare()` for one-time setup:

```python
@executor_class(cache=True, behavior_version=1)
class EmbeddingExecutor:
    spec: EmbeddingSpec
    model: Any = None
    
    def prepare(self) -> None:
        # Load model once before any execution
        self.model = SentenceTransformer(self.spec.model_name)
    
    def __call__(self, text: str) -> Vector[Float32, Literal[384]]:
        return self.model.encode(text)
```

### Pattern 4: Batch Processing

Use `batching=True` for operations that benefit from batch execution:

```python
@executor_class(batching=True, max_batch_size=32, behavior_version=1)
class BatchEmbeddingExecutor:
    spec: EmbeddingSpec
    model: Any = None
    
    def prepare(self) -> None:
        self.model = SentenceTransformer(self.spec.model_name)
    
    def __call__(self, texts: list[str]) -> list[Vector[Float32, Literal[384]]]:
        # Process batch at once
        return self.model.encode_batch(texts)
    
    def analyze(self) -> type:
        # Return element type (not list type) for batched operations
        return Vector[Float32, Literal[384]]
```

### Pattern 5: Nullable Handling

Operations automatically propagate `None` for required arguments:

```python
@op.function(behavior_version=1)
def process_optional(data: str) -> str:
    # If data is None, this function won't be called
    # The executor returns None automatically
    return data.upper()

@op.function(behavior_version=1)
def process_optional_explicit(data: str | None) -> str | None:
    # For optional arguments, you must handle None explicitly
    if data is None:
        return None
    return data.upper()
```

**Sources:**
- [python/cocoindex/op.py:370-391]()

### Pattern 6: Custom Analysis

Override `analyze()` to compute output types dynamically:

```python
@executor_class(behavior_version=1)
class DynamicOutputExecutor:
    spec: DynamicSpec
    
    def analyze(self) -> type:
        # Compute output type based on spec
        if self.spec.output_format == "list":
            return list[str]
        else:
            return str
    
    def __call__(self, data: Any) -> Any:
        if self.spec.output_format == "list":
            return [process_item(data)]
        else:
            return process_item(data)
```

**Sources:**
- [python/cocoindex/op.py:336-351]()

## Summary

The operation registration system in CocoIndex provides a flexible decorator-based API for extending the framework:

| Decorator | Use Case | Key Requirements |
|-----------|----------|------------------|
| `@function` | Simple stateless transformations | Type-annotated arguments and return |
| `@executor_class` | Configurable operations with state | `spec` field, `__call__` method, type annotations |
| `@source_connector` | Custom data sources | `create()`, `list()`, `get_value()` methods |
| `@target_connector` | Custom export targets | `get_persistent_key()`, `apply_setup_change()`, `mutate()` methods |

All operations are registered with the Rust execution engine through PyO3 bindings, enabling seamless integration of Python code into high-performance data processing pipelines.

For more details:
- Type system and schema analysis: [Type System and Schema Analysis](#4.2)
- Data encoding/decoding: [Encoders and Decoders](#4.3)
- Execution options: [OpArgs and Execution Options](#4.4)

**Sources:**
- [python/cocoindex/op.py:1-1544]()

## Setup State Management

Targets must track and evolve their setup states across flow updates. The setup system uses versioned states with compatibility checking:

### Setup State Lifecycle

```mermaid
stateDiagram-v2
    [*] --> NoState: Initial deployment
    NoState --> Desired: build() creates setup
    
    Desired --> Comparing: Next flow update
    Comparing --> Compatible: Schema unchanged
    Comparing --> PartialCompatible: Backward-compatible change
    Comparing --> NotCompatible: Breaking change
    
    Compatible --> NoChange: diff_setup_states()
    PartialCompatible --> Update: diff_setup_states()
    NotCompatible --> Rebuild: diff_setup_states()
    
    NoChange --> [*]
    Update --> ApplyChanges: apply_setup_changes()
    Rebuild --> DropAndRecreate: apply_setup_changes()
    
    ApplyChanges --> [*]
    DropAndRecreate --> [*]
```

### SetupStateCompatibility Enum

The `SetupStateCompatibility` enum [src/ops/interface.rs:239-250]() determines how state changes are applied:

| Variant | Meaning | Data Loss | Action |
|---------|---------|-----------|--------|
| `Compatible` | Fully compatible | None | In-place update, data preserved |
| `PartialCompatible` | Backward-compatible | Some fields lost | Update with reindexing |
| `NotCompatible` | Breaking change | All data lost | Full rebuild required |

Targets implement `check_state_compatibility()` to return this enum based on state comparison [src/ops/interface.rs:292-296]().

### CombinedState Structure

The `CombinedState<T>` [src/setup/states.rs:43-120]() represents the union of current and staged states:

```rust
pub struct CombinedState<T> {
    pub current: Option<T>,           // Currently applied state
    pub staging: Vec<StateChange<T>>, // Pending changes
    pub legacy_state_key: Option<serde_json::Value>, // Old key format
}
```

Methods:
- `possible_versions()`: Iterator over all possible state versions [src/setup/states.rs:84-88]()
- `always_exists()`: True if no staging deletes [src/setup/states.rs:90-92]()
- `has_state_diff()`: Check if states differ from desired [src/setup/states.rs:110-119]()

**Sources:**
- [src/ops/interface.rs:239-250]()
- [src/setup/states.rs:43-120]()
- [src/setup/driver.rs:341-505]()

---

# Page: Type System and Schema Analysis

# Type System and Schema Analysis

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [docs/docs/core/data_types.mdx](docs/docs/core/data_types.mdx)
- [python/cocoindex/op.py](python/cocoindex/op.py)
- [python/cocoindex/tests/test_typing.py](python/cocoindex/tests/test_typing.py)
- [python/cocoindex/typing.py](python/cocoindex/typing.py)

</details>



## Purpose and Scope

This document details the CocoIndex type system and schema analysis mechanism, which bridges Python type annotations to the Rust engine's type representation. It covers how Python types are analyzed, validated, and encoded into schemas that the engine can understand.

For information about how types are used in operation registration, see [Operation Registration and Decorators](#4.1). For details on how data values are encoded/decoded based on these types, see [Encoders and Decoders](#4.3). For user-facing type documentation, refer to the external documentation on data types.

## Overview

The type system in CocoIndex serves as a critical bridge between Python's dynamic typing and the Rust engine's statically-typed execution model. All type analysis happens at flow definition time, before any data is processed. This enables:

- **Schema inference**: Determining output types without processing actual data
- **Type safety**: Validating type compatibility across Python-Rust boundary
- **Target schema generation**: Creating database schemas with correct column types and vector dimensions
- **Optimization**: Enabling the Rust engine to allocate appropriate data structures

The type system is language-agnostic at its core, defined independently of Python or Rust, allowing CocoIndex to potentially support multiple language frontends in the future.

**Sources:** [python/cocoindex/typing.py:1-50](), [docs/docs/core/data_types.mdx:9-22]()

## Type Analysis Architecture

The following diagram shows how type analysis flows from Python annotations through analysis to engine representation:

```mermaid
graph TB
    PyAnnotation["Python Type Annotation<br/>(e.g., list[Person])"]
    AnalyzeFunc["analyze_type_info()<br/>[typing.py:301-403]"]
    AnalyzedInfo["AnalyzedTypeInfo<br/>core_type, base_type,<br/>variant, attrs, nullable"]
    
    AnalyzedInfo --> EncodeFunc["encode_enriched_type_info()<br/>[typing.py:510-522]"]
    EncodeFunc --> EngineSchema["Engine Schema (dict)<br/>EnrichedValueType JSON"]
    
    PyAnnotation --> AnalyzeFunc
    AnalyzeFunc --> AnalyzedInfo
    
    AnalyzeFunc --> Variants["Type Variant Analysis"]
    
    Variants --> AnalyzedAnyType["AnalyzedAnyType<br/>Missing or Any"]
    Variants --> AnalyzedBasicType["AnalyzedBasicType<br/>Primitives (str, int, etc.)"]
    Variants --> AnalyzedListType["AnalyzedListType<br/>Lists, NDArray, Vector"]
    Variants --> AnalyzedStructType["AnalyzedStructType<br/>dataclass, NamedTuple, Pydantic"]
    Variants --> AnalyzedDictType["AnalyzedDictType<br/>dict, Mapping (KTable)"]
    Variants --> AnalyzedUnionType["AnalyzedUnionType<br/>Union types"]
    Variants --> AnalyzedUnknownType["AnalyzedUnknownType<br/>Unsupported types"]
    
    EngineSchema --> RustEngine["Rust Engine<br/>ValueType processing"]
```

**Sources:** [python/cocoindex/typing.py:174-284](), [python/cocoindex/typing.py:301-403](), [python/cocoindex/typing.py:510-522]()

## The analyze_type_info Function

The `analyze_type_info` function is the entry point for type analysis. It takes any Python type annotation and returns an `AnalyzedTypeInfo` object.

### Function Signature

Located at [python/cocoindex/typing.py:301-403](), the function:

```python
def analyze_type_info(t: Any) -> AnalyzedTypeInfo
```

### Processing Steps

1. **Extract Annotations**: Unwraps `Annotated` types to extract metadata like `TypeKind`, `TypeAttr`, and `VectorInfo` [python/cocoindex/typing.py:306-335]()

2. **Determine Base Type**: Uses `typing.get_origin()` to get the generic base (e.g., `list` from `list[int]`) [python/cocoindex/typing.py:311-321]()

3. **Extract Type Arguments**: Gets generic parameters using `typing.get_args()` [python/cocoindex/typing.py:319]()

4. **Analyze Variant**: Determines which type variant this annotation represents [python/cocoindex/typing.py:336-395]()

5. **Handle Unions and Nullability**: Processes `Union` and `Optional` types, extracting nullability [python/cocoindex/typing.py:358-369]()

6. **Return AnalyzedTypeInfo**: Packages all information into the result object [python/cocoindex/typing.py:397-403]()

**Sources:** [python/cocoindex/typing.py:301-403]()

## AnalyzedTypeInfo Structure

The `AnalyzedTypeInfo` dataclass encapsulates all analyzed information about a type:

| Field | Type | Description |
|-------|------|-------------|
| `core_type` | `Any` | Type without annotations (e.g., `int`, `list[int]`, `dict[str, int]`) |
| `base_type` | `Any` | Type without parameters (e.g., `int`, `list`, `dict`) |
| `variant` | `AnalyzedTypeVariant` | Specific variant classification |
| `attrs` | `dict[str, Any] \| None` | Custom attributes from `TypeAttr` annotations |
| `nullable` | `bool` | Whether the type can be `None` (default: `False`) |

The `variant` field is a union of seven possible type variants, each representing a different category of type.

**Sources:** [python/cocoindex/typing.py:286-299]()

## Type Variants

### Type Variant Hierarchy

```mermaid
graph TB
    AnalyzedTypeVariant["AnalyzedTypeVariant<br/>(Union Type)"]
    
    AnalyzedTypeVariant --> AnalyzedAnyType["AnalyzedAnyType<br/>No specific type info"]
    AnalyzedTypeVariant --> AnalyzedBasicType["AnalyzedBasicType<br/>kind: str<br/>(Str, Int64, Float32, etc.)"]
    AnalyzedTypeVariant --> AnalyzedListType["AnalyzedListType<br/>elem_type: Any<br/>vector_info: VectorInfo | None"]
    AnalyzedTypeVariant --> AnalyzedStructType["AnalyzedStructType<br/>struct_type: type<br/>fields property"]
    AnalyzedTypeVariant --> AnalyzedUnionType["AnalyzedUnionType<br/>variant_types: list[Any]"]
    AnalyzedTypeVariant --> AnalyzedDictType["AnalyzedDictType<br/>key_type: Any<br/>value_type: Any"]
    AnalyzedTypeVariant --> AnalyzedUnknownType["AnalyzedUnknownType<br/>Unsupported type"]
```

**Sources:** [python/cocoindex/typing.py:174-283]()

### AnalyzedAnyType

Represents missing or `Any` type annotations. Used when no specific type information is available.

**When It Occurs:**
- Function parameter annotated with `Any`
- Missing type annotation (equivalent to `inspect.Parameter.empty`)

**Engine Behavior:** The engine cannot proceed with `AnalyzedAnyType` for return values. It's acceptable for function arguments where the engine already knows the type.

**Sources:** [python/cocoindex/typing.py:174-177](), [python/cocoindex/typing.py:340-341]()

### AnalyzedBasicType

Represents primitive scalar types with a `kind` string identifier.

**Supported Kinds:**

| Kind | Python Types | Notes |
|------|--------------|-------|
| `Bytes` | `bytes` | Binary data |
| `Str` | `str` | Text strings |
| `Bool` | `bool` | Boolean values |
| `Int64` | `int`, `np.int64`, `Int64` | 64-bit integers |
| `Float32` | `np.float32`, `Float32` | 32-bit floats |
| `Float64` | `float`, `np.float64`, `Float64` | 64-bit floats |
| `Uuid` | `uuid.UUID` | UUIDs |
| `Date` | `datetime.date` | Dates without time |
| `Time` | `datetime.time` | Time without date |
| `LocalDateTime` | `LocalDateTime` | Datetime without timezone |
| `OffsetDateTime` | `datetime.datetime`, `OffsetDateTime` | Datetime with timezone |
| `TimeDelta` | `datetime.timedelta` | Time durations |
| `Range` | `Range` (tuple[int, int]) | Start and end offsets |
| `Json` | `Json` | Arbitrary JSON data |

**Annotated Types:** CocoIndex exports type aliases like `Int64`, `Float32`, `Range` which are `Annotated` versions of base Python types with explicit `TypeKind` markers [python/cocoindex/typing.py:61-67]().

**Sources:** [python/cocoindex/typing.py:180-186](), [python/cocoindex/typing.py:336-346](), [python/cocoindex/typing.py:371-395]()

### AnalyzedListType

Represents collection types: lists, sequences, NumPy arrays, and vectors.

**Fields:**
- `elem_type`: Type of elements in the collection
- `vector_info`: Optional `VectorInfo(dim=...)` for fixed-dimension vectors

**Python Type Mapping:**

| Python Type | CocoIndex Interpretation |
|-------------|--------------------------|
| `list[T]` | List of elements of type `T` |
| `Sequence[T]` | List of elements of type `T` |
| `NDArray[dtype]` | Vector with NumPy dtype element type |
| `Vector[dtype]` | Vector with element type and optional dimension |
| `Vector[dtype, Literal[N]]` | Vector with fixed dimension `N` |

**Vector Dimension Specification:** When `vector_info.dim` is not `None`, it represents a fixed-dimension vector. This is critical for vector database targets that require knowing dimensions at schema creation time [python/cocoindex/typing.py:42-44](), [python/cocoindex/typing.py:469-483]().

**Sources:** [python/cocoindex/typing.py:188-195](), [python/cocoindex/typing.py:80-104](), [python/cocoindex/typing.py:347-353]()

### AnalyzedStructType

Represents structured types with named fields: dataclasses, NamedTuples, and Pydantic models.

**Supported Struct Types:**

| Python Type | Validation | Mutability | Notes |
|-------------|-----------|------------|-------|
| `@dataclass` | `dataclasses.is_dataclass()` | Mutable (unless `frozen=True`) | Standard Python dataclasses |
| `NamedTuple` | `issubclass(..., tuple)` with `_fields` | Immutable | Tuple-based structs |
| `BaseModel` | `issubclass(..., pydantic.BaseModel)` | Depends on config | Requires `pydantic` package |

**Field Iteration:** The `fields` property provides an iterator over `AnalyzedStructFieldInfo` objects, each containing:
- `name`: Field name
- `type_hint`: Type annotation (from `get_type_hints()`)
- `default_value`: Default value or `inspect.Parameter.empty`
- `description`: Documentation (for Pydantic models)

**Implementation:** [python/cocoindex/typing.py:208-249]()

**Sources:** [python/cocoindex/typing.py:208-250](), [python/cocoindex/typing.py:124-141](), [python/cocoindex/typing.py:342-343]()

### AnalyzedDictType

Represents dictionary/mapping types, which become `KTable` types in CocoIndex (keyed tables).

**Fields:**
- `key_type`: Type of dictionary keys
- `value_type`: Type of dictionary values

**Python Type Mapping:**

| Python Type | CocoIndex Type |
|-------------|----------------|
| `dict[K, V]` | `KTable` with key type `K` and value struct `V` |
| `Mapping[K, V]` | `KTable` with key type `K` and value struct `V` |

**Constraint:** The `value_type` must be a struct type (dataclass, NamedTuple, or Pydantic model) for encoding to succeed [python/cocoindex/typing.py:486-494]().

**Sources:** [python/cocoindex/typing.py:260-267](), [python/cocoindex/typing.py:354-357](), [python/cocoindex/typing.py:486-494]()

### AnalyzedUnionType

Represents union types (`T1 | T2 | ...`), allowing values to be one of multiple types.

**Fields:**
- `variant_types`: List of possible types in the union

**Nullability Handling:** If `None` is one of the variant types, it's removed from `variant_types` and `nullable` is set to `True` on the `AnalyzedTypeInfo` [python/cocoindex/typing.py:358-369]().

**Sources:** [python/cocoindex/typing.py:252-258](), [python/cocoindex/typing.py:358-369]()

### AnalyzedUnknownType

Represents types that CocoIndex doesn't support. Attempting to encode an unknown type raises a `ValueError`.

**Common Unknown Types:**
- `set`, `frozenset`
- Custom classes that aren't dataclasses/NamedTuples/Pydantic models
- Complex generic types not covered by other variants

**Sources:** [python/cocoindex/typing.py:269-272](), [python/cocoindex/typing.py:458-459]()

## Schema Encoding Pipeline

After analysis, types are encoded into a dictionary representation that the Rust engine can deserialize. This encoding happens in two stages:

### Encoding Flow Diagram

```mermaid
graph LR
    AnalyzedTypeInfo["AnalyzedTypeInfo"]
    
    AnalyzedTypeInfo --> EncodeEnrichedTypeInfo["encode_enriched_type_info()<br/>[typing.py:510-522]"]
    
    EncodeEnrichedTypeInfo --> EncodeType["_encode_type()<br/>[typing.py:452-508]"]
    
    EncodeType --> BasicJSON["Basic Type JSON<br/>{kind: 'Str'}"]
    EncodeType --> VectorJSON["Vector Type JSON<br/>{kind: 'Vector', element_type, dimension}"]
    EncodeType --> StructJSON["Struct Type JSON<br/>{kind: 'Struct', fields: [...]}"]
    EncodeType --> TableJSON["Table Type JSON<br/>{kind: 'KTable'/'LTable', row, num_key_parts}"]
    EncodeType --> UnionJSON["Union Type JSON<br/>{kind: 'Union', types: [...]}"]
    
    EncodeEnrichedTypeInfo --> EnrichedJSON["EnrichedValueType JSON<br/>{type: {...}, nullable, attrs}"]
    
    EnrichedJSON --> RustEngine["Rust Engine<br/>serde deserialization"]
```

**Sources:** [python/cocoindex/typing.py:452-522]()

### encode_enriched_type_info Function

Located at [python/cocoindex/typing.py:510-522](), this function converts `AnalyzedTypeInfo` to engine JSON:

**Output Structure:**
```json
{
  "type": { "kind": "...", ... },
  "nullable": true,  // if type is nullable
  "attrs": { ... }   // if type has attributes
}
```

**Type Encoding:** Delegates to `_encode_type()` for the `"type"` field [python/cocoindex/typing.py:452-508]().

**Attribute Propagation:** Custom attributes from `TypeAttr` annotations are included in the `"attrs"` field [python/cocoindex/typing.py:516-517]().

**Error Handling:** Raises `ValueError` for `AnalyzedAnyType` or `AnalyzedUnknownType` [python/cocoindex/typing.py:455-459]().

**Sources:** [python/cocoindex/typing.py:510-522]()

### Struct Schema Encoding

For struct types, the encoding process uses `_encode_struct_schema()` [python/cocoindex/typing.py:406-449]() which:

1. **Iterates Fields**: Calls `field.fields` to get all struct fields
2. **Recursively Encodes**: Each field type is analyzed and encoded
3. **Adds Metadata**: Field names, descriptions (for Pydantic), and optionally struct documentation
4. **Handles Keys**: For `KTable` types, key fields are prepended to the field list

**Key Field Handling:** When encoding a `dict[K, V]` type:
- If `K` is a basic type, a single key field named `_key` is added [python/cocoindex/typing.py:108]()
- If `K` is a struct, all its fields become key fields
- The `num_key_parts` indicates how many leading fields are keys [python/cocoindex/typing.py:432-442]()

**Sources:** [python/cocoindex/typing.py:406-449]()

## Engine Schema Representation

The engine schema is represented by Python classes that mirror the Rust type system. These classes provide bidirectional encoding/decoding between Python dictionaries and typed objects.

### Schema Class Hierarchy

```mermaid
graph TB
    ValueType["ValueType<br/>(Type Alias)"]
    
    ValueType --> BasicValueType["BasicValueType<br/>kind: str<br/>vector: VectorTypeSchema?<br/>union: UnionTypeSchema?"]
    ValueType --> StructType["StructType<br/>kind='Struct'<br/>fields: list[FieldSchema]"]
    ValueType --> TableType["TableType<br/>kind='KTable'|'LTable'<br/>row: StructSchema<br/>num_key_parts: int?"]
    
    BasicValueType --> VectorTypeSchema["VectorTypeSchema<br/>element_type: BasicValueType<br/>dimension: int?"]
    BasicValueType --> UnionTypeSchema["UnionTypeSchema<br/>variants: list[BasicValueType]"]
    
    StructType --> StructSchema["StructSchema<br/>fields: list[FieldSchema]<br/>description: str?"]
    TableType --> StructSchema
    
    StructSchema --> FieldSchema["FieldSchema<br/>name: str<br/>value_type: EnrichedValueType<br/>description: str?"]
    
    FieldSchema --> EnrichedValueType["EnrichedValueType<br/>type: ValueType<br/>nullable: bool<br/>attrs: dict?"]
    
    EnrichedValueType --> ValueType
```

**Sources:** [python/cocoindex/typing.py:549-813]()

### BasicValueType

Represents primitive types and vectors/unions. Located at [python/cocoindex/typing.py:599-667]().

**Fields:**
- `kind`: One of the primitive type kind strings (e.g., `"Str"`, `"Int64"`, `"Vector"`)
- `vector`: Present only when `kind == "Vector"`
- `union`: Present only when `kind == "Union"`

**Codec Methods:**
- `decode(obj: dict) -> BasicValueType`: Deserializes from engine JSON
- `encode() -> dict`: Serializes to engine JSON

**Sources:** [python/cocoindex/typing.py:599-667]()

### StructType and StructSchema

`StructSchema` is the base class containing field definitions and optional description [python/cocoindex/typing.py:732-756](). `StructType` extends it with `kind='Struct'` [python/cocoindex/typing.py:758-773]().

**Key Difference:**
- `StructSchema`: Used for describing row structure in tables
- `StructType`: Used for standalone struct values

Both share the same field list representation.

**Sources:** [python/cocoindex/typing.py:732-773]()

### TableType

Represents collection types: `KTable` (keyed) and `LTable` (list) [python/cocoindex/typing.py:775-811]().

**Fields:**
- `kind`: Either `"KTable"` or `"LTable"`
- `row`: `StructSchema` describing row structure
- `num_key_parts`: Number of leading fields that form the key (KTable only)

**KTable Example:**
```python
# dict[str, Person] where Person has fields (first_name, last_name, dob)
TableType(
    kind="KTable",
    row=StructSchema(fields=[
        FieldSchema(name="_key", value_type=EnrichedValueType(type=BasicValueType(kind="Str"))),
        FieldSchema(name="first_name", value_type=EnrichedValueType(type=BasicValueType(kind="Str"))),
        FieldSchema(name="last_name", value_type=EnrichedValueType(type=BasicValueType(kind="Str"))),
        FieldSchema(name="dob", value_type=EnrichedValueType(type=BasicValueType(kind="Date")))
    ]),
    num_key_parts=1
)
```

**Sources:** [python/cocoindex/typing.py:775-811]()

### FieldSchema

Represents a single field in a struct or table [python/cocoindex/typing.py:704-730]().

**Fields:**
- `name`: Field name
- `value_type`: `EnrichedValueType` containing the field's type and metadata
- `description`: Optional documentation string (from Pydantic field descriptions)

**Sources:** [python/cocoindex/typing.py:704-730]()

### EnrichedValueType

Wraps a `ValueType` with additional metadata [python/cocoindex/typing.py:669-702]().

**Fields:**
- `type`: The actual `ValueType` (BasicValueType, StructType, or TableType)
- `nullable`: Whether the value can be `None`
- `attrs`: Optional dictionary of custom attributes

This is the top-level type representation that gets passed to/from the Rust engine.

**Sources:** [python/cocoindex/typing.py:669-702]()

## Type Analysis in Operation Registration

When operations are registered (functions, sources, targets), the type system validates arguments and determines return types. This happens in the `analyze_schema` method of operation executors.

### Function Operation Type Flow

```mermaid
graph TB
    PyFunction["Python Function<br/>with type annotations"]
    
    PyFunction --> RegisterOp["_register_op_factory()<br/>[op.py:189-414]"]
    
    RegisterOp --> WrappedExecutor["_WrappedExecutor<br/>created per operation"]
    
    WrappedExecutor --> AnalyzeSchema["analyze_schema(*args, **kwargs)<br/>[op.py:225-358]"]
    
    AnalyzeSchema --> ProcessArgs["Process each argument<br/>[op.py:237-263]"]
    
    ProcessArgs --> AnalyzeArgType["analyze_type_info(arg_param.annotation)<br/>For each parameter"]
    
    AnalyzeArgType --> CreateDecoder["make_engine_value_decoder()<br/>[op.py:254-256]"]
    
    AnalyzeSchema --> AnalyzeReturn["analyze_type_info(expected_return)<br/>[op.py:331]"]
    
    AnalyzeReturn --> CreateEncoder["make_engine_value_encoder()<br/>[op.py:332-334]"]
    
    AnalyzeReturn --> EncodeType["encode_enriched_type_info()<br/>[op.py:356]"]
    
    EncodeType --> ReturnToEngine["Return encoded type to engine"]
```

**Sources:** [python/cocoindex/op.py:189-414]()

### Argument Type Processing

For each operation argument [python/cocoindex/op.py:237-263]():

1. **Analyze Parameter Type**: Calls `analyze_type_info()` on the parameter's type annotation
2. **Decode Engine Type**: The engine passes the actual argument type as `_engine.OpArgSchema`
3. **Create Decoder**: Builds a decoder to convert engine values to Python values
4. **Validate Nullability**: Checks if required (non-optional) arguments might receive `None`
5. **Store ArgInfo**: Saves decoder and required flag for execution time

**Special Handling:**
- **Batching Functions**: Uses `_make_batched_engine_value_decoder()` to handle lists of values [python/cocoindex/op.py:179-186]()
- **Argument Relationships**: Extracts `arg_relationship` metadata to track provenance [python/cocoindex/op.py:243-246]()

**Sources:** [python/cocoindex/op.py:237-263](), [python/cocoindex/op.py:179-186]()

### Return Type Processing

For the operation's return value [python/cocoindex/op.py:331-358]():

1. **Analyze Return Type**: Calls `analyze_type_info()` on the return annotation
2. **Create Encoder**: Builds an encoder to convert Python return values to engine format
3. **Handle Custom analyze() Method**: If the executor provides an `analyze()` method, use its return type
4. **Adjust for Batching**: For batching functions, extract element type from list return
5. **Add Attributes**: Includes relationship attributes if `arg_relationship` was specified
6. **Mark Nullable**: Sets nullable if any required argument could be `None`
7. **Encode for Engine**: Calls `encode_enriched_type_info()` to produce final JSON

**Sources:** [python/cocoindex/op.py:331-358]()

## Type Analysis Examples

### Example 1: Basic Function Type Analysis

```python
@cocoindex.op.function(behavior_version=1)
def format_date(date: datetime.date) -> str:
    return date.strftime("%Y-%m-%d")
```

**Analysis Process:**
1. `date` parameter: `analyze_type_info(datetime.date)` → `AnalyzedTypeInfo(variant=AnalyzedBasicType(kind="Date"))`
2. Return type: `analyze_type_info(str)` → `AnalyzedTypeInfo(variant=AnalyzedBasicType(kind="Str"))`
3. Encoded return: `{"type": {"kind": "Str"}}`

**Sources:** [python/cocoindex/typing.py:301-403](), [python/cocoindex/op.py:331-334]()

### Example 2: Vector Type with Dimension

```python
from typing import Literal
import numpy as np
from numpy.typing import NDArray

@cocoindex.op.function(behavior_version=1)
def embed_text(text: str) -> cocoindex.Vector[np.float32, Literal[768]]:
    # ... embedding logic
    return embedding_array
```

**Analysis Process:**
1. `Vector[np.float32, Literal[768]]` is an `Annotated[NDArray[np.float32], VectorInfo(dim=768)]`
2. `analyze_type_info()` extracts `VectorInfo(dim=768)` from annotations
3. Recognizes `NDArray[np.float32]` → `AnalyzedListType(elem_type=np.float32, vector_info=VectorInfo(dim=768))`
4. `elem_type` analysis: `np.float32` → `AnalyzedBasicType(kind="Float32")`
5. Encoded return: `{"type": {"kind": "Vector", "element_type": {"kind": "Float32"}, "dimension": 768}}`

**Sources:** [python/cocoindex/typing.py:80-104](), [python/cocoindex/typing.py:347-353](), [python/cocoindex/typing.py:469-483]()

### Example 3: Struct Type with Nested Fields

```python
from dataclasses import dataclass

@dataclass
class Person:
    name: str
    age: int
    dob: datetime.date

@cocoindex.op.function(behavior_version=1)
def parse_person_data(data: dict) -> Person:
    return Person(
        name=data["name"],
        age=data["age"],
        dob=datetime.date.fromisoformat(data["dob"])
    )
```

**Analysis Process:**
1. Return type: `analyze_type_info(Person)` → `AnalyzedTypeInfo(variant=AnalyzedStructType(struct_type=Person))`
2. Struct field iteration via `AnalyzedStructFieldInfo`:
   - `name: str` → `{"name": "name", "type": {"kind": "Str"}}`
   - `age: int` → `{"name": "age", "type": {"kind": "Int64"}}`
   - `dob: datetime.date` → `{"name": "dob", "type": {"kind": "Date"}}`
3. Encoded return: `{"type": {"kind": "Struct", "fields": [...]}}`

**Sources:** [python/cocoindex/typing.py:208-249](), [python/cocoindex/typing.py:406-449]()

### Example 4: KTable Type (Dictionary)

```python
@cocoindex.op.function(behavior_version=1)
def group_people(people: list[Person]) -> dict[str, Person]:
    return {p.name: p for p in people}
```

**Analysis Process:**
1. Return type: `analyze_type_info(dict[str, Person])`
2. Recognized as `AnalyzedDictType(key_type=str, value_type=Person)`
3. Key type: `str` → `AnalyzedBasicType(kind="Str")` → creates `_key` field
4. Value type: `Person` → `AnalyzedStructType(struct_type=Person)` → expands to fields
5. Encoded: `{"type": {"kind": "KTable", "row": {"fields": [{"name": "_key", ...}, {"name": "name", ...}, ...]}, "num_key_parts": 1}}`

**Sources:** [python/cocoindex/typing.py:354-357](), [python/cocoindex/typing.py:486-499]()

### Example 5: Optional/Nullable Type

```python
@cocoindex.op.function(behavior_version=1)
def optional_upper(text: str | None) -> str | None:
    if text is None:
        return None
    return text.upper()
```

**Analysis Process:**
1. Parameter: `analyze_type_info(str | None)` 
2. Recognizes union with `None` → extracts `str`, sets `nullable=True`
3. Result: `AnalyzedTypeInfo(variant=AnalyzedBasicType(kind="Str"), nullable=True)`
4. Encoded: `{"type": {"kind": "Str"}, "nullable": true}`

**Sources:** [python/cocoindex/typing.py:358-369]()

## Type Validation and Error Handling

The type system enforces constraints at different stages:

### Analysis-Time Validation

| Validation | Location | Error Message |
|------------|----------|---------------|
| NDArray without dtype | [typing.py:161-170]() | "NDArray for Vector must use a concrete numpy dtype" |
| Unsupported dtype | [typing.py:165-170]() | "Unsupported NumPy dtype in NDArray" |
| Unknown type | [typing.py:458-459]() | "Unsupported type annotation: ..." |
| Missing type | [typing.py:455-456]() | "Specific type annotation is expected" |
| KTable non-struct value | [typing.py:486-490]() | "KTable value must have a Struct type" |
| LTable with vector info | [typing.py:472-474]() | "LTable type must not have a vector info" |

### Operation Registration Validation

| Validation | Location | Error Message |
|------------|----------|---------------|
| Missing spec field | [op.py:429-430]() | "Expect a `spec` field with type hint" |
| Too many arguments | [op.py:269-271]() | "Too many arguments passed in" |
| Unexpected keyword | [op.py:304-306]() | "Unexpected keyword argument passed in" |
| Missing arguments | [op.py:328-329]() | "Missing arguments: ..." |
| Batching with multiple args | [op.py:203-204]() | "Batching is only supported for single argument functions" |

**Sources:** [python/cocoindex/typing.py:161-170](), [python/cocoindex/typing.py:455-490](), [python/cocoindex/op.py:203-329]()

## Utility Functions and Type Checking

### Type Checking Utilities

The module provides several helper functions for type identification:

| Function | Purpose | Location |
|----------|---------|----------|
| `is_numpy_number_type(t)` | Check if type is numpy numeric | [typing.py:120-122]() |
| `is_namedtuple_type(t)` | Check if type is NamedTuple | [typing.py:124-126]() |
| `is_pydantic_model(t)` | Check if type is Pydantic BaseModel | [typing.py:128-136]() |
| `is_struct_type(t)` | Check if dataclass/NamedTuple/Pydantic | [typing.py:138-141]() |

### Type Extraction Utilities

| Function | Purpose | Location |
|----------|---------|----------|
| `extract_ndarray_elem_dtype(t)` | Get dtype from NDArray type | [typing.py:111-117]() |
| `resolve_forward_ref(t)` | Resolve string type references | [typing.py:543-546]() |

### Encoding/Decoding Utilities

| Function | Purpose | Location |
|----------|---------|----------|
| `encode_enriched_type(t)` | Convert Python type to engine JSON | [typing.py:533-540]() |
| `decode_engine_field_schemas(objs)` | Deserialize field schemas | [typing.py:816-817]() |
| `decode_engine_value_type(obj)` | Deserialize ValueType | [typing.py:820-830]() |
| `encode_engine_value_type(vt)` | Serialize ValueType | [typing.py:832-835]() |

**Sources:** [python/cocoindex/typing.py:111-141](), [python/cocoindex/typing.py:533-546](), [python/cocoindex/typing.py:816-835]()

## Special Type Constructs

### Vector Type

The `Vector` type is a custom generic that provides dimension-aware vector annotations:

```python
# Vector without dimension
embedding: Vector[np.float32]

# Vector with fixed dimension  
embedding: Vector[np.float32, Literal[768]]

# Works with non-numpy types too
tags: Vector[str, Literal[5]]
```

**Implementation:** `Vector.__class_getitem__()` creates appropriate `Annotated` types with `VectorInfo` [python/cocoindex/typing.py:80-104]().

**Backend Selection:**
- NumPy numeric types → `NDArray[dtype]`
- Other types → `list[dtype]`

**Sources:** [python/cocoindex/typing.py:69-105]()

### Type Aliases for Clarity

CocoIndex exports type aliases to disambiguate Python types that map to multiple CocoIndex types:

| Alias | Python Base | CocoIndex Type | Purpose |
|-------|-------------|----------------|---------|
| `Int64` | `int` | Int64 | Explicit 64-bit integer |
| `Float32` | `float` | Float32 | 32-bit float (vs default Float64) |
| `Float64` | `float` | Float64 | Explicit 64-bit float |
| `Range` | `tuple[int, int]` | Range | Character range (start, end) |
| `LocalDateTime` | `datetime.datetime` | LocalDateTime | Datetime without timezone |
| `OffsetDateTime` | `datetime.datetime` | OffsetDateTime | Datetime with timezone |
| `Json` | `Any` | Json | Arbitrary JSON data |

These are `Annotated` types with `TypeKind` markers [python/cocoindex/typing.py:61-67]().

**Sources:** [python/cocoindex/typing.py:61-67]()

### DtypeRegistry

Validates and maps NumPy dtypes to CocoIndex type kinds [python/cocoindex/typing.py:144-172]().

**Supported Mappings:**

| NumPy dtype | CocoIndex Kind |
|-------------|----------------|
| `np.float32` | `Float32` |
| `np.float64` | `Float64` |
| `np.int64` | `Int64` |

**Validation:** Raises `ValueError` for unsupported dtypes like `np.int32` or `np.uint64`.

**Sources:** [python/cocoindex/typing.py:144-172]()

## Integration with Rust Engine

### Type JSON Format

The engine expects types in a specific JSON format. Here's the complete schema:

**EnrichedValueType:**
```json
{
  "type": { /* ValueType */ },
  "nullable": true,     // optional
  "attrs": { /* ... */ } // optional
}
```

**ValueType Variants:**

**Basic Type:**
```json
{
  "kind": "Str"  // or "Int64", "Float32", etc.
}
```

**Vector Type:**
```json
{
  "kind": "Vector",
  "element_type": { "kind": "Float32" },
  "dimension": 768  // optional
}
```

**Union Type:**
```json
{
  "kind": "Union",
  "types": [
    { "kind": "Int64" },
    { "kind": "Float64" }
  ]
}
```

**Struct Type:**
```json
{
  "kind": "Struct",
  "fields": [
    {
      "name": "field1",
      "type": { "kind": "Str" },
      "description": "..."  // optional
    }
  ],
  "description": "..."  // optional
}
```

**Table Type (KTable):**
```json
{
  "kind": "KTable",
  "row": {
    "fields": [
      { "name": "_key", "type": { "kind": "Str" } },
      { "name": "value1", "type": { "kind": "Int64" } }
    ]
  },
  "num_key_parts": 1
}
```

**Table Type (LTable):**
```json
{
  "kind": "LTable",
  "row": {
    "fields": [
      { "name": "col1", "type": { "kind": "Str" } }
    ]
  }
}
```

**Sources:** [python/cocoindex/typing.py:452-522](), [python/cocoindex/typing.py:599-811]()

### Engine Communication

The Rust engine receives type information through:

1. **Operation Registration**: When `_engine.register_function_factory()` is called [python/cocoindex/op.py:410-412]()
2. **Schema Analysis**: The `analyze_schema()` method returns encoded type [python/cocoindex/op.py:356-358]()
3. **Target Connectors**: Target registration passes table types [python/cocoindex/op.py:703-704]()

The engine uses these schemas to:
- Allocate appropriate data structures
- Validate data flow compatibility
- Generate target database schemas
- Optimize memory layouts and execution plans

**Sources:** [python/cocoindex/op.py:356-358](), [python/cocoindex/op.py:410-412](), [python/cocoindex/op.py:703-704]()

---

# Page: Encoders and Decoders

# Encoders and Decoders

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [docs/docs/core/data_types.mdx](docs/docs/core/data_types.mdx)
- [python/cocoindex/op.py](python/cocoindex/op.py)
- [python/cocoindex/tests/test_typing.py](python/cocoindex/tests/test_typing.py)
- [python/cocoindex/typing.py](python/cocoindex/typing.py)

</details>



## Purpose and Scope

The encoder and decoder system provides the critical data marshalling layer between Python and Rust in CocoIndex. This system handles automatic conversion of data values as they cross the Python-Rust boundary, ensuring type-safe data flow through the execution pipeline.

This page covers:
- How encoders convert Python values to Rust engine format
- How decoders convert Rust engine values back to Python
- Type analysis that drives encoder/decoder generation
- Special handling for batching, None values, and different operation types

For information about the broader type system and type annotations, see [Type System and Schema Analysis](#4.2). For details on how operations are registered and use these encoders/decoders, see [Operation Registration and Decorators](#4.1).

## System Overview

CocoIndex is built with a Rust execution engine and Python user interface. Every data value that flows through an operation must cross the Python-Rust boundary at least twice: once when entering the Rust engine (encoding) and once when returning to Python (decoding).

The encoder/decoder system provides:
- **Automatic type conversion** based on Python type annotations
- **Zero-copy optimization** where possible for performance
- **Type-safe marshalling** with validation at registration time
- **Special handling** for optional values, batching, and complex types

```mermaid
graph LR
    subgraph "Python Space"
        PyValue["Python Value<br/>(e.g., str, dict, list)"]
        PyReturn["Python Return Value"]
    end
    
    subgraph "Boundary Layer"
        Encoder["Encoder<br/>make_engine_value_encoder"]
        Decoder["Decoder<br/>make_engine_value_decoder"]
    end
    
    subgraph "Rust Engine Space"
        EngineValue["Engine Value<br/>(serialized format)"]
        EngineResult["Engine Result"]
    end
    
    PyValue -->|"encode()"| Encoder
    Encoder --> EngineValue
    EngineValue -->|"processing"| EngineResult
    EngineResult --> Decoder
    Decoder -->|"decode()"| PyReturn
```

**Sources:** [python/cocoindex/op.py:26-32](), [python/cocoindex/op.py:179-186]()

## Encoder Creation and Types

Encoders convert Python values into the format expected by the Rust engine. The system provides specialized encoders for different use cases.

### Encoder Factory Functions

The encoder factory functions create encoder callables during operation registration:

| Function | Purpose | Used For |
|----------|---------|----------|
| `make_engine_value_encoder(analyzed_type_info)` | Encodes regular values | Function return values, source values |
| `make_engine_key_encoder(analyzed_type_info)` | Encodes key values | Source keys, target keys |

The key difference between value and key encoders is that key encoders handle both simple keys (single field) and composite keys (multiple fields in a struct).

### Encoder Usage in Operation Registration

```mermaid
graph TB
    subgraph "Operation Registration"
        RegStart["@executor_class or @function decorator"]
        AnalyzeReturn["Analyze return type annotation"]
        CreateEncoder["make_engine_value_encoder(analyzed_type_info)"]
    end
    
    subgraph "Runtime Execution"
        FuncCall["Function called with args"]
        PyReturn["Python return value"]
        EncoderCall["encoder(python_value)"]
        EngineVal["Engine value (dict/list/primitive)"]
    end
    
    RegStart --> AnalyzeReturn
    AnalyzeReturn -->|"AnalyzedTypeInfo"| CreateEncoder
    CreateEncoder -->|"store encoder"| FuncCall
    FuncCall --> PyReturn
    PyReturn --> EncoderCall
    EncoderCall --> EngineVal
```

During operation registration, the system:
1. Analyzes the return type annotation
2. Creates an appropriate encoder function
3. Stores the encoder in the wrapped executor
4. At runtime, applies the encoder to every return value

**Sources:** [python/cocoindex/op.py:331-334](), [python/cocoindex/op.py:388-390]()

### Example: Function Encoder

```mermaid
sequenceDiagram
    participant Reg as Registration Phase
    participant Wrap as _WrappedExecutor
    participant Runtime as Runtime Phase
    participant Encoder as encoder()
    
    Reg->>Wrap: analyze_schema(*args, **kwargs)
    Wrap->>Wrap: analyze_type_info(return_annotation)
    Wrap->>Wrap: make_engine_value_encoder(analyzed_type_info)
    Note over Wrap: Store encoder in self._result_encoder
    
    Runtime->>Wrap: __call__(*args, **kwargs)
    Wrap->>Wrap: Execute function → python_result
    Wrap->>Encoder: self._result_encoder(python_result)
    Encoder->>Runtime: engine_value (dict/list/etc)
```

**Sources:** [python/cocoindex/op.py:331-334](), [python/cocoindex/op.py:388-390]()

## Decoder Creation and Types

Decoders convert Rust engine values back to Python types. The system provides multiple decoder types for different scenarios.

### Decoder Factory Functions

| Function | Signature | Purpose |
|----------|-----------|---------|
| `make_engine_value_decoder(field_path, src_type, dst_type_info)` | Returns `Callable[[Any], Any]` | Decodes function arguments |
| `make_engine_key_decoder(field_path, key_fields_schema, dst_type_info)` | Returns `Callable[[Any], Any]` | Decodes source/target keys |
| `make_engine_struct_decoder(field_path, value_fields_schema, dst_type_info)` | Returns `Callable[[Any], Any]` | Decodes struct values (targets) |
| `_make_batched_engine_value_decoder(field_path, src_type, dst_type_info)` | Returns `Callable[[Any], list]` | Decodes batched arguments |

The `field_path` parameter is used for error reporting, tracking the location of decoding failures in nested structures.

**Sources:** [python/cocoindex/op.py:179-186](), [python/cocoindex/op.py:247-256](), [python/cocoindex/op.py:681-683](), [python/cocoindex/op.py:885-891]()

### Decoder Usage in Function Operations

```mermaid
graph TB
    subgraph "analyze_schema Phase"
        AnalyzeArg["For each argument parameter"]
        GetEngineType["Get engine type from OpArgSchema"]
        DecodeEngineType["EnrichedValueType.decode(arg.value_type)"]
        AnalyzePyType["analyze_type_info(param.annotation)"]
        CreateDecoder["make_engine_value_decoder(field_path, enriched.type, type_info)"]
    end
    
    subgraph "Runtime __call__ Phase"
        GetEngineValue["Receive engine value"]
        CheckRequired["Check if required and None"]
        ApplyDecoder["decoder(engine_value)"]
        PyValue["Python value ready for function"]
    end
    
    AnalyzeArg --> GetEngineType
    GetEngineType --> DecodeEngineType
    DecodeEngineType --> AnalyzePyType
    AnalyzePyType --> CreateDecoder
    
    CreateDecoder -->|"store in _ArgInfo"| GetEngineValue
    GetEngineValue --> CheckRequired
    CheckRequired -->|"not None or optional"| ApplyDecoder
    CheckRequired -->|"None and required"| ReturnNone["Return None (propagate)"]
    ApplyDecoder --> PyValue
```

**Sources:** [python/cocoindex/op.py:236-263](), [python/cocoindex/op.py:370-390]()

### Batching Decoder

For operations with `batching=True`, arguments are passed as lists and need special handling:

```python
# python/cocoindex/op.py:179-186
def _make_batched_engine_value_decoder(
    field_path: list[str], src_type: ValueType, dst_type_info: AnalyzedTypeInfo
) -> Callable[[Any], Any]:
    if not isinstance(dst_type_info.variant, AnalyzedListType):
        raise ValueError("Expected arguments for batching function to be a list type")
    elem_type_info = analyze_type_info(dst_type_info.variant.elem_type)
    base_decoder = make_engine_value_decoder(field_path, src_type, elem_type_info)
    return lambda value: [base_decoder(v) for v in value]
```

The batched decoder wraps the element decoder and applies it to each item in the batch.

**Sources:** [python/cocoindex/op.py:179-186](), [python/cocoindex/op.py:249-252]()

## Type Analysis Pipeline

The encoder/decoder system relies on type analysis to understand Python type annotations and convert them to engine schemas.

### Type Analysis Flow

```mermaid
graph TB
    subgraph "Input"
        PyAnnotation["Python Type Annotation<br/>e.g., list[Person], dict[str, int]"]
    end
    
    subgraph "analyze_type_info Function"
        ExtractOrigin["Extract typing.get_origin()"]
        ExtractArgs["Extract typing.get_args()"]
        ExtractAnnotations["Extract Annotated metadata"]
        DetermineVariant["Determine variant type:<br/>Basic, List, Struct, Dict, Union"]
        CreateAnalyzed["Create AnalyzedTypeInfo"]
    end
    
    subgraph "Output"
        AnalyzedInfo["AnalyzedTypeInfo<br/>core_type, base_type, variant, attrs, nullable"]
    end
    
    PyAnnotation --> ExtractOrigin
    ExtractOrigin --> ExtractArgs
    ExtractArgs --> ExtractAnnotations
    ExtractAnnotations --> DetermineVariant
    DetermineVariant --> CreateAnalyzed
    CreateAnalyzed --> AnalyzedInfo
```

**Sources:** [python/cocoindex/typing.py:301-403]()

### AnalyzedTypeInfo Structure

The `AnalyzedTypeInfo` dataclass contains all information needed for encoding/decoding:

```python
# python/cocoindex/typing.py:286-299
@dataclasses.dataclass
class AnalyzedTypeInfo:
    """Analyzed info of a Python type."""
    core_type: Any          # Type without annotations, e.g., list[int]
    base_type: Any          # Type without parameters, e.g., list
    variant: AnalyzedTypeVariant  # One of: Any, Basic, List, Struct, Union, Dict, Unknown
    attrs: dict[str, Any] | None  # Custom attributes from Annotated
    nullable: bool = False        # Whether type allows None
```

The `variant` field determines which encoder/decoder strategy to use:

| Variant Type | Example | Encoding Strategy |
|--------------|---------|-------------------|
| `AnalyzedBasicType` | `str`, `int`, `float` | Direct primitive conversion |
| `AnalyzedListType` | `list[str]`, `NDArray[np.float32]` | Iterate and encode elements |
| `AnalyzedStructType` | `@dataclass Person` | Field-by-field encoding |
| `AnalyzedDictType` | `dict[str, Person]` | Key-value pair encoding |
| `AnalyzedUnionType` | `str \| int` | Type tag + value encoding |

**Sources:** [python/cocoindex/typing.py:174-283](), [python/cocoindex/typing.py:286-299]()

### Schema Encoding

After type analysis, `encode_enriched_type_info` converts the analyzed type to engine format:

```mermaid
graph LR
    subgraph "Python Side"
        AnalyzedType["AnalyzedTypeInfo"]
        EncodeFunc["encode_enriched_type_info()"]
    end
    
    subgraph "Engine Format"
        EnrichedDict["dict with 'type' key<br/>+ optional 'attrs', 'nullable'"]
    end
    
    AnalyzedType --> EncodeFunc
    EncodeFunc --> EnrichedDict
    
    style EnrichedDict text-align:left
```

Example encoding for `str | None`:
```python
{
    "type": {"kind": "Str"},
    "nullable": True
}
```

**Sources:** [python/cocoindex/typing.py:510-522]()

## Encoder/Decoder Usage by Operation Type

Different operation types use encoders and decoders in specific ways.

### Function Operations

```mermaid
graph TB
    subgraph "Function Registration"
        FuncDef["@executor_class decorated class<br/>or @function decorated func"]
        ExtractSig["Extract signature parameters"]
        ForEachArg["For each argument"]
        CreateArgDecoder["Create decoder from OpArgSchema"]
        StoreArgInfo["Store in _args_info / _kwargs_info"]
        CreateReturnEncoder["Create encoder for return type"]
        StoreEncoder["Store in _result_encoder"]
    end
    
    subgraph "Function Execution"
        ReceiveArgs["__call__(*engine_args, **engine_kwargs)"]
        DecodeArgs["Decode each argument"]
        CheckNone["Check None propagation"]
        CallFunc["Call Python function"]
        EncodeReturn["Encode return value"]
        ReturnToEngine["Return engine value"]
    end
    
    FuncDef --> ExtractSig
    ExtractSig --> ForEachArg
    ForEachArg --> CreateArgDecoder
    CreateArgDecoder --> StoreArgInfo
    ExtractSig --> CreateReturnEncoder
    CreateReturnEncoder --> StoreEncoder
    
    StoreArgInfo -.->|"runtime"| ReceiveArgs
    StoreEncoder -.->|"runtime"| EncodeReturn
    
    ReceiveArgs --> DecodeArgs
    DecodeArgs --> CheckNone
    CheckNone -->|"no None or optional"| CallFunc
    CheckNone -->|"None on required arg"| ReturnNone["Return None"]
    CallFunc --> EncodeReturn
    EncodeReturn --> ReturnToEngine
```

**Sources:** [python/cocoindex/op.py:225-263](), [python/cocoindex/op.py:331-334](), [python/cocoindex/op.py:370-390]()

### Source Connectors

```mermaid
graph TB
    subgraph "Source Registration"
        SourceReg["@source_connector(spec_cls, key_type, value_type)"]
        AnalyzeKeyType["analyze_type_info(key_type)"]
        AnalyzeValueType["analyze_type_info(value_type)"]
        CreateKeyEncoder["make_engine_key_encoder(key_type_info)"]
        CreateKeyDecoder["make_engine_key_decoder(key_fields_schema, key_type_info)"]
        CreateValueEncoder["make_engine_value_encoder(value_type_info)"]
    end
    
    subgraph "Source Execution"
        ListMethod["list(options) → Iterator"]
        YieldRow["Yield PartialSourceRow"]
        EncodeKey["key_encoder(row.key)"]
        EncodeValue["value_encoder(row.value)"]
        
        GetValue["get_value(raw_key, options)"]
        DecodeKey["key_decoder(raw_key)"]
        FetchValue["Fetch from storage"]
        EncodeResult["Encode result"]
    end
    
    SourceReg --> AnalyzeKeyType
    SourceReg --> AnalyzeValueType
    AnalyzeKeyType --> CreateKeyEncoder
    AnalyzeKeyType --> CreateKeyDecoder
    AnalyzeValueType --> CreateValueEncoder
    
    CreateKeyEncoder -.-> EncodeKey
    CreateKeyDecoder -.-> DecodeKey
    CreateValueEncoder -.-> EncodeValue
    CreateValueEncoder -.-> EncodeResult
    
    ListMethod --> YieldRow
    YieldRow --> EncodeKey
    YieldRow --> EncodeValue
    
    GetValue --> DecodeKey
    DecodeKey --> FetchValue
    FetchValue --> EncodeResult
```

**Key encoding example:**
- Simple key `str` → single field `{"_key": "value"}`
- Composite key `(str, int)` → multiple fields `{"id": "value", "version": 123}`

**Sources:** [python/cocoindex/op.py:562-635]()

### Target Connectors

```mermaid
graph TB
    subgraph "Target Registration"
        TargetReg["@target_connector(spec_cls, persistent_key_type, setup_state_cls)"]
        AnalyzeMutateType["Analyze mutate(*args) signature<br/>tuple[SpecType, dict[K, V]]"]
        ExtractKeyType["Extract K from dict[K, V]"]
        ExtractValueType["Extract V from dict[K, V]"]
    end
    
    subgraph "Export Context Creation"
        CreateContext["create_export_context(name, spec, schemas, ...)"]
        DecodeKeyFields["decode_engine_field_schemas(raw_key_fields)"]
        DecodeValueFields["decode_engine_field_schemas(raw_value_fields)"]
        CreateKeyDecoder["make_engine_key_decoder(field_path, key_fields, key_type_info)"]
        CreateStructDecoder["make_engine_struct_decoder(field_path, value_fields, value_type_info)"]
        StoreInContext["Store decoders in _TargetConnectorContext"]
    end
    
    subgraph "Mutation Execution"
        ReceiveMutation["mutate(*args) with engine mutations"]
        DecodeKey["key_decoder(raw_key)"]
        DecodeValue["value_decoder(raw_value)"]
        ApplyToTarget["Apply to target database"]
    end
    
    TargetReg --> AnalyzeMutateType
    AnalyzeMutateType --> ExtractKeyType
    AnalyzeMutateType --> ExtractValueType
    
    ExtractKeyType -.-> CreateKeyDecoder
    ExtractValueType -.-> CreateStructDecoder
    
    CreateContext --> DecodeKeyFields
    CreateContext --> DecodeValueFields
    DecodeKeyFields --> CreateKeyDecoder
    DecodeValueFields --> CreateStructDecoder
    CreateKeyDecoder --> StoreInContext
    CreateStructDecoder --> StoreInContext
    
    StoreInContext -.-> ReceiveMutation
    ReceiveMutation --> DecodeKey
    ReceiveMutation --> DecodeValue
    DecodeKey --> ApplyToTarget
    DecodeValue --> ApplyToTarget
```

**Sources:** [python/cocoindex/op.py:867-905](), [python/cocoindex/op.py:1006-1019]()

## None Value Propagation

The decoder system implements special handling for `None` values to support optional types and avoid unnecessary computation.

### None Propagation Logic

```mermaid
graph TD
    Start["Function argument received"]
    CheckValue{"Value == None?"}
    CheckRequired{"Argument is_required<br/>(not Optional)?"}
    SkipExecution["Skip function execution<br/>Return None immediately"]
    DecodeValue["Decode value normally"]
    ExecuteFunction["Execute function"]
    
    Start --> CheckValue
    CheckValue -->|"Yes"| CheckRequired
    CheckValue -->|"No"| DecodeValue
    CheckRequired -->|"Yes (required)"| SkipExecution
    CheckRequired -->|"No (optional)"| DecodeValue
    DecodeValue --> ExecuteFunction
```

Implementation in `_WrappedExecutor.__call__`:

```python
# python/cocoindex/op.py:370-390
async def __call__(self, *args: Any, **kwargs: Any) -> Any:
    decoded_args = []
    for arg_info, arg in zip(self._args_info, args):
        if arg_info.is_required and arg is None:
            return None  # Skip execution, propagate None
        decoded_args.append(arg_info.decoder(arg))
    
    # Similar for kwargs...
    
    output = await self._acall(*decoded_args, **decoded_kwargs)
    return self._result_encoder(output)
```

The `is_required` flag is determined from the type annotation:
- `str` → required (is_required=True)
- `str | None` or `Optional[str]` → optional (is_required=False)

**Sources:** [python/cocoindex/op.py:257-258](), [python/cocoindex/op.py:372-386]()

### Nullable Return Types

When any input to a function is optional and has a `None` value, the return type is automatically marked as nullable:

```python
# python/cocoindex/op.py:354-356
if potentially_missing_required_arg:
    analyzed_result_type.nullable = True
encoded_type = encode_enriched_type_info(analyzed_result_type)
```

This ensures that downstream operations correctly handle the possibility of `None` results.

**Sources:** [python/cocoindex/op.py:258-259](), [python/cocoindex/op.py:354-356]()

## Batching Support

Operations with `OpArgs(batching=True)` receive multiple input values at once for efficiency. The encoder/decoder system adapts to handle batched data.

### Batched Decoder Creation

```mermaid
graph TB
    subgraph "Batching Configuration"
        OpArgs["OpArgs(batching=True)"]
        SingleArg["Must have exactly 1 argument"]
        ArgTypeCheck["Argument annotation must be list[T]"]
    end
    
    subgraph "Decoder Creation"
        ExtractElemType["Extract T from list[T]"]
        CreateElemDecoder["make_engine_value_decoder(field_path, src_type, elem_type_info)"]
        WrapInList["lambda value: [elem_decoder(v) for v in value]"]
        BatchedDecoder["Batched decoder ready"]
    end
    
    subgraph "Runtime"
        EngineValueList["Engine sends list of values"]
        ApplyBatchDecoder["Apply batched decoder"]
        PythonList["list[T] ready for function"]
    end
    
    OpArgs --> SingleArg
    SingleArg --> ArgTypeCheck
    ArgTypeCheck --> ExtractElemType
    ExtractElemType --> CreateElemDecoder
    CreateElemDecoder --> WrapInList
    WrapInList --> BatchedDecoder
    
    BatchedDecoder -.-> ApplyBatchDecoder
    EngineValueList --> ApplyBatchDecoder
    ApplyBatchDecoder --> PythonList
```

**Sources:** [python/cocoindex/op.py:179-186](), [python/cocoindex/op.py:202-204](), [python/cocoindex/op.py:249-252]()

### Batched Return Type Analysis

For batching functions, the return type annotation must be `list[R]` where `R` is the actual return type per item:

```python
# python/cocoindex/op.py:340-351
if op_args.batching:
    if not isinstance(analyzed_expected_return_type.variant, AnalyzedListType):
        raise ValueError("Expected return type for batching function to be a list type")
    analyzed_result_type = analyze_type_info(
        analyzed_expected_return_type.variant.elem_type
    )
else:
    analyzed_result_type = analyzed_expected_return_type
```

The encoder is created for the element type `R`, and the engine handles the list batching automatically.

**Sources:** [python/cocoindex/op.py:340-351]()

## Error Handling and Debugging

The encoder/decoder system includes error context to help debug type mismatches.

### Field Path Tracking

All decoder factory functions accept a `field_path: list[str]` parameter that tracks the location in nested structures:

```python
# Example field paths:
["text"]                    # Top-level argument "text"
["person", "address"]       # Nested field in struct
["<key>"]                   # Key decoding
["<value>", "embedding"]    # Value field in target mutation
```

When decoding fails, the field path helps identify exactly where the type mismatch occurred.

**Sources:** [python/cocoindex/op.py:179-180](), [python/cocoindex/op.py:254-255](), [python/cocoindex/op.py:885-886]()

### Type Validation Errors

Type encoding validates that annotations are specific and supported:

```python
# python/cocoindex/typing.py:414-421
try:
    type_info = encode_enriched_type_info(analyzed_type)
except ValueError as e:
    e.add_note(
        f"Failed to encode annotation for field - "
        f"{struct_info.struct_type.__name__}.{name}: {analyzed_type.core_type}"
    )
    raise
```

Common errors include:
- Using `Any` in return type annotations (not specific enough)
- Using unsupported types like `set` or custom classes without `@dataclass`
- Missing type annotations entirely

**Sources:** [python/cocoindex/typing.py:414-421](), [python/cocoindex/typing.py:456-459]()

## Summary

The encoder/decoder system provides the critical bridge between Python's flexible type system and Rust's performance-oriented execution:

| Component | Purpose | Key Functions |
|-----------|---------|---------------|
| **Encoders** | Python → Rust conversion | `make_engine_value_encoder`, `make_engine_key_encoder` |
| **Decoders** | Rust → Python conversion | `make_engine_value_decoder`, `make_engine_key_decoder`, `make_engine_struct_decoder` |
| **Type Analysis** | Parse Python annotations | `analyze_type_info`, `encode_enriched_type_info` |
| **Special Handling** | Edge cases | Batching, None propagation, key encoding |

The system operates in two phases:
1. **Registration time**: Analyze type annotations and create encoder/decoder functions
2. **Runtime**: Apply encoders/decoders to actual data values

This design ensures zero runtime reflection overhead while maintaining type safety through static analysis.

**Sources:** [python/cocoindex/op.py:26-32](), [python/cocoindex/typing.py:301-403](), [python/cocoindex/typing.py:510-522]()

---

# Page: OpArgs and Execution Options

# OpArgs and Execution Options

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [docs/docs/core/data_types.mdx](docs/docs/core/data_types.mdx)
- [python/cocoindex/op.py](python/cocoindex/op.py)
- [python/cocoindex/tests/test_typing.py](python/cocoindex/tests/test_typing.py)
- [python/cocoindex/typing.py](python/cocoindex/typing.py)

</details>



## Overview

`OpArgs` is a configuration class that controls how operations (functions, sources, and targets) are executed by the CocoIndex engine. It provides fine-grained control over performance optimizations, resource allocation, and data lineage tracking.

**Purpose**: Configure execution behavior including GPU usage, caching, batching, timeouts, and argument relationship tracking for custom operations.

**Scope**: This page covers the `OpArgs` configuration system and how it affects operation execution. For information about operation registration and type marshalling, see [Operation Registration and Decorators](#4.1). For details on the broader operation system, see [Operation System and Type Marshalling](#4).

**Key Concepts**:
- Control execution environment (GPU vs CPU)
- Enable/disable result caching with version control
- Configure batch processing for performance
- Set execution timeouts
- Track data lineage through argument relationships

Sources: [python/cocoindex/op.py:149-171]()

---

## OpArgs Configuration Structure

The `OpArgs` dataclass defines execution options that can be passed to operation decorators:

```mermaid
graph TB
    OpArgs["OpArgs<br/>python/cocoindex/op.py:149-171"]
    
    OpArgs --> gpu["gpu: bool<br/>Default: False"]
    OpArgs --> cache["cache: bool<br/>Default: False"]
    OpArgs --> batching["batching: bool<br/>Default: False"]
    OpArgs --> max_batch["max_batch_size: int | None<br/>Default: None"]
    OpArgs --> behavior_v["behavior_version: int | None<br/>Default: None"]
    OpArgs --> timeout["timeout: datetime.timedelta | None<br/>Default: None"]
    OpArgs --> arg_rel["arg_relationship: tuple[ArgRelationship, str] | None<br/>Default: None"]
    
    gpu --> |"subprocess"| executor_stub["executor_stub()<br/>subprocess_exec.py"]
    cache --> enable_cache["enable_cache()<br/>op.py:392-393"]
    behavior_v --> cache_inv["Cache invalidation"]
    batching --> batch_opts["batching_options()<br/>op.py:401-407"]
    timeout --> timeout_method["timeout()<br/>op.py:398-399"]
    arg_rel --> attrs["Output attributes<br/>op.py:243-247"]
    
    style OpArgs fill:#f9f9f9
```

**OpArgs Dataclass Structure**

| Field | Type | Default | Purpose |
|-------|------|---------|---------|
| `gpu` | `bool` | `False` | Execute operation on GPU in separate process |
| `cache` | `bool` | `False` | Enable result caching for incremental processing |
| `batching` | `bool` | `False` | Enable batch processing of multiple inputs |
| `max_batch_size` | `int \| None` | `None` | Maximum number of items per batch (only with `batching=True`) |
| `behavior_version` | `int \| None` | `None` | Version number for cache invalidation (required if `cache=True`) |
| `timeout` | `datetime.timedelta \| None` | `None` | Maximum execution time (default: 300s) |
| `arg_relationship` | `tuple[ArgRelationship, str] \| None` | `None` | Tracks lineage between input and output data |

Sources: [python/cocoindex/op.py:149-171]()

---

## Using OpArgs with Decorators

### With @function Decorator

The `@function` decorator accepts `OpArgs` fields as keyword arguments:

```python
import cocoindex
from cocoindex import op
import datetime

# Simple function with caching
@op.function(cache=True, behavior_version=1)
def embed_text(text: str) -> list[float]:
    """Generate text embedding with caching enabled."""
    # ... embedding logic ...
    return embedding

# GPU-accelerated function with batching
@op.function(
    gpu=True, 
    batching=True, 
    max_batch_size=32,
    timeout=datetime.timedelta(minutes=5)
)
def batch_embed(texts: list[str]) -> list[list[float]]:
    """Generate embeddings in batches on GPU."""
    # ... batch embedding logic ...
    return embeddings

# Function with argument relationship tracking
@op.function(
    cache=True,
    behavior_version=1,
    arg_relationship=(op.ArgRelationship.CHUNKS_BASE_TEXT, "content")
)
def split_text(content: str, chunk_size: int) -> list[str]:
    """Split text into chunks, tracking original text."""
    # ... chunking logic ...
    return chunks
```

### With @executor_class Decorator

For class-based executors, pass `OpArgs` fields to the `@executor_class` decorator:

```python
from cocoindex.op import executor_class, FunctionSpec

class MyFunctionSpec(FunctionSpec):
    param1: str
    param2: int

@executor_class(
    cache=True,
    behavior_version=2,
    gpu=True
)
class MyExecutor:
    spec: MyFunctionSpec
    
    def analyze(self) -> type:
        """Return the output type."""
        return str
    
    def prepare(self) -> None:
        """Initialize resources (optional)."""
        # Load models, etc.
        pass
    
    def __call__(self, input_data: str) -> str:
        """Execute the operation."""
        # ... processing logic ...
        return result
```

Sources: [python/cocoindex/op.py:417-481]()

---

## Execution Control Patterns

```mermaid
graph LR
    subgraph "Decorator Application"
        Decorator["@op.function(**kwargs)"]
        OpArgsObj["OpArgs(**kwargs)<br/>op.py:150"]
    end
    
    subgraph "Registration"
        Register["_register_op_factory()<br/>op.py:189-414"]
        Wrapped["_WrappedExecutor<br/>op.py:206-408"]
    end
    
    subgraph "Execution Modes"
        GPUCheck{"gpu=True?"}
        ExecutorStub["executor_stub()<br/>subprocess_exec.py<br/>Separate process"]
        DirectExec["Direct execution<br/>Same process"]
    end
    
    subgraph "Runtime Behavior"
        CacheCheck{"cache=True?"}
        CacheLookup["Check cache with<br/>behavior_version"]
        BatchCheck{"batching=True?"}
        BatchProcess["Batch inputs<br/>max_batch_size"]
        TimeoutCheck["Apply timeout<br/>default: 300s"]
        Execute["Execute operation"]
    end
    
    Decorator --> OpArgsObj
    OpArgsObj --> Register
    Register --> Wrapped
    Wrapped --> GPUCheck
    
    GPUCheck -->|Yes| ExecutorStub
    GPUCheck -->|No| DirectExec
    
    ExecutorStub --> CacheCheck
    DirectExec --> CacheCheck
    
    CacheCheck -->|Yes| CacheLookup
    CacheCheck -->|No| BatchCheck
    CacheLookup --> BatchCheck
    
    BatchCheck -->|Yes| BatchProcess
    BatchCheck -->|No| TimeoutCheck
    BatchProcess --> TimeoutCheck
    
    TimeoutCheck --> Execute
```

Sources: [python/cocoindex/op.py:189-414](), [python/cocoindex/op.py:417-444](), [python/cocoindex/op.py:458-481]()

### GPU Execution

**Purpose**: Offload compute-intensive operations to GPU hardware in a separate process.

**Implementation**: When `gpu=True`, the operation is wrapped with `executor_stub()` which spawns a separate process for execution:

```mermaid
graph TB
    MainProc["Main Process"]
    GPUProc["GPU Process<br/>subprocess"]
    
    MainProc -->|"Input data"| ExecutorStub["executor_stub()<br/>subprocess_exec.py:24"]
    ExecutorStub -->|"Serialize & send"| GPUProc
    GPUProc -->|"Execute on GPU"| Executor["Operation Executor"]
    Executor -->|"Results"| GPUProc
    GPUProc -->|"Serialize & return"| ExecutorStub
    ExecutorStub -->|"Output data"| MainProc
```

**When to use**:
- Operations that benefit from GPU acceleration (embeddings, neural networks)
- Prevents GPU memory issues by isolating GPU usage
- Allows multiple operations to share GPU resources sequentially

**Code location**: [python/cocoindex/op.py:217-222]()

Sources: [python/cocoindex/op.py:217-222](), [python/cocoindex/subprocess_exec.py:24]()

### Caching

**Purpose**: Enable incremental processing by caching operation results based on input checksums.

**Requirements**:
- `cache=True`: Enable caching
- `behavior_version`: Required integer version number

**Behavior version**: Cache is invalidated when `behavior_version` changes. Increment this number when:
- Operation logic changes
- Model versions change
- Dependencies are updated
- Output format changes

**Implementation**: Cache is checked during execution:

```python
# Example: Caching with version control
@op.function(cache=True, behavior_version=3)
def process_document(text: str) -> dict:
    """Process document with caching.
    
    behavior_version=3: Updated to use improved algorithm
    """
    # Expensive processing
    return result
```

**Cache lookup flow**:

```mermaid
graph TB
    Input["Input data"]
    Hash["Compute input checksum"]
    CacheLookup{"Cache hit?<br/>+ behavior_version match?"}
    CacheHit["Return cached result"]
    Execute["Execute operation"]
    CacheStore["Store result in cache"]
    Output["Return result"]
    
    Input --> Hash
    Hash --> CacheLookup
    CacheLookup -->|Yes| CacheHit
    CacheLookup -->|No| Execute
    CacheHit --> Output
    Execute --> CacheStore
    CacheStore --> Output
```

**Code location**: [python/cocoindex/op.py:392-396]()

Sources: [python/cocoindex/op.py:392-396](), [python/cocoindex/op.py:154-158]()

### Batching

**Purpose**: Process multiple inputs together for improved throughput.

**Requirements**:
- `batching=True`: Enable batch processing
- Function must accept `list[T]` and return `list[R]`
- Single-argument functions only

**Configuration**:
- `max_batch_size`: Optional limit on batch size (default: engine decides)

**Type transformation**: When `batching=True`, the engine automatically:
1. Collects multiple inputs into a list
2. Passes the list to your function
3. Distributes results back to individual items

```python
# Batching example
@op.function(batching=True, max_batch_size=32)
def embed_batch(texts: list[str]) -> list[cocoindex.Vector[float, Literal[768]]]:
    """Process up to 32 texts at once."""
    # texts will be a list of strings (batch)
    # Must return a list of embeddings (same length)
    return embeddings
```

**Batching validation and encoding**:

```mermaid
graph TB
    Validate["Validate batching<br/>op.py:202-204"]
    Check1{"Single argument?"}
    Error1["ValueError:<br/>Only single-arg supported"]
    
    Analyze["analyze_schema()<br/>op.py:225-358"]
    Check2{"Arg type is list?"}
    Check3{"Return type is list?"}
    Error2["ValueError:<br/>Expected list types"]
    
    Decoder["_make_batched_engine_value_decoder<br/>op.py:179-186"]
    BatchDecode["Decode each item<br/>in batch"]
    
    Encoder["_result_encoder<br/>op.py:332-334"]
    BatchEncode["Encode batch results"]
    
    Validate --> Check1
    Check1 -->|No| Error1
    Check1 -->|Yes| Analyze
    
    Analyze --> Check2
    Check2 -->|No| Error2
    Check2 -->|Yes| Check3
    Check3 -->|No| Error2
    Check3 -->|Yes| Decoder
    
    Decoder --> BatchDecode
    BatchEncode --> Encoder
```

**Code location**: [python/cocoindex/op.py:202-204](), [python/cocoindex/op.py:249-252](), [python/cocoindex/op.py:340-349](), [python/cocoindex/op.py:401-407]()

Sources: [python/cocoindex/op.py:179-186](), [python/cocoindex/op.py:202-204](), [python/cocoindex/op.py:249-252](), [python/cocoindex/op.py:340-349](), [python/cocoindex/op.py:401-407]()

### Timeout

**Purpose**: Prevent operations from hanging indefinitely.

**Configuration**:
- `timeout`: `datetime.timedelta` object
- Default: 300 seconds (5 minutes) if not specified

**Example**:

```python
import datetime

# Short timeout for fast operations
@op.function(timeout=datetime.timedelta(seconds=30))
def quick_process(data: str) -> str:
    return data.upper()

# Longer timeout for expensive operations
@op.function(timeout=datetime.timedelta(minutes=10))
def expensive_llm_call(prompt: str) -> str:
    # LLM API call
    return response
```

**Code location**: [python/cocoindex/op.py:398-399]()

Sources: [python/cocoindex/op.py:158-159](), [python/cocoindex/op.py:398-399]()

---

## Argument Relationships and Data Lineage

### ArgRelationship Enum

`ArgRelationship` specifies semantic relationships between input arguments and output values, enabling data lineage tracking and optimization:

```mermaid
graph LR
    ArgRelEnum["ArgRelationship<br/>op.py:141-147"]
    
    ArgRelEnum --> EMB["EMBEDDING_ORIGIN_TEXT<br/>'cocoindex.io/embedding_origin_text'"]
    ArgRelEnum --> CHUNK["CHUNKS_BASE_TEXT<br/>'cocoindex.io/chunk_base_text'"]
    ArgRelEnum --> RECT["RECTS_BASE_IMAGE<br/>'cocoindex.io/rects_base_image'"]
    
    EMB --> EmbUse["Embedding derived from<br/>specified text argument"]
    CHUNK --> ChunkUse["Chunks derived from<br/>specified text argument"]
    RECT --> RectUse["Rectangles/regions from<br/>specified image argument"]
```

**Available Relationships**:

| Relationship | Value | Meaning |
|--------------|-------|---------|
| `EMBEDDING_ORIGIN_TEXT` | `"cocoindex.io/embedding_origin_text"` | Output embedding was generated from specified text input |
| `CHUNKS_BASE_TEXT` | `"cocoindex.io/chunk_base_text"` | Output chunks were split from specified text input |
| `RECTS_BASE_IMAGE` | `"cocoindex.io/rects_base_image"` | Output rectangles/regions extracted from specified image input |

Sources: [python/cocoindex/op.py:141-147]()

### Usage Pattern

Argument relationships are specified as a tuple: `(ArgRelationship, argument_name)`

```python
from cocoindex.op import ArgRelationship

@op.function(
    arg_relationship=(ArgRelationship.CHUNKS_BASE_TEXT, "text")
)
def split_text(text: str, chunk_size: int) -> list[str]:
    """Split text into chunks.
    
    The 'text' argument is the base text for the output chunks.
    """
    chunks = []
    for i in range(0, len(text), chunk_size):
        chunks.append(text[i:i+chunk_size])
    return chunks

@op.function(
    arg_relationship=(ArgRelationship.EMBEDDING_ORIGIN_TEXT, "text")
)
def embed_text(text: str) -> cocoindex.Vector[float, Literal[384]]:
    """Generate embedding from text.
    
    The 'text' argument is the origin of the embedding.
    """
    # Generate embedding
    return embedding
```

### Implementation Details

During schema analysis, argument relationships are stored as attributes on the output type:

```mermaid
graph TB
    OpArgs["OpArgs with<br/>arg_relationship"]
    
    Analyze["analyze_schema()<br/>op.py:225-358"]
    
    ProcessArg["process_arg()<br/>op.py:237-263"]
    
    CheckRel{"arg_relationship<br/>defined?"}
    
    ExtractArg["Extract related_attr<br/>and related_arg_name<br/>op.py:243-246"]
    
    AddAttr["Add to attributes<br/>attributes[related_attr.value] =<br/>actual_arg.analyzed_value<br/>op.py:246"]
    
    AnalyzeResult["analyzed_result_type<br/>op.py:331-356"]
    
    SetAttrs["Set analyzed_result_type.attrs<br/>op.py:352-353"]
    
    Encode["encode_enriched_type_info()<br/>op.py:356"]
    
    OpArgs --> Analyze
    Analyze --> ProcessArg
    ProcessArg --> CheckRel
    CheckRel -->|Yes| ExtractArg
    CheckRel -->|No| AnalyzeResult
    ExtractArg --> AddAttr
    AddAttr --> AnalyzeResult
    AnalyzeResult --> SetAttrs
    SetAttrs --> Encode
```

**Attribute storage flow**:

1. During `analyze_schema()`, the relationship is checked for each argument
2. If the argument name matches `arg_relationship[1]`, the attribute key is set
3. The attribute value is the analyzed type information of the input argument
4. Attributes are attached to the output type's `attrs` dictionary
5. Encoded in the type information sent to the Rust engine

**Code location**: [python/cocoindex/op.py:243-247](), [python/cocoindex/op.py:352-353]()

Sources: [python/cocoindex/op.py:141-147](), [python/cocoindex/op.py:243-247](), [python/cocoindex/op.py:352-353]()

### Benefits

1. **Data Lineage**: Track which input data produced which output
2. **Query Optimization**: Retrieve original text when querying embeddings
3. **Debugging**: Understand data transformations through the pipeline
4. **CocoInsight Integration**: Visualize data relationships in the UI

---

## Complete Configuration Example

Here's a comprehensive example combining multiple `OpArgs` features:

```python
import cocoindex
from cocoindex import op
import datetime
from typing import Literal

# GPU-accelerated batch embedding with caching and lineage
@op.function(
    gpu=True,                                              # Run on GPU
    cache=True,                                            # Enable caching
    behavior_version=2,                                    # Cache version
    batching=True,                                         # Process in batches
    max_batch_size=64,                                     # Max 64 items per batch
    timeout=datetime.timedelta(minutes=5),                 # 5 minute timeout
    arg_relationship=(op.ArgRelationship.EMBEDDING_ORIGIN_TEXT, "texts")
)
def batch_embed_with_lineage(
    texts: list[str]
) -> list[cocoindex.Vector[cocoindex.Float32, Literal[768]]]:
    """
    Generate 768-dimensional embeddings in batches.
    
    Features:
    - Processes up to 64 texts per batch
    - Runs on GPU in separate process
    - Results are cached (v2)
    - Tracks original text for each embedding
    - 5 minute timeout per batch
    """
    # Load model once (happens in prepare() if using executor_class)
    model = load_embedding_model()
    
    # Generate embeddings for batch
    embeddings = model.encode(texts)
    
    return embeddings

# Class-based executor with full control
class CustomEmbedSpec(op.FunctionSpec):
    model_name: str
    normalize: bool

@op.executor_class(
    gpu=True,
    cache=True,
    behavior_version=1,
    timeout=datetime.timedelta(minutes=2)
)
class CustomEmbedExecutor:
    spec: CustomEmbedSpec
    model: Any = None
    
    def analyze(self) -> type:
        """Specify output type."""
        return cocoindex.Vector[cocoindex.Float32, Literal[768]]
    
    def prepare(self) -> None:
        """Load model once during initialization."""
        self.model = load_model(self.spec.model_name)
    
    def __call__(self, text: str) -> list[float]:
        """Process single text."""
        embedding = self.model.encode(text)
        if self.spec.normalize:
            embedding = normalize(embedding)
        return embedding
```

Sources: [python/cocoindex/op.py:149-171](), [python/cocoindex/op.py:458-481](), [python/cocoindex/op.py:417-444]()

---

## Best Practices

### Caching
- **Always increment** `behavior_version` when changing operation logic
- Use caching for expensive operations (LLM calls, embeddings)
- Avoid caching for fast operations (simple string manipulation)

### GPU Execution
- Enable for compute-intensive operations (neural networks, embeddings)
- Combine with batching for better GPU utilization
- Monitor memory usage in GPU process

### Batching
- Set `max_batch_size` based on GPU memory constraints
- Larger batches = better throughput but more memory
- Typical ranges: 16-128 for embeddings

### Timeouts
- Set generous timeouts for external API calls (LLM providers)
- Use shorter timeouts for internal operations
- Default 300s is usually sufficient

### Argument Relationships
- Use for tracking data lineage through transformations
- Essential for debugging and query optimization
- Enables rich metadata in CocoInsight visualization

Sources: [python/cocoindex/op.py:149-171]()

---

# Page: Built-in Data Sources

# Built-in Data Sources

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [.cargo/config.toml](.cargo/config.toml)
- [docs/docs/core/basics.md](docs/docs/core/basics.md)
- [docs/docs/core/flow_def.mdx](docs/docs/core/flow_def.mdx)
- [docs/docs/core/flow_methods.mdx](docs/docs/core/flow_methods.mdx)

</details>



This page documents the built-in source connectors provided by CocoIndex for importing data into indexing flows. Sources are the entry points where data enters a flow, providing structured data that can be transformed and exported to targets.

For information about defining custom sources, see [Custom Source Implementation](#8.3). For information about processing functions that transform source data, see [Built-in Processing Functions](#6). For the complete flow definition system, see [Flow Definition System](#3).

---

## Overview

A **source connector** is a CocoIndex operation that reads data from external systems and makes it available to the indexing flow as a table (KTable) with key and value fields. Each source connector implements the source interface defined in the operation system.

```mermaid
graph TB
    subgraph "External Systems"
        FS["Local Filesystem"]
        S3["Amazon S3"]
        Azure["Azure Blob Storage"]
        GDrive["Google Drive"]
        PG["PostgreSQL Database"]
    end
    
    subgraph "CocoIndex Source Connectors"
        LF["LocalFile"]
        AS3["AmazonS3"]
        AB["AzureBlob"]
        GD["GoogleDrive"]
        PS["Postgres"]
    end
    
    subgraph "Flow Data"
        Table["KTable<br/>Key: identifier<br/>Value: content fields"]
    end
    
    FS --> LF
    S3 --> AS3
    Azure --> AB
    GDrive --> GD
    PG --> PS
    
    LF --> Table
    AS3 --> Table
    AB --> Table
    GD --> Table
    PS --> Table
    
    Table --> Transform["Transform Functions"]
```

**Diagram: Source Connector Architecture**

Sources: [python/cocoindex/op.py:86-88](), [README.md:86-92](), Diagram 1 and Diagram 3 from architecture

---

## Source Interface and Operations

### Adding a Source to a Flow

Sources are added to flows using the `FlowBuilder.add_source()` method, which takes a source specification and optional parameters:

```python
data_scope["documents"] = flow_builder.add_source(
    source_spec,
    refresh_interval=datetime.timedelta(minutes=5),  # Optional
    max_inflight_rows=100,                           # Optional
    max_inflight_bytes=100*1024*1024                 # Optional
)
```

The source specification is an instance of a `SourceSpec` subclass that defines the source type and its configuration. Each built-in source provides its own spec class.

Sources: [docs/docs/core/flow_def.mdx:116-159](), [python/cocoindex/op.py:86-88]()

### Refresh Interval and Change Capture

The `refresh_interval` parameter enables **live update mode** for sources. When specified, CocoIndex periodically polls the source for changes:

| Parameter | Type | Purpose |
|-----------|------|---------|
| `refresh_interval` | `datetime.timedelta` | Interval between source refresh checks in live update mode |

During each refresh cycle, CocoIndex:
1. Lists rows in the data source
2. Compares metadata (checksums, modification times) against internal state
3. Only processes rows that have changed
4. Skips unchanged rows to minimize computation

If no changes are detected, only the list operation is performed (typically low cost).

Some sources also support **native change capture mechanisms** beyond polling:
- **Postgres source**: Listens to PostgreSQL NOTIFY events for real-time updates
- **AmazonS3 source**: Can watch S3 event notifications for bucket changes
- **GoogleDrive source**: Polls recent modified files via Drive API

Sources: [docs/docs/core/flow_def.mdx:141-167](), [docs/docs/core/flow_methods.mdx:205-214]()

### Source Concurrency Control

Sources support concurrency limits to prevent system overload:

```python
data_scope["documents"] = flow_builder.add_source(
    source_spec,
    max_inflight_rows=10,           # Max rows processed concurrently
    max_inflight_bytes=100*1024*1024  # Max memory footprint (100MB)
)
```

Global limits can also be configured via `GlobalExecutionOptions` or environment variables (`COCOINDEX_SOURCE_MAX_INFLIGHT_ROWS`, `COCOINDEX_SOURCE_MAX_INFLIGHT_BYTES`). When both global and per-source limits are set, both are enforced independently.

Sources: [docs/docs/core/flow_def.mdx:365-414]()

### Source Data Model

Each source produces a **KTable** with:
- **Key fields**: One or more fields that uniquely identify each row
- **Value fields**: Content fields specific to the source type

The source connector implementation defines the key and value types through the `@source_connector` decorator:

```python
@source_connector(
    spec_cls=MySourceSpec,
    key_type=str,              # Single key field
    value_type=MyValueType     # Struct type for value fields
)
class MySourceConnector:
    ...
```

For composite keys, use a frozen dataclass or NamedTuple as the key type.

Sources: [python/cocoindex/op.py:707-727](), [python/cocoindex/op.py:638-705]()

---

## Source Connector Implementation Details

```mermaid
graph TB
    subgraph "Source Connector Interface"
        SC["@source_connector"]
        Create["create(spec) → executor"]
        List["list(options) → Iterator[PartialSourceRow]"]
        GetValue["get_value(key, options) → PartialSourceRowData"]
        ProvidesOrdinal["provides_ordinal() → bool"]
    end
    
    subgraph "Data Structures"
        SourceReadOptions["SourceReadOptions<br/>- include_ordinal<br/>- include_content_version_fp<br/>- include_value"]
        PartialSourceRow["PartialSourceRow<br/>- key<br/>- data: PartialSourceRowData"]
        PartialSourceRowData["PartialSourceRowData<br/>- value<br/>- ordinal<br/>- content_version_fp"]
    end
    
    subgraph "Engine Integration"
        KeyEncoder["Key Encoder<br/>Python → Rust"]
        ValueEncoder["Value Encoder<br/>Python → Rust"]
        TableType["EnrichedValueType<br/>TableType(KTable)"]
    end
    
    SC --> Create
    Create --> List
    Create --> GetValue
    Create --> ProvidesOrdinal
    
    List --> SourceReadOptions
    GetValue --> SourceReadOptions
    List --> PartialSourceRow
    GetValue --> PartialSourceRowData
    PartialSourceRow --> PartialSourceRowData
    
    PartialSourceRow --> KeyEncoder
    PartialSourceRowData --> ValueEncoder
    KeyEncoder --> TableType
    ValueEncoder --> TableType
```

**Diagram: Source Connector Implementation Architecture**

### SourceReadOptions

The `SourceReadOptions` class controls what data the engine requests from a source:

| Field | Type | Purpose |
|-------|------|---------|
| `include_ordinal` | `bool` | Request row ordinal if available (for ordering) |
| `include_content_version_fp` | `bool` | Request content fingerprint (for change detection) |
| `include_value` | `bool` | Request row value (for list operation optimization) |

When `include_value=True` in `list()`, sources can optionally return values inline to avoid subsequent `get_value()` calls.

Sources: [python/cocoindex/op.py:489-514]()

### PartialSourceRowData

The `PartialSourceRowData` struct represents data returned by sources:

| Field | Type | Purpose |
|-------|------|---------|
| `value` | `V \| None \| NON_EXISTENCE` | Row value, None if not requested, NON_EXISTENCE if row deleted |
| `ordinal` | `int \| NO_ORDINAL \| None` | Row ordinal for ordering, NO_ORDINAL if unsupported |
| `content_version_fp` | `bytes \| None` | Content fingerprint for change detection |

The `ordinal` field enables ordered processing and helps avoid race conditions in concurrent updates. The `content_version_fp` enables early change detection without computing the full value hash.

Sources: [python/cocoindex/op.py:524-542]()

### Source Executor Lifecycle

```mermaid
sequenceDiagram
    participant Engine
    participant Connector as "_SourceConnector"
    participant Executor as "Source Executor"
    participant External as "External System"
    
    Engine->>Connector: create_executor(spec)
    Connector->>Executor: create(spec)
    Executor->>External: connect/authenticate
    Executor-->>Connector: executor instance
    Connector->>Executor: provides_ordinal()
    Executor-->>Connector: bool
    Connector-->>Engine: _SourceExecutorContext
    
    Engine->>Connector: list_async(options)
    Connector->>Executor: list(SourceReadOptions)
    Executor->>External: list rows
    External-->>Executor: row metadata/data
    Executor-->>Connector: Iterator[PartialSourceRow]
    Connector->>Connector: encode keys & values
    Connector-->>Engine: AsyncIterator[encoded data]
    
    Engine->>Connector: get_value_async(key, options)
    Connector->>Connector: decode key
    Connector->>Executor: get_value(key, SourceReadOptions)
    Executor->>External: fetch row data
    External-->>Executor: row data
    Executor-->>Connector: PartialSourceRowData
    Connector->>Connector: encode value
    Connector-->>Engine: encoded data
```

**Diagram: Source Executor Execution Flow**

Sources: [python/cocoindex/op.py:638-705](), [python/cocoindex/op.py:545-636]()

---

## Built-in Source Connectors

### LocalFile

The `LocalFile` source reads files from the local filesystem. It's the simplest source and commonly used for development and small-scale deployments.

**Configuration:**

```python
cocoindex.sources.LocalFile(
    path="directory_path"
)
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `path` | `str` | Yes | Directory path containing files to index |

**Output Schema:**

| Field | Type | Key/Value | Description |
|-------|------|-----------|-------------|
| `filename` | *Str* | Key | Relative file path from the source directory |
| `content` | *Str* | Value | File contents as text |

**Example:**

```python
@cocoindex.flow_def(name="LocalFileExample")
def local_file_flow(flow_builder: cocoindex.FlowBuilder, data_scope: cocoindex.DataScope):
    data_scope["documents"] = flow_builder.add_source(
        cocoindex.sources.LocalFile(path="markdown_files"),
        refresh_interval=datetime.timedelta(minutes=1)
    )
    
    with data_scope["documents"].row() as doc:
        # doc["filename"] is the file path
        # doc["content"] is the file contents
        ...
```

**Change Capture:** Supports refresh interval. During refresh, checks file modification times to detect changes.

Sources: [README.md:56-62](), [docs/docs/getting_started/quickstart.md:56-68]()

### AmazonS3

The `AmazonS3` source reads objects from AWS S3 buckets.

**Configuration:**

```python
cocoindex.sources.AmazonS3(
    bucket="bucket-name",
    prefix="optional/prefix/",
    region="us-east-1"
)
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `bucket` | `str` | Yes | S3 bucket name |
| `prefix` | `str` | No | Key prefix to filter objects (defaults to root) |
| `region` | `str` | No | AWS region (inferred if not specified) |

**Authentication:** Uses standard AWS credentials from environment variables (`AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`) or IAM roles.

**Output Schema:**

| Field | Type | Key/Value | Description |
|-------|------|-----------|-------------|
| `key` | *Str* | Key | S3 object key |
| `content` | *Bytes* | Value | Object contents as bytes |

**Change Capture:** 
- Supports refresh interval with LastModified timestamp checking
- Can be configured to watch S3 event notifications for real-time updates

Sources: [README.md:189](), Example references

### AzureBlob

The `AzureBlob` source reads blobs from Azure Blob Storage containers.

**Configuration:**

```python
cocoindex.sources.AzureBlob(
    container="container-name",
    prefix="optional/prefix/",
    connection_string="...",
    # OR use sas_token
    sas_token=cocoindex.add_transient_auth_entry("sas_token_value")
)
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `container` | `str` | Yes | Azure Blob container name |
| `prefix` | `str` | No | Blob prefix to filter objects |
| `connection_string` | `str` | Conditional | Azure Storage connection string (if not using SAS token) |
| `sas_token` | `TransientAuthEntryReference[str]` | Conditional | SAS token for authentication (if not using connection string) |

**Authentication:** Either `connection_string` or `sas_token` must be provided. Use `sas_token` with `add_transient_auth_entry()` to avoid exposing secrets in specs.

**Output Schema:**

| Field | Type | Key/Value | Description |
|-------|------|-----------|-------------|
| `name` | *Str* | Key | Blob name (path within container) |
| `content` | *Bytes* | Value | Blob contents as bytes |

**Change Capture:** Supports refresh interval with blob metadata checking.

Sources: [README.md:190](), [docs/docs/core/flow_def.mdx:512-523]()

### GoogleDrive

The `GoogleDrive` source reads files from Google Drive folders.

**Configuration:**

```python
cocoindex.sources.GoogleDrive(
    folder_id="google-drive-folder-id",
    credentials=cocoindex.add_transient_auth_entry({
        "type": "service_account",
        "project_id": "...",
        "private_key": "...",
        "client_email": "...",
        # ... other OAuth2 fields
    })
)
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `folder_id` | `str` | Yes | Google Drive folder ID to index |
| `credentials` | `TransientAuthEntryReference[dict]` | Yes | OAuth2 credentials (service account or user credentials) |

**Authentication:** Requires Google Drive API credentials. Use service account credentials for server applications or OAuth2 user credentials for user-specific access. Always use `add_transient_auth_entry()` to protect credentials.

**Output Schema:**

| Field | Type | Key/Value | Description |
|-------|------|-----------|-------------|
| `file_id` | *Str* | Key | Google Drive file ID |
| `name` | *Str* | Value | File name |
| `content` | *Str* or *Bytes* | Value | File contents (text or binary depending on file type) |
| `mime_type` | *Str* | Value | MIME type of the file |

**Change Capture:**
- Supports refresh interval
- Polls Drive API for recently modified files
- Efficient for detecting changes in large folders

Sources: [README.md:191](), [examples/meeting_notes_graph/README.md]()

### Postgres

The `Postgres` source reads rows from PostgreSQL database tables.

**Configuration:**

```python
cocoindex.sources.Postgres(
    database_url="postgresql://user:pass@host:5432/dbname",
    table="source_table_name",
    schema="public"  # optional
)
```

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `database_url` | `str` | Yes | PostgreSQL connection URL |
| `table` | `str` | Yes | Table name to read from |
| `schema` | `str` | No | Schema name (defaults to "public") |

**Output Schema:**

The output schema matches the source table schema. Key fields are determined by the table's primary key. Value fields include all non-key columns.

**Example:**

For a source table:
```sql
CREATE TABLE articles (
    id SERIAL PRIMARY KEY,
    title TEXT NOT NULL,
    content TEXT NOT NULL,
    created_at TIMESTAMP DEFAULT NOW()
);
```

The source produces a KTable with:
- Key: `id` (*Int64*)
- Value fields: `title` (*Str*), `content` (*Str*), `created_at` (*OffsetDateTime*)

**Change Capture:**
- Supports refresh interval with row-level change detection
- **Native support**: Can listen to PostgreSQL NOTIFY events for real-time updates
- Most efficient for streaming database changes to other systems

Sources: Architecture diagrams, [docs/docs/core/flow_methods.mdx:211]()

---

## Type Mapping and Schema

```mermaid
graph LR
    subgraph "Source Connector Definition"
        Decorator["@source_connector<br/>spec_cls, key_type, value_type"]
    end
    
    subgraph "Python Types"
        KeyType["Key Type<br/>str, int, or frozen Struct"]
        ValueType["Value Type<br/>dataclass, NamedTuple, Pydantic"]
    end
    
    subgraph "Type Analysis"
        Analyze["analyze_type_info()"]
        KeyInfo["AnalyzedTypeInfo<br/>for key"]
        ValueInfo["AnalyzedTypeInfo<br/>for value"]
    end
    
    subgraph "Engine Schema"
        KeyFields["Key Fields<br/>FieldSchema[]"]
        ValueFields["Value Fields<br/>FieldSchema[]"]
        TableType["TableType(KTable)<br/>row: StructSchema<br/>num_key_parts"]
    end
    
    subgraph "Flow Data"
        KTable["KTable<Key, Value><br/>Available in flow"]
    end
    
    Decorator --> KeyType
    Decorator --> ValueType
    
    KeyType --> Analyze
    ValueType --> Analyze
    Analyze --> KeyInfo
    Analyze --> ValueInfo
    
    KeyInfo --> KeyFields
    ValueInfo --> ValueFields
    
    KeyFields --> TableType
    ValueFields --> TableType
    
    TableType --> KTable
```

**Diagram: Source Type Analysis and Schema Generation**

Each source connector specifies its output types through the `@source_connector` decorator parameters:

```python
@source_connector(
    spec_cls=LocalFileSpec,
    key_type=str,                    # Single key field "_key"
    value_type=LocalFileValue        # Struct with content fields
)
```

For composite keys, use a frozen dataclass or NamedTuple:

```python
@dataclass(frozen=True)
class CompositeKey:
    part1: str
    part2: int

@source_connector(
    spec_cls=MySourceSpec,
    key_type=CompositeKey,          # Multiple key fields: part1, part2
    value_type=MyValueType
)
```

The engine analyzes these types using the type system to create the appropriate `TableType(KTable)` schema with correctly determined `num_key_parts`.

Sources: [python/cocoindex/op.py:652-692](), [python/cocoindex/typing.py:406-449](), [docs/docs/core/data_types.mdx:219-269]()

---

## Integration with Flow Definition

Sources integrate with the flow definition system through the `FlowBuilder`:

```mermaid
sequenceDiagram
    participant User as "Flow Definition"
    participant FB as "FlowBuilder"
    participant Engine as "CocoIndex Engine"
    participant SC as "Source Connector"
    
    User->>FB: add_source(spec, refresh_interval=...)
    FB->>FB: dump_engine_object(spec)
    FB->>Engine: register_source_operation
    Engine->>Engine: store source in flow DAG
    Engine->>SC: register operation type
    FB-->>User: DataSlice (KTable type)
    
    Note over User,FB: Flow definition continues
    
    User->>User: data_scope["docs"].row()
    User->>User: transform operations...
    
    Note over FB,Engine: During flow execution
    
    Engine->>SC: create_executor(spec)
    SC->>SC: get_table_type()
    SC-->>Engine: TableType schema
    Engine->>SC: list_async(options)
    SC-->>Engine: row iterator
    Engine->>Engine: process rows through DAG
```

**Diagram: Source Integration with Flow Execution**

The `add_source()` method:
1. Serializes the source spec using `dump_engine_object()`
2. Registers the source operation with the engine
3. Returns a `DataSlice` representing the source table
4. The `DataSlice` has a `TableType(KTable)` type determined by the source's key and value types

During execution:
1. Engine calls `create_executor()` to instantiate the source
2. Engine requests table type schema
3. Engine calls `list_async()` to iterate rows
4. Engine processes rows through the transformation DAG
5. Engine tracks row states for incremental processing

Sources: [python/cocoindex/op.py:652-704](), [docs/docs/core/flow_def.mdx:113-131]()

---

## Best Practices

### Choosing Key Types

Select key types that:
- **Uniquely identify rows**: Keys must be stable and unique
- **Support equality comparison**: All key types must be comparable
- **Are immutable**: Keys should never change for a given row

Good key examples:
- File paths for filesystem sources
- Object keys for cloud storage
- Primary key columns for database sources
- Composite keys for multi-field identifiers

### Using Refresh Intervals

Choose refresh intervals based on:
- **Update frequency**: How often source data changes
- **Latency requirements**: How quickly updates must propagate
- **System load**: Shorter intervals increase load

Typical patterns:
```python
# Fast updates (every 30 seconds) for frequently changing data
refresh_interval=datetime.timedelta(seconds=30)

# Moderate updates (every 5 minutes) for typical document indexing
refresh_interval=datetime.timedelta(minutes=5)

# Slow updates (every hour) for relatively static data
refresh_interval=datetime.timedelta(hours=1)
```

### Protecting Credentials

Always use auth entries for credentials:

```python
# ✅ Good: Credentials in auth registry
credentials = cocoindex.add_transient_auth_entry({"private_key": "..."})
flow_builder.add_source(cocoindex.sources.GoogleDrive(
    folder_id="...",
    credentials=credentials
))

# ❌ Bad: Credentials in spec (visible in cocoindex show, logs, etc.)
flow_builder.add_source(cocoindex.sources.GoogleDrive(
    folder_id="...",
    credentials={"private_key": "..."}  # NEVER DO THIS
))
```

Sources: [docs/docs/core/flow_def.mdx:439-526]()

### Handling Binary vs Text Content

Sources return content as `bytes` or `str` depending on the data type:
- `LocalFile`: Returns `str` (text content)
- `AmazonS3`: Returns `bytes` (binary content)
- `AzureBlob`: Returns `bytes` (binary content)
- `Postgres`: Returns typed data per column schema

Convert bytes to text when needed:

```python
with data_scope["s3_objects"].row() as obj:
    # Convert bytes to string if needed
    obj["text_content"] = obj["content"].transform(
        lambda b: b.decode('utf-8')
    )
```

### Memory Management with Large Sources

For sources with many rows, use concurrency limits:

```python
data_scope["large_dataset"] = flow_builder.add_source(
    source_spec,
    max_inflight_rows=50,              # Process 50 rows at a time
    max_inflight_bytes=500*1024*1024   # Max 500MB in memory
)
```

This prevents memory exhaustion when indexing large datasets. Tune based on:
- Available RAM
- Average row size
- Downstream operation memory requirements

Sources: [docs/docs/core/flow_def.mdx:365-414]()

---

## Summary

CocoIndex provides five built-in source connectors for common data sources:

| Source | Use Case | Key Type | Change Capture |
|--------|----------|----------|----------------|
| `LocalFile` | Local filesystem documents | `str` (filename) | Refresh interval + mtime |
| `AmazonS3` | Cloud object storage | `str` (object key) | Refresh interval + S3 events |
| `AzureBlob` | Azure cloud storage | `str` (blob name) | Refresh interval + blob metadata |
| `GoogleDrive` | Google Drive documents | `str` (file ID) | Refresh interval + Drive API |
| `Postgres` | Database tables | Table PK columns | Refresh interval + NOTIFY events |

All sources:
- Implement the source connector interface defined in [python/cocoindex/op.py:638-727]()
- Support `refresh_interval` for live updates
- Provide KTable output with key and value fields
- Integrate with incremental processing and state management
- Support concurrency control via `max_inflight_rows` and `max_inflight_bytes`

For custom source implementation details, see [Custom Source Implementation](#8.3).

Sources: [python/cocoindex/op.py:86-88](), [python/cocoindex/op.py:638-727](), [README.md:86-92](), All built-in source documentation

---

# Page: Built-in Processing Functions

# Built-in Processing Functions

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [.cargo/config.toml](.cargo/config.toml)
- [docs/docs/ops/functions.md](docs/docs/ops/functions.md)
- [python/cocoindex/functions/_engine_builtin_specs.py](python/cocoindex/functions/_engine_builtin_specs.py)
- [rust/cocoindex/src/llm/litellm.rs](rust/cocoindex/src/llm/litellm.rs)
- [rust/cocoindex/src/llm/openrouter.rs](rust/cocoindex/src/llm/openrouter.rs)
- [rust/cocoindex/src/llm/vllm.rs](rust/cocoindex/src/llm/vllm.rs)
- [rust/cocoindex/src/ops/functions/embed_text.rs](rust/cocoindex/src/ops/functions/embed_text.rs)

</details>



## Purpose and Scope

Built-in processing functions are transformation operations implemented in the Rust engine and exposed to Python via the operation system. These functions are invoked through `DataSlice.transform()` and execute as part of the dataflow pipeline.

This page catalogs all built-in functions by category, specifying their input/output schemas, Rust implementation locations, and configuration options. For implementation details:
- Text chunking functions: see [Text Processing and Chunking](#6.1)
- Embedding functions: see [Embedding and AI Functions](#6.2)

Related pages:
- Custom function implementation: see [Operation Registration and Decorators](#4.1)
- LLM provider configuration: see [LLM Integration](#7)
- Type system details: see [Type System and Schema Analysis](#4.2)

Sources: [docs/docs/ops/functions.md:1-262](), [python/cocoindex/functions/_engine_builtin_specs.py:1-70]()

## Function Architecture

Built-in functions follow a two-layer architecture: Python specification classes (`FunctionSpec` subclasses) define the interface, while Rust factories (`SimpleFunctionFactoryBase` implementations) execute the logic.

```mermaid
graph TB
    subgraph Python["Python Layer (python/cocoindex/functions/)"]
        ParseJsonSpec["ParseJson(FunctionSpec)"]
        SplitRecursivelySpec["SplitRecursively(FunctionSpec)"]
        EmbedTextSpec["EmbedText(FunctionSpec)"]
        ExtractByLlmSpec["ExtractByLlm(FunctionSpec)"]
        ColPaliSpec["ColPaliEmbedImage(FunctionSpec)"]
    end
    
    subgraph Registration["Operation Registration (python/cocoindex/op.py)"]
        RegisterFactory["_register_op_factory()"]
        AnalyzeType["analyze_type_info()"]
        MakeEncoder["make_engine_value_encoder()"]
        MakeDecoder["make_engine_value_decoder()"]
    end
    
    subgraph RustEngine["Rust Engine (rust/cocoindex/src/ops/functions/)"]
        ParseJsonFactory["parse_json::Factory"]
        SplitFactory["split_recursively::Factory"]
        EmbedFactory["embed_text::Factory"]
        ExtractFactory["extract_by_llm::Factory"]
        ColPaliFactory["colpali::Factory"]
    end
    
    subgraph Execution["Execution Layer"]
        SimpleFunctionExecutor["SimpleFunctionExecutor"]
        BatchedFunctionExecutor["BatchedFunctionExecutor"]
        SubprocessExecutor["SubprocessExecutor (GPU)"]
    end
    
    ParseJsonSpec --> RegisterFactory
    SplitRecursivelySpec --> RegisterFactory
    EmbedTextSpec --> RegisterFactory
    ExtractByLlmSpec --> RegisterFactory
    ColPaliSpec --> RegisterFactory
    
    RegisterFactory --> AnalyzeType
    AnalyzeType --> MakeEncoder
    AnalyzeType --> MakeDecoder
    
    MakeEncoder --> ParseJsonFactory
    MakeEncoder --> SplitFactory
    MakeEncoder --> EmbedFactory
    MakeEncoder --> ExtractFactory
    MakeEncoder --> ColPaliFactory
    
    ParseJsonFactory --> SimpleFunctionExecutor
    SplitFactory --> SimpleFunctionExecutor
    EmbedFactory --> BatchedFunctionExecutor
    ExtractFactory --> BatchedFunctionExecutor
    ColPaliFactory --> SubprocessExecutor
    
    MakeDecoder --> SimpleFunctionExecutor
    MakeDecoder --> BatchedFunctionExecutor
    MakeDecoder --> SubprocessExecutor
```

**Registration and Execution**

Each built-in function has a Python `FunctionSpec` class in [python/cocoindex/functions/_engine_builtin_specs.py:1-70]() and a corresponding Rust `Factory` implementation in [rust/cocoindex/src/ops/functions/](). During flow compilation, `_register_op_factory()` registers the spec with the Rust engine, and `analyze_type_info()` creates type-safe encoders/decoders. At runtime, the Rust factory's `analyze()` method validates input types and `build_executor()` creates the executor (simple, batched, or subprocess-based for GPU).

Sources: [python/cocoindex/functions/_engine_builtin_specs.py:1-70](), [rust/cocoindex/src/ops/functions/embed_text.rs:98-177]()

## Function Categories

Built-in functions are organized into four categories based on their implementation and dependencies:

```mermaid
graph TB
    subgraph BuiltinFunctions["Built-in Functions (rust/cocoindex/src/ops/functions/)"]
        ParseJson["ParseJson<br/>parse_json.rs"]
        DetectLang["DetectProgrammingLanguage<br/>detect_programming_language.rs"]
        SplitRecursively["SplitRecursively<br/>split_recursively.rs"]
        SplitBySeparators["SplitBySeparators<br/>split_by_separators.rs"]
    end
    
    subgraph PythonHosted["Python-Hosted Functions (requires extras)"]
        SentenceTransformer["SentenceTransformerEmbed<br/>python/cocoindex/functions/sentence_transformer_embed.py"]
        ColPaliImage["ColPaliEmbedImage<br/>python/cocoindex/functions/colpali.py"]
        ColPaliQuery["ColPaliEmbedQuery<br/>python/cocoindex/functions/colpali.py"]
    end
    
    subgraph LlmApiBased["LLM API-Based Functions"]
        EmbedText["EmbedText<br/>rust/cocoindex/src/ops/functions/embed_text.rs"]
        ExtractByLlm["ExtractByLlm<br/>rust/cocoindex/src/ops/functions/extract_by_llm.rs"]
    end
    
    subgraph Dependencies["External Dependencies"]
        RustCore["Base Rust engine"]
        SentenceTransformersLib["sentence-transformers library"]
        ColPaliLib["colpali-engine library"]
        LlmClients["LLM API clients:<br/>OpenAI, Gemini, Vertex AI,<br/>Ollama, Voyage, etc."]
    end
    
    ParseJson --> RustCore
    DetectLang --> RustCore
    SplitRecursively --> RustCore
    SplitBySeparators --> RustCore
    
    SentenceTransformer --> SentenceTransformersLib
    ColPaliImage --> ColPaliLib
    ColPaliQuery --> ColPaliLib
    
    EmbedText --> LlmClients
    ExtractByLlm --> LlmClients
```

**Category Summary**

| Category | Rust Implementation | Python Extras | External APIs |
|----------|-------------------|---------------|---------------|
| **Text Parsing** | ✓ | - | - |
| `ParseJson` | [rust/cocoindex/src/ops/functions/parse_json.rs]() | - | - |
| `DetectProgrammingLanguage` | [rust/cocoindex/src/ops/functions/detect_programming_language.rs]() | - | - |
| **Text Chunking** | ✓ | - | - |
| `SplitRecursively` | [rust/cocoindex/src/ops/functions/split_recursively.rs]() | - | - |
| `SplitBySeparators` | [rust/cocoindex/src/ops/functions/split_by_separators.rs]() | - | - |
| **Local Embeddings** | - | ✓ | - |
| `SentenceTransformerEmbed` | - | `pip install 'cocoindex[embeddings]'` | - |
| `ColPaliEmbedImage` | - | `pip install 'cocoindex[colpali]'` | - |
| `ColPaliEmbedQuery` | - | `pip install 'cocoindex[colpali]'` | - |
| **API-Based Processing** | ✓ | - | ✓ |
| `EmbedText` | [rust/cocoindex/src/ops/functions/embed_text.rs]() | - | See [LLM Integration](#7) |
| `ExtractByLlm` | [rust/cocoindex/src/ops/functions/extract_by_llm.rs]() | - | See [LLM Integration](#7) |

Sources: [rust/cocoindex/src/ops/functions/embed_text.rs:1-237](), [python/cocoindex/functions/_engine_builtin_specs.py:1-70](), [docs/docs/ops/functions.md:1-262]()

## Function Reference

### Data Parsing Functions

#### ParseJson

Parses JSON text into a structured JSON object using the `serde_json` parser.

```python
ParseJson()
```

**Spec:** [python/cocoindex/functions/_engine_builtin_specs.py:10-11]()  
**Implementation:** [rust/cocoindex/src/ops/functions/parse_json.rs]()

| Field | Type | Description |
|-------|------|-------------|
| **Spec Fields** | | |
| *(none)* | | Spec has no configuration fields |
| **Input Arguments** | | |
| `text` | `Str` | JSON text to parse |
| `language` | `Optional[Str]` | Reserved for future use (default: `"json"`) |
| **Output** | `Json` | Parsed JSON object represented as `Value::Json` |

**Example:**
```python
doc["parsed"] = doc["json_text"].transform(cocoindex.functions.ParseJson())
```

Sources: [docs/docs/ops/functions.md:8-17](), [python/cocoindex/functions/_engine_builtin_specs.py:10-11]()

#### DetectProgrammingLanguage

Detects programming language from file extension by matching against a built-in registry.

```python
DetectProgrammingLanguage()
```

**Spec:** [python/cocoindex/functions/_engine_builtin_specs.py:23-24]()  
**Implementation:** [rust/cocoindex/src/ops/functions/detect_programming_language.rs]()

| Field | Type | Description |
|-------|------|-------------|
| **Spec Fields** | | |
| *(none)* | | Spec has no configuration fields |
| **Input Arguments** | | |
| `filename` | `Str` | Filename with extension to detect |
| **Output** | `Str` or `Null` | Language name matching [`tree-sitter-language-pack`](https://github.com/Goldziher/tree-sitter-language-pack) naming, or `Null` if extension is unrecognized |

**Example:**
```python
doc["language"] = doc["filename"].transform(
    cocoindex.functions.DetectProgrammingLanguage()
)
```

Sources: [docs/docs/ops/functions.md:19-29](), [python/cocoindex/functions/_engine_builtin_specs.py:23-24]()

### Text Chunking Functions

#### SplitRecursively

Splits text into chunks using hierarchical boundary detection. For recognized languages, splits at high-level boundaries (sections, classes) before falling back to low-level boundaries (paragraphs, sentences).

```python
SplitRecursively(custom_languages=None)
```

**Spec:** [python/cocoindex/functions/_engine_builtin_specs.py:27-30]()  
**Implementation:** [rust/cocoindex/src/ops/functions/split_recursively.rs]()

| Field | Type | Description |
|-------|------|-------------|
| **Spec Fields** | | |
| `custom_languages` | `list[CustomLanguageSpec]` | Custom language definitions with `language_name`, `separators_regex`, and `aliases` |
| **Input Arguments** | | |
| `text` | `Str` | Text to split into chunks |
| `chunk_size` | `Int64` | Maximum chunk size in bytes |
| `min_chunk_size` | `Int64` | Minimum chunk size (default: `chunk_size / 2`) |
| `chunk_overlap` | `Optional[Int64]` | Maximum overlap between adjacent chunks (default: `None`) |
| `language` | `Optional[Str]` | Language name, alias, or file extension (case-insensitive). If `None` or unrecognized, treated as plain text. |
| **Output** | `KTable` | Each row represents a chunk: |
| → `location` | `Range` | Location range for the chunk |
| → `text` | `Str` | Chunk text content |
| → `start` | `Struct` | Start position with `offset` (Int64), `line` (Int64), `column` (Int64) |
| → `end` | `Struct` | End position with `offset` (Int64), `line` (Int64), `column` (Int64) |

**Language Matching:** The `language` field is matched (case-insensitive) against:
1. `custom_languages` entries by `language_name` or `aliases`
2. Built-in language registry (29 languages supported)
3. If no match, text is treated as plain article with paragraph/sentence splitting

**Supported Built-in Languages:** c, cpp, csharp, css, dtd, fortran, go, html, java, javascript, json, kotlin, markdown, pascal, php, python, r, ruby, rust, scala, solidity, sql, swift, toml, tsx, typescript, xml, yaml. See [docs/docs/ops/functions.md:98-128]() for aliases and file extensions.

**Example:**
```python
doc["chunks"] = doc["content"].transform(
    cocoindex.functions.SplitRecursively(),
    language="markdown",
    chunk_size=2000,
    chunk_overlap=500
)
```

Sources: [docs/docs/ops/functions.md:32-128](), [python/cocoindex/functions/_engine_builtin_specs.py:27-30]()

### Embedding Functions

#### SentenceTransformerEmbed

Embeds text using local `sentence-transformers` models. Executed in a subprocess with GPU allocation support.

```python
SentenceTransformerEmbed(model, args=None)
```

**Installation:** `pip install 'cocoindex[embeddings]'`  
**Spec:** [python/cocoindex/functions/_engine_builtin_specs.py]()  
**Implementation:** [python/cocoindex/functions/sentence_transformer_embed.py]()

| Field | Type | Description |
|-------|------|-------------|
| **Spec Fields** | | |
| `model` | `str` | SentenceTransformer model identifier (HuggingFace model name) |
| `args` | `dict[str, Any]` | Optional arguments passed to `SentenceTransformer()` constructor (e.g., `{"trust_remote_code": True}`) |
| **Input Arguments** | | |
| `text` | `Str` | Text to embed |
| **Output** | `Vector[Float32, N]` | Embedding vector with dimension N determined by model |

**GPU Execution:** This function uses `@executor_class` with `enable_subprocess=True`, executing in an isolated subprocess to manage GPU resources. Supports `OpArgs(gpu=...)` for GPU allocation.

**Example:**
```python
chunk["embedding"] = chunk["text"].transform(
    cocoindex.functions.SentenceTransformerEmbed(
        model="sentence-transformers/all-MiniLM-L6-v2"
    )
)
```

Sources: [docs/docs/ops/functions.md:133-157](), [python/cocoindex/functions/sentence_transformer_embed.py]()

#### EmbedText

Embeds text using LLM API embedding endpoints. Supports OpenAI, Gemini, Vertex AI, Ollama, Voyage, and other providers.

```python
EmbedText(api_type, model, address=None, output_dimension=None, 
          expected_output_dimension=None, task_type=None, api_config=None, api_key=None)
```

**Spec:** [python/cocoindex/functions/_engine_builtin_specs.py:51-62]()  
**Implementation:** [rust/cocoindex/src/ops/functions/embed_text.rs:1-237]()

| Field | Type | Description |
|-------|------|-------------|
| **Spec Fields** | | |
| `api_type` | `LlmApiType` | API provider enum (see [LLM Integration](#7)) |
| `model` | `str` | Model identifier (e.g., `"text-embedding-3-small"`) |
| `address` | `Optional[str]` | Custom API endpoint URL (uses default if `None`) |
| `output_dimension` | `Optional[int]` | Dimension parameter sent to API (for APIs supporting dimension reduction) |
| `expected_output_dimension` | `Optional[int]` | Expected output dimension for validation and type schema. Defaults to `output_dimension`, then model's known default. |
| `task_type` | `Optional[str]` | Task type hint for embedding optimization (e.g., `"retrieval_document"`, `"retrieval_query"`) |
| `api_config` | `Optional[VertexAiConfig]` | Provider-specific config (currently Vertex AI only) |
| `api_key` | `Optional[TransientAuthEntryReference[str]]` | API key reference from auth registry |
| **Input Arguments** | | |
| `text` | `Str` | Text to embed |
| **Output** | `Vector[Float32, N]` | Embedding vector with dimension N |

**Dimension Handling:** If `expected_output_dimension` is unspecified, the function attempts to infer it from:
1. `output_dimension` (if provided)
2. Known default dimensions for common models (internal registry in Rust)
3. If inference fails, raises error requiring explicit specification

**Batching:** Implements `BatchedFunctionExecutor` with default `max_batch_size=64` for efficient API calls. See [rust/cocoindex/src/ops/functions/embed_text.rs:37-43]().

**Example:**
```python
chunk["embedding"] = chunk["text"].transform(
    cocoindex.functions.EmbedText(
        api_type=cocoindex.LlmApiType.OPENAI,
        model="text-embedding-3-small",
        output_dimension=1536
    )
)
```

Sources: [docs/docs/ops/functions.md:184-211](), [rust/cocoindex/src/ops/functions/embed_text.rs:1-237](), [python/cocoindex/functions/_engine_builtin_specs.py:51-62]()

#### ColPali Functions

ColPali functions enable multimodal document retrieval using the `colpali-engine` library. These functions support all ColVision model families (ColPali, ColQwen2, ColSmol) and use late interaction scoring between multi-vector image and query embeddings.

**Installation:** `pip install 'cocoindex[colpali]'`  
**Implementation:** [python/cocoindex/functions/colpali.py]()

##### ColPaliEmbedImage

Embeds images into multi-vector representations using ColVision models.

```python
ColPaliEmbedImage(model)
```

| Field | Type | Description |
|-------|------|-------------|
| **Spec Fields** | | |
| `model` | `str` | ColVision model identifier (e.g., `"vidore/colpali-v1.2"`) |
| **Input Arguments** | | |
| `img_bytes` | `Bytes` | Image data in bytes format |
| **Output** | `Vector[Vector[Float32, N]]` | Multi-vector with variable number of patches and fixed hidden dimension N per patch |

##### ColPaliEmbedQuery

Embeds text queries into multi-vector representations for late interaction with ColVision image embeddings.

```python
ColPaliEmbedQuery(model)
```

| Field | Type | Description |
|-------|------|-------------|
| **Spec Fields** | | |
| `model` | `str` | ColVision model identifier (must match the image embedding model) |
| **Input Arguments** | | |
| `query` | `Str` | Text query for retrieval |
| **Output** | `Vector[Vector[Float32, N]]` | Multi-vector with variable number of tokens and fixed hidden dimension N per token |

**Supported Model Families:**
- **ColPali** (`colpali-*`): PaliGemma-based, general document retrieval
- **ColQwen2** (`colqwen-*`): Qwen2-VL-based, multilingual (29+ languages)
- **ColSmol** (`colsmol-*`): Lightweight models for resource-constrained environments
- See [complete model list](https://github.com/illuin-tech/colpali#list-of-colvision-models)

**Late Interaction Scoring:** Query-image matching uses MaxSim scoring: `score = sum(max(query_token · image_patch for each image_patch) for each query_token)`.

**Example:**
```python
page["image_embedding"] = page["image_bytes"].transform(
    cocoindex.functions.ColPaliEmbedImage(
        model="vidore/colpali-v1.2"
    )
)
```

Sources: [docs/docs/ops/functions.md:213-262](), [python/cocoindex/functions/colpali.py]()

### LLM Extraction Functions

#### ExtractByLlm

Extracts structured data from text using LLM API with JSON schema guidance. The LLM is prompted to generate output matching the schema derived from a Python dataclass.

```python
ExtractByLlm(llm_spec, output_type, instruction=None)
```

**Spec:** [python/cocoindex/functions/_engine_builtin_specs.py:64-69]()  
**Implementation:** [rust/cocoindex/src/ops/functions/extract_by_llm.rs]()

| Field | Type | Description |
|-------|------|-------------|
| **Spec Fields** | | |
| `llm_spec` | `LlmSpec` | LLM configuration with `api_type`, `model`, `address`, etc. (see [LLM Integration](#7)) |
| `output_type` | `type` | Python dataclass or type annotation defining output schema. Converted to JSON schema for LLM guidance. |
| `instruction` | `Optional[str]` | Additional instruction appended to system prompt |
| **Input Arguments** | | |
| `text` | `Str` | Text to extract information from |
| **Output** | As specified by `output_type` | Structured data matching the schema. Type is determined by `analyze_type_info(output_type)`. |

**Schema Generation:** The `output_type` is analyzed by [python/cocoindex/type_inference.py]() to generate a JSON schema. This schema is sent to the LLM via structured output APIs (OpenAI's response format, Gemini's schema, etc.) or as prompt guidance for APIs without native structured output support.

**Dataclass Best Practices:**
- Use descriptive field names (LLM sees them in schema)
- Add docstrings to dataclass and fields (included in schema descriptions)
- Annotate optional fields as `Type | None` or `Optional[Type]`
- Use type hints compatible with CocoIndex type system (see [Type System](#4.2))

**Example:**
```python
@dataclasses.dataclass
class ModuleInfo:
    """Information extracted from a Python module's documentation."""
    title: str
    description: str
    classes: list[str]  # List of class names
    methods: list[str]  # List of method names

doc["module_info"] = doc["markdown"].transform(
    cocoindex.functions.ExtractByLlm(
        llm_spec=cocoindex.LlmSpec(
            api_type=cocoindex.LlmApiType.OPENAI,
            model="gpt-4o"
        ),
        output_type=ModuleInfo,
        instruction="Extract module information from the Python documentation."
    )
)
```

Sources: [docs/docs/ops/functions.md:159-182](), [python/cocoindex/functions/_engine_builtin_specs.py:64-69](), [examples/manuals_llm_extraction/main.py:37-56]()

## Usage Patterns in Flows

Built-in functions are used within flow definitions through the `.transform()` method. The following diagram shows common usage patterns:

```mermaid
graph TB
    subgraph FlowStructure["Flow Structure"]
        Source["add_source()<br/>LocalFile, S3, etc."]
        Row1["row() context<br/>document iteration"]
        Transform1["transform()<br/>SplitRecursively"]
        Row2["row() context<br/>chunk iteration"]
        Transform2["transform()<br/>SentenceTransformerEmbed"]
        Collect["collect()<br/>gather results"]
        Export["export()<br/>to target"]
    end
    
    subgraph Functions["Function Invocations"]
        SplitSpec["SplitRecursively()<br/>spec instantiation"]
        SplitInput["Input: text, chunk_size,<br/>language parameters"]
        SplitOutput["Output: KTable with<br/>location, text fields"]
        
        EmbedSpec["SentenceTransformerEmbed()<br/>spec instantiation"]
        EmbedInput["Input: text field"]
        EmbedOutput["Output: Vector[Float32, N]"]
    end
    
    Source --> Row1
    Row1 --> Transform1
    Transform1 --> SplitSpec
    SplitSpec --> SplitInput
    SplitInput --> SplitOutput
    SplitOutput --> Row2
    
    Row2 --> Transform2
    Transform2 --> EmbedSpec
    EmbedSpec --> EmbedInput
    EmbedInput --> EmbedOutput
    EmbedOutput --> Collect
    
    Collect --> Export
```

**Common Patterns:**

1. **Document Chunking Pattern:**
```python
doc["chunks"] = doc["content"].transform(
    cocoindex.functions.SplitRecursively(),
    language="markdown",
    chunk_size=2000,
    chunk_overlap=500
)
```

2. **Embedding Pattern:**
```python
chunk["embedding"] = chunk["text"].transform(
    cocoindex.functions.SentenceTransformerEmbed(
        model="sentence-transformers/all-MiniLM-L6-v2"
    )
)
```

3. **LLM Extraction Pattern:**
```python
doc["extracted"] = doc["text"].transform(
    cocoindex.functions.ExtractByLlm(
        llm_spec=cocoindex.LlmSpec(
            api_type=cocoindex.LlmApiType.OPENAI,
            model="gpt-4o"
        ),
        output_type=MyDataclass
    )
)
```

4. **Nested Processing Pattern:**
```python
with data_scope["documents"].row() as doc:
    doc["chunks"] = doc["content"].transform(SplitRecursively(), ...)
    
    with doc["chunks"].row() as chunk:
        chunk["embedding"] = chunk["text"].transform(
            SentenceTransformerEmbed(...)
        )
```

Sources: [docs/docs/getting_started/quickstart.md:74-111](), [examples/manuals_llm_extraction/main.py:90-139]()

## Installation Requirements and Dependencies

Built-in functions have different installation and runtime dependencies:

```mermaid
graph TB
    subgraph BaseInstall["Base Installation: pip install cocoindex"]
        ParseJson["ParseJson"]
        DetectLang["DetectProgrammingLanguage"]
        SplitRecursively["SplitRecursively"]
        SplitBySeparators["SplitBySeparators"]
        EmbedText["EmbedText"]
        ExtractByLlm["ExtractByLlm"]
    end
    
    subgraph EmbeddingsExtra["Extra: pip install 'cocoindex[embeddings]'"]
        SentenceTransformer["SentenceTransformerEmbed"]
        SentenceTransformersLib["sentence-transformers library"]
    end
    
    subgraph ColpaliExtra["Extra: pip install 'cocoindex[colpali]'"]
        ColPaliImage["ColPaliEmbedImage"]
        ColPaliQuery["ColPaliEmbedQuery"]
        ColpaliEngineLib["colpali-engine library"]
    end
    
    subgraph RuntimeDeps["Runtime Dependencies"]
        ApiKeys["LLM API Keys:<br/>OPENAI_API_KEY,<br/>GEMINI_API_KEY, etc."]
        VectorDB["Vector Databases:<br/>Postgres+pgvector,<br/>Qdrant"]
    end
    
    SentenceTransformer --> SentenceTransformersLib
    ColPaliImage --> ColpaliEngineLib
    ColPaliQuery --> ColpaliEngineLib
    
    EmbedText -.requires.-> ApiKeys
    ExtractByLlm -.requires.-> ApiKeys
    
    SentenceTransformer -.optional.-> VectorDB
    EmbedText -.optional.-> VectorDB
    ColPaliImage -.optional.-> VectorDB
```

**Installation Matrix**

| Function | Python Package | Optional Extras | Runtime Requirements |
|----------|---------------|-----------------|---------------------|
| `ParseJson` | `cocoindex` | - | - |
| `DetectProgrammingLanguage` | `cocoindex` | - | - |
| `SplitRecursively` | `cocoindex` | - | - |
| `SplitBySeparators` | `cocoindex` | - | - |
| `SentenceTransformerEmbed` | `cocoindex[embeddings]` | `sentence-transformers`, `torch` | GPU optional |
| `EmbedText` | `cocoindex` | - | LLM API key |
| `ExtractByLlm` | `cocoindex` | - | LLM API key |
| `ColPaliEmbedImage` | `cocoindex[colpali]` | `colpali-engine`, `torch` | GPU optional |
| `ColPaliEmbedQuery` | `cocoindex[colpali]` | `colpali-engine`, `torch` | GPU optional |

**LLM API Authentication**

Functions using LLM APIs (`EmbedText`, `ExtractByLlm`) require authentication via environment variables or auth registry:

| Provider | Environment Variable | Alternative Auth Method |
|----------|---------------------|------------------------|
| OpenAI | `OPENAI_API_KEY` | `api_key` spec field |
| Gemini | `GEMINI_API_KEY` | `api_key` spec field |
| Anthropic | `ANTHROPIC_API_KEY` | `api_key` spec field |
| Voyage | `VOYAGE_API_KEY` | `api_key` spec field |
| Vertex AI | Application Default Credentials | `api_config` with project/location |
| Ollama | - (local) | - |
| OpenRouter | `OPENROUTER_API_KEY` | `api_key` spec field |
| vLLM | `VLLM_API_KEY` (optional) | `api_key` spec field |
| LiteLLM | `LITELLM_API_KEY` (optional) | `api_key` spec field |

See [LLM Integration](#7) and [Authentication and Secrets Management](#10.1) for configuration details.

Sources: [docs/docs/ops/functions.md:138-144](), [rust/cocoindex/src/llm/openrouter.rs:1-22](), [rust/cocoindex/src/llm/vllm.rs:1-22](), [rust/cocoindex/src/llm/litellm.rs:1-22]()

## Execution Model and Performance Characteristics

Built-in functions execute via different executor types with distinct performance characteristics:

```mermaid
graph TB
    subgraph ExecutionPath["Function Execution Path"]
        Transform["DataSlice.transform()"]
        EngineCall["Rust Engine Call<br/>via PyO3 FFI"]
        Factory["Factory::analyze()"]
        Executor["Factory::build_executor()"]
    end
    
    subgraph ExecutorTypes["Executor Types"]
        Simple["SimpleFunctionExecutor<br/>Synchronous, no batching"]
        Batched["BatchedFunctionExecutor<br/>Async, auto-batching"]
        Subprocess["SubprocessExecutor<br/>GPU isolation"]
    end
    
    subgraph ExecutionFeatures["Execution Features"]
        Cache["Content-based Caching<br/>SHA256 fingerprint"]
        Retry["Retry Logic<br/>Exponential backoff"]
        GPU["GPU Allocation<br/>OpArgs.gpu"]
        Batch["Dynamic Batching<br/>max_batch_size"]
    end
    
    Transform --> EngineCall
    EngineCall --> Factory
    Factory --> Executor
    
    Executor --> Simple
    Executor --> Batched
    Executor --> Subprocess
    
    Simple --> Cache
    Batched --> Cache
    Batched --> Retry
    Batched --> Batch
    Subprocess --> GPU
    Subprocess --> Cache
```

**Executor Type Assignment**

| Function | Executor Type | Rationale |
|----------|--------------|-----------|
| `ParseJson` | `SimpleFunctionExecutor` | Fast, deterministic parsing |
| `DetectProgrammingLanguage` | `SimpleFunctionExecutor` | Lookup-based, no I/O |
| `SplitRecursively` | `SimpleFunctionExecutor` | CPU-bound, single-item processing |
| `SplitBySeparators` | `SimpleFunctionExecutor` | CPU-bound, single-item processing |
| `SentenceTransformerEmbed` | `SubprocessExecutor` | GPU-enabled, Python-hosted |
| `EmbedText` | `BatchedFunctionExecutor` | API calls benefit from batching |
| `ExtractByLlm` | `BatchedFunctionExecutor` | LLM API calls (batching disabled by design) |
| `ColPaliEmbedImage` | `SubprocessExecutor` | GPU-enabled, Python-hosted |
| `ColPaliEmbedQuery` | `SubprocessExecutor` | GPU-enabled, Python-hosted |

**Caching Behavior**

All functions support content-based caching via [rust/cocoindex/src/incremental/]() infrastructure:
- Cache key: SHA256 hash of serialized input arguments
- Cache storage: PostgreSQL internal state database
- Cache hit: Result returned immediately without re-execution
- Configurable via `OpArgs(cache=True/False)`

**Batching Configuration**

`EmbedText` implements batching in [rust/cocoindex/src/ops/functions/embed_text.rs:37-43]():
```
max_batch_size: Some(64)  // Safe default for most embedding APIs
```

Batching reduces API latency by grouping multiple input texts into single API requests.

**GPU Execution**

Functions using `SubprocessExecutor` run in isolated subprocesses:
- Subprocess lifecycle: [rust/cocoindex/src/ops/executor_stub/]()
- GPU allocation: Specified via `OpArgs(gpu="0")` or `OpArgs(gpu="0,1")`
- Process pool: Reuses subprocesses across invocations
- Crash recovery: Parent watchdog detects subprocess crashes and restarts

**Retry Logic**

Functions calling external APIs (LLM, embedding) use exponential backoff for transient failures:
- Implemented in [rust/cocoindex/src/llm/]() client implementations
- Retryable errors: Network timeouts, rate limits (429), server errors (500-503)
- Non-retryable errors: Authentication failures (401, 403), invalid requests (400)

For execution configuration, see [OpArgs and Execution Options](#4.4).

Sources: [rust/cocoindex/src/ops/functions/embed_text.rs:32-96](), [python/cocoindex/functions/sentence_transformer_embed.py]()

## Type System Integration

Built-in functions integrate with CocoIndex's type system to ensure type safety across the Python-Rust boundary. Each function declares input and output types that are validated during flow compilation.

**Type Mapping Examples:**

| Python Type Annotation | CocoIndex Type | Function Example |
|------------------------|----------------|------------------|
| `str` | `Str` | `SplitRecursively` input |
| `bytes` | `Bytes` | `ColPaliEmbedImage` input |
| `list[dict]` | `KTable` | `SplitRecursively` output |
| `list[float]` | `Vector[Float32, N]` | `SentenceTransformerEmbed` output |
| `dataclass` | `Struct` | `ExtractByLlm` output |
| `dict` | `Json` | `ParseJson` output |

The type analysis system automatically generates encoders and decoders for data transfer between Python and Rust. This ensures that function outputs correctly match expected input types in subsequent transformations.

For details on the type system, see [Type System and Schema Analysis](#4.2).

Sources: [docs/docs/ops/functions.md:1-253]()

---

# Page: Text Processing and Chunking

# Text Processing and Chunking

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [.cargo/config.toml](.cargo/config.toml)
- [docs/docs/ops/functions.md](docs/docs/ops/functions.md)
- [python/cocoindex/functions/_engine_builtin_specs.py](python/cocoindex/functions/_engine_builtin_specs.py)
- [rust/cocoindex/src/llm/litellm.rs](rust/cocoindex/src/llm/litellm.rs)
- [rust/cocoindex/src/llm/openrouter.rs](rust/cocoindex/src/llm/openrouter.rs)
- [rust/cocoindex/src/llm/vllm.rs](rust/cocoindex/src/llm/vllm.rs)
- [rust/cocoindex/src/ops/functions/embed_text.rs](rust/cocoindex/src/ops/functions/embed_text.rs)

</details>



## Purpose and Scope

This document covers CocoIndex's built-in text processing and chunking functions. These operations handle text manipulation, document splitting, JSON parsing, and programming language detection. They are typically applied early in data pipelines to prepare raw text for downstream operations like embedding or LLM extraction.

For embedding functions that operate on processed text, see [Embedding and AI Functions](#6.2). For LLM-based extraction, see [LLM Integration](#7).

Sources: [docs/docs/ops/functions.md:1-132](), [python/cocoindex/functions/_engine_builtin_specs.py:1-70]()

## Text Processing Function Catalog

CocoIndex provides four core text processing functions, each registered as a `FunctionSpec` in the operation system:

| Function | Purpose | Input Types | Output Type |
|----------|---------|-------------|-------------|
| `ParseJson` | Parse text to JSON objects | `text: Str`, `language: Str` (optional) | `Json` |
| `DetectProgrammingLanguage` | Detect language from filename | `filename: Str` | `Str` or `Null` |
| `SplitRecursively` | Intelligent document chunking | `text: Str`, `chunk_size: Int64`, `language: Str` (optional) | `KTable` with chunk metadata |
| `SplitBySeparators` | Simple regex-based splitting | `text: Str`, `separators_regex: list[str]` | `KTable` with chunk metadata |

```mermaid
graph LR
    subgraph "Text Processing Functions"
        ParseJson["ParseJson"]
        DetectProgrammingLanguage["DetectProgrammingLanguage"]
        SplitRecursively["SplitRecursively"]
        SplitBySeparators["SplitBySeparators"]
    end
    
    subgraph "Python Layer"
        ParseJsonSpec["ParseJson (FunctionSpec)"]
        DetectProgLangSpec["DetectProgrammingLanguage (FunctionSpec)"]
        SplitRecursivelySpec["SplitRecursively (FunctionSpec)"]
        SplitBySeparatorsSpec["SplitBySeparators (FunctionSpec)"]
        CustomLanguageSpec["CustomLanguageSpec"]
    end
    
    subgraph "Rust Engine"
        ParseJsonFactory["ParseJson Factory"]
        DetectProgLangFactory["DetectProgrammingLanguage Factory"]
        SplitRecursivelyFactory["SplitRecursively Factory"]
        SplitBySeparatorsFactory["SplitBySeparators Factory"]
    end
    
    ParseJson --> ParseJsonSpec
    DetectProgrammingLanguage --> DetectProgLangSpec
    SplitRecursively --> SplitRecursivelySpec
    SplitBySeparators --> SplitBySeparatorsSpec
    
    SplitRecursivelySpec --> CustomLanguageSpec
    
    ParseJsonSpec -.registered with.-> ParseJsonFactory
    DetectProgLangSpec -.registered with.-> DetectProgLangFactory
    SplitRecursivelySpec -.registered with.-> SplitRecursivelyFactory
    SplitBySeparatorsSpec -.registered with.-> SplitBySeparatorsFactory
```

Sources: [python/cocoindex/functions/_engine_builtin_specs.py:10-49](), [docs/docs/ops/functions.md:8-132]()

## ParseJson: JSON Text Parsing

`ParseJson` converts text strings into structured JSON objects. This function is useful when source data contains JSON-formatted strings that need to be parsed before further processing.

### Specification

The `ParseJson` class extends `op.FunctionSpec` with no configuration fields:

```python
class ParseJson(op.FunctionSpec):
    """Parse a text into a JSON object."""
```

### Input and Output Schema

| Field | Type | Required | Default | Description |
|-------|------|----------|---------|-------------|
| `text` | `Str` | Yes | - | Source text to parse as JSON |
| `language` | `Str` | No | `"json"` | Language identifier (only `"json"` supported) |

**Output**: `Json` - The parsed JSON object

### Usage Example

```python
from cocoindex import flow_def
from cocoindex.functions import ParseJson

@flow_def
def parse_config_flow(builder):
    data = builder.add_source(...)
    # Parse JSON strings from source
    parsed = data.transform(ParseJson())
    # Access nested JSON fields
    extracted = parsed.select(field="parsed.config.setting")
```

Sources: [python/cocoindex/functions/_engine_builtin_specs.py:10-12](), [docs/docs/ops/functions.md:8-17]()

## DetectProgrammingLanguage: Language Detection

`DetectProgrammingLanguage` analyzes filename extensions to determine the programming language. This is essential for applying language-specific chunking strategies in `SplitRecursively`.

### Specification

```python
class DetectProgrammingLanguage(op.FunctionSpec):
    """Detect the programming language of a file."""
```

### Input and Output Schema

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `filename` | `Str` | Yes | Filename with extension to analyze |

**Output**: `Str` or `Null` - Language name matching [tree-sitter-language-pack](https://github.com/Goldziher/tree-sitter-language-pack?tab=readme-ov-file#available-languages) naming, or `Null` if unrecognized

### Supported Extensions

The function recognizes file extensions for 28 programming languages. Examples include:

- `.py` → `"python"`
- `.js` → `"javascript"`
- `.rs` → `"rust"`
- `.java` → `"java"`
- `.go` → `"go"`
- `.cpp`, `.cc`, `.h` → `"cpp"`

### Usage Example

```python
from cocoindex import flow_def
from cocoindex.functions import DetectProgrammingLanguage, SplitRecursively

@flow_def
def code_chunking_flow(builder):
    files = builder.add_source(...)
    # Detect language from filename
    with_lang = files.transform(DetectProgrammingLanguage(), filename="path")
    # Use detected language for smart chunking
    chunks = with_lang.transform(
        SplitRecursively(),
        text="content",
        language="detected_lang",
        chunk_size=1000
    )
```

Sources: [python/cocoindex/functions/_engine_builtin_specs.py:23-25](), [docs/docs/ops/functions.md:19-29]()

## SplitRecursively: Intelligent Document Chunking

`SplitRecursively` is the primary chunking function, implementing hierarchical text splitting based on language-specific structure. It attempts to split at high-level boundaries first (e.g., sections, paragraphs), falling back to lower-level boundaries (e.g., sentences, words) if chunks remain too large.

### Specification

```python
@dataclasses.dataclass
class CustomLanguageSpec:
    """Custom language specification."""
    language_name: str
    separators_regex: list[str]
    aliases: list[str] = dataclasses.field(default_factory=list)

class SplitRecursively(op.FunctionSpec):
    """Split a document (in string) recursively."""
    custom_languages: list[CustomLanguageSpec] = dataclasses.field(default_factory=list)
```

### Language Hierarchy and Splitting Strategy

```mermaid
graph TB
    Input["Input Text + Language"]
    
    subgraph "Language Matching"
        CustomLang["Check custom_languages"]
        BuiltinLang["Check 28 builtin languages"]
        PlainText["Fallback: Plain text"]
    end
    
    subgraph "Hierarchical Splitting"
        Level1["Level 1: Sections (e.g., # headers)"]
        Level2["Level 2: Subsections (e.g., ## headers)"]
        Level3["Level 3: Paragraphs"]
        Level4["Level 4: Sentences"]
        Level5["Level 5: Words"]
    end
    
    subgraph "Chunk Assembly"
        CheckSize["Check chunk size"]
        TooLarge["Chunk > chunk_size"]
        TooSmall["Chunk < min_chunk_size"]
        Acceptable["min_chunk_size <= Chunk <= chunk_size"]
        Overlap["Apply chunk_overlap"]
    end
    
    Output["KTable with chunk metadata"]
    
    Input --> CustomLang
    CustomLang --> BuiltinLang
    BuiltinLang --> PlainText
    
    CustomLang -.match.-> Level1
    BuiltinLang -.match.-> Level1
    PlainText -.match.-> Level1
    
    Level1 --> CheckSize
    Level2 --> CheckSize
    Level3 --> CheckSize
    Level4 --> CheckSize
    Level5 --> CheckSize
    
    CheckSize --> TooLarge
    CheckSize --> TooSmall
    CheckSize --> Acceptable
    
    TooLarge -.split at next level.-> Level2
    Level2 -.if still too large.-> Level3
    Level3 -.if still too large.-> Level4
    Level4 -.if still too large.-> Level5
    
    TooSmall -.merge with adjacent.-> Acceptable
    Acceptable --> Overlap
    Overlap --> Output
```

### Configuration Parameters

| Parameter | Type | Required | Default | Description |
|-----------|------|----------|---------|-------------|
| `custom_languages` | `list[CustomLanguageSpec]` | No | `[]` | Custom language definitions with regex separators |

### Input Arguments

| Argument | Type | Required | Default | Description |
|----------|------|----------|---------|-------------|
| `text` | `Str` | Yes | - | Text to split into chunks |
| `chunk_size` | `Int64` | Yes | - | Maximum chunk size in bytes |
| `min_chunk_size` | `Int64` | No | `chunk_size / 2` | Minimum chunk size in bytes |
| `chunk_overlap` | `Int64` | No | `None` | Maximum overlap between adjacent chunks in bytes |
| `language` | `Str` | No | `None` | Language name, alias, or file extension |

### Output Schema

Returns a `KTable` where each row represents a chunk with the following fields:

| Field | Type | Description |
|-------|------|-------------|
| `location` | `Range` | Character range of the chunk in original text |
| `text` | `Str` | Text content of the chunk |
| `start` | `Struct` | Start position metadata |
| `start.offset` | `Int64` | Byte offset (0-indexed) |
| `start.line` | `Int64` | Line number (1-indexed) |
| `start.column` | `Int64` | Column number (1-indexed) |
| `end` | `Struct` | End position metadata |
| `end.offset` | `Int64` | Byte offset (0-indexed) |
| `end.line` | `Int64` | Line number (1-indexed) |
| `end.column` | `Int64` | Column number (1-indexed) |

### Supported Languages

`SplitRecursively` has built-in support for 28 programming and markup languages:

| Language | Aliases | Extensions |
|----------|---------|------------|
| `c` | - | `.c` |
| `cpp` | `c++` | `.cpp`, `.cc`, `.cxx`, `.h`, `.hpp` |
| `csharp` | `cs` | `.cs` |
| `css` | - | `.css`, `.scss` |
| `go` | `golang` | `.go` |
| `html` | - | `.html`, `.htm` |
| `java` | - | `.java` |
| `javascript` | `js` | `.js` |
| `json` | - | `.json` |
| `kotlin` | - | `.kt`, `.kts` |
| `markdown` | `md` | `.md`, `.mdx` |
| `python` | - | `.py` |
| `ruby` | - | `.rb` |
| `rust` | `rs` | `.rs` |
| `sql` | - | `.sql` |
| `typescript` | `ts` | `.ts` |
| `yaml` | - | `.yaml`, `.yml` |

(Full table available in [docs/docs/ops/functions.md:98-128]())

### Custom Language Configuration

Define custom chunking strategies using `CustomLanguageSpec`:

```python
from cocoindex.functions import SplitRecursively, CustomLanguageSpec

# Define custom markdown variant with specific separators
custom_md = CustomLanguageSpec(
    language_name="custom_markdown",
    aliases=["cmd", "custom_md"],
    separators_regex=[
        r"\n# ",      # Level 1 headers
        r"\n## ",     # Level 2 headers
        r"\n### ",    # Level 3 headers
        r"\n\n",      # Paragraphs
        r"\. ",       # Sentences
    ]
)

# Use in chunking
chunker = SplitRecursively(custom_languages=[custom_md])
```

### Usage Examples

**Basic Markdown Chunking**:
```python
from cocoindex.functions import SplitRecursively

@flow_def
def markdown_chunking_flow(builder):
    docs = builder.add_source(...)
    chunks = docs.transform(
        SplitRecursively(),
        text="content",
        chunk_size=1000,
        min_chunk_size=400,
        language="markdown"
    )
    # Each chunk in 'chunks' has metadata: text, location, start, end
```

**Code Chunking with Overlap**:
```python
@flow_def
def code_chunking_flow(builder):
    code_files = builder.add_source(...)
    chunks = code_files.transform(
        SplitRecursively(),
        text="content",
        chunk_size=2000,
        chunk_overlap=200,  # 200 bytes overlap between chunks
        language="python"
    )
```

**Language-Agnostic Chunking**:
```python
@flow_def
def plain_text_flow(builder):
    text_data = builder.add_source(...)
    # Without language parameter, treats as plain text
    # Uses article-style splitting: blank lines, punctuation, whitespace
    chunks = text_data.transform(
        SplitRecursively(),
        text="content",
        chunk_size=500
    )
```

Sources: [python/cocoindex/functions/_engine_builtin_specs.py:15-31](), [docs/docs/ops/functions.md:31-132]()

## SplitBySeparators: Simple Regex-Based Splitting

`SplitBySeparators` provides a simpler alternative to `SplitRecursively`, splitting text using explicit regex patterns without hierarchical fallback logic. The output schema matches `SplitRecursively` for drop-in compatibility.

### Specification

```python
class SplitBySeparators(op.FunctionSpec):
    """
    Split text by specified regex separators only.
    Output schema matches SplitRecursively for drop-in compatibility.
    """
    separators_regex: list[str] = dataclasses.field(default_factory=list)
    keep_separator: Literal["NONE", "LEFT", "RIGHT"] = "NONE"
    include_empty: bool = False
    trim: bool = True
```

### Configuration Parameters

| Parameter | Type | Default | Description |
|-----------|------|---------|-------------|
| `separators_regex` | `list[str]` | `[]` | List of regex patterns to split by |
| `keep_separator` | `Literal["NONE", "LEFT", "RIGHT"]` | `"NONE"` | Where to attach separators: discard (`NONE`), start of chunk (`LEFT`), or end of chunk (`RIGHT`) |
| `include_empty` | `bool` | `False` | Whether to include empty chunks in output |
| `trim` | `bool` | `True` | Whether to trim whitespace from chunks |

### Input and Output Schema

**Input**: 
- `text` (`Str`): Text to split

**Output**: `KTable` with the same schema as `SplitRecursively`:
- `location`, `text`, `start`, `end` fields

### Usage Example

```python
from cocoindex.functions import SplitBySeparators

@flow_def
def paragraph_splitting_flow(builder):
    documents = builder.add_source(...)
    
    # Split by blank lines (2+ newlines)
    paragraphs = documents.transform(
        SplitBySeparators(
            separators_regex=[r"\n\n+"],
            keep_separator="NONE",
            include_empty=False,
            trim=True
        ),
        text="content"
    )
```

**Split by Multiple Separators**:
```python
# Split by sections and paragraphs
splitter = SplitBySeparators(
    separators_regex=[
        r"\n#+\s",    # Markdown headers
        r"\n\n",      # Blank lines
    ],
    keep_separator="LEFT"  # Keep headers with their content
)
```

Sources: [python/cocoindex/functions/_engine_builtin_specs.py:33-49]()

## Text Processing in Data Flows

Text processing functions integrate into CocoIndex flows as transformations on `DataSlice` objects:

```mermaid
graph LR
    subgraph "Flow Definition"
        AddSource["builder.add_source()"]
        DataSlice1["DataSlice (raw)"]
        Transform1["slice.transform(ParseJson())"]
        DataSlice2["DataSlice (parsed)"]
        Transform2["slice.transform(DetectProgrammingLanguage())"]
        DataSlice3["DataSlice (with language)"]
        Transform3["slice.transform(SplitRecursively())"]
        DataSlice4["DataSlice (chunked KTable)"]
        Transform4["slice.unnest()"]
        DataSlice5["DataSlice (individual chunks)"]
    end
    
    subgraph "Rust Engine Execution"
        SourceReader["Source Reader"]
        ParseJsonExec["ParseJson Executor"]
        DetectProgLangExec["DetectProgrammingLanguage Executor"]
        SplitRecursivelyExec["SplitRecursively Executor"]
        UnnestExec["Unnest Executor"]
    end
    
    AddSource --> DataSlice1
    DataSlice1 --> Transform1
    Transform1 --> DataSlice2
    DataSlice2 --> Transform2
    Transform2 --> DataSlice3
    DataSlice3 --> Transform3
    Transform3 --> DataSlice4
    DataSlice4 --> Transform4
    Transform4 --> DataSlice5
    
    Transform1 -.executes.-> ParseJsonExec
    Transform2 -.executes.-> DetectProgLangExec
    Transform3 -.executes.-> SplitRecursivelyExec
    Transform4 -.executes.-> UnnestExec
    
    SourceReader --> ParseJsonExec
    ParseJsonExec --> DetectProgLangExec
    DetectProgLangExec --> SplitRecursivelyExec
    SplitRecursivelyExec --> UnnestExec
```

### Complete Pipeline Example

```python
from cocoindex import flow_def
from cocoindex.sources import LocalFile
from cocoindex.functions import (
    DetectProgrammingLanguage,
    SplitRecursively,
)

@flow_def
def code_indexing_flow(builder):
    # 1. Read code files from local directory
    code_files = builder.add_source(
        LocalFile(path="./src/**/*.py"),
        fields={"path": "path", "content": "content"}
    )
    
    # 2. Detect programming language from file path
    with_language = code_files.transform(
        DetectProgrammingLanguage(),
        filename="path"
    )
    
    # 3. Split files into chunks using language-aware strategy
    chunked = with_language.transform(
        SplitRecursively(),
        text="content",
        chunk_size=1500,
        min_chunk_size=750,
        chunk_overlap=100,
        language="detected_lang"
    )
    
    # 4. Unnest KTable to get individual chunks
    chunks = chunked.unnest("chunks")
    
    # 5. Access chunk fields for downstream processing
    chunk_texts = chunks.select(field="chunks.text")
    chunk_locations = chunks.select(field="chunks.start.line")
    
    # 6. Continue with embedding, export, etc.
    # (See page 6.2 for embedding functions)
```

Sources: [docs/docs/ops/functions.md:31-132](), [python/cocoindex/functions/_engine_builtin_specs.py:1-49]()

## Chunking Strategy Selection Guide

| Use Case | Recommended Function | Configuration |
|----------|---------------------|---------------|
| **Source code files** | `SplitRecursively` | Set `language` from `DetectProgrammingLanguage` |
| **Markdown documentation** | `SplitRecursively` | `language="markdown"`, `chunk_size=1000-2000` |
| **Structured articles** | `SplitRecursively` | `language=None` (plain text mode) |
| **Simple paragraph splitting** | `SplitBySeparators` | `separators_regex=[r"\n\n+"]` |
| **Custom format** | `SplitRecursively` + `CustomLanguageSpec` | Define separators hierarchy |
| **Fixed-pattern splitting** | `SplitBySeparators` | Explicit regex patterns |

### Chunk Size Recommendations

| Content Type | Recommended `chunk_size` | Reasoning |
|--------------|-------------------------|-----------|
| Code | 1500-3000 bytes | Preserve function/class boundaries |
| Markdown | 1000-2000 bytes | Preserve section coherence |
| Plain text | 500-1000 bytes | Maintain semantic units (paragraphs) |
| Technical docs | 1500-2500 bytes | Balance context with granularity |

**Best Practice**: Set `min_chunk_size` to 40-60% of `chunk_size` to give the algorithm flexibility in finding optimal boundaries.

Sources: [docs/docs/ops/functions.md:54-60]()

## Implementation Notes

### Type System Integration

All text processing functions integrate with CocoIndex's type system through the operation registration mechanism:

1. Python `FunctionSpec` classes define configuration schema
2. Functions are registered with the Rust engine via `_register_op_factory` 
3. Input/output types are analyzed by `analyze_type_info`
4. Runtime values are encoded/decoded across the FFI boundary

See [Operation Registration and Decorators](#4.1) for registration details and [Type System and Schema Analysis](#4.2) for type handling.

### Batching and Caching

Text processing functions support:
- **Automatic batching**: Multiple text inputs processed together for efficiency
- **Result memoization**: Cached based on input content fingerprints (see [Incremental Processing and State Management](#9))
- **GPU acceleration**: Not applicable (CPU-only operations)

Configuration via `OpArgs` (see [OpArgs and Execution Options](#4.4)):
```python
# Example: disable caching for debugging
chunks = data.transform(
    SplitRecursively(),
    text="content",
    chunk_size=1000,
    op_args={"cache": False}
)
```

Sources: [docs/docs/ops/functions.md:1-132]()

---

# Page: Embedding and AI Functions

# Embedding and AI Functions

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [.cargo/config.toml](.cargo/config.toml)
- [docs/docs/ops/functions.md](docs/docs/ops/functions.md)
- [python/cocoindex/functions/_engine_builtin_specs.py](python/cocoindex/functions/_engine_builtin_specs.py)
- [rust/cocoindex/src/llm/litellm.rs](rust/cocoindex/src/llm/litellm.rs)
- [rust/cocoindex/src/llm/openrouter.rs](rust/cocoindex/src/llm/openrouter.rs)
- [rust/cocoindex/src/llm/vllm.rs](rust/cocoindex/src/llm/vllm.rs)
- [rust/cocoindex/src/ops/functions/embed_text.rs](rust/cocoindex/src/ops/functions/embed_text.rs)

</details>



This page documents the built-in functions in CocoIndex that leverage AI models for embedding generation and structured data extraction. These functions enable semantic search, multimodal retrieval, and intelligent data processing using large language models.

For text chunking and preprocessing functions, see [Text Processing and Chunking](#6.1). For LLM provider configuration and API integration details, see [LLM Integration](#7).

## Overview

CocoIndex provides three categories of AI-powered functions:

| Category | Functions | Primary Use Case |
|----------|-----------|-----------------|
| **Local Embedding** | `SentenceTransformerEmbed` | Text embedding using local transformer models |
| **API-based Embedding** | `EmbedText` | Text embedding via LLM APIs (OpenAI, Gemini, etc.) |
| **Structured Extraction** | `ExtractByLlm` | Extract structured data from unstructured text |
| **Multimodal Retrieval** | `ColPaliEmbedImage`, `ColPaliEmbedQuery` | Document and query embedding for visual search |

```mermaid
graph TB
    subgraph "Input Data Types"
        TextIn["Text Input"]
        ImageIn["Image Input<br/>(bytes)"]
    end
    
    subgraph "Embedding Functions"
        SentenceT["SentenceTransformerEmbed<br/>Local Models"]
        EmbedT["EmbedText<br/>10 LLM APIs"]
        ColPaliImg["ColPaliEmbedImage<br/>Multimodal Models"]
        ColPaliQ["ColPaliEmbedQuery<br/>Multimodal Models"]
    end
    
    subgraph "Extraction Functions"
        ExtractLlm["ExtractByLlm<br/>Structured Output"]
    end
    
    subgraph "Output Types"
        VectorOut["Vector[Float32, N]<br/>Fixed dimension"]
        MultiVectorOut["Vector[Vector[Float32, N]]<br/>Multi-vector"]
        StructOut["Dataclass/Struct<br/>Typed schema"]
    end
    
    TextIn --> SentenceT
    TextIn --> EmbedT
    TextIn --> ColPaliQ
    TextIn --> ExtractLlm
    ImageIn --> ColPaliImg
    
    SentenceT --> VectorOut
    EmbedT --> VectorOut
    ColPaliImg --> MultiVectorOut
    ColPaliQ --> MultiVectorOut
    ExtractLlm --> StructOut
```

**Sources:** [docs/docs/ops/functions.md:1-253]()

## SentenceTransformerEmbed

`SentenceTransformerEmbed` embeds text using the [SentenceTransformer](https://huggingface.co/sentence-transformers) library, which runs models locally on your machine or GPU. This is ideal for development, privacy-sensitive use cases, or when you want full control over the embedding model.

### Specification

The function spec is defined as `cocoindex.functions.SentenceTransformerEmbed` and takes the following fields:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `model` | `str` | Yes | HuggingFace model identifier (e.g., `"sentence-transformers/all-MiniLM-L6-v2"`) |
| `args` | `dict[str, Any]` | No | Additional arguments passed to SentenceTransformer constructor (e.g., `{"trust_remote_code": True}`) |

### Input and Output

**Input data:**
- `text` (*Str*): The text to embed

**Return:** *Vector[Float32, N]*, where *N* is the output dimension determined by the model (e.g., 384 for `all-MiniLM-L6-v2`, 768 for `all-mpnet-base-v2`)

### Dependency Installation

This function requires an optional dependency:

```sh
pip install 'cocoindex[embeddings]'
```

This installs the `sentence-transformers` library along with its dependencies.

### Usage Example

```python
chunk["embedding"] = chunk["text"].transform(
    cocoindex.functions.SentenceTransformerEmbed(
        model="sentence-transformers/all-MiniLM-L6-v2"
    )
)
```

For GPU acceleration or custom model loading:

```python
chunk["embedding"] = chunk["text"].transform(
    cocoindex.functions.SentenceTransformerEmbed(
        model="sentence-transformers/all-mpnet-base-v2",
        args={"device": "cuda", "trust_remote_code": True}
    )
)
```

**Model Selection Guidance:**

| Model | Dimensions | Speed | Quality | Use Case |
|-------|-----------|-------|---------|----------|
| `all-MiniLM-L6-v2` | 384 | Fast | Good | General purpose, resource-constrained |
| `all-mpnet-base-v2` | 768 | Medium | Better | Balanced quality/performance |
| `all-MiniLM-L12-v2` | 384 | Medium | Better | Better quality at 384-dim |

**Sources:** [docs/docs/ops/functions.md:125-149](), [docs/docs/getting_started/quickstart.md:93-118](), [examples/pdf_elements_embedding/README.md:1-72]()

## EmbedText

`EmbedText` embeds text using various LLM APIs that support text embedding. Unlike `SentenceTransformerEmbed`, this function calls external APIs, making it suitable for production deployments where you want to leverage managed embedding services.

### Specification

The function spec is defined as `cocoindex.functions.EmbedText` and takes the following fields:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `api_type` | `cocoindex.LlmApiType` | Yes | The LLM API provider (see supported types below) |
| `model` | `str` | Yes | The model identifier specific to the API provider |
| `address` | `str` | No | Custom API endpoint address (uses default if not specified) |
| `output_dimension` | `int` | No | Expected output dimension; required for unknown models |
| `task_type` | `str` | No | Task-specific optimization hint (provider-dependent) |

### Supported API Providers

The following table shows which LLM API types support text embedding:

| API Provider | `LlmApiType` Enum | Supports Embedding | Environment Variable |
|--------------|-------------------|-------------------|---------------------|
| OpenAI | `OPENAI` | ✅ | `OPENAI_API_KEY` |
| Ollama | `OLLAMA` | ✅ | None (local) |
| Google Gemini | `GEMINI` | ✅ | `GEMINI_API_KEY` |
| Vertex AI | `VERTEX_AI` | ✅ | ADC credentials |
| Voyage | `VOYAGE` | ✅ | `VOYAGE_API_KEY` |
| Anthropic | `ANTHROPIC` | ❌ | N/A |
| LiteLLM | `LITE_LLM` | ❌ | N/A |
| OpenRouter | `OPEN_ROUTER` | ❌ | N/A |
| vLLM | `VLLM` | ❌ | N/A |
| Bedrock | `BEDROCK` | ❌ | N/A |

### Input and Output

**Input data:**
- `text` (*Str*): The text to embed

**Return:** *Vector[Float32, N]*, where *N* is the dimension determined by the model

```mermaid
graph LR
    TextInput["text: Str"]
    
    subgraph "EmbedText Function"
        APISelect["Select API<br/>LlmApiType"]
        ModelSelect["Select Model<br/>model: str"]
        APICall["Call External API<br/>address (optional)"]
    end
    
    VectorOut["Vector[Float32, N]"]
    
    TextInput --> APISelect
    APISelect --> ModelSelect
    ModelSelect --> APICall
    APICall --> VectorOut
    
    EnvVars["Environment Variables<br/>API Keys"]
    EnvVars -.-> APICall
```

### Usage Examples

**OpenAI API:**

```python
chunk["embedding"] = chunk["text"].transform(
    cocoindex.functions.EmbedText(
        api_type=cocoindex.LlmApiType.OPENAI,
        model="text-embedding-3-small",
        output_dimension=1536,
    )
)
```

**Ollama (local):**

```python
chunk["embedding"] = chunk["text"].transform(
    cocoindex.functions.EmbedText(
        api_type=cocoindex.LlmApiType.OLLAMA,
        model="nomic-embed-text",
        address="http://localhost:11434",
    )
)
```

**Google Gemini with task type:**

```python
chunk["embedding"] = chunk["text"].transform(
    cocoindex.functions.EmbedText(
        api_type=cocoindex.LlmApiType.GEMINI,
        model="text-embedding-004",
        task_type="SEMANTICS_SIMILARITY",
        output_dimension=1536,
    )
)
```

**Voyage AI (specialized for code):**

```python
chunk["embedding"] = chunk["text"].transform(
    cocoindex.functions.EmbedText(
        api_type=cocoindex.LlmApiType.VOYAGE,
        model="voyage-code-3",
        task_type="document",
    )
)
```

### Task Type Usage

Some providers support `task_type` to optimize embeddings for specific use cases:

| Provider | Supported Task Types | Documentation |
|----------|---------------------|---------------|
| Gemini | `SEMANTICS_SIMILARITY`, `RETRIEVAL_QUERY`, `RETRIEVAL_DOCUMENT`, etc. | [Gemini Task Types](https://ai.google.dev/gemini-api/docs/embeddings#supported-task-types) |
| Vertex AI | Same as Gemini | [Vertex AI Parameters](https://cloud.google.com/vertex-ai/generative-ai/docs/model-reference/text-embeddings-api#parameter-list) |
| Voyage | `document`, `query` | [Voyage API Docs](https://docs.voyageai.com/reference/embeddings-api) |

**Sources:** [docs/docs/ops/functions.md:175-202](), [docs/docs/ai/llm.mdx:14-31](), [docs/docs/ai/llm.mdx:56-69]()

## ExtractByLlm

`ExtractByLlm` extracts structured information from unstructured text using large language models. This function is essential for data extraction pipelines where you need to convert free-form text into typed, queryable data structures.

### Specification

The function spec is defined as `cocoindex.functions.ExtractByLlm` and takes the following fields:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `llm_spec` | `cocoindex.LlmSpec` | Yes | LLM provider configuration (see [LLM Integration](#7)) |
| `output_type` | `type` | Yes | Python type (dataclass) defining the output schema |
| `instruction` | `str` | No | Additional context or instructions for the LLM |

### LlmSpec Configuration

The `llm_spec` field uses `cocoindex.LlmSpec` dataclass defined in [python/cocoindex/llm.py:40-48]():

```python
@dataclass
class LlmSpec:
    api_type: LlmApiType           # Required: API provider
    model: str                      # Required: Model identifier
    address: str | None = None      # Optional: Custom endpoint
    api_config: VertexAiConfig | OpenAiConfig | None = None  # Optional: Provider-specific config
```

Supported `api_type` values for text generation:

```mermaid
graph TB
    LlmSpec["LlmSpec Configuration"]
    
    subgraph "Text Generation APIs"
        OpenAI["OPENAI<br/>gpt-4o, gpt-4o-mini"]
        Ollama["OLLAMA<br/>llama3.2, etc."]
        Gemini["GEMINI<br/>gemini-2.0-flash"]
        VertexAI["VERTEX_AI<br/>gemini-2.0-flash"]
        Anthropic["ANTHROPIC<br/>claude-3-5-sonnet"]
        LiteLLM["LITE_LLM<br/>deepseek-chat"]
        OpenRouter["OPEN_ROUTER<br/>deepseek/deepseek-r1"]
        VLLM["VLLM<br/>custom models"]
        Bedrock["BEDROCK<br/>claude-3-5-haiku"]
    end
    
    LlmSpec --> OpenAI
    LlmSpec --> Ollama
    LlmSpec --> Gemini
    LlmSpec --> VertexAI
    LlmSpec --> Anthropic
    LlmSpec --> LiteLLM
    LlmSpec --> OpenRouter
    LlmSpec --> VLLM
    LlmSpec --> Bedrock
```

### Input and Output

**Input data:**
- `text` (*Str*): The unstructured text to extract information from

**Return:** The type specified by `output_type`. The LLM generates structured output matching the schema of the output type.

### Output Type Definition

The `output_type` must be a valid CocoIndex type (see [Type System and Schema Analysis](#4.2)). Typically, you define a Python dataclass:

```python
import dataclasses

@dataclasses.dataclass
class ModuleInfo:
    """Information about a Python module."""
    title: str
    description: str
    classes: list[ClassInfo]
    methods: list[MethodInfo]

@dataclasses.dataclass
class ClassInfo:
    """Information about a class."""
    name: str
    description: str
    methods: list[MethodInfo]
```

**Best practices for output types:**

1. **Use clear field names:** The LLM uses field names as hints
2. **Add docstrings:** Class and field docstrings guide the LLM
3. **Mark optional fields:** Use `SomeType | None` or `typing.Optional[SomeType]` for fields that may not always be present
4. **Use nested structures:** Complex hierarchies are fully supported
5. **Use lists for collections:** `list[T]` for variable-length collections

### Usage Examples

**Basic extraction with Ollama:**

```python
doc["module_info"] = doc["markdown"].transform(
    cocoindex.functions.ExtractByLlm(
        llm_spec=cocoindex.LlmSpec(
            api_type=cocoindex.LlmApiType.OLLAMA,
            model="llama3.2",
        ),
        output_type=ModuleInfo,
        instruction="Please extract Python module information from the manual.",
    )
)
```

**Extraction with OpenAI:**

```python
doc["module_info"] = doc["markdown"].transform(
    cocoindex.functions.ExtractByLlm(
        llm_spec=cocoindex.LlmSpec(
            api_type=cocoindex.LlmApiType.OPENAI,
            model="gpt-4o"
        ),
        output_type=ModuleInfo,
    )
)
```

**Extraction with Gemini:**

```python
doc["patient_info"] = doc["form_text"].transform(
    cocoindex.functions.ExtractByLlm(
        llm_spec=cocoindex.LlmSpec(
            api_type=cocoindex.LlmApiType.GEMINI,
            model="gemini-2.0-flash"
        ),
        output_type=PatientIntake,
        instruction="Extract all patient information, marking optional fields as None if not present.",
    )
)
```

**Extraction with Anthropic:**

```python
doc["topics"] = doc["text"].transform(
    cocoindex.functions.ExtractByLlm(
        llm_spec=cocoindex.LlmSpec(
            api_type=cocoindex.LlmApiType.ANTHROPIC,
            model="claude-3-5-sonnet-latest"
        ),
        output_type=TopicList,
    )
)
```

### Data Flow

```mermaid
graph TB
    UnstructuredText["Unstructured Text<br/>Markdown, PDF, Forms"]
    
    subgraph "ExtractByLlm Processing"
        Schema["output_type Schema<br/>Dataclass Definition"]
        LLM["LLM API Call<br/>llm_spec configuration"]
        Validation["Schema Validation<br/>Type checking"]
    end
    
    StructuredData["Structured Output<br/>Typed Dataclass Instance"]
    
    UnstructuredText --> LLM
    Schema -.->|"Schema passed as context"| LLM
    LLM --> Validation
    Validation --> StructuredData
    
    Instruction["instruction field<br/>(optional)"]
    Instruction -.->|"Additional guidance"| LLM
```

**Sources:** [docs/docs/ops/functions.md:150-174](), [examples/manuals_llm_extraction/main.py:1-140](), [docs/docs/ai/llm.mdx:34-53](), [python/cocoindex/llm.py:40-48]()

## ColPali Functions

ColPali functions enable multimodal document retrieval using ColVision models from the [colpali-engine library](https://github.com/illuin-tech/colpali). These models use late interaction between image patch embeddings and text token embeddings for retrieval, making them ideal for visual document search without OCR.

### Supported Models

All models from the colpali-engine library are supported:

| Model Family | Example Models | Best For |
|--------------|---------------|----------|
| **ColPali** | `vidore/colpali-v1.2` | General document retrieval (PaliGemma-based) |
| **ColQwen2** | `vidore/colqwen2.5-v0.2` | Multilingual (29+ languages), general vision (Qwen2-VL-based) |
| **ColSmol** | `vidore/colsmol-v1.0` | Resource-constrained environments (lightweight) |

See the [complete model list](https://github.com/illuin-tech/colpali#list-of-colvision-models) for all available models.

### Dependency Installation

ColPali functions require an optional dependency:

```sh
pip install 'cocoindex[colpali]'
```

### ColPaliEmbedImage

Embeds images into multi-vector representations for visual document search.

**Specification:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `model` | `str` | Yes | ColVision model identifier (e.g., `"vidore/colpali-v1.2"`) |

**Input data:**
- `img_bytes` (*Bytes*): The image data in bytes format

**Return:** *Vector[Vector[Float32, N]]*, where *N* is the hidden dimension determined by the model. This is a multi-vector format with variable number of patches and fixed hidden dimension per patch.

**Usage example:**

```python
with doc["images"].row() as img:
    img["embedding"] = img["img_bytes"].transform(
        cocoindex.functions.ColPaliEmbedImage(
            model="vidore/colpali-v1.2"
        )
    )
```

### ColPaliEmbedQuery

Embeds text queries into multi-vector representations compatible with ColVision image embeddings for late interaction scoring (MaxSim).

**Specification:**

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `model` | `str` | Yes | ColVision model identifier (must match the image embedding model) |

**Input data:**
- `query` (*Str*): The text query to embed

**Return:** *Vector[Vector[Float32, N]]*, where *N* is the hidden dimension determined by the model. This is a multi-vector format with variable number of tokens and fixed hidden dimension per token.

**Usage example:**

```python
# At query time (not in the indexing flow)
query_embedding = cocoindex.functions.ColPaliEmbedQuery(
    model="vidore/colpali-v1.2"
).execute(query_text)
```

### Multi-Vector Representation

Unlike traditional single-vector embeddings, ColPali uses multi-vector representations:

```mermaid
graph TB
    subgraph "Traditional Embedding"
        SingleVecImg["Image"] --> SingleEmbed["Single Vector<br/>[Float32 × 768]"]
        SingleVecQuery["Query Text"] --> SingleQueryEmbed["Single Vector<br/>[Float32 × 768]"]
    end
    
    subgraph "ColPali Multi-Vector"
        MultiVecImg["Image<br/>512×512 px"] --> Patches["Image Patches<br/>Variable count"]
        Patches --> MultiImgEmbed["Multi-Vector<br/>[[Float32 × N] × M patches]"]
        
        MultiVecQuery["Query Text"] --> Tokens["Query Tokens<br/>Variable count"]
        Tokens --> MultiQueryEmbed["Multi-Vector<br/>[[Float32 × N] × K tokens]"]
    end
    
    subgraph "Retrieval"
        MaxSim["MaxSim Late Interaction<br/>max_i(sum_j(q_i · d_j))"]
    end
    
    SingleEmbed --> Cosine["Cosine Similarity"]
    SingleQueryEmbed --> Cosine
    
    MultiImgEmbed --> MaxSim
    MultiQueryEmbed --> MaxSim
```

**Late interaction (MaxSim):** For each query token, find the maximum similarity across all document patches, then sum these maxima. This allows fine-grained matching between query terms and image regions.

### End-to-End Example

```python
@cocoindex.flow_def(name="VisualDocumentSearch")
def visual_search_flow(
    flow_builder: cocoindex.FlowBuilder, 
    data_scope: cocoindex.DataScope
):
    # Read PDF files
    data_scope["documents"] = flow_builder.add_source(
        cocoindex.sources.LocalFile(path="pdfs", binary=True)
    )
    
    doc_images = data_scope.add_collector()
    
    with data_scope["documents"].row() as doc:
        # Extract images from PDF pages
        doc["pages"] = doc["content"].transform(ExtractPdfPages())
        
        with doc["pages"].row() as page:
            page["images"] = page["content"].transform(ExtractImages())
            
            with page["images"].row() as img:
                # Embed image with ColPali
                img["embedding"] = img["img_bytes"].transform(
                    cocoindex.functions.ColPaliEmbedImage(
                        model="vidore/colpali-v1.2"
                    )
                )
                
                doc_images.collect(
                    filename=doc["filename"],
                    page_num=page["page_num"],
                    embedding=img["embedding"]
                )
    
    # Export to vector database with multi-vector support
    doc_images.export(
        "visual_docs",
        cocoindex.targets.Qdrant(),
        primary_key_fields=["filename", "page_num"],
        vector_indexes=[
            cocoindex.VectorIndexDef(
                field_name="embedding",
                metric=cocoindex.VectorSimilarityMetric.COSINE_SIMILARITY
            )
        ]
    )
```

**Sources:** [docs/docs/ops/functions.md:203-253](), [README.md:201-201]()

## Function Selection Guide

Choose the appropriate embedding or AI function based on your requirements:

### When to Use SentenceTransformerEmbed

✅ **Use when:**
- You want local, offline embedding generation
- Privacy is critical (no data sent to external APIs)
- You have GPU resources available for acceleration
- You need consistent, reproducible embeddings
- Development and prototyping on sample data

❌ **Avoid when:**
- You need the latest state-of-art embedding models
- You want managed infrastructure and don't want to handle model updates
- Resource constraints prevent running local models

### When to Use EmbedText

✅ **Use when:**
- You want managed, production-grade embedding services
- You need access to the latest models (e.g., OpenAI's ada-002 successor)
- You prefer API-based pricing over infrastructure costs
- You want minimal local resource usage
- You need provider-specific features (e.g., Gemini task types, Voyage code embeddings)

❌ **Avoid when:**
- Privacy regulations prevent sending data to external APIs
- API costs are prohibitive for your scale
- You need offline operation

### When to Use ExtractByLlm

✅ **Use when:**
- Converting unstructured text to structured data
- You need typed, schema-validated outputs
- Processing documents like forms, invoices, or manuals
- Building knowledge graphs from text
- Extracting entities with complex relationships

❌ **Avoid when:**
- Simple pattern matching or regex would suffice
- You need high throughput with strict latency requirements (LLM calls have higher latency)
- Schema is unknown or highly variable

### When to Use ColPali Functions

✅ **Use when:**
- Searching visual documents (PDFs, scanned pages, diagrams)
- You want to avoid OCR preprocessing
- Documents contain significant non-textual information (tables, charts, layouts)
- Multilingual document search (ColQwen2 models)

❌ **Avoid when:**
- Documents are pure text without visual elements
- You already have high-quality text extraction pipelines
- Resource constraints prevent running vision models

### Performance Considerations

| Function | Latency | Throughput | GPU Required | Cost Model |
|----------|---------|------------|--------------|------------|
| `SentenceTransformerEmbed` | Low-Medium | High | Optional (3-10× speedup) | Infrastructure only |
| `EmbedText` | Medium | Medium-High | No | Per-token API pricing |
| `ExtractByLlm` | High | Low-Medium | No | Per-token API pricing |
| `ColPaliEmbedImage` | Medium-High | Low-Medium | Recommended | Infrastructure only |
| `ColPaliEmbedQuery` | Low | High | Optional | Infrastructure only |

**Batching optimization:** All functions support batching through `OpArgs` configuration (see [OpArgs and Execution Options](#4.4)), which significantly improves throughput for API-based functions by reducing per-request overhead.

```python
# Enable batching for better throughput
chunk["embedding"] = chunk["text"].transform(
    cocoindex.functions.EmbedText(
        api_type=cocoindex.LlmApiType.OPENAI,
        model="text-embedding-3-small"
    ),
    # OpArgs configuration
    batching=cocoindex.BatchingSpec(max_batch_size=100)
)
```

**Sources:** [docs/docs/ops/functions.md:1-253](), [docs/docs/ai/llm.mdx:1-469](), [examples/manuals_llm_extraction/main.py:1-140]()

---

# Page: LLM Integration

# LLM Integration

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [.cargo/config.toml](.cargo/config.toml)
- [docs/docs/ops/functions.md](docs/docs/ops/functions.md)
- [python/cocoindex/functions/_engine_builtin_specs.py](python/cocoindex/functions/_engine_builtin_specs.py)
- [rust/cocoindex/src/llm/litellm.rs](rust/cocoindex/src/llm/litellm.rs)
- [rust/cocoindex/src/llm/openrouter.rs](rust/cocoindex/src/llm/openrouter.rs)
- [rust/cocoindex/src/llm/vllm.rs](rust/cocoindex/src/llm/vllm.rs)
- [rust/cocoindex/src/ops/functions/embed_text.rs](rust/cocoindex/src/ops/functions/embed_text.rs)

</details>



This document provides comprehensive coverage of Large Language Model (LLM) integration in CocoIndex. It details the `LlmSpec` configuration system, the `LlmApiType` enumeration, and configuration instructions for all 10 supported LLM providers.

CocoIndex uses LLMs for two primary tasks: **text generation** (for structured extraction) and **text embedding** (for semantic search). The framework provides a unified interface across multiple providers, allowing you to switch between OpenAI, Ollama, Gemini, and others with minimal configuration changes.

For information about the built-in functions that consume these LLM specifications (such as `ExtractByLlm` and `EmbedText`), see [Built-in Processing Functions](#6) and [Embedding and AI Functions](#6.2).

---

## LLM Integration Architecture

CocoIndex's LLM integration operates at the boundary between the Python configuration layer and the Rust execution engine. Users configure LLM providers through Python dataclasses, which are serialized and passed to the Rust engine for actual API calls.

**LLM Integration Flow**

```mermaid
graph TB
    subgraph "Python Layer"
        UserCode["User Flow Definition<br/>@flow_def"]
        LlmSpec["LlmSpec Configuration<br/>python/cocoindex/llm.py:41-48"]
        LlmApiType["LlmApiType Enum<br/>python/cocoindex/llm.py:5-18"]
        Functions["Built-in Functions<br/>ExtractByLlm, EmbedText"]
    end
    
    subgraph "Serialization"
        TypeMarshalling["Type Marshalling System<br/>Encode to Rust"]
    end
    
    subgraph "Rust Engine"
        LlmExecutor["LLM Executor<br/>Function Implementation"]
        ApiClients["HTTP Clients<br/>OpenAI, Ollama, etc."]
        RetryMechanism["Retry & Error Handling<br/>cocoindex_utils"]
    end
    
    subgraph "External Services"
        OpenAI["OpenAI API"]
        Ollama["Ollama Local"]
        Gemini["Google Gemini"]
        VertexAI["Vertex AI"]
        Anthropic["Anthropic API"]
        Voyage["Voyage API"]
        LiteLLM["LiteLLM Proxy"]
        OpenRouter["OpenRouter"]
        vLLM["vLLM Server"]
        Bedrock["AWS Bedrock"]
    end
    
    UserCode --> LlmSpec
    LlmSpec --> LlmApiType
    LlmSpec --> Functions
    Functions --> TypeMarshalling
    TypeMarshalling --> LlmExecutor
    LlmExecutor --> ApiClients
    ApiClients --> RetryMechanism
    
    RetryMechanism --> OpenAI
    RetryMechanism --> Ollama
    RetryMechanism --> Gemini
    RetryMechanism --> VertexAI
    RetryMechanism --> Anthropic
    RetryMechanism --> Voyage
    RetryMechanism --> LiteLLM
    RetryMechanism --> OpenRouter
    RetryMechanism --> vLLM
    RetryMechanism --> Bedrock
```

**Sources:** [python/cocoindex/llm.py:1-48](), [docs/docs/ai/llm.mdx:1-469]()

---

## LlmSpec Configuration

The `LlmSpec` dataclass is the primary configuration interface for LLM-powered functions. It is defined in [python/cocoindex/llm.py:41-48]() and has the following structure:

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `api_type` | `LlmApiType` | Yes | The LLM provider type (e.g., `OPENAI`, `OLLAMA`) |
| `model` | `str` | Yes | The model identifier (e.g., `"gpt-4o"`, `"llama3.2"`) |
| `address` | `str \| None` | No | Custom API endpoint URL (overrides provider defaults) |
| `api_config` | `VertexAiConfig \| OpenAiConfig \| None` | No | Provider-specific configuration |

### Basic Usage Pattern

```python
# OpenAI example
spec = cocoindex.LlmSpec(
    api_type=cocoindex.LlmApiType.OPENAI,
    model="gpt-4o"
)

# Ollama with custom address
spec = cocoindex.LlmSpec(
    api_type=cocoindex.LlmApiType.OLLAMA,
    model="llama3.2:latest",
    address="http://localhost:11434"
)

# Vertex AI with api_config
spec = cocoindex.LlmSpec(
    api_type=cocoindex.LlmApiType.VERTEX_AI,
    model="gemini-2.0-flash",
    api_config=cocoindex.llm.VertexAiConfig(project="my-project-id")
)
```

The `LlmSpec` is passed to functions like `ExtractByLlm` (for text generation) or to the embedding configuration in `EmbedText` (for text embedding).

**Sources:** [python/cocoindex/llm.py:41-48](), [docs/docs/ai/llm.mdx:43-53]()

---

## LlmApiType Enumeration

The `LlmApiType` enum defines all supported LLM providers. It is defined in [python/cocoindex/llm.py:5-18]():

```python
class LlmApiType(Enum):
    OPENAI = "OpenAi"
    OLLAMA = "Ollama"
    GEMINI = "Gemini"
    VERTEX_AI = "VertexAi"
    ANTHROPIC = "Anthropic"
    LITE_LLM = "LiteLlm"
    OPEN_ROUTER = "OpenRouter"
    VOYAGE = "Voyage"
    VLLM = "Vllm"
    BEDROCK = "Bedrock"
```

**Provider Capabilities Matrix**

```mermaid
graph LR
    subgraph "Text Generation Providers"
        GenOpenAI["OPENAI"]
        GenOllama["OLLAMA"]
        GenGemini["GEMINI"]
        GenVertexAI["VERTEX_AI"]
        GenAnthropic["ANTHROPIC"]
        GenLiteLLM["LITE_LLM"]
        GenOpenRouter["OPEN_ROUTER"]
        GenVLLM["VLLM"]
        GenBedrock["BEDROCK"]
    end
    
    subgraph "Text Embedding Providers"
        EmbedOpenAI["OPENAI"]
        EmbedOllama["OLLAMA"]
        EmbedGemini["GEMINI"]
        EmbedVertexAI["VERTEX_AI"]
        EmbedVoyage["VOYAGE"]
    end
    
    subgraph "Functions"
        ExtractByLlm["ExtractByLlm<br/>Structured Extraction"]
        EmbedText["EmbedText<br/>Vector Embeddings"]
    end
    
    GenOpenAI --> ExtractByLlm
    GenOllama --> ExtractByLlm
    GenGemini --> ExtractByLlm
    GenVertexAI --> ExtractByLlm
    GenAnthropic --> ExtractByLlm
    GenLiteLLM --> ExtractByLlm
    GenOpenRouter --> ExtractByLlm
    GenVLLM --> ExtractByLlm
    GenBedrock --> ExtractByLlm
    
    EmbedOpenAI --> EmbedText
    EmbedOllama --> EmbedText
    EmbedGemini --> EmbedText
    EmbedVertexAI --> EmbedText
    EmbedVoyage --> EmbedText
```

**Sources:** [python/cocoindex/llm.py:5-18](), [docs/docs/ai/llm.mdx:18-31]()

---

## Text Generation vs Text Embedding

CocoIndex supports two distinct LLM task types, each with different provider support:

### Text Generation

Text generation is used by `ExtractByLlm` to extract structured information from unstructured text. The LLM generates text output according to a schema defined by Python dataclasses.

**Supported Providers:** `OPENAI`, `OLLAMA`, `GEMINI`, `VERTEX_AI`, `ANTHROPIC`, `LITE_LLM`, `OPEN_ROUTER`, `VLLM`, `BEDROCK`

**Configuration Example:**
```python
llm_spec = cocoindex.LlmSpec(
    api_type=cocoindex.LlmApiType.OPENAI,
    model="gpt-4o"
)

doc["extracted"] = doc["text"].transform(
    cocoindex.functions.ExtractByLlm(
        llm_spec=llm_spec,
        output_type=MyDataClass,
        instruction="Extract relevant information"
    )
)
```

### Text Embedding

Text embedding converts text into vector representations for semantic similarity search. Configuration is provided directly to the `EmbedText` function rather than through `LlmSpec`.

**Supported Providers:** `OPENAI`, `OLLAMA`, `GEMINI`, `VERTEX_AI`, `VOYAGE`

**Configuration Example:**
```python
doc["embedding"] = doc["text"].transform(
    cocoindex.functions.EmbedText(
        api_type=cocoindex.LlmApiType.OPENAI,
        model="text-embedding-3-small",
        output_dimension=1536
    )
)
```

**Sources:** [docs/docs/ai/llm.mdx:33-68](), [examples/manuals_llm_extraction/main.py:105-127]()

---

## Provider Configuration Guide

### OpenAI

**Supported Tasks:** Text Generation ✅ | Text Embedding ✅

**Environment Variables:**
- `OPENAI_API_KEY` (required) - Your OpenAI API key from [platform.openai.com/api-keys](https://platform.openai.com/api-keys)
- `OPENAI_API_BASE` (optional) - Custom API endpoint

**Provider-Specific Config:**
The `api_config` field accepts `OpenAiConfig` from [python/cocoindex/llm.py:30-38]():

| Field | Type | Description |
|-------|------|-------------|
| `org_id` | `str \| None` | OpenAI organization ID |
| `project_id` | `str \| None` | OpenAI project ID |

**Text Generation Example:**
```python
cocoindex.LlmSpec(
    api_type=cocoindex.LlmApiType.OPENAI,
    model="gpt-4o",
    api_config=cocoindex.llm.OpenAiConfig(
        org_id="org-xxx",
        project_id="proj-xxx"
    )
)
```

**Text Embedding Example:**
```python
cocoindex.functions.EmbedText(
    api_type=cocoindex.LlmApiType.OPENAI,
    model="text-embedding-3-small",
    output_dimension=1536
)
```

**Sources:** [docs/docs/ai/llm.mdx:74-117](), [python/cocoindex/llm.py:30-38]()

---

### Ollama

**Supported Tasks:** Text Generation ✅ | Text Embedding ✅

**Setup Requirements:**
1. Download and install Ollama from [ollama.com/download](https://ollama.com/download)
2. Pull models: `ollama pull llama3.2`
3. For embedding: `ollama pull nomic-embed-text`

**Default Address:** `http://localhost:11434`

**Text Generation Example:**
```python
cocoindex.LlmSpec(
    api_type=cocoindex.LlmApiType.OLLAMA,
    model="llama3.2:latest",
    address="http://localhost:11434"  # optional
)
```

**Text Embedding Example:**
```python
cocoindex.functions.EmbedText(
    api_type=cocoindex.LlmApiType.OLLAMA,
    model="nomic-embed-text",
    address="http://localhost:11434"  # optional
)
```

**Sources:** [docs/docs/ai/llm.mdx:119-168](), [examples/manuals_llm_extraction/main.py:107-111]()

---

### Google Gemini

**Supported Tasks:** Text Generation ✅ | Text Embedding ✅

**Environment Variables:**
- `GEMINI_API_KEY` (required) - API key from [aistudio.google.com/apikey](https://aistudio.google.com/apikey)

**Note:** Gemini uses Google AI Studio APIs for prototyping. For production use, see [Vertex AI](#vertex-ai).

**Text Generation Example:**
```python
cocoindex.LlmSpec(
    api_type=cocoindex.LlmApiType.GEMINI,
    model="gemini-2.5-flash"
)
```

**Text Embedding Example:**
```python
cocoindex.functions.EmbedText(
    api_type=cocoindex.LlmApiType.GEMINI,
    model="text-embedding-004",
    task_type="SEMANTICS_SIMILARITY",  # optional
    output_dimension=1536  # optional
)
```

**Embedding Task Types:**
- `RETRIEVAL_QUERY`
- `RETRIEVAL_DOCUMENT`
- `SEMANTIC_SIMILARITY`
- `CLASSIFICATION`
- `CLUSTERING`
- `QUESTION_ANSWERING`
- `FACT_VERIFICATION`

**Sources:** [docs/docs/ai/llm.mdx:170-217](), [examples/manuals_llm_extraction/main.py:115-117]()

---

### Vertex AI

**Supported Tasks:** Text Generation ✅ | Text Embedding ✅

**Setup Requirements:**
1. Enable Vertex AI API in [Google Cloud Console](https://console.cloud.google.com/)
2. Enable billing for the project
3. Set up Application Default Credentials (ADC):
   ```bash
   gcloud auth application-default login
   ```

**Provider-Specific Config:**
The `api_config` field accepts `VertexAiConfig` from [python/cocoindex/llm.py:20-28]():

| Field | Type | Required | Description |
|-------|------|----------|-------------|
| `project` | `str` | Yes | Google Cloud project ID |
| `region` | `str \| None` | No | GCP region (defaults to `"global"`) |

**Text Generation Example:**
```python
cocoindex.LlmSpec(
    api_type=cocoindex.LlmApiType.VERTEX_AI,
    model="gemini-2.0-flash",
    api_config=cocoindex.llm.VertexAiConfig(
        project="your-project-id",
        region="us-central1"
    )
)
```

**Text Embedding Example:**
```python
cocoindex.functions.EmbedText(
    api_type=cocoindex.LlmApiType.VERTEX_AI,
    model="text-embedding-005",
    task_type="SEMANTICS_SIMILARITY",  # optional
    output_dimension=1536,  # optional
    api_config=cocoindex.llm.VertexAiConfig(project="your-project-id")
)
```

**Sources:** [docs/docs/ai/llm.mdx:219-281](), [python/cocoindex/llm.py:20-28]()

---

### Anthropic

**Supported Tasks:** Text Generation ✅ | Text Embedding ❌

**Environment Variables:**
- `ANTHROPIC_API_KEY` (required) - API key from [console.anthropic.com/settings/keys](https://console.anthropic.com/settings/keys)

**Text Generation Example:**
```python
cocoindex.LlmSpec(
    api_type=cocoindex.LlmApiType.ANTHROPIC,
    model="claude-3-5-sonnet-latest"
)
```

**Popular Models:**
- `claude-3-5-sonnet-latest`
- `claude-3-5-haiku-latest`
- `claude-3-opus-latest`

**Sources:** [docs/docs/ai/llm.mdx:282-302](), [examples/manuals_llm_extraction/main.py:118-120]()

---

### Voyage

**Supported Tasks:** Text Generation ❌ | Text Embedding ✅

**Environment Variables:**
- `VOYAGE_API_KEY` (required) - API key from [dashboard.voyageai.com/organization/api-keys](https://dashboard.voyageai.com/organization/api-keys)

**Text Embedding Example:**
```python
cocoindex.functions.EmbedText(
    api_type=cocoindex.LlmApiType.VOYAGE,
    model="voyage-code-3",
    task_type="document"  # or "query"
)
```

**Task Types:**
- `"document"` - For indexing documents
- `"query"` - For search queries

**Sources:** [docs/docs/ai/llm.mdx:304-326]()

---

### LiteLLM

**Supported Tasks:** Text Generation ✅ | Text Embedding ❌

**Setup Requirements:**
1. Install LiteLLM proxy: `pip install 'litellm[proxy]'`
2. Create `config.yml` with provider configuration
3. Run proxy: `litellm --config config.yml`

**Environment Variables:**
- `LITELLM_API_KEY` (required)
- Provider-specific keys (e.g., `DEEPSEEK_API_KEY`, `GROQ_API_KEY`)

**Configuration Flow:**

```mermaid
graph LR
    subgraph "Provider APIs"
        DeepSeek["DeepSeek API<br/>DEEPSEEK_API_KEY"]
        Groq["Groq API<br/>GROQ_API_KEY"]
        OtherLLM["Other LLM Providers"]
    end
    
    subgraph "LiteLLM Proxy"
        ConfigYML["config.yml<br/>Model Mappings"]
        ProxyServer["LiteLLM Server<br/>http://127.0.0.1:4000"]
    end
    
    subgraph "CocoIndex"
        LlmSpecLite["LlmSpec<br/>LITE_LLM type"]
        ExtractFunc["ExtractByLlm Function"]
    end
    
    DeepSeek --> ConfigYML
    Groq --> ConfigYML
    OtherLLM --> ConfigYML
    ConfigYML --> ProxyServer
    ProxyServer --> LlmSpecLite
    LlmSpecLite --> ExtractFunc
```

**Example config.yml:**
```yaml
model_list:
  - model_name: deepseek-chat
    litellm_params:
      model: deepseek/deepseek-chat
      api_key: os.environ/DEEPSEEK_API_KEY
```

**Text Generation Example:**
```python
cocoindex.LlmSpec(
    api_type=cocoindex.LlmApiType.LITE_LLM,
    model="deepseek-chat",
    address="http://127.0.0.1:4000"
)
```

**Sources:** [docs/docs/ai/llm.mdx:328-391]()

---

### OpenRouter

**Supported Tasks:** Text Generation ✅ | Text Embedding ❌

**Environment Variables:**
- `OPENROUTER_API_KEY` (required) - API key from [openrouter.ai/settings/keys](https://openrouter.ai/settings/keys)

**Text Generation Example:**
```python
cocoindex.LlmSpec(
    api_type=cocoindex.LlmApiType.OPEN_ROUTER,
    model="deepseek/deepseek-r1:free"
)
```

**Use Case:** Provides access to multiple LLM providers through a single API, with pay-per-use pricing.

**Sources:** [docs/docs/ai/llm.mdx:393-412]()

---

### vLLM

**Supported Tasks:** Text Generation ✅ | Text Embedding ❌

**Setup Requirements:**
1. Install vLLM: `pip install vllm`
2. Run vLLM server:
   ```bash
   vllm serve deepseek-ai/deepseek-coder-1.3b-instruct
   ```

**Default Address:** `http://127.0.0.1:8000/v1`

**Text Generation Example:**
```python
cocoindex.LlmSpec(
    api_type=cocoindex.LlmApiType.VLLM,
    model="deepseek-ai/deepseek-coder-1.3b-instruct",
    address="http://127.0.0.1:8000/v1"
)
```

**Use Case:** Self-hosted high-performance LLM inference server for production deployments.

**Sources:** [docs/docs/ai/llm.mdx:414-443]()

---

### Bedrock

**Supported Tasks:** Text Generation ✅ | Text Embedding ❌

**Environment Variables:**
- `AWS_ACCESS_KEY_ID` (required)
- `AWS_SECRET_ACCESS_KEY` (required)
- `AWS_SESSION_TOKEN` (optional)

**Text Generation Example:**
```python
cocoindex.LlmSpec(
    api_type=cocoindex.LlmApiType.BEDROCK,
    model="us.anthropic.claude-3-5-haiku-20241022-v1:0"
)
```

**Use Case:** Access AWS-managed LLM models (including Anthropic Claude, Amazon Titan, etc.) with enterprise features like VPC integration and CloudWatch logging.

**Sources:** [docs/docs/ai/llm.mdx:445-469](), [examples/manuals_llm_extraction/main.py:121-123]()

---

## Provider Summary Table

| Provider | API Type Enum | Generation | Embedding | Auth Method | Special Config |
|----------|---------------|------------|-----------|-------------|----------------|
| OpenAI | `OPENAI` | ✅ | ✅ | `OPENAI_API_KEY` | `OpenAiConfig` |
| Ollama | `OLLAMA` | ✅ | ✅ | None (local) | Custom address |
| Gemini | `GEMINI` | ✅ | ✅ | `GEMINI_API_KEY` | Task types |
| Vertex AI | `VERTEX_AI` | ✅ | ✅ | ADC credentials | `VertexAiConfig` (required) |
| Anthropic | `ANTHROPIC` | ✅ | ❌ | `ANTHROPIC_API_KEY` | None |
| Voyage | `VOYAGE` | ❌ | ✅ | `VOYAGE_API_KEY` | Task types |
| LiteLLM | `LITE_LLM` | ✅ | ❌ | `LITELLM_API_KEY` | config.yml |
| OpenRouter | `OPEN_ROUTER` | ✅ | ❌ | `OPENROUTER_API_KEY` | None |
| vLLM | `VLLM` | ✅ | ❌ | None (self-hosted) | Custom address |
| Bedrock | `BEDROCK` | ✅ | ❌ | AWS credentials | None |

**Sources:** [docs/docs/ai/llm.mdx:18-31](), [python/cocoindex/llm.py:5-18]()

---

## Authentication and Environment Variables

CocoIndex LLM integrations follow a consistent authentication pattern using environment variables. This design allows credentials to be managed separately from code and supports multiple deployment environments.

**Authentication Flow:**

```mermaid
graph TB
    subgraph "Configuration Phase"
        UserCode["User Flow Definition"]
        LlmSpec["LlmSpec Creation"]
        EnvVars["Environment Variables<br/>OPENAI_API_KEY, etc."]
    end
    
    subgraph "Execution Phase"
        RustEngine["Rust Engine"]
        EnvLookup["Environment Lookup"]
        ApiRequest["HTTP API Request"]
    end
    
    subgraph "External API"
        LLMProvider["LLM Provider<br/>OpenAI, Anthropic, etc."]
    end
    
    UserCode --> LlmSpec
    EnvVars -.->|read at runtime| EnvLookup
    LlmSpec --> RustEngine
    RustEngine --> EnvLookup
    EnvLookup --> ApiRequest
    ApiRequest -->|Authorization: Bearer| LLMProvider
```

**Environment Variable Conventions:**

| Provider | Required Variables | Optional Variables |
|----------|-------------------|-------------------|
| OpenAI | `OPENAI_API_KEY` | `OPENAI_API_BASE` |
| Ollama | None | None |
| Gemini | `GEMINI_API_KEY` | None |
| Vertex AI | ADC credentials | None |
| Anthropic | `ANTHROPIC_API_KEY` | None |
| Voyage | `VOYAGE_API_KEY` | None |
| LiteLLM | `LITELLM_API_KEY`, provider keys | None |
| OpenRouter | `OPENROUTER_API_KEY` | None |
| vLLM | None | None |
| Bedrock | `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY` | `AWS_SESSION_TOKEN` |

**Setting Environment Variables:**

```bash
# Linux/macOS
export OPENAI_API_KEY="sk-..."
export ANTHROPIC_API_KEY="sk-ant-..."

# Windows PowerShell
$env:OPENAI_API_KEY="sk-..."
$env:ANTHROPIC_API_KEY="sk-ant-..."

# Docker
docker run -e OPENAI_API_KEY="sk-..." ...
```

For production deployments, use secure secret management systems (AWS Secrets Manager, HashiCorp Vault, Kubernetes Secrets) rather than plain environment variables.

**Sources:** [docs/docs/ai/llm.mdx:76-77](), [docs/docs/ai/llm.mdx:176-177](), [docs/docs/ai/llm.mdx:284-285](), [docs/docs/ai/llm.mdx:306-307](), [docs/docs/ai/llm.mdx:329](), [docs/docs/ai/llm.mdx:394-395](), [docs/docs/ai/llm.mdx:447-451]()

---

## Custom Address Configuration

The `address` field in `LlmSpec` allows you to override default API endpoints. This is useful for:
- Using proxy servers
- Self-hosted LLM servers (Ollama, vLLM)
- Custom API gateways
- Regional endpoints

**Address Priority:**
1. `address` parameter in `LlmSpec` (highest priority)
2. Provider-specific environment variable (e.g., `OPENAI_API_BASE`)
3. Provider default endpoint

**Example Use Cases:**

```python
# Self-hosted Ollama on custom port
cocoindex.LlmSpec(
    api_type=cocoindex.LlmApiType.OLLAMA,
    model="llama3.2",
    address="http://gpu-server:8080"
)

# OpenAI via corporate proxy
cocoindex.LlmSpec(
    api_type=cocoindex.LlmApiType.OPENAI,
    model="gpt-4o",
    address="https://api-proxy.company.com/v1"
)

# vLLM on specific server
cocoindex.LlmSpec(
    api_type=cocoindex.LlmApiType.VLLM,
    model="my-model",
    address="http://10.0.1.50:8000/v1"
)
```

**Sources:** [docs/docs/ai/llm.mdx:79](), [docs/docs/ai/llm.mdx:138-141](), [docs/docs/ai/llm.mdx:437-439]()

---

## Integration with Built-in Functions

LLM specifications are consumed by two primary built-in functions:

### ExtractByLlm

Uses text generation to extract structured data. Accepts `LlmSpec` as configuration.

```python
doc["extracted"] = doc["text"].transform(
    cocoindex.functions.ExtractByLlm(
        llm_spec=cocoindex.LlmSpec(
            api_type=cocoindex.LlmApiType.OPENAI,
            model="gpt-4o"
        ),
        output_type=MyDataClass,
        instruction="Extract relevant fields"
    )
)
```

### EmbedText

Uses text embedding for vector representation. Takes LLM parameters directly (not via `LlmSpec`).

```python
doc["vector"] = doc["text"].transform(
    cocoindex.functions.EmbedText(
        api_type=cocoindex.LlmApiType.OPENAI,
        model="text-embedding-3-small"
    )
)
```

For detailed documentation of these functions, see [Built-in Processing Functions](#6) and [Embedding and AI Functions](#6.2).

**Sources:** [docs/docs/ai/llm.mdx:36-68](), [examples/manuals_llm_extraction/main.py:105-127]()

---

# Page: Storage Backends and Export Targets

# Storage Backends and Export Targets

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [docs/docs/core/basics.md](docs/docs/core/basics.md)
- [docs/docs/core/flow_def.mdx](docs/docs/core/flow_def.mdx)
- [docs/docs/core/flow_methods.mdx](docs/docs/core/flow_methods.mdx)
- [docs/docs/targets/postgres.md](docs/docs/targets/postgres.md)
- [python/cocoindex/targets/__init__.py](python/cocoindex/targets/__init__.py)
- [python/cocoindex/targets/_engine_builtin_specs.py](python/cocoindex/targets/_engine_builtin_specs.py)

</details>



This page provides an overview of CocoIndex's export target system, which defines where and how processed data is written to external storage systems. Export targets are the final stage of a data processing flow, responsible for persisting collected data to databases, vector stores, knowledge graphs, or custom destinations.

For detailed information about defining data processing flows and collecting data before export, see [Flow Definition System](#3). For information about implementing custom export destinations, see [Custom Target Implementation](#8.3).

## Purpose and Role

Export targets serve as the bridge between CocoIndex's data processing pipeline and external storage systems. After data is sourced, transformed, and collected within a flow, export targets handle:

- **Schema Creation**: Setting up tables, collections, or graph structures in the target system
- **Data Synchronization**: Writing inserts, updates, and deletes to keep the target in sync
- **Incremental Updates**: Only writing changed data, leveraging CocoIndex's state tracking
- **Index Management**: Creating and maintaining indexes for efficient querying

Export targets are defined in the `export()` method called on a `DataCollector` object within a flow definition.

**Flow Architecture with Export Targets**

```mermaid
graph LR
    Source["add_source()"]
    Transform["transform()"]
    Collector["add_collector()"]
    Collect["collect()"]
    Export["export()"]
    Target["External Storage<br/>(Postgres/Qdrant/Neo4j/Kuzu)"]
    
    Source --> Transform
    Transform --> Collector
    Collector --> Collect
    Collect --> Export
    Export --> Target
    
    Note1["DataScope operations"]
    Note2["DataCollector operations"]
    
    Source -.-> Note1
    Transform -.-> Note1
    Collector -.-> Note2
    Collect -.-> Note2
    Export -.-> Note2
```

Sources: [docs/sidebars.ts:46-58](), [python/cocoindex/targets/_engine_builtin_specs.py:1-154]()

## Target Architecture

All export targets in CocoIndex implement a two-phase operation model:

### Setup Operations

Setup operations manage the lifecycle of storage infrastructure without specific data:

- **Creation**: Initialize tables, collections, directories, or graph schemas
- **Updates**: Apply configuration changes to existing structures
- **Deletion**: Clean up resources when a target is removed

These operations are triggered by `cocoindex setup` and `cocoindex drop` commands, or when the target specification changes between flow updates.

### Data Operations

Data operations handle actual data mutations:

- **Insert**: Add new records when data is first collected
- **Update**: Modify existing records when source data changes
- **Delete**: Remove records when source data is deleted

These operations are triggered during `cocoindex update` runs and are applied incrementally based on state tracking.

**Target Operation Flow**

```mermaid
stateDiagram-v2
    [*] --> NoTarget: Flow defined without target
    NoTarget --> SetupPending: export() called in flow
    SetupPending --> Ready: cocoindex setup
    
    Ready --> DataUpdate: cocoindex update
    DataUpdate --> Ready: Complete
    
    Ready --> SpecChanged: Spec modified in code
    SpecChanged --> Ready: cocoindex update
    
    Ready --> TargetRemoved: export() removed from flow
    TargetRemoved --> [*]: cocoindex drop
    
    note right of SetupPending
        apply_setup_change()
        previous=None, current=Spec
    end note
    
    note right of DataUpdate
        mutate()
        receives insertions/updates/deletes
    end note
    
    note right of SpecChanged
        apply_setup_change()
        previous=OldSpec, current=NewSpec
    end note
    
    note right of TargetRemoved
        apply_setup_change()
        previous=Spec, current=None
    end note
```

Sources: [docs/docs/custom_ops/custom_targets.mdx:20-33](), [python/cocoindex/targets/_engine_builtin_specs.py:20-154]()

## Built-in Targets Overview

CocoIndex provides built-in targets for common storage systems. Each target is optimized for specific use cases and query patterns.

| Target | Type | Primary Use Case | Key Features | Spec Class |
|--------|------|------------------|--------------|------------|
| **Postgres** | Relational + Vector | General-purpose storage with vector search | Native pgvector support, ACID compliance, full SQL | `cocoindex.targets.Postgres` |
| **Qdrant** | Vector Database | High-performance vector similarity search | Purpose-built for embeddings, filtering, HNSW indexes | `cocoindex.targets.Qdrant` |
| **Neo4j** | Graph Database | Complex relationship querying and graph analytics | Cypher queries, property graphs, path finding | `cocoindex.targets.Neo4j` |
| **Kuzu** | Embedded Graph | In-process graph analytics with DuckDB integration | Embedded deployment, analytical queries | `cocoindex.targets.Kuzu` |
| **Custom** | User-defined | Any destination not covered above | Full control over setup and data operations | `cocoindex.op.TargetSpec` |

**Target Classification by Storage Model**

```mermaid
graph TD
    Targets["Export Targets"]
    
    Vector["Vector Storage"]
    Graph["Graph Storage"]
    Custom["Custom Storage"]
    
    Postgres["Postgres<br/>cocoindex.targets.Postgres"]
    Qdrant["Qdrant<br/>cocoindex.targets.Qdrant"]
    
    Neo4j["Neo4j<br/>cocoindex.targets.Neo4j"]
    Kuzu["Kuzu<br/>cocoindex.targets.Kuzu"]
    
    UserTarget["@cocoindex.op.target_connector<br/>User-defined class"]
    
    Targets --> Vector
    Targets --> Graph
    Targets --> Custom
    
    Vector --> Postgres
    Vector --> Qdrant
    
    Graph --> Neo4j
    Graph --> Kuzu
    
    Custom --> UserTarget
    
    PgDetails["- Relational + Vector<br/>- pgvector extension<br/>- HNSW, IVFFlat indexes<br/>- Same DB as internal storage"]
    QdrantDetails["- Pure vector DB<br/>- gRPC connection<br/>- Collection-based<br/>- High-performance similarity"]
    
    Neo4jDetails["- Property graph<br/>- Nodes and Relationships<br/>- Cypher queries<br/>- Cloud or self-hosted"]
    KuzuDetails["- Embedded graph DB<br/>- DuckDB-style analytics<br/>- API server connection<br/>- In-process performance"]
    
    CustomDetails["- TargetSpec subclass<br/>- @target_connector decorator<br/>- Implement setup & mutate<br/>- Full control"]
    
    Postgres -.-> PgDetails
    Qdrant -.-> QdrantDetails
    Neo4j -.-> Neo4jDetails
    Kuzu -.-> KuzuDetails
    UserTarget -.-> CustomDetails
```

Sources: [python/cocoindex/targets/_engine_builtin_specs.py:20-154](), [docs/sidebars.ts:46-58]()

## Target Specifications and Connections

Each target is configured through a spec class that defines connection details and behavior. Specs are instantiated when calling `export()` on a collector.

### Common Connection Patterns

**Authentication Registry Pattern**

Built-in targets use the authentication registry to manage credentials securely. Connection specs are registered once and referenced by targets:

```python
# Register connection credentials (typically in settings)
cocoindex.register_auth(
    "my_qdrant",
    cocoindex.targets.QdrantConnection(
        grpc_url="http://localhost:6334",
        api_key="secret_key"
    )
)

# Reference in target spec
collector.export(
    "embeddings",
    cocoindex.targets.Qdrant(
        collection_name="documents",
        connection=cocoindex.auth_reference("my_qdrant")
    ),
    primary_key_fields=["id"]
)
```

**Connection Spec Types by Target**

```mermaid
classDiagram
    class TargetSpec {
        <<abstract>>
    }
    
    class Postgres {
        +database: AuthEntryReference[DatabaseConnectionSpec]
        +table_name: str
        +schema: str
        +column_options: dict
    }
    
    class Qdrant {
        +collection_name: str
        +connection: AuthEntryReference[QdrantConnection]
    }
    
    class Neo4j {
        +connection: AuthEntryReference[Neo4jConnection]
        +mapping: Nodes | Relationships
    }
    
    class Kuzu {
        +connection: AuthEntryReference[KuzuConnection]
        +mapping: Nodes | Relationships
    }
    
    class DatabaseConnectionSpec {
        +url: str
        +user: str
        +password: str
        +max_connections: int
        +min_connections: int
    }
    
    class QdrantConnection {
        +grpc_url: str
        +api_key: str
    }
    
    class Neo4jConnection {
        +uri: str
        +user: str
        +password: str
        +db: str
    }
    
    class KuzuConnection {
        +api_server_url: str
    }
    
    TargetSpec <|-- Postgres
    TargetSpec <|-- Qdrant
    TargetSpec <|-- Neo4j
    TargetSpec <|-- Kuzu
    
    Postgres --> DatabaseConnectionSpec
    Qdrant --> QdrantConnection
    Neo4j --> Neo4jConnection
    Kuzu --> KuzuConnection
```

Sources: [python/cocoindex/targets/_engine_builtin_specs.py:12-154](), [docs/docs/targets/postgres.md:44-46]()

## Choosing a Target

The choice of export target depends on your application's query patterns and data characteristics:

### Vector Search Applications

Use when your primary operation is finding similar items based on embeddings (e.g., semantic search, recommendation systems).

**Postgres (pgvector)** - Choose when:
- You need relational queries alongside vector search
- You want to use the same database as CocoIndex's internal storage
- You require ACID transactions
- You have structured data with foreign key relationships

**Qdrant** - Choose when:
- You need maximum vector search performance
- You have very large vector collections (millions+ of vectors)
- You need advanced filtering during similarity search
- Vector search is your primary query pattern

See [Vector Databases: Postgres and Qdrant](#8.1) for detailed configuration.

### Knowledge Graph Applications

Use when your data has complex relationships that need to be traversed or analyzed (e.g., entity networks, hierarchies, dependency graphs).

**Neo4j** - Choose when:
- You need mature graph query capabilities (Cypher)
- You require cloud-hosted graph databases
- You need enterprise features and ecosystem support
- You have complex relationship patterns

**Kuzu** - Choose when:
- You prefer embedded/in-process deployment
- You need DuckDB-style analytical graph queries
- You want faster analytical aggregations over graphs
- You need integration with the DuckDB ecosystem

See [Knowledge Graphs: Neo4j and Kuzu](#8.2) for detailed configuration.

### Custom Applications

Use when none of the built-in targets match your requirements:
- Exporting to file systems (local, S3, GCS)
- Writing to APIs or message queues
- Custom storage systems or proprietary databases
- Special transformation or formatting requirements

See [Custom Target Implementation](#8.3) for a complete guide.

**Decision Tree**

```mermaid
graph TD
    Start["Select Export Target"]
    
    UseCase{"Primary Query<br/>Pattern?"}
    
    VectorSearch["Vector<br/>Similarity"]
    GraphQuery["Graph<br/>Traversal"]
    OtherUse["Other/Custom"]
    
    VectorNeed{"Additional<br/>Requirements?"}
    GraphNeed{"Deployment<br/>Preference?"}
    
    UsePg["Postgres<br/>with pgvector"]
    UseQdrant["Qdrant"]
    
    UseNeo4j["Neo4j"]
    UseKuzu["Kuzu"]
    
    UseCustom["Custom Target"]
    
    Start --> UseCase
    
    UseCase -->|"Find similar items"| VectorSearch
    UseCase -->|"Traverse relationships"| GraphQuery
    UseCase -->|"File export, API, etc"| OtherUse
    
    VectorSearch --> VectorNeed
    VectorNeed -->|"Need SQL queries<br/>or ACID"| UsePg
    VectorNeed -->|"Pure vector search<br/>performance"| UseQdrant
    
    GraphQuery --> GraphNeed
    GraphNeed -->|"Cloud/enterprise<br/>Cypher queries"| UseNeo4j
    GraphNeed -->|"Embedded/analytical<br/>DuckDB-style"| UseKuzu
    
    OtherUse --> UseCustom
    
    PgNote["Table with vector columns<br/>HNSW/IVFFlat indexes<br/>Full SQL support"]
    QdrantNote["Collections of vectors<br/>gRPC API<br/>Advanced filtering"]
    Neo4jNote["Property graph<br/>Nodes & Relationships<br/>Cypher DSL"]
    KuzuNote["Embedded graph<br/>Analytical queries<br/>API server"]
    CustomNote["Implement TargetSpec<br/>Define setup & mutate<br/>Full control"]
    
    UsePg -.-> PgNote
    UseQdrant -.-> QdrantNote
    UseNeo4j -.-> Neo4jNote
    UseKuzu -.-> KuzuNote
    UseCustom -.-> CustomNote
```

Sources: [docs/docs/targets/postgres.md:1-105](), [python/cocoindex/targets/_engine_builtin_specs.py:46-154]()

## Basic Usage Pattern

All export targets follow the same usage pattern within a flow definition:

```python
@cocoindex.flow_def(name="MyFlow")
def my_flow(flow_builder: cocoindex.FlowBuilder, data_scope: cocoindex.DataScope):
    # 1. Add data source
    data_scope["documents"] = flow_builder.add_source(...)
    
    # 2. Transform and process
    data_scope["chunks"] = data_scope["documents"].transform(...)
    
    # 3. Create collector
    output = data_scope.add_collector()
    
    # 4. Collect data into collector
    with data_scope["chunks"].row() as chunk:
        output.collect(
            id=chunk["chunk_id"],
            text=chunk["text"],
            embedding=chunk["embedding"]
        )
    
    # 5. Export to target(s)
    output.export(
        "doc_embeddings",                    # Target name
        cocoindex.targets.Postgres(          # Target spec
            table_name="document_embeddings"
        ),
        primary_key_fields=["id"],           # Required: defines row identity
        vector_indexes=[                      # Optional: create indexes
            cocoindex.index.VectorIndexDef(
                field_name="embedding",
                metric=cocoindex.index.VectorSimilarityMetric.COSINE_SIMILARITY
            )
        ]
    )
```

**Key Parameters for export()**

| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `target_name` | `str` | Yes | Unique identifier for this target within the flow |
| `spec` | `TargetSpec` | Yes | Target specification (Postgres, Qdrant, Neo4j, Kuzu, or custom) |
| `primary_key_fields` | `Sequence[str]` | Yes | Field(s) that uniquely identify each row |
| `vector_indexes` | `Sequence[VectorIndexDef]` | No | Vector indexes to create (for vector-capable targets) |
| `attachments` | `Sequence[TargetAttachmentSpec]` | No | Additional setup operations (e.g., `PostgresSqlCommand`) |
| `setup_by_user` | `bool` | No | If True, skip automatic setup/drop (manual management) |

**Export Method Call Flow**

```mermaid
sequenceDiagram
    participant User as Flow Definition
    participant Collector as DataCollector
    participant Engine as CocoIndex Engine
    participant Target as Target System
    
    User->>Collector: collect(id=..., text=..., embedding=...)
    Note over Collector: Rows accumulated<br/>in collector
    
    User->>Collector: export("name", spec, primary_key_fields=[...])
    Collector->>Engine: Register export target
    Note over Engine: Target stored in flow config
    
    User->>Engine: cocoindex setup
    Engine->>Target: apply_setup_change(previous=None, current=spec)
    Target-->>Engine: Schema/collection created
    
    User->>Engine: cocoindex update
    Engine->>Engine: Process flow, detect changes
    Engine->>Target: mutate({id1: row1_data, id2: row2_data, ...})
    Target-->>Engine: Data written
    
    User->>User: Modify spec in code
    User->>Engine: cocoindex update
    Engine->>Target: apply_setup_change(previous=old_spec, current=new_spec)
    Target-->>Engine: Configuration updated
```

Sources: [docs/docs/custom_ops/custom_targets.mdx:36-37](), [python/cocoindex/targets/_engine_builtin_specs.py:20-154]()

## Target Index Management

Targets that support vector or graph operations can define indexes to optimize query performance. Index definitions are specified in the `export()` call and are created during `cocoindex setup`.

### Vector Indexes

Vector indexes accelerate similarity search operations. They are supported by Postgres (pgvector) and Qdrant targets.

**Index Configuration Classes**

```mermaid
classDiagram
    class VectorIndexDef {
        +field_name: str
        +metric: VectorSimilarityMetric
        +method: VectorIndexMethod
    }
    
    class VectorSimilarityMetric {
        <<enumeration>>
        COSINE_SIMILARITY
        L2_DISTANCE
        INNER_PRODUCT
    }
    
    class VectorIndexMethod {
        <<interface>>
    }
    
    class HnswVectorIndexMethod {
        +kind: "Hnsw"
        +m: int
        +ef_construction: int
    }
    
    class IvfFlatVectorIndexMethod {
        +kind: "IvfFlat"
        +lists: int
    }
    
    VectorIndexDef --> VectorSimilarityMetric
    VectorIndexDef --> VectorIndexMethod
    VectorIndexMethod <|-- HnswVectorIndexMethod
    VectorIndexMethod <|-- IvfFlatVectorIndexMethod
```

Example:

```python
output.export(
    "embeddings",
    cocoindex.targets.Postgres(table_name="docs"),
    primary_key_fields=["id"],
    vector_indexes=[
        cocoindex.index.VectorIndexDef(
            field_name="embedding",
            metric=cocoindex.index.VectorSimilarityMetric.COSINE_SIMILARITY,
            method=cocoindex.index.HnswVectorIndexMethod(
                m=16,
                ef_construction=64
            )
        )
    ]
)
```

Sources: [python/cocoindex/index.py:1-51](), [python/cocoindex/targets/_engine_builtin_specs.py:70-78]()

### Graph Indexes

Graph targets (Neo4j, Kuzu) can declare vector indexes on node properties for similarity search within graph structures. These are specified using declaration specs rather than in the `export()` call.

Example for Neo4j:

```python
# Declare node structure with vector index
flow_builder.add_declaration(
    cocoindex.targets.Neo4jDeclaration(
        connection=cocoindex.auth_reference("neo4j_conn"),
        nodes_label="Document",
        primary_key_fields=["id"],
        vector_indexes=[
            cocoindex.index.VectorIndexDef(
                field_name="embedding",
                metric=cocoindex.index.VectorSimilarityMetric.COSINE_SIMILARITY
            )
        ]
    )
)
```

See [Knowledge Graphs: Neo4j and Kuzu](#8.2) for more details on graph-specific indexing.

Sources: [python/cocoindex/targets/_engine_builtin_specs.py:123-131](), [python/cocoindex/index.py:32-51]()

## Target Attachments

Some targets support attachments—additional setup operations that extend the target's capabilities. Attachments are specified in the `attachments` parameter of `export()`.

### PostgresSqlCommand Attachment

The `PostgresSqlCommand` attachment allows executing arbitrary SQL during Postgres target setup and teardown. This is useful for operations not covered by the core target spec, such as:

- Creating specialized indexes (GIN, GiST, full-text search)
- Setting up triggers or stored procedures
- Granting permissions
- Creating views or materialized views

Example usage:

```python
output.export(
    "documents",
    cocoindex.targets.Postgres(table_name="documents"),
    primary_key_fields=["id"],
    attachments=[
        cocoindex.targets.PostgresSqlCommand(
            name="fts_index",
            setup_sql=(
                "CREATE INDEX IF NOT EXISTS documents_text_fts "
                "ON documents USING GIN (to_tsvector('english', text));"
            ),
            teardown_sql="DROP INDEX IF EXISTS documents_text_fts;"
        )
    ]
)
```

**Attachment Lifecycle**

```mermaid
stateDiagram-v2
    [*] --> NoAttachment: Target without attachment
    NoAttachment --> AttachmentAdded: Add PostgresSqlCommand to export()
    AttachmentAdded --> SetupExecuted: cocoindex setup
    
    SetupExecuted --> SetupUpdated: Modify setup_sql
    SetupUpdated --> SetupExecuted: cocoindex setup
    
    SetupExecuted --> TeardownExecuted: Remove attachment or target
    TeardownExecuted --> [*]: cocoindex drop
    
    note right of AttachmentAdded
        Attachment defined with:
        - name (unique ID)
        - setup_sql
        - teardown_sql (optional)
    end note
    
    note right of SetupExecuted
        setup_sql executed
        Must be idempotent
        (CREATE IF NOT EXISTS)
    end note
    
    note right of TeardownExecuted
        teardown_sql executed
        Uses saved version
        (DROP IF EXISTS)
    end note
```

Sources: [python/cocoindex/targets/_engine_builtin_specs.py:29-35](), [docs/docs/targets/postgres.md:58-96]()

## Summary

Export targets in CocoIndex provide a unified interface for writing processed data to diverse storage systems:

- **Vector Databases** ([#8.1](#8.1)): Postgres with pgvector and Qdrant for similarity search
- **Graph Databases** ([#8.2](#8.2)): Neo4j and Kuzu for relationship-based queries
- **Custom Targets** ([#8.3](#8.3)): User-defined targets for any destination

All targets follow the same setup/data operation model and are configured through spec classes in the `export()` method. The authentication registry provides secure credential management, while index definitions optimize query performance.

The next sections provide detailed documentation for each target category.

Sources: [python/cocoindex/targets/_engine_builtin_specs.py:1-154](), [docs/sidebars.ts:46-58](), [docs/docs/custom_ops/custom_targets.mdx:1-282]()

---

# Page: Vector Databases: Postgres and Qdrant

# Vector Databases: Postgres and Qdrant

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [docs/docs/targets/postgres.md](docs/docs/targets/postgres.md)
- [python/cocoindex/targets/__init__.py](python/cocoindex/targets/__init__.py)
- [python/cocoindex/targets/_engine_builtin_specs.py](python/cocoindex/targets/_engine_builtin_specs.py)

</details>



## Purpose and Scope

This document covers the two primary vector database backends supported by CocoIndex: **Postgres with pgvector** and **Qdrant**. These targets enable storage and retrieval of vector embeddings for semantic search applications. The document explains:

- Vector index configuration (`VectorIndexDef`, similarity metrics, index methods)
- Postgres vector storage with the `Postgres` target spec
- Qdrant vector storage with the `Qdrant` target spec
- Data mapping, schema generation, and operational considerations

For information about graph databases (Neo4j, Kuzu), see [8.2](#8.2). For implementing custom storage backends, see [8.3](#8.3). For the flow-level export operation that uses these targets, see [3.2](#3.2).

---

## Vector Index Configuration System

### VectorIndexDef and IndexOptions

CocoIndex provides a unified vector index configuration system that works across different storage backends. The core abstraction is `VectorIndexDef`, which specifies how vector fields should be indexed.

```mermaid
classDiagram
    class VectorIndexDef {
        +str field_name
        +VectorSimilarityMetric metric
        +VectorIndexMethod|None method
    }
    
    class VectorSimilarityMetric {
        <<enumeration>>
        COSINE_SIMILARITY
        L2_DISTANCE
        INNER_PRODUCT
    }
    
    class HnswVectorIndexMethod {
        +str kind = "Hnsw"
        +int|None m
        +int|None ef_construction
    }
    
    class IvfFlatVectorIndexMethod {
        +str kind = "IvfFlat"
        +int|None lists
    }
    
    class IndexOptions {
        +Sequence~str~ primary_key_fields
        +Sequence~VectorIndexDef~ vector_indexes
    }
    
    VectorIndexDef --> VectorSimilarityMetric
    VectorIndexDef --> HnswVectorIndexMethod
    VectorIndexDef --> IvfFlatVectorIndexMethod
    IndexOptions --> VectorIndexDef
```

Sources: [python/cocoindex/index.py:1-51]()

### Similarity Metrics

The `VectorSimilarityMetric` enum defines three distance/similarity functions for vector search:

| Metric | Enum Value | Description | Use Case |
|--------|-----------|-------------|----------|
| **Cosine Similarity** | `VectorSimilarityMetric.COSINE_SIMILARITY` | Measures angle between vectors, normalized to [-1, 1] | Text embeddings, semantic similarity (most common) |
| **L2 Distance** | `VectorSimilarityMetric.L2_DISTANCE` | Euclidean distance between vectors | Spatial data, feature vectors |
| **Inner Product** | `VectorSimilarityMetric.INNER_PRODUCT` | Dot product of vectors | Pre-normalized embeddings, efficiency-optimized retrieval |

Sources: [python/cocoindex/index.py:6-10]()

### Index Methods

CocoIndex supports two vector index algorithms through the `VectorIndexMethod` union type:

#### HNSW (Hierarchical Navigable Small World)

The `HnswVectorIndexMethod` configures HNSW indexes, which provide fast approximate nearest neighbor search:

```python
HnswVectorIndexMethod(
    kind="Hnsw",
    m=16,              # Number of bi-directional links per node
    ef_construction=64 # Size of dynamic candidate list during construction
)
```

**Parameters:**
- `m` (int, optional): Controls index connectivity. Higher values improve recall but increase memory usage. Typical range: 8-64. Default varies by backend.
- `ef_construction` (int, optional): Determines construction quality. Higher values improve index quality but slow down indexing. Typical range: 64-512.

Sources: [python/cocoindex/index.py:13-19]()

#### IVFFlat (Inverted File with Flat Quantization)

The `IvfFlatVectorIndexMethod` configures IVFFlat indexes, which partition vectors into clusters:

```python
IvfFlatVectorIndexMethod(
    kind="IvfFlat",
    lists=100  # Number of inverted lists (clusters)
)
```

**Parameters:**
- `lists` (int, optional): Number of clusters. Rule of thumb: `sqrt(num_rows)` for small datasets, `num_rows / 1000` for large datasets.

Sources: [python/cocoindex/index.py:22-27]()

---

## Postgres Vector Storage

### Overview and Data Mapping

The `Postgres` target spec exports data to PostgreSQL tables with pgvector extension support. Each export target maps to a unique table in the database.

```mermaid
graph LR
    subgraph "CocoIndex Flow"
        Collector["DataScope.collect()"]
        Export["DataScope.export()"]
    end
    
    subgraph "Postgres Target Spec"
        PostgresSpec["cocoindex.targets.Postgres"]
        DatabaseConnSpec["DatabaseConnectionSpec"]
        ColumnOpts["PostgresColumnOptions"]
        Attachment["PostgresSqlCommand"]
    end
    
    subgraph "Postgres Database"
        Table["Table: app__flow__target"]
        Columns["Columns: id, text, embedding"]
        VectorIndex["Vector Index: HNSW/IVFFlat"]
    end
    
    Collector --> Export
    Export --> PostgresSpec
    PostgresSpec --> DatabaseConnSpec
    PostgresSpec --> ColumnOpts
    PostgresSpec --> Attachment
    PostgresSpec --> Table
    Table --> Columns
    Table --> VectorIndex
```

Sources: [python/cocoindex/targets/_engine_builtin_specs.py:20-28](), [docs/docs/targets/postgres.md:13-25]()

#### Element Mapping Table

| CocoIndex Element | Postgres Element | Notes |
|-------------------|------------------|-------|
| Export target | Table | Unique table per target; naming: `[app__]flow__target` |
| Collected row | Table row | Each call to `collect()` produces one row |
| Field | Column | Field names map directly to column names |
| Vector field | `vector(N)` or `jsonb` column | Fixed-dimension numeric vectors → `vector(N)`, others → `jsonb` |
| Primary key fields | Primary key constraint | Specified in `export()` call |
| Vector index | pgvector index | HNSW or IVFFlat index on vector columns |

Sources: [docs/docs/targets/postgres.md:13-31]()

### Postgres Target Spec

The `Postgres` class is defined at [python/cocoindex/targets/_engine_builtin_specs.py:20-27]() with the following fields:

```python
class Postgres(op.TargetSpec):
    database: AuthEntryReference[DatabaseConnectionSpec] | None = None
    table_name: str | None = None
    schema: str | None = None
    column_options: dict[str, PostgresColumnOptions] | None = None
```

#### Field Reference

**`database`** (`AuthEntryReference[DatabaseConnectionSpec] | None`)
- Connection to the Postgres database
- If `None`, uses the internal storage database (see [9](#9) for internal state management)
- References `DatabaseConnectionSpec` through the auth registry (see [10.1](#10.1))
- `DatabaseConnectionSpec` contains: `url`, `user`, `password`, `max_connections`, `min_connections`

**`table_name`** (`str | None`)
- Name of the target table
- If unspecified, auto-generates as `[AppNamespace__]FlowName__TargetName`
- Examples: `DemoFlow__doc_embeddings`, `Staging__DemoFlow__doc_embeddings`
- Must be explicitly set if using the `schema` parameter

**`schema`** (`str | None`)
- PostgreSQL schema for the table (e.g., `"analytics"`, `"staging"`)
- Default: `"public"` schema
- CocoIndex automatically creates the schema if it doesn't exist
- Requires explicit `table_name` when specified

**`column_options`** (`dict[str, PostgresColumnOptions] | None`)
- Per-column configuration overrides
- Key: column name, Value: `PostgresColumnOptions` instance
- Currently supports vector type specification (`"vector"` vs `"halfvec"`)

Sources: [python/cocoindex/targets/_engine_builtin_specs.py:20-27](), [docs/docs/targets/postgres.md:40-56]()

### PostgresColumnOptions

The `PostgresColumnOptions` dataclass provides fine-grained control over column types:

```python
@dataclass
class PostgresColumnOptions:
    type: Literal["vector", "halfvec"] | None = None
```

**Use Case: Half-Precision Vectors**

By default, CocoIndex maps vector fields to `vector` type (full precision). To reduce storage by 50%, specify `"halfvec"`:

```python
collector.export(
    "embeddings",
    cocoindex.targets.Postgres(
        table_name="doc_embeddings",
        column_options={
            "embedding": PostgresColumnOptions(type="halfvec")
        }
    ),
    primary_key_fields=["id"],
    vector_indexes=[
        VectorIndexDef(
            field_name="embedding",
            metric=VectorSimilarityMetric.COSINE_SIMILARITY,
            method=HnswVectorIndexMethod(m=16, ef_construction=64)
        )
    ]
)
```

Sources: [python/cocoindex/targets/_engine_builtin_specs.py:12-18](), [docs/docs/targets/postgres.md:52-56]()

### Type Mapping Rules

#### Vector Type Mapping

CocoIndex applies the following logic for vector fields:

```mermaid
flowchart TD
    VectorField["Vector Field in Flow"]
    CheckDim{"Fixed dimension?<br/>Vector[Float32/Float64/Int64, N]"}
    CheckOverride{"column_options<br/>specifies type?"}
    UseOverride["Use specified type:<br/>vector or halfvec"]
    UseDefault["Use vector(N)"]
    UseJsonb["Use jsonb"]
    
    VectorField --> CheckDim
    CheckDim -->|Yes| CheckOverride
    CheckDim -->|No| UseJsonb
    CheckOverride -->|Yes| UseOverride
    CheckOverride -->|No| UseDefault
```

**Key Rule:** Only fixed-dimension numeric vectors (`Vector[Float32, N]`, `Vector[Float64, N]`, `Vector[Int64, N]`) map to `vector(N)`. All other vector types (dynamic dimension, lists, etc.) map to `jsonb`.

Sources: [docs/docs/targets/postgres.md:26-31]()

#### String Sanitization

**Important:** PostgreSQL cannot store U+0000 (NUL) characters in text or jsonb. CocoIndex automatically strips these characters during export.

Example:
```python
# Input: "Hello\x00World"
# Stored in Postgres: "HelloWorld"
```

Sources: [docs/docs/targets/postgres.md:33-38]()

### PostgresSqlCommand Attachment

The `PostgresSqlCommand` attachment executes arbitrary SQL during setup/teardown, enabling advanced indexing and database features not directly modeled by the target spec.

```python
class PostgresSqlCommand(op.TargetAttachmentSpec):
    name: str
    setup_sql: str
    teardown_sql: str | None = None
```

Sources: [python/cocoindex/targets/_engine_builtin_specs.py:29-35]()

#### Example: Full-Text Search Index

```python
collector.export(
    "doc_embeddings",
    cocoindex.targets.Postgres(table_name="doc_embeddings"),
    primary_key_fields=["id"],
    vector_indexes=[
        VectorIndexDef(
            field_name="embedding",
            metric=VectorSimilarityMetric.COSINE_SIMILARITY
        )
    ],
    attachments=[
        cocoindex.targets.PostgresSqlCommand(
            name="fts",
            setup_sql=(
                "CREATE INDEX IF NOT EXISTS doc_embeddings_text_fts "
                "ON doc_embeddings USING GIN (to_tsvector('english', text));"
            ),
            teardown_sql="DROP INDEX IF EXISTS doc_embeddings_text_fts;"
        )
    ]
)
```

#### Execution Guarantees

1. **Idempotence Required:** Both `setup_sql` and `teardown_sql` must be idempotent (use `IF NOT EXISTS`, `IF EXISTS`)
2. **Multiple Statements:** Use semicolons to separate statements
3. **Upsert Behavior:** Updated `setup_sql` re-executes during setup
4. **Saved Teardown:** CocoIndex saves `teardown_sql`; executes when attachment removed
5. **Automatic Execution:** `teardown_sql` runs automatically when attachment or target deleted

Sources: [docs/docs/targets/postgres.md:59-96]()

---

## Qdrant Vector Storage

### Overview

Qdrant is a specialized vector database optimized for high-performance similarity search. The `Qdrant` target spec exports data to Qdrant collections.

```mermaid
graph LR
    subgraph "CocoIndex Flow"
        Export["DataScope.export()"]
    end
    
    subgraph "Qdrant Target Spec"
        QdrantSpec["cocoindex.targets.Qdrant"]
        QdrantConn["QdrantConnection"]
    end
    
    subgraph "Qdrant Server"
        Collection["Collection: my_embeddings"]
        Points["Points with Vectors"]
        Payload["Payload: Metadata Fields"]
        Index["HNSW Index"]
    end
    
    Export --> QdrantSpec
    QdrantSpec --> QdrantConn
    QdrantSpec --> Collection
    Collection --> Points
    Collection --> Payload
    Collection --> Index
```

Sources: [python/cocoindex/targets/_engine_builtin_specs.py:37-51]()

### Qdrant Target Spec

```python
@dataclass
class Qdrant(op.TargetSpec):
    collection_name: str
    connection: AuthEntryReference[QdrantConnection] | None = None
```

**`collection_name`** (`str`, required)
- Name of the Qdrant collection to store vectors
- CocoIndex creates the collection if it doesn't exist
- Unlike Postgres, no auto-generated naming; must be explicitly specified

**`connection`** (`AuthEntryReference[QdrantConnection] | None`)
- Connection specification for the Qdrant server
- If `None`, requires Qdrant connection via environment variables or settings
- References `QdrantConnection` through auth registry

Sources: [python/cocoindex/targets/_engine_builtin_specs.py:46-51]()

### QdrantConnection

```python
@dataclass
class QdrantConnection:
    grpc_url: str
    api_key: str | None = None
```

**`grpc_url`** (`str`, required)
- gRPC endpoint URL for Qdrant server
- Examples:
  - Local: `"http://localhost:6334"`
  - Cloud: `"https://xyz-example.eu-central.aws.cloud.qdrant.io:6334"`

**`api_key`** (`str | None`)
- API key for authentication (required for Qdrant Cloud)
- Should be stored in environment variables or auth registry, not hardcoded

Sources: [python/cocoindex/targets/_engine_builtin_specs.py:37-43]()

### Example Usage

```python
from cocoindex import flow_def, FlowBuilder
from cocoindex.targets import Qdrant, QdrantConnection
from cocoindex.index import VectorIndexDef, VectorSimilarityMetric, HnswVectorIndexMethod
from cocoindex.auth_registry import register_auth

# Register connection in auth registry
register_auth(
    "qdrant_prod",
    QdrantConnection(
        grpc_url="https://xyz.cloud.qdrant.io:6334",
        api_key="your-api-key-here"  # Better: use environment variable
    )
)

@flow_def
def embedding_pipeline(flow: FlowBuilder):
    # ... data processing ...
    
    collector.export(
        "embeddings",
        Qdrant(
            collection_name="document_embeddings",
            connection="qdrant_prod"  # Reference to registered auth
        ),
        primary_key_fields=["doc_id"],
        vector_indexes=[
            VectorIndexDef(
                field_name="embedding",
                metric=VectorSimilarityMetric.COSINE_SIMILARITY,
                method=HnswVectorIndexMethod(m=16, ef_construction=128)
            )
        ]
    )
```

Sources: [python/cocoindex/targets/_engine_builtin_specs.py:37-51]()

---

## Comparison: Postgres vs Qdrant

### Feature Matrix

| Feature | Postgres + pgvector | Qdrant | Notes |
|---------|-------------------|--------|-------|
| **Deployment** | Self-hosted or managed (AWS RDS, etc.) | Self-hosted or Qdrant Cloud | Postgres reuses existing infrastructure |
| **Index Methods** | HNSW, IVFFlat | HNSW (default) | Both support HNSW |
| **Vector Types** | `vector`, `halfvec` | Native vector storage | Qdrant optimized for vectors |
| **Scalar Filtering** | Full SQL support | Payload filters | Postgres more flexible for complex queries |
| **Performance** | Good for < 1M vectors | Optimized for millions of vectors | Qdrant scales better |
| **Relational Data** | Native support (JOIN, transactions) | Not supported | Use Postgres for relational requirements |
| **Schema Management** | Automatic table/column creation | Automatic collection creation | Both handle schema automatically |
| **Attachments** | `PostgresSqlCommand` for custom SQL | Not supported | Postgres enables custom indexes, triggers |
| **Storage Overhead** | Higher (full RDBMS) | Lower (specialized) | Qdrant more storage-efficient |

### Decision Guide

```mermaid
flowchart TD
    Start["Choose Vector Database"]
    HasPostgres{"Already using<br/>Postgres?"}
    NeedsRelational{"Need relational<br/>queries or JOINs?"}
    VectorCount{"Vector count?"}
    
    UsePostgres["Use Postgres + pgvector"]
    UseQdrant["Use Qdrant"]
    
    Start --> HasPostgres
    HasPostgres -->|Yes| NeedsRelational
    HasPostgres -->|No| VectorCount
    NeedsRelational -->|Yes| UsePostgres
    NeedsRelational -->|No| VectorCount
    VectorCount -->|"< 1M vectors"| UsePostgres
    VectorCount -->|"> 1M vectors"| UseQdrant
```

**Use Postgres when:**
- Already using Postgres for application data
- Need to join vector results with relational tables
- Vector count < 1 million
- Want to minimize infrastructure complexity (one database)
- Need complex filtering with SQL
- Using `PostgresSqlCommand` for custom indexes/triggers

**Use Qdrant when:**
- Handling millions of vectors
- Pure similarity search workload (no complex JOINs)
- Need maximum vector search performance
- Willing to manage separate vector database
- Cloud deployment with Qdrant Cloud

Sources: [python/cocoindex/targets/_engine_builtin_specs.py:20-51](), [docs/docs/targets/postgres.md:1-105]()

---

## Integration with Flow System

### Export Operation Configuration

Both Postgres and Qdrant targets are used through the `DataScope.export()` method. The complete signature includes:

```python
collector.export(
    target_name: str,
    target_spec: Postgres | Qdrant,
    primary_key_fields: Sequence[str],
    vector_indexes: Sequence[VectorIndexDef] = (),
    attachments: Sequence[PostgresSqlCommand] = ()  # Postgres only
)
```

The `vector_indexes` parameter accepts `VectorIndexDef` instances that work uniformly across both backends:

```mermaid
graph TD
    subgraph "User Code"
        FlowDef["@flow_def function"]
        Export["collector.export()"]
        VectorIdx["VectorIndexDef instances"]
    end
    
    subgraph "Target Specs"
        PostgresSpec["Postgres spec"]
        QdrantSpec["Qdrant spec"]
    end
    
    subgraph "Engine Processing"
        Engine["Rust Engine"]
        PgBackend["Postgres Backend"]
        QdBackend["Qdrant Backend"]
    end
    
    FlowDef --> Export
    Export --> VectorIdx
    Export --> PostgresSpec
    Export --> QdrantSpec
    PostgresSpec --> Engine
    QdrantSpec --> Engine
    VectorIdx --> Engine
    Engine --> PgBackend
    Engine --> QdBackend
```

Sources: [python/cocoindex/targets/_engine_builtin_specs.py:20-51](), [python/cocoindex/index.py:33-41]()

### Type Safety and Validation

The operation system (see [4](#4)) performs type analysis on collected rows to ensure compatibility with target specs. The Rust engine validates:

1. Primary key fields exist in the schema
2. Vector index fields exist and have compatible types
3. Schema changes are compatible with existing data (see [9.2](#9.2))

This validation occurs during the `setup` phase, before any data is written.

Sources: [python/cocoindex/index.py:1-51]()

---

# Page: Knowledge Graphs: Neo4j and Kuzu

# Knowledge Graphs: Neo4j and Kuzu

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [docs/docs/targets/postgres.md](docs/docs/targets/postgres.md)
- [python/cocoindex/targets/__init__.py](python/cocoindex/targets/__init__.py)
- [python/cocoindex/targets/_engine_builtin_specs.py](python/cocoindex/targets/_engine_builtin_specs.py)

</details>



## Purpose and Scope

This document covers the graph database target connectors in CocoIndex: Neo4j and Kuzu. These targets enable exporting data processing results as knowledge graphs with nodes and relationships. Both targets support declarative schema definition, primary key constraints, and vector indexes for semantic search over graph elements.

For vector-only storage without graph structure, see [Vector Databases: Postgres and Qdrant](#8.1). For implementing custom storage backends, see [Custom Target Implementation](#8.3).

## Graph Database Overview

CocoIndex supports two graph database backends:

| Database | Type | Description | Use Case |
|----------|------|-------------|----------|
| **Neo4j** | Client-server | Industry-standard labeled property graph database | Production deployments, large-scale graphs, ACID transactions |
| **Kuzu** | Embedded/server | High-performance embedded graph database with optional API server | Embedded applications, local development, columnar storage |

Both targets use the same mapping model but have different connection specifications and capabilities.

Sources: [python/cocoindex/targets/_engine_builtin_specs.py:106-154]()

## Graph Data Mapping Model

CocoIndex maps tabular data from collectors to graph elements (nodes and relationships) using a declarative mapping specification.

```mermaid
graph TB
    subgraph "CocoIndex Layer"
        Collector["DataScope.collect()"]
        Rows["Collected Rows"]
        Fields["Row Fields"]
    end
    
    subgraph "Mapping Layer"
        Mapping["Nodes or Relationships"]
        NodeMap["Nodes mapping"]
        RelMap["Relationships mapping"]
        FieldMapping["TargetFieldMapping"]
    end
    
    subgraph "Graph Database"
        Node["Graph Node"]
        Relationship["Graph Relationship"]
        Properties["Node/Rel Properties"]
    end
    
    Collector --> Rows
    Rows --> Fields
    Fields --> Mapping
    
    Mapping --> NodeMap
    Mapping --> RelMap
    
    NodeMap --> Node
    RelMap --> Relationship
    
    FieldMapping --> Properties
    
    Node --> Properties
    Relationship --> Properties
```

**Mapping Rules:**

- **For Nodes**: Each collected row creates one graph node with a specified label. Row fields become node properties.
- **For Relationships**: Each collected row creates one relationship between two referenced nodes. Row fields specify source/target node identities and relationship properties.

Sources: [python/cocoindex/targets/_engine_builtin_specs.py:54-103]()

## Mapping Specifications

### Nodes Mapping

The `Nodes` class maps rows to graph nodes with a single label.

```python
@dataclass
class Nodes:
    """Spec to map a row to a graph node."""
    kind = "Node"
    label: str
```

**Fields:**
- `label` (`str`): The node label in the graph database

**Example:**
```python
# Map rows to Person nodes
collector.export(
    "people",
    cocoindex.targets.Neo4j(
        connection=auth.neo4j_conn,
        mapping=cocoindex.targets.Nodes(label="Person")
    ),
    primary_key_fields=["person_id"]
)
```

Each row in the collector becomes a `Person` node. If a row has fields `{person_id: "123", name: "Alice", age: 30}`, the resulting node will have those properties.

Sources: [python/cocoindex/targets/_engine_builtin_specs.py:80-88]()

### Relationships Mapping

The `Relationships` class maps rows to graph relationships connecting two nodes.

```python
@dataclass
class Relationships:
    """Spec to map a row to a graph relationship."""
    kind = "Relationship"
    rel_type: str
    source: NodeFromFields
    target: NodeFromFields
```

**Fields:**
- `rel_type` (`str`): The relationship type
- `source` (`NodeFromFields`): Specification for the source node
- `target` (`NodeFromFields`): Specification for the target node

Sources: [python/cocoindex/targets/_engine_builtin_specs.py:89-98]()

### Field Mapping Components

```mermaid
classDiagram
    class TargetFieldMapping {
        +str source
        +str|None target
    }
    
    class NodeFromFields {
        +str label
        +list~TargetFieldMapping~ fields
    }
    
    class Relationships {
        +str rel_type
        +NodeFromFields source
        +NodeFromFields target
    }
    
    Relationships --> NodeFromFields : source
    Relationships --> NodeFromFields : target
    NodeFromFields --> TargetFieldMapping : fields
```

**TargetFieldMapping** (`TargetFieldMapping`):
- `source` (`str`): Field name in the collected row
- `target` (`str | None`): Property name in the graph node (defaults to same as source)

**NodeFromFields** (`NodeFromFields`):
- `label` (`str`): Node label to match/create
- `fields` (`list[TargetFieldMapping]`): Mappings from row fields to node properties for identification

**Example:**
```python
# Map rows to KNOWS relationships between Person nodes
collector.export(
    "friendships",
    cocoindex.targets.Neo4j(
        connection=auth.neo4j_conn,
        mapping=cocoindex.targets.Relationships(
            rel_type="KNOWS",
            source=cocoindex.targets.NodeFromFields(
                label="Person",
                fields=[
                    cocoindex.targets.TargetFieldMapping(source="person1_id")
                ]
            ),
            target=cocoindex.targets.NodeFromFields(
                label="Person",
                fields=[
                    cocoindex.targets.TargetFieldMapping(source="person2_id")
                ]
            )
        )
    ),
    primary_key_fields=["person1_id", "person2_id"]
)
```

This creates `(Person {person1_id})-[:KNOWS]->(Person {person2_id})` relationships. The source and target nodes must already exist (typically created by a separate `Nodes` export).

Sources: [python/cocoindex/targets/_engine_builtin_specs.py:54-69]()

## Neo4j Target

### Connection Specification

```python
@dataclass
class Neo4jConnection:
    """Connection spec for Neo4j."""
    uri: str
    user: str
    password: str
    db: str | None = None
```

**Fields:**
- `uri` (`str`): Neo4j connection URI (e.g., `bolt://localhost:7687`)
- `user` (`str`): Username for authentication
- `password` (`str`): Password for authentication
- `db` (`str | None`): Database name (optional, defaults to Neo4j's default database)

Sources: [python/cocoindex/targets/_engine_builtin_specs.py:106-114]()

### Target Specification

```python
class Neo4j(op.TargetSpec):
    """Graph storage powered by Neo4j."""
    connection: AuthEntryReference[Neo4jConnection]
    mapping: Nodes | Relationships
```

**Fields:**
- `connection` (auth reference to `Neo4jConnection`): Database connection credentials
- `mapping` (`Nodes | Relationships`): How to map rows to graph elements

**Example Usage:**

```python
import cocoindex
from cocoindex.auth_registry import AuthEntry

# Register Neo4j connection
auth = cocoindex.auth_registry.GlobalAuthRegistry()
auth.neo4j_conn = AuthEntry(
    cocoindex.targets.Neo4jConnection(
        uri="bolt://localhost:7687",
        user="neo4j",
        password="password",
        db="mydb"
    )
)

@cocoindex.flow_def
def build_knowledge_graph(builder: cocoindex.FlowBuilder):
    # Create entity nodes
    entities = builder.add_source(...).transform(...)
    entities_collected = entities.collect()
    
    entities_collected.export(
        "entities",
        cocoindex.targets.Neo4j(
            connection=auth.neo4j_conn,
            mapping=cocoindex.targets.Nodes(label="Entity")
        ),
        primary_key_fields=["entity_id"]
    )
    
    # Create relationships
    relations = entities.transform(...).collect()
    relations.export(
        "relations",
        cocoindex.targets.Neo4j(
            connection=auth.neo4j_conn,
            mapping=cocoindex.targets.Relationships(
                rel_type="RELATED_TO",
                source=cocoindex.targets.NodeFromFields(
                    label="Entity",
                    fields=[cocoindex.targets.TargetFieldMapping(source="source_id")]
                ),
                target=cocoindex.targets.NodeFromFields(
                    label="Entity",
                    fields=[cocoindex.targets.TargetFieldMapping(source="target_id")]
                )
            )
        ),
        primary_key_fields=["source_id", "target_id"]
    )
```

Sources: [python/cocoindex/targets/_engine_builtin_specs.py:116-121]()

## Kuzu Target

### Connection Specification

```python
@dataclass
class KuzuConnection:
    """Connection spec for Kuzu."""
    api_server_url: str
```

**Fields:**
- `api_server_url` (`str`): URL to the Kuzu API server endpoint

Kuzu operates differently from Neo4j: it can run embedded within an application or via an HTTP API server. The CocoIndex connector communicates with Kuzu through its HTTP API.

Sources: [python/cocoindex/targets/_engine_builtin_specs.py:133-138]()

### Target Specification

```python
class Kuzu(op.TargetSpec):
    """Graph storage powered by Kuzu."""
    connection: AuthEntryReference[KuzuConnection]
    mapping: Nodes | Relationships
```

**Fields:**
- `connection` (auth reference to `KuzuConnection`): API server connection
- `mapping` (`Nodes | Relationships`): How to map rows to graph elements

The mapping specification is identical to Neo4j, enabling easy switching between backends.

**Example Usage:**

```python
import cocoindex
from cocoindex.auth_registry import AuthEntry

auth = cocoindex.auth_registry.GlobalAuthRegistry()
auth.kuzu_conn = AuthEntry(
    cocoindex.targets.KuzuConnection(
        api_server_url="http://localhost:8000"
    )
)

@cocoindex.flow_def
def build_graph(builder: cocoindex.FlowBuilder):
    nodes = builder.add_source(...).transform(...).collect()
    
    nodes.export(
        "documents",
        cocoindex.targets.Kuzu(
            connection=auth.kuzu_conn,
            mapping=cocoindex.targets.Nodes(label="Document")
        ),
        primary_key_fields=["doc_id"]
    )
```

Sources: [python/cocoindex/targets/_engine_builtin_specs.py:140-145]()

## Declaration Specifications

Declarations define graph schemas independently of data exports. They are used to declare nodes that are referenced by relationships but may not be directly exported from a collector.

### Neo4jDeclaration

```python
class Neo4jDeclaration(op.DeclarationSpec):
    """Declarations for Neo4j."""
    kind = "Neo4j"
    connection: AuthEntryReference[Neo4jConnection]
    nodes_label: str
    primary_key_fields: Sequence[str]
    vector_indexes: Sequence[index.VectorIndexDef] = ()
```

**Fields:**
- `connection` (auth reference to `Neo4jConnection`): Database connection
- `nodes_label` (`str`): Label for the declared node type
- `primary_key_fields` (`Sequence[str]`): Fields that uniquely identify nodes
- `vector_indexes` (`Sequence[VectorIndexDef]`): Vector indexes for semantic search

Sources: [python/cocoindex/targets/_engine_builtin_specs.py:123-131]()

### KuzuDeclaration

```python
class KuzuDeclaration(op.DeclarationSpec):
    """Declarations for Kuzu."""
    kind = "Kuzu"
    connection: AuthEntryReference[KuzuConnection]
    nodes_label: str
    primary_key_fields: Sequence[str]
```

**Fields:**
- `connection` (auth reference to `KuzuConnection`): API server connection
- `nodes_label` (`str`): Label for the declared node type
- `primary_key_fields` (`Sequence[str]`): Fields that uniquely identify nodes

**Note:** Kuzu declarations do not currently support vector indexes in the declaration spec.

Sources: [python/cocoindex/targets/_engine_builtin_specs.py:147-154]()

### When to Use Declarations

```mermaid
graph LR
    subgraph "Without Declaration"
        Rel1["Relationships export"]
        Rel1 --> Error["Error: Referenced nodes<br/>must exist or be declared"]
    end
    
    subgraph "With Declaration"
        Decl["Declaration for<br/>referenced nodes"]
        Rel2["Relationships export"]
        Decl --> Rel2
        Rel2 --> Success["Success: Schema defined"]
    end
```

Use declarations when:
1. **Relationships reference nodes not exported in the same flow**: The declaration establishes the schema for nodes that will be created by relationship mappings
2. **Multiple flows share node types**: Declare shared node schemas in a common location
3. **Defining vector indexes**: Vector indexes must be declared on nodes before data export

**Example:**

```python
@cocoindex.flow_def
def build_graph(builder: cocoindex.FlowBuilder):
    # Declare Person nodes (not directly exported)
    builder.add_declaration(
        cocoindex.targets.Neo4jDeclaration(
            connection=auth.neo4j_conn,
            nodes_label="Person",
            primary_key_fields=["person_id"],
            vector_indexes=[
                cocoindex.index.VectorIndexDef(
                    field_name="bio_embedding",
                    metric=cocoindex.index.VectorSimilarityMetric.COSINE_SIMILARITY
                )
            ]
        )
    )
    
    # Export relationships that create Person nodes
    friendships = builder.add_source(...).transform(...).collect()
    friendships.export(
        "friendships",
        cocoindex.targets.Neo4j(
            connection=auth.neo4j_conn,
            mapping=cocoindex.targets.Relationships(
                rel_type="KNOWS",
                source=cocoindex.targets.NodeFromFields(
                    label="Person",
                    fields=[cocoindex.targets.TargetFieldMapping(source="person1_id")]
                ),
                target=cocoindex.targets.NodeFromFields(
                    label="Person",
                    fields=[cocoindex.targets.TargetFieldMapping(source="person2_id")]
                )
            )
        ),
        primary_key_fields=["person1_id", "person2_id"]
    )
```

Sources: [python/cocoindex/targets/_engine_builtin_specs.py:123-154]()

## Vector Indexes in Graph Databases

Both Neo4j and Kuzu support vector indexes on node properties for semantic search capabilities.

### Vector Index Definition

```python
@dataclass
class VectorIndexDef:
    """Define a vector index on a field."""
    field_name: str
    metric: VectorSimilarityMetric
    method: VectorIndexMethod | None = None
```

**Similarity Metrics:**
- `COSINE_SIMILARITY`: Cosine similarity (normalized dot product)
- `L2_DISTANCE`: Euclidean distance
- `INNER_PRODUCT`: Dot product

**Index Methods:**
- `HnswVectorIndexMethod`: HNSW (Hierarchical Navigable Small World) algorithm
  - `m` (`int | None`): Number of bi-directional links per node
  - `ef_construction` (`int | None`): Size of dynamic candidate list during construction
- `IvfFlatVectorIndexMethod`: IVF-Flat (Inverted File with Flat vectors)
  - `lists` (`int | None`): Number of inverted lists

### Example with Vector Index

```python
@cocoindex.flow_def
def semantic_graph(builder: cocoindex.FlowBuilder):
    # Process documents with embeddings
    docs = builder.add_source(
        cocoindex.sources.LocalFile(path="docs/*.md")
    )
    
    docs_embedded = docs.transform(
        cocoindex.functions.EmbedText(
            llm_spec=llm_spec,
            input_field="content",
            output_field="embedding"
        )
    )
    
    docs_collected = docs_embedded.collect()
    
    # Export with vector index
    docs_collected.export(
        "documents",
        cocoindex.targets.Neo4j(
            connection=auth.neo4j_conn,
            mapping=cocoindex.targets.Nodes(label="Document")
        ),
        primary_key_fields=["doc_id"],
        vector_indexes=[
            cocoindex.index.VectorIndexDef(
                field_name="embedding",
                metric=cocoindex.index.VectorSimilarityMetric.COSINE_SIMILARITY,
                method=cocoindex.index.HnswVectorIndexMethod(
                    m=16,
                    ef_construction=200
                )
            )
        ]
    )
```

This creates a vector index on the `embedding` property of `Document` nodes, enabling efficient semantic search queries against the knowledge graph.

Sources: [python/cocoindex/index.py:1-51](), [python/cocoindex/targets/_engine_builtin_specs.py:72-78]()

## Graph Target Comparison

```mermaid
graph TB
    subgraph "Neo4j"
        Neo4jConn["Neo4jConnection<br/>bolt://uri, user, password, db"]
        Neo4jTarget["Neo4j TargetSpec"]
        Neo4jDecl["Neo4jDeclaration<br/>supports vector_indexes"]
        Neo4jDB[("Neo4j Database<br/>Client-Server")]
    end
    
    subgraph "Kuzu"
        KuzuConn["KuzuConnection<br/>api_server_url"]
        KuzuTarget["Kuzu TargetSpec"]
        KuzuDecl["KuzuDeclaration<br/>no vector indexes yet"]
        KuzuDB[("Kuzu Database<br/>Embedded/Server")]
    end
    
    subgraph "Shared Mapping Layer"
        NodesMapping["Nodes mapping"]
        RelMapping["Relationships mapping"]
        FieldMap["TargetFieldMapping"]
    end
    
    Neo4jTarget --> NodesMapping
    Neo4jTarget --> RelMapping
    KuzuTarget --> NodesMapping
    KuzuTarget --> RelMapping
    
    NodesMapping --> FieldMap
    RelMapping --> FieldMap
    
    Neo4jConn --> Neo4jTarget
    Neo4jConn --> Neo4jDecl
    Neo4jTarget --> Neo4jDB
    
    KuzuConn --> KuzuTarget
    KuzuConn --> KuzuDecl
    KuzuTarget --> KuzuDB
```

| Feature | Neo4j | Kuzu |
|---------|-------|------|
| **Architecture** | Client-server database | Embedded database with optional API server |
| **Connection** | Bolt protocol (`bolt://`) with auth | HTTP API server URL |
| **Deployment** | Separate Neo4j server required | Can run embedded or as server |
| **Maturity** | Industry standard, production-ready | Modern, high-performance alternative |
| **Vector Indexes** | Supported in declarations | Not yet supported in declarations |
| **Use Case** | Large-scale production graphs | Embedded apps, local development |
| **Mapping API** | Identical (`Nodes`, `Relationships`) | Identical (`Nodes`, `Relationships`) |

**Migration Between Backends:**

The identical mapping API allows easy migration between Neo4j and Kuzu:

```python
# Switch from Neo4j to Kuzu by changing only the target spec
# mapping=cocoindex.targets.Nodes(label="Entity") stays the same

# Neo4j version:
collector.export(
    "entities",
    cocoindex.targets.Neo4j(connection=auth.neo4j_conn, mapping=mapping),
    primary_key_fields=["id"]
)

# Kuzu version:
collector.export(
    "entities",
    cocoindex.targets.Kuzu(connection=auth.kuzu_conn, mapping=mapping),
    primary_key_fields=["id"]
)
```

Sources: [python/cocoindex/targets/_engine_builtin_specs.py:106-154]()

## Primary Keys and Constraints

Both graph targets require `primary_key_fields` to be specified during export. These fields:

1. **Uniquely identify nodes**: For `Nodes` mappings, primary key fields identify unique nodes
2. **Define relationship identity**: For `Relationships` mappings, primary keys identify unique relationships
3. **Enable incremental updates**: CocoIndex tracks changes using primary keys to determine which nodes/relationships to update or delete

**Node Primary Keys:**
```python
# Single field primary key
collector.export(
    "users",
    target_spec,
    primary_key_fields=["user_id"]
)

# Composite primary key
collector.export(
    "users",
    target_spec,
    primary_key_fields=["namespace", "user_id"]
)
```

**Relationship Primary Keys:**
```python
# Relationship identified by source and target
collector.export(
    "follows",
    target_spec,
    primary_key_fields=["follower_id", "followed_id"]
)

# Relationship with additional identifying field
collector.export(
    "interactions",
    target_spec,
    primary_key_fields=["user_id", "item_id", "interaction_type"]
)
```

The graph database enforces uniqueness constraints based on these primary keys during data ingestion.

Sources: [python/cocoindex/targets/_engine_builtin_specs.py:72-78](), [python/cocoindex/index.py:44-51]()

---

# Page: Custom Target Implementation

# Custom Target Implementation

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [docs/docs/core/data_types.mdx](docs/docs/core/data_types.mdx)
- [docs/docs/targets/postgres.md](docs/docs/targets/postgres.md)
- [python/cocoindex/op.py](python/cocoindex/op.py)
- [python/cocoindex/targets/__init__.py](python/cocoindex/targets/__init__.py)
- [python/cocoindex/targets/_engine_builtin_specs.py](python/cocoindex/targets/_engine_builtin_specs.py)
- [python/cocoindex/tests/test_typing.py](python/cocoindex/tests/test_typing.py)
- [python/cocoindex/typing.py](python/cocoindex/typing.py)

</details>



## Purpose and Scope

This document provides a comprehensive guide to implementing custom target connectors in CocoIndex. Custom targets allow you to export processed data to any destination—databases, cloud storage, file systems, APIs, or other external systems. This page covers the target specification system, the connector interface, lifecycle methods, and implementation patterns.

For information about using built-in targets (Postgres, Qdrant, Neo4j, Kuzu), see [Storage Backends and Export Targets](#8). For general information about the operation system and type marshalling, see [Operation System and Type Marshalling](#4).

---

## Custom Target Architecture

Custom targets in CocoIndex consist of two components that work together to handle data export:

```mermaid
graph TB
    subgraph "User Code"
        TargetSpec["TargetSpec subclass<br/>Configuration parameters"]
        ConnectorClass["Connector class<br/>@target_connector decorator"]
    end
    
    subgraph "CocoIndex Engine"
        TargetConnectorInternal["_TargetConnector<br/>python/cocoindex/op.py:780"]
        TargetConnectorContext["_TargetConnectorContext<br/>python/cocoindex/op.py:735"]
        Engine["Engine<br/>Rust execution engine"]
    end
    
    subgraph "External System"
        Database[("Database")]
        Storage[("Cloud Storage")]
        API[("API Endpoint")]
        FileSystem[("File System")]
    end
    
    TargetSpec -->|"defines config"| ConnectorClass
    ConnectorClass -->|"registered via<br/>target_connector()"| TargetConnectorInternal
    TargetConnectorInternal -->|"creates"| TargetConnectorContext
    TargetConnectorContext -->|"manages"| Engine
    ConnectorClass -->|"writes data to"| Database
    ConnectorClass -->|"writes data to"| Storage
    ConnectorClass -->|"writes data to"| API
    ConnectorClass -->|"writes data to"| FileSystem
```

**Sources:** [python/cocoindex/op.py:94-104](), [python/cocoindex/op.py:707-727](), [python/cocoindex/op.py:780-806](), [docs/docs/custom_ops/custom_targets.mdx:13-33]()

---

## Target Specification

A `TargetSpec` defines the configuration parameters for a custom target. It inherits from `cocoindex.op.TargetSpec` and uses Python's dataclass mechanism.

### Defining a TargetSpec

```python
class CustomTarget(cocoindex.op.TargetSpec):
    """
    Configuration for the custom target.
    All fields must be JSON-serializable.
    """
    location: str
    batch_size: int = 100
    timeout: datetime.timedelta | None = None
```

**Key Requirements:**
- All fields must have types serializable by the `json` module
- Subclasses can be instantiated like dataclasses: `CustomTarget(location="...", batch_size=50)`
- The spec is automatically converted to a dataclass by the `SpecMeta` metaclass

**Sources:** [python/cocoindex/op.py:94-96](), [python/cocoindex/op.py:66-83](), [docs/docs/custom_ops/custom_targets.mdx:34-58]()

---

## Target Connector Interface

The target connector implements the actual data export logic. It is decorated with `@cocoindex.op.target_connector(spec_cls=CustomTarget)` and defines both setup methods and data methods.

### Registration and Structure

```mermaid
graph LR
    subgraph "Registration Process"
        Decorator["@target_connector<br/>spec_cls=CustomTarget"]
        ConnectorClass["CustomTargetConnector<br/>User-defined class"]
        InternalConnector["_TargetConnector<br/>op.py:780"]
        Engine["_engine.register_target_connector()"]
    end
    
    Decorator -->|"wraps"| ConnectorClass
    ConnectorClass -->|"instantiates"| InternalConnector
    InternalConnector -->|"registers with"| Engine
```

**Connector Method Categories:**

| Method Category | Purpose | When Called |
|----------------|---------|-------------|
| **Setup Methods** | Manage target infrastructure | During setup, drop, or spec changes |
| **Data Methods** | Handle data mutations | During flow update operations |

**Sources:** [python/cocoindex/op.py:707-727](), [python/cocoindex/op.py:780-866](), [docs/docs/custom_ops/custom_targets.mdx:60-108]()

---

## Setup Methods

Setup methods manage the target's infrastructure lifecycle—creating, configuring, and cleaning up resources.

### Method Signature Flow

```mermaid
stateDiagram-v2
    [*] --> get_persistent_key: Target added to flow
    get_persistent_key --> get_setup_state: Key generated
    get_setup_state --> check_state_compatibility: State extracted
    check_state_compatibility --> apply_setup_change: Compatibility checked
    apply_setup_change --> [*]: Setup complete
    
    note right of get_persistent_key
        Returns stable identifier
        for this target instance
    end note
    
    note right of check_state_compatibility
        Determines if target
        needs recreation
    end note
```

### `get_persistent_key(spec, target_name) -> PersistentKey`

**Required.** Returns a unique, stable identifier for the target instance.

```python
@staticmethod
def get_persistent_key(spec: CustomTarget, target_name: str) -> PersistentKey:
    """
    Return stable identifier for this target.
    Used to track target state across runs.
    """
    return spec.location  # or any JSON-serializable value
```

**Key Points:**
- The persistent key must be stable across different runs
- If a previously existing key is no longer returned, CocoIndex assumes the target is deleted
- The return type must be JSON-serializable
- The key is passed to other setup methods (`apply_setup_change`, `describe`)

**Sources:** [python/cocoindex/op.py:809-814](), [python/cocoindex/op.py:907-914](), [docs/docs/custom_ops/custom_targets.mdx:114-119]()

### `apply_setup_change(key, previous, current) -> None`

**Required.** Applies infrastructure changes when target configuration changes.

**Method Signature:**
```python
@staticmethod
def apply_setup_change(
    key: PersistentKey,
    previous: CustomTarget | None,
    current: CustomTarget | None
) -> None:
    """
    Apply setup changes to external system.
    
    Cases:
    - previous=None, current=Spec: Create new target
    - previous=Spec, current=Spec: Update configuration
    - previous=Spec, current=None: Delete target
    """
```

**Implementation Pattern:**

| `previous` | `current` | Action |
|-----------|----------|--------|
| `None` | `CustomTarget` | Create new resources (e.g., table, directory) |
| `CustomTarget` | `CustomTarget` | Update configuration if needed |
| `CustomTarget` | `None` | Clean up and delete resources |

**Sources:** [python/cocoindex/op.py:812-814](), [python/cocoindex/op.py:986-1005](), [docs/docs/custom_ops/custom_targets.mdx:122-133]()

### `get_setup_state(spec, key_fields_schema, value_fields_schema, index_options) -> SetupState`

**Optional.** Extracts the setup state from the target spec. If not provided, the spec itself is used as the setup state.

```python
@staticmethod
def get_setup_state(
    spec: CustomTarget,
    key_fields_schema: list[FieldSchema],
    value_fields_schema: list[FieldSchema],
    index_options: IndexOptions
) -> SetupState:
    """
    Extract setup state that determines if target needs recreation.
    Return value must be JSON-serializable.
    """
    # Return state that captures infrastructure configuration
    return SetupState(...)
```

**Purpose:** The setup state represents the infrastructure configuration that, when changed, may require target recreation. This allows separating configuration that affects infrastructure (e.g., table schema) from configuration that doesn't (e.g., batch size).

**Sources:** [python/cocoindex/op.py:917-939](), [docs/docs/custom_ops/custom_targets.mdx:122-133]()

### `check_state_compatibility(desired_state, existing_state) -> TargetStateCompatibility`

**Optional.** Determines if an existing target is compatible with the desired configuration.

```python
@staticmethod
def check_state_compatibility(
    desired_state: SetupState,
    existing_state: SetupState
) -> TargetStateCompatibility:
    """
    Check if existing target can be reused.
    
    Returns:
    - COMPATIBLE: No changes needed
    - PARTIALLY_COMPATIBLE: Can be updated incrementally
    - NOT_COMPATIBLE: Must recreate target
    """
```

**Compatibility Enum:**

| Value | Meaning |
|-------|---------|
| `COMPATIBLE` | No changes needed, existing target can be used as-is |
| `PARTIALLY_COMPATIBLE` | Changes needed but can be applied incrementally |
| `NOT_COMPATIBLE` | Target must be recreated |

**Default Behavior:** If not provided, compares states for equality—`COMPATIBLE` if equal, `PARTIALLY_COMPATIBLE` otherwise.

**Sources:** [python/cocoindex/op.py:772-777](), [python/cocoindex/op.py:941-958]()

### `describe(key) -> str`

**Optional.** Returns a human-readable description of the target for logging and debugging.

```python
@staticmethod
def describe(key: PersistentKey) -> str:
    """Return human-readable description."""
    return f"CustomTarget at {key}"
```

**Sources:** [python/cocoindex/op.py:979-984](), [docs/docs/custom_ops/custom_targets.mdx:134-137]()

---

## Data Methods

Data methods handle the actual data operations—inserting, updating, and deleting records in the target.

### Data Method Execution Flow

```mermaid
sequenceDiagram
    participant Engine
    participant Connector
    participant ExternalSystem as "External System"
    
    Note over Engine: Flow update begins
    Engine->>Connector: prepare(spec)
    Connector-->>Engine: PreparedSpec
    
    loop For each mutation batch
        Engine->>Connector: mutate(*all_mutations)
        Note over Connector: all_mutations contains<br/>(PreparedSpec, dict[Key, Value|None])
        
        loop For each mutation
            alt Value is None
                Connector->>ExternalSystem: Delete row
            else Value exists
                Connector->>ExternalSystem: Upsert row
            end
        end
        
        Connector-->>Engine: Done
    end
    
    Note over Engine: Flow update complete
```

### `prepare(spec) -> PreparedSpec`

**Optional.** Performs one-time setup before mutations, such as establishing connections or validating configuration.

```python
@staticmethod
def prepare(spec: CustomTarget) -> PreparedCustomTarget:
    """
    Prepare for execution. Called once before mutations.
    
    Common uses:
    - Establish database connections
    - Validate configuration
    - Create session objects
    """
    connection = establish_connection(spec.location)
    return PreparedCustomTarget(connection=connection, spec=spec)
```

If not provided, the original spec is passed directly to `mutate()`.

**Sources:** [python/cocoindex/op.py:960-977](), [docs/docs/custom_ops/custom_targets.mdx:163-177]()

### `mutate(*all_mutations) -> None`

**Required.** Applies data changes to the target. This is the core method that writes data to the external system.

**Method Signature:**
```python
@staticmethod
def mutate(
    *all_mutations: tuple[PreparedCustomTarget, dict[DataKeyType, DataValueType | None]],
) -> None:
    """
    Apply data mutations to the target.
    
    Args:
        all_mutations: Variable number of mutation batches.
            Each batch is (PreparedSpec, mutations_dict)
            where mutations_dict maps keys to values (or None for deletion)
    """
```

**Mutation Structure:**

```mermaid
graph TD
    AllMutations["*all_mutations<br/>tuple of batches"]
    Batch1["Batch 1<br/>(PreparedSpec, dict)"]
    Batch2["Batch 2<br/>(PreparedSpec, dict)"]
    Dict["dict[DataKeyType, DataValueType | None]"]
    Entry1["'key1' → Value1"]
    Entry2["'key2' → None (delete)"]
    Entry3["'key3' → Value3"]
    
    AllMutations --> Batch1
    AllMutations --> Batch2
    Batch1 --> Dict
    Dict --> Entry1
    Dict --> Entry2
    Dict --> Entry3
```

**Sources:** [python/cocoindex/op.py:794-795](), [python/cocoindex/op.py:1006-1040](), [docs/docs/custom_ops/custom_targets.mdx:144-161]()

---

## Type System for Mutations

The mutation dictionary uses CocoIndex's type system to represent row keys and values.

### Key and Value Types

```mermaid
graph TB
    subgraph "Mutation Dictionary"
        MutDict["dict[DataKeyType, DataValueType | None]"]
    end
    
    subgraph "DataKeyType (Immutable)"
        BasicKey["Basic type<br/>str, int, etc."]
        StructKey["Struct type<br/>frozen dataclass or NamedTuple"]
    end
    
    subgraph "DataValueType"
        StructValue["Struct type<br/>dataclass, NamedTuple, or dict[str, Any]"]
        None["None<br/>(indicates deletion)"]
    end
    
    MutDict --> BasicKey
    MutDict --> StructKey
    MutDict --> StructValue
    MutDict --> None
```

### Type Mapping

| Role | Python Types | Constraints |
|------|--------------|-------------|
| **DataKeyType** (single field) | `str`, `int`, `uuid.UUID`, etc. | Must be a [key type](#key-types) |
| **DataKeyType** (multi-field) | `@dataclass(frozen=True)` or `NamedTuple` | Must be immutable |
| **DataValueType** | `dataclass`, `NamedTuple`, `dict[str, Any]` | Represents non-key fields |

### Example Type Patterns

**Single-field key with struct value:**
```python
# Key: filename (str)
# Value: author and html fields
dict[str, LocalFileTargetValues | None]

@dataclasses.dataclass
class LocalFileTargetValues:
    author: str
    html: str
```

**Multi-field key with struct value:**
```python
# Key: id_kind and id (composite)
# Value: name and email fields
dict[PersonKey, PersonData | None]

@dataclasses.dataclass(frozen=True)
class PersonKey:
    id_kind: str
    id: str

@dataclasses.dataclass
class PersonData:
    name: str
    email: str
```

**Sources:** [python/cocoindex/typing.py:213-267](), [python/cocoindex/op.py:820-891](), [docs/docs/custom_ops/custom_targets.mdx:144-161](), [docs/docs/core/data_types.mdx:218-269]()

---

## Target Connector Lifecycle

This diagram shows when each connector method is called during different CocoIndex operations.

```mermaid
stateDiagram-v2
    [*] --> FlowSetup: cocoindex flow.setup()
    
    state FlowSetup {
        [*] --> GetPersistentKey
        GetPersistentKey --> GetSetupState
        GetSetupState --> CheckCompatibility
        CheckCompatibility --> ApplySetupChange
        ApplySetupChange --> [*]
    }
    
    FlowSetup --> Ready: Setup complete
    
    state FlowUpdate {
        [*] --> Prepare
        Prepare --> Mutate1: First batch
        Mutate1 --> Mutate2: Second batch
        Mutate2 --> MutateN: More batches...
        MutateN --> [*]
    }
    
    Ready --> FlowUpdate: cocoindex flow.update()
    FlowUpdate --> Ready: Update complete
    
    state FlowDrop {
        [*] --> GetPersistentKey2: Get keys
        GetPersistentKey2 --> ApplySetupChange2: current=None
        ApplySetupChange2 --> [*]
    }
    
    Ready --> FlowDrop: cocoindex flow.drop()
    FlowDrop --> [*]: Dropped
    
    Ready --> FlowSpecChange: Spec modified
    
    state FlowSpecChange {
        [*] --> GetPersistentKey3
        GetPersistentKey3 --> GetSetupState2
        GetSetupState2 --> CheckCompatibility2
        CheckCompatibility2 --> ApplySetupChange3
        ApplySetupChange3 --> [*]
    }
    
    FlowSpecChange --> Ready: Setup updated
```

**Sources:** [python/cocoindex/op.py:867-1068](), [docs/docs/custom_ops/custom_targets.mdx:20-32]()

---

## Internal Implementation Details

### _TargetConnector Class

The `_TargetConnector` class in [python/cocoindex/op.py:780-1068]() wraps user-defined connector classes and provides the interface to the Rust engine.

**Key Responsibilities:**

| Method | Purpose |
|--------|---------|
| `create_export_context()` | Creates context with decoded schemas and type information |
| `get_persistent_key()` | Calls user's method and serializes result |
| `get_setup_state()` | Extracts setup state using user's method or spec |
| `check_state_compatibility()` | Checks compatibility using user's method or default logic |
| `prepare_async()` | Calls optional `prepare()` method asynchronously |
| `apply_setup_changes_async()` | Applies batched setup changes |
| `_decode_mutation()` | Converts engine format to Python objects |

### Type Analysis for Mutate Method

The `_analyze_mutate_mutation_type()` method [python/cocoindex/op.py:824-865]() extracts type information from the `mutate()` signature:

```python
# Expected signature:
def mutate(*args: tuple[SpecType, dict[KeyType, ValueType]]) -> None:
    ...
```

**Validation:**
- Must have exactly one parameter of type `*args`
- Parameter annotation must be `tuple[SpecType, dict[KeyType, ValueType]]`
- Extracts `KeyType` and `ValueType` for encoder/decoder generation

**Sources:** [python/cocoindex/op.py:824-865](), [python/cocoindex/op.py:1006-1040]()

### Mutation Decoding Process

```mermaid
graph TB
    EngineFormat["Engine Format<br/>list[(key, value_or_none)]"]
    KeyDecoder["make_engine_key_decoder()"]
    ValueDecoder["make_engine_struct_decoder()"]
    PythonKey["Python Key Object"]
    PythonValue["Python Value Object or None"]
    PythonDict["dict[Key, Value | None]"]
    
    EngineFormat --> KeyDecoder
    EngineFormat --> ValueDecoder
    KeyDecoder --> PythonKey
    ValueDecoder --> PythonValue
    PythonKey --> PythonDict
    PythonValue --> PythonDict
```

**Sources:** [python/cocoindex/op.py:1006-1021](), [python/cocoindex/engine_value.py]()

---

## Best Practices

### Idempotency

Both `apply_setup_change()` and `mutate()` must be idempotent—calling them multiple times with the same arguments should produce the same effect.

**Why:** CocoIndex may retry operations after failures or crashes. Idempotency ensures the system can safely resume from intermediate states.

**Examples:**

```python
# Setup method - idempotent directory creation
@staticmethod
def apply_setup_change(key: str, previous: CustomTarget | None, current: CustomTarget | None) -> None:
    if current is None:
        # Delete: no-op if directory doesn't exist
        shutil.rmtree(key, ignore_errors=True)
    elif previous is None:
        # Create: no-op if directory exists
        os.makedirs(key, exist_ok=True)

# Mutate method - idempotent row operations
@staticmethod
def mutate(*all_mutations) -> None:
    for spec, mutations in all_mutations:
        for key, value in mutations.items():
            if value is None:
                # Delete: no-op if row doesn't exist
                cursor.execute("DELETE FROM table WHERE id = ?", (key,))
            else:
                # Upsert: idempotent
                cursor.execute(
                    "INSERT INTO table VALUES (?, ?) ON CONFLICT DO UPDATE",
                    (key, value)
                )
```

**Sources:** [docs/docs/custom_ops/custom_targets.mdx:180-190]()

### Error Handling

**Setup Methods:** Raise exceptions for configuration errors or infrastructure failures. CocoIndex will propagate these to the user.

**Data Methods:** 
- For transient errors (network timeouts), consider retry logic within the method
- For data errors, decide whether to fail the entire batch or skip invalid rows
- Log errors with sufficient context for debugging

### Batch Processing

The `mutate()` method receives multiple batches in `all_mutations`. This allows for:
- Processing multiple targets in a single call (when specs are identical)
- Optimizing database round-trips by batching operations
- Implementing transaction boundaries per batch

**Sources:** [python/cocoindex/op.py:1006-1040](), [docs/docs/custom_ops/custom_targets.mdx:144-161]()

### Resource Management

Use the `prepare()` method to establish long-lived connections:

```python
@staticmethod
def prepare(spec: CustomTarget) -> PreparedCustomTarget:
    connection = create_connection(spec.database_url)
    return PreparedCustomTarget(connection=connection)

@staticmethod
def mutate(*all_mutations: tuple[PreparedCustomTarget, dict]) -> None:
    for prepared, mutations in all_mutations:
        # Reuse connection from prepare()
        with prepared.connection.cursor() as cursor:
            # Apply mutations
            ...
```

**Sources:** [python/cocoindex/op.py:960-977](), [docs/docs/custom_ops/custom_targets.mdx:163-177]()

---

## Complete Example

This example implements a custom target that exports data to individual JSON files in a directory.

```python
import os
import json
import dataclasses
from typing import NamedTuple
import cocoindex

# 1. Define target spec
class JsonFileTarget(cocoindex.op.TargetSpec):
    """Export data to JSON files in a directory."""
    output_dir: str
    pretty: bool = False

# 2. Define value dataclass
@dataclasses.dataclass
class DocumentData:
    """Document fields to export."""
    title: str
    content: str
    author: str

# 3. Define prepared spec (with connection state)
class PreparedJsonFileTarget(NamedTuple):
    output_dir: str
    pretty: bool

# 4. Implement connector
@cocoindex.op.target_connector(spec_cls=JsonFileTarget)
class JsonFileTargetConnector:
    @staticmethod
    def get_persistent_key(spec: JsonFileTarget, target_name: str) -> str:
        """Use directory path as persistent key."""
        return os.path.abspath(spec.output_dir)
    
    @staticmethod
    def apply_setup_change(
        key: str,
        previous: JsonFileTarget | None,
        current: JsonFileTarget | None
    ) -> None:
        """Create or delete output directory."""
        if current is None:
            # Delete directory (idempotent)
            import shutil
            shutil.rmtree(key, ignore_errors=True)
        elif previous is None:
            # Create directory (idempotent)
            os.makedirs(key, exist_ok=True)
    
    @staticmethod
    def describe(key: str) -> str:
        return f"JSON files in {key}"
    
    @staticmethod
    def prepare(spec: JsonFileTarget) -> PreparedJsonFileTarget:
        """Validate directory exists."""
        if not os.path.isdir(spec.output_dir):
            raise ValueError(f"Directory does not exist: {spec.output_dir}")
        return PreparedJsonFileTarget(
            output_dir=spec.output_dir,
            pretty=spec.pretty
        )
    
    @staticmethod
    def mutate(
        *all_mutations: tuple[PreparedJsonFileTarget, dict[str, DocumentData | None]],
    ) -> None:
        """Write or delete JSON files."""
        for prepared, mutations in all_mutations:
            for doc_id, doc_data in mutations.items():
                file_path = os.path.join(prepared.output_dir, f"{doc_id}.json")
                
                if doc_data is None:
                    # Delete file (idempotent)
                    if os.path.exists(file_path):
                        os.remove(file_path)
                else:
                    # Write file (idempotent)
                    data = dataclasses.asdict(doc_data)
                    indent = 2 if prepared.pretty else None
                    with open(file_path, 'w') as f:
                        json.dump(data, f, indent=indent)
```

**Usage in a flow:**

```python
@cocoindex.flow_def(name="ExportToJsonFiles")
def export_flow(flow_builder: cocoindex.FlowBuilder, data_scope: cocoindex.DataScope) -> None:
    # Add source
    data_scope["docs"] = flow_builder.add_source(...)
    
    # Create collector
    output = data_scope.add_collector()
    
    # Process and collect data
    with data_scope["docs"].row() as doc:
        output.collect(
            doc_id=doc["id"],
            title=doc["title"],
            content=doc["processed_content"],
            author=doc["author"]
        )
    
    # Export to custom target
    output.export(
        "JsonFiles",
        JsonFileTarget(output_dir="./output", pretty=True),
        primary_key_fields=["doc_id"]
    )
```

**Sources:** [docs/docs/custom_ops/custom_targets.mdx:192-282](), [examples/custom_output_files]()

---

## Registration Mechanism

The `@target_connector` decorator registers the connector with the engine:

```mermaid
sequenceDiagram
    participant User as "User Code"
    participant Decorator as "@target_connector"
    participant Internal as "_TargetConnector"
    participant Engine as "_engine"
    
    User->>Decorator: @target_connector(spec_cls=CustomTarget)
    Decorator->>Decorator: Validate spec_cls is TargetSpec
    Decorator->>Internal: Create _TargetConnector instance
    Internal->>Internal: Analyze mutate() signature
    Internal->>Internal: Extract KeyType, ValueType
    Decorator->>Engine: register_target_connector(name, connector)
    Engine-->>Decorator: Registration complete
    Decorator-->>User: Return connector class
```

**Key Steps:**
1. Decorator validates `spec_cls` is a `TargetSpec` subclass [python/cocoindex/op.py:718-719]()
2. Creates `_TargetConnector` wrapper [python/cocoindex/op.py:723]()
3. Analyzes `mutate()` signature to extract type information [python/cocoindex/op.py:820-865]()
4. Registers with engine using spec class name [python/cocoindex/op.py:724]()

**Sources:** [python/cocoindex/op.py:707-727](), [python/cocoindex/op.py:780-806]()

---

## Summary Table

| Component | Required | Purpose | Key Methods/Fields |
|-----------|----------|---------|-------------------|
| **TargetSpec** | Yes | Configuration parameters | Fields with JSON-serializable types |
| **Target Connector** | Yes | Data export implementation | `get_persistent_key()`, `apply_setup_change()`, `mutate()` |
| **Setup Methods** | Partially | Infrastructure management | `get_persistent_key()`, `apply_setup_change()` required; others optional |
| **Data Methods** | Partially | Data operations | `mutate()` required; `prepare()` optional |
| **Persistent Key** | Yes | Stable target identifier | JSON-serializable value |
| **Setup State** | Optional | Infrastructure configuration | Extracted by `get_setup_state()` or defaults to spec |
| **Prepared Spec** | Optional | Runtime state | Returned by `prepare()` |

**Sources:** [python/cocoindex/op.py:94-96](), [python/cocoindex/op.py:707-1068](), [docs/docs/custom_ops/custom_targets.mdx:1-282]()

---

# Page: Incremental Processing and State Management

# Incremental Processing and State Management

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [Cargo.lock](Cargo.lock)
- [Cargo.toml](Cargo.toml)
- [python/cocoindex/__init__.py](python/cocoindex/__init__.py)
- [python/cocoindex/cli.py](python/cocoindex/cli.py)
- [python/cocoindex/flow.py](python/cocoindex/flow.py)
- [python/cocoindex/lib.py](python/cocoindex/lib.py)
- [python/cocoindex/setup.py](python/cocoindex/setup.py)
- [rust/cocoindex/Cargo.toml](rust/cocoindex/Cargo.toml)
- [rust/cocoindex/src/lib_context.rs](rust/cocoindex/src/lib_context.rs)
- [rust/cocoindex/src/prelude.rs](rust/cocoindex/src/prelude.rs)
- [rust/py_utils/Cargo.toml](rust/py_utils/Cargo.toml)
- [rust/py_utils/src/future.rs](rust/py_utils/src/future.rs)
- [rust/utils/Cargo.toml](rust/utils/Cargo.toml)
- [rust/utils/src/error.rs](rust/utils/src/error.rs)
- [rust/utils/src/prelude.rs](rust/utils/src/prelude.rs)
- [rust/utils/src/retryable.rs](rust/utils/src/retryable.rs)

</details>



CocoIndex achieves efficient data updates through incremental processing, which detects and processes only changed source data. The system maintains processing state in an internal PostgreSQL database, using checksums and ordinals to minimize recomputation.

## Overview

When you run `cocoindex update`, CocoIndex doesn't reprocess all data from scratch. Instead, it:

1. **Tracks previously processed data** in an internal Postgres database
2. **Detects changes** using a three-level strategy (ordinals, content checksums, logic checksums)
3. **Reprocesses only what changed** - new rows, modified content, or rows affected by flow logic changes
4. **Maintains consistency** through atomic four-phase updates per source row

This approach ensures minimal computational overhead while keeping target data synchronized with sources.

```mermaid
graph LR
    Source["Source Data"]
    Detect["Change<br/>Detection"]
    Tracking["source_tracking<br/>table"]
    Process["Transform<br/>Pipeline"]
    Target["Target<br/>Storage"]
    
    Source -->|"Read"| Detect
    Tracking -->|"Compare"| Detect
    Detect -->|"Changed rows<br/>only"| Process
    Process -->|"Upsert/Delete"| Target
    Process -->|"Update state"| Tracking
```

**Core Principle**: CocoIndex uses content-addressed caching. Each source row's processing state is keyed by `(source_id, source_key)` and versioned by content checksums, enabling precise change detection.

Sources: [python/cocoindex/flow.py:766-787](), [README.md:94-105]()

## Internal State Storage

CocoIndex uses a dedicated PostgreSQL database (configured via `COCOINDEX_DATABASE_URL`) to persist incremental processing state. This is separate from your data sources and target databases.

### Database Components

The internal database contains:

| Component | Purpose |
|-----------|---------|
| `source_tracking` table | Tracks processing state for each source row |
| `flow_contexts` table | Stores flow metadata and setup state |
| `memoization_info` | Caches intermediate computation results |
| Transaction isolation | Ensures consistency during concurrent updates |

### Why PostgreSQL?

PostgreSQL provides:
- **ACID transactions** for atomic state updates
- **Row-level locking** (`SELECT ... FOR UPDATE`) for concurrency control
- **JSONB columns** for flexible metadata storage (checksums, target keys)
- **Connection pooling** via the Rust `deadpool-postgres` library

The internal database is automatically initialized when you first run `cocoindex setup` or `cocoindex update`.

Sources: [python/cocoindex/setting.py:12-56](), [docs/docs/getting_started/quickstart.md:142-145]()

## Change Detection Strategy

CocoIndex uses a three-level hierarchy to determine whether a source row needs reprocessing. Each level trades precision for computational cost, with faster checks attempted first.

### Three-Level Detection Hierarchy

**Level 1: Ordinal-Based Detection**

Sources like PostgreSQL and S3 provide ordinals (modification timestamps, etags) that indicate when data changed. If the ordinal hasn't advanced, CocoIndex skips reprocessing without reading content.

```mermaid
graph TB
    ReadTracking["Read source_tracking<br/>table row"]
    CompareOrdinal{"Current ordinal<br/>vs stored?"}
    Skip["SKIP: num_no_change++"]
    CheckContent["Level 2:<br/>Content Fingerprint"]
    
    ReadTracking --> CompareOrdinal
    CompareOrdinal -->|"Same or older"| Skip
    CompareOrdinal -->|"Advanced"| CheckContent
```

**Example ordinal comparison logic:**

| Stored | Current | Logic Changed? | Action |
|--------|---------|----------------|--------|
| `100` | `100` | No | Skip (no change) |
| `100` | `105` | No | Proceed to Level 2 |
| `100` | `100` | Yes | Process (logic changed) |
| `None` | `105` | - | Process (new row) |

**Level 2: Content Fingerprint Detection**

When an ordinal advances, CocoIndex computes a SHA-256 checksum of the source row's content. If it matches the stored checksum, the row is skipped (handles cases where metadata changed but content didn't).

```mermaid
sequenceDiagram
    participant Row as "RowIndexer"
    participant Hash as "SHA-256<br/>Hasher"
    participant DB as "source_tracking"
    
    Row->>Hash: Compute content hash
    Hash-->>Row: content_fp
    Row->>DB: BEGIN; SELECT ... FOR UPDATE
    DB-->>Row: processed_source_fp
    
    alt Fingerprints match
        Row->>DB: UPDATE ordinal only
        Row->>DB: COMMIT
        Note over Row: Skip (num_no_change++)
    else Fingerprints differ
        Row->>DB: ROLLBACK
        Note over Row: Proceed to Level 3
    end
```

The `try_collapse()` method in [rust/cocoindex/src/execution/row_indexer.rs:428-530]() implements this optimization by updating only the ordinal if content is unchanged.

**Level 3: Logic Fingerprint Detection**

The logic fingerprint is a checksum of the entire flow definition:
- Function specs (`SplitRecursively`, `SentenceTransformerEmbed`, etc.)
- Export configurations (target specs, primary keys, vector indexes)
- Schema versions

When flow logic changes (e.g., you modify chunk size or change the embedding model), all source rows are marked for reprocessing, even if source content is unchanged.

```mermaid
graph LR
    FlowDef["@flow_def<br/>function body"]
    FuncSpecs["FunctionSpec<br/>instances"]
    ExportSpecs["TargetSpec<br/>instances"]
    Schemas["Field schemas"]
    
    FlowDef --> LogicFP["ExecutionPlanLogicFingerprint<br/>(SHA-256)"]
    FuncSpecs --> LogicFP
    ExportSpecs --> LogicFP
    Schemas --> LogicFP
    
    LogicFP --> Stored["process_logic_fingerprint<br/>(stored)"]
    LogicFP --> Current["Logic fingerprint<br/>(current)"]
    
    Stored --> Compare{"Match?"}
    Current --> Compare
    Compare -->|"Yes"| SkipReproc["Use cached<br/>results"]
    Compare -->|"No"| Reproc["Reprocess all<br/>affected rows"]
```

Sources: [python/cocoindex/flow.py:766-787](), [python/cocoindex/cli.py:439-485]()

## Source Tracking Table

The `source_tracking` table is the heart of incremental processing. Each row represents the processing state of one source row, keyed by `(source_id, source_key)`.

### Schema and Key Columns

**Tracking Table: `source_tracking`**

| Column | Type | Purpose |
|--------|------|---------|
| `source_id` | INTEGER | Identifies which source connector (e.g., LocalFile, Postgres) |
| `source_key` | JSONB | Primary key from the source (e.g., `{"filename": "doc.md"}`) |
| `processed_source_ordinal` | BIGINT | Last processed ordinal (timestamp, etag, etc.) |
| `processed_source_fp` | BYTEA | SHA-256 of last processed content |
| `process_logic_fingerprint` | BYTEA | SHA-256 of flow logic at last processing |
| `process_ordinal` | BIGINT | Monotonic sequence number (total ordering) |
| `max_process_ordinal` | BIGINT | Highest attempted processing (for error tracking) |
| `process_time_micros` | BIGINT | Unix timestamp of last successful processing |
| `target_keys` | JSONB | Keys in target storage (e.g., `{"doc_embeddings": [...]}`) |
| `staging_target_keys` | JSONB | Target keys staged during precommit phase |
| `memoization_info` | JSONB | Cached intermediate computation results |

**Primary Key**: `(source_id, source_key)`

**Key Design Choices:**

- **JSONB for keys**: Source keys can be composite (e.g., `{"bucket": "data", "key": "file.txt"}`)
- **Ordinal sequencing**: `process_ordinal` provides total ordering of all updates across all sources
- **Staged mutations**: `staging_target_keys` enables atomic two-phase updates

### In-Memory State: `SourceIndexingContext`

CocoIndex maintains in-memory state per source to coordinate concurrent updates and track scan generations.

**Tracking per-source state: `SourceIndexingContext`**

```mermaid
graph TB
    subgraph Context["SourceIndexingContext"]
        State["Mutex&lt;SourceIndexingState&gt;"]
        UpdateSem["update_sem:<br/>Semaphore(1)"]
        Pending["pending_update:<br/>Option&lt;JoinHandle&gt;"]
    end
    
    subgraph InnerState["SourceIndexingState"]
        Rows["rows: HashMap&lt;KeyValue,<br/>SourceRowIndexingState&gt;"]
        Gen["scan_generation: u64"]
        Retry["rows_to_retry: Vec"]
    end
    
    subgraph RowState["SourceRowIndexingState"]
        Version["source_version:<br/>(ordinal, kind)"]
        Sem["processing_sem:<br/>Semaphore(1)"]
        Touch["touched_generation"]
    end
    
    Context --> InnerState
    InnerState --> RowState
```

**Scan Generation Mechanism**: Each full source scan increments `scan_generation`. Rows not "touched" in the current generation are marked for deletion (they disappeared from the source).

**Per-row semaphore**: `processing_sem` prevents concurrent processing of the same source row, avoiding race conditions when multiple change notifications arrive simultaneously.

Sources: [python/cocoindex/flow.py:598-688](), [python/cocoindex/cli.py:391-485]()

## Batch Update Flow

When you run `cocoindex update main.py`, CocoIndex executes a one-time batch update:

1. **Read source list**: Fetch all source keys with ordinals (optimized read)
2. **Compare with tracking**: Check each key against `source_tracking` table
3. **Process changed rows**: Apply transformations and update targets
4. **Detect deletions**: Rows not seen in this scan are deleted from targets
5. **Return statistics**: Report inserted, updated, deleted, and unchanged counts

**Batch Update Sequence: `Flow.update()`**

```mermaid
sequenceDiagram
    participant CLI as "cocoindex update"
    participant Flow as "Flow.update()"
    participant Source as "SourceIndexer"
    participant Track as "source_tracking"
    participant Process as "process_source_row()"
    
    CLI->>Flow: flow.update()
    Flow->>Source: update_once()
    Source->>Source: Increment scan_generation
    
    loop For each source batch
        Source->>Source: list(include_ordinal=true,<br/>include_value=false)
        Note over Source: Optimized: read only ordinals<br/>when little change expected
        
        par Parallel row processing
            Source->>Track: Read tracking info
            Track-->>Source: (ordinal, fingerprints)
            
            alt Ordinal unchanged
                Source->>Source: Skip (num_no_change++)
            else Ordinal changed
                Source->>Process: Fetch value & process
            end
        end
    end
    
    Source->>Source: Detect deletions<br/>(touched_generation < scan_generation)
    
    loop For each deleted row
        Source->>Process: process_source_row(NonExistence)
    end
    
    Source-->>Flow: IndexUpdateInfo
    Flow-->>CLI: Print statistics
```

**Optimized Reading**: When `expect_little_diff` is true, CocoIndex reads only ordinals initially, fetching full content only for changed rows. This dramatically reduces I/O for large sources with few changes.

Sources: [python/cocoindex/flow.py:766-787](), [python/cocoindex/cli.py:439-485]()

## Four-Phase Update Process

Each source row update proceeds through four atomic phases, ensuring consistency even under concurrent updates and failures.

### Phase 1: Check and Optimize

Read tracking state and apply the three-level change detection strategy to skip unnecessary work.

**Phase 1 Flow: Change Detection**

```mermaid
graph TB
    Start["process_source_row()"]
    ReadTrack["Read source_tracking<br/>FOR UPDATE"]
    L1{"Level 1:<br/>Ordinal?"}
    L2["Level 2:<br/>Compute content_fp"]
    L2Check{"FP match?"}
    L3["Level 3:<br/>Fetch value<br/>& evaluate"]
    Skip["SKIP:<br/>num_no_change++"]
    OptUpdate["UPDATE ordinal<br/>COMMIT"]
    Precommit["Phase 2:<br/>Precommit"]
    
    Start --> ReadTrack
    ReadTrack --> L1
    L1 -->|"Unchanged"| Skip
    L1 -->|"Advanced"| L2
    L2 --> L2Check
    L2Check -->|"Yes"| OptUpdate
    L2Check -->|"No"| L3
    L3 --> Precommit
```

**Key optimization**: The `try_collapse()` method attempts to update only the ordinal without full reprocessing when content is unchanged.

Sources: [python/cocoindex/flow.py:766-787]()

### Phase 2: Precommit

Stage target keys in a transaction, preparing mutations while holding row-level locks to prevent race conditions.

**Phase 2: Precommit Transaction**

```mermaid
sequenceDiagram
    participant Row as "RowIndexer"
    participant DB as "source_tracking"
    participant Builder as "TrackingInfoBuilder"
    
    Row->>DB: BEGIN TRANSACTION
    Row->>DB: SELECT ... FOR UPDATE<br/>(row-level lock)
    
    alt Newer ordinal exists
        Note over Row: Another update won the race
        Row->>DB: ROLLBACK
        Row-->>Row: Skip this update
    end
    
    Row->>Builder: Load target_keys (old)
    Row->>Builder: Compute target_keys (new)
    
    loop For each export target
        Builder->>Builder: Diff old vs new keys
        Builder->>Builder: Mark for upsert/delete
    end
    
    Row->>DB: UPDATE staging_target_keys<br/>SET process_ordinal++
    Row->>DB: COMMIT
    
    Note over Row: Staged mutations ready
```

The `process_ordinal` field is incremented and provides a total ordering of all updates across all sources and flows.

**Why staging?** If Phase 3 (mutation) fails, the tracking table is not corrupted. The next update will retry using the staged keys.

Sources: [python/cocoindex/flow.py:401-447]()

### Phase 3: Mutate

Apply mutations to all target storage systems in parallel. This phase does not modify the tracking table.

**Phase 3: Parallel Target Mutations**

```mermaid
graph TB
    Staged["staging_target_keys:<br/>prepared mutations"]
    Group["Group by target<br/>(Postgres, Qdrant, Neo4j)"]
    
    subgraph Parallel["Parallel Execution"]
        PG["Postgres:<br/>UPSERT + DELETE"]
        QD["Qdrant:<br/>upsert_points()<br/>delete_points()"]
        Neo["Neo4j:<br/>MERGE nodes<br/>DELETE rels"]
    end
    
    Staged --> Group
    Group --> PG
    Group --> QD
    Group --> Neo
    
    PG --> Join["await all"]
    QD --> Join
    Neo --> Join
    
    Join --> Phase4["Phase 4:<br/>Commit"]
```

**Error handling**: If a target mutation fails, the error is logged and `num_errors` is incremented, but other targets are still updated. The tracking table is not updated, so the next update will retry.

Sources: [python/cocoindex/flow.py:401-447]()

### Phase 4: Commit

Finalize the tracking record, copying `staging_target_keys` to `target_keys` and updating all fingerprints.

**Phase 4: Finalize Tracking**

```mermaid
sequenceDiagram
    participant Row as "RowIndexer"
    participant DB as "source_tracking"
    participant Stats as "IndexUpdateInfo"
    
    Row->>DB: UPDATE source_tracking SET<br/>  processed_source_ordinal = current<br/>  processed_source_fp = computed<br/>  process_logic_fingerprint = current<br/>  target_keys = staging_target_keys<br/>  memoization_info = cached_results<br/>WHERE (source_id, source_key)
    
    Row->>Stats: Increment counter
    
    alt New row
        Stats->>Stats: num_insertions++
    else Updated row
        alt Logic changed
            Stats->>Stats: num_reprocesses++
        else Content changed
            Stats->>Stats: num_updates++
        end
    else Deleted row
        Stats->>Stats: num_deletions++
    end
    
    Note over Row: Update complete
```

**Atomicity**: All four phases are designed such that interruptions (crashes, network failures) leave the system in a consistent state for retry.

Sources: [python/cocoindex/flow.py:766-787]()

## Update Statistics

After `cocoindex update` completes, it prints statistics showing what changed:

```
documents: 3 added, 0 removed, 0 updated
```

**Index Update Statistics: `IndexUpdateInfo`**

```mermaid
graph TB
    subgraph Stats["IndexUpdateInfo"]
        Insert["num_insertions:<br/>New source rows"]
        Update["num_updates:<br/>Content changed"]
        Delete["num_deletions:<br/>Rows disappeared"]
        NoChange["num_no_change:<br/>Skipped (unchanged)"]
        Reproc["num_reprocesses:<br/>Logic changed"]
        Errors["num_errors:<br/>Failed rows"]
    end
    
    subgraph InProcess["In-Process Tracking"]
        Starts["num_starts:<br/>Rows started"]
        Ends["num_ends:<br/>Rows completed"]
        Current["Currently processing:<br/>num_starts - num_ends"]
    end
    
    Stats --> Display["cocoindex update output"]
    InProcess --> Display
```

**Statistics Breakdown:**

| Counter | Meaning | When Incremented |
|---------|---------|------------------|
| `num_insertions` | New rows | First time processing a source key |
| `num_updates` | Content changed | Ordinal or content fingerprint advanced, logic same |
| `num_reprocesses` | Logic changed | Flow definition changed, forcing recomputation |
| `num_deletions` | Rows disappeared | Source key no longer exists in source |
| `num_no_change` | Skipped | Ordinal unchanged, content same, logic same |
| `num_errors` | Failed | Any phase raised an exception |

**In-process counters**: `num_starts` and `num_ends` track concurrent processing. The difference (`num_starts - num_ends`) shows how many rows are currently being processed.

Sources: [python/cocoindex/flow.py:766-787](), [docs/docs/getting_started/quickstart.md:150-158]()

## Concurrency Control

### Row-Level Locking

Each source row has a dedicated semaphore preventing concurrent processing of the same key:

```mermaid
graph TB
    Row1["Row key: 'user_123'"]
    Row2["Row key: 'user_456'"]
    Row3["Row key: 'user_789'"]
    
    Sem1["Semaphore(1)"]
    Sem2["Semaphore(1)"]
    Sem3["Semaphore(1)"]
    
    Row1 --> Sem1
    Row2 --> Sem2
    Row3 --> Sem3
    
    Sem1 -.->|"Guards"| Process1["process_source_row()"]
    Sem2 -.->|"Guards"| Process2["process_source_row()"]
    Sem3 -.->|"Guards"| Process3["process_source_row()"]
```

This prevents data races when multiple change notifications arrive for the same row.

Sources: [src/execution/source_indexer.rs:35-46](), [src/execution/source_indexer.rs:200-203]()

### Source-Level Update Coordination

The `update_sem` semaphore ensures only one full source scan runs at a time, preventing redundant list operations:

```rust
let _permit = slf.update_sem.acquire().await?;
{
    let mut pending_update = slf.pending_update.lock().unwrap();
    *pending_update = None;
}
slf.update_once(&pool, &update_stats, &update_options).await?;
```

Multiple callers to `update()` share the same pending update future, coalescing concurrent requests.

Sources: [src/execution/source_indexer.rs:526-565]()

## Error Handling and Retry

### Row-Level Error Isolation

Processing errors are isolated to individual rows and do not halt the entire update:

```rust
if let Err(e) = process_and_ack.await {
    update_stats.num_errors.inc(1);
    error!("Error in processing row from flow `{flow}` source `{source}` with key: {key}");
}
```

Failed rows are tracked via `max_process_ordinal` to enable retry on subsequent updates.

Sources: [src/execution/source_indexer.rs:512-523](), [src/execution/source_indexer.rs:486-496]()

### Change Stream Retry

Change streams implement exponential backoff retry with configurable limits:

```mermaid
sequenceDiagram
    participant Task as "Change Stream Task"
    participant Stream as "ChangeStream"
    participant Retry as "Retry Logic"
    
    loop Forever
        Task->>Stream: next()
        
        alt Success
            Stream-->>Task: Change message
            Task->>Task: Process change
        else Retryable error
            Stream-->>Task: Error
            Task->>Retry: Check backoff
            Retry->>Retry: Sleep (5s..60s)
            Retry-->>Task: Retry
        else Non-retryable error
            Stream-->>Task: Error
            Task->>Task: Log error, continue
        end
    end
```

This ensures transient network failures don't permanently disable live updates.

Sources: [src/execution/live_updater.rs:152-179](), [src/utils/retryable.rs:111-158]()

---

# Page: Live Updates and Change Capture

# Live Updates and Change Capture

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [docs/docs/core/basics.md](docs/docs/core/basics.md)
- [docs/docs/core/flow_def.mdx](docs/docs/core/flow_def.mdx)
- [docs/docs/core/flow_methods.mdx](docs/docs/core/flow_methods.mdx)
- [python/cocoindex/__init__.py](python/cocoindex/__init__.py)
- [python/cocoindex/cli.py](python/cocoindex/cli.py)
- [python/cocoindex/flow.py](python/cocoindex/flow.py)
- [python/cocoindex/lib.py](python/cocoindex/lib.py)
- [python/cocoindex/setup.py](python/cocoindex/setup.py)

</details>



## Purpose and Scope

This document describes CocoIndex's live update system, which enables continuous monitoring of data sources and automatic propagation of changes to target indexes. Live updates allow flows to react to source changes in real-time rather than requiring manual batch updates. This system builds on top of CocoIndex's incremental processing capabilities to provide efficient, long-running data synchronization.

For information about one-time batch updates, see [Flow Lifecycle and Methods](#3.3). For details on incremental processing and state management, see [Incremental Processing and State Management](#9). For setup and teardown of flow backends, see [Setup and Drop Operations](#9.2).

Sources: [docs/docs/core/flow_methods.mdx:1-367](), [docs/docs/core/basics.md:63-97]()

---

## Overview

Live updates transform CocoIndex flows from batch processing systems into continuously synchronized data pipelines. When live update mode is enabled, CocoIndex:

1. Performs an initial one-time update to bring targets up-to-date
2. Activates change capture mechanisms for all configured sources
3. Continuously monitors sources for changes
4. Applies incremental updates to targets as changes are detected
5. Continues running until explicitly aborted

The system only updates data that has changed, leveraging CocoIndex's checksum-based incremental processing to minimize computation and I/O.

Sources: [docs/docs/core/flow_methods.mdx:110-131](), [python/cocoindex/flow.py:598-688]()

---

## Change Capture Mechanisms

CocoIndex supports two categories of change capture mechanisms that can be used independently or together:

### Refresh Interval

The `refresh_interval` parameter provides a universal change detection mechanism applicable to any data source. When specified on a source, CocoIndex periodically lists all rows in the source and compares metadata (such as last modified time) against cached state to identify changes.

**Configuration:**

```python
data_scope["documents"] = flow_builder.add_source(
    cocoindex.sources.LocalFile(...),
    refresh_interval=datetime.timedelta(minutes=5)
)
```

During each refresh cycle:
- CocoIndex lists all source keys
- Compares checksums/timestamps with cached values
- Identifies added, modified, or deleted rows
- Triggers reprocessing only for changed data

If no changes are detected, only the listing operation occurs—typically inexpensive for most sources.

Sources: [docs/docs/core/flow_def.mdx:140-168](), [python/cocoindex/flow.py:476-543]()

### Source-Specific Mechanisms

Individual source types provide optimized change capture mechanisms that avoid periodic polling:

| Source Type | Mechanism | Description |
|------------|-----------|-------------|
| **Postgres** | LISTEN/NOTIFY | Subscribes to PostgreSQL change notifications for real-time updates |
| **AmazonS3** | Event Notifications | Watches S3 bucket events (ObjectCreated, ObjectRemoved) via EventBridge or SNS |
| **GoogleDrive** | Changes API | Polls Google Drive's changes feed for recent modifications |

These mechanisms provide lower latency change detection compared to refresh intervals and reduce unnecessary listing operations. Refer to individual source documentation for configuration details.

Sources: [docs/docs/core/flow_methods.mdx:205-212]()

---

## FlowLiveUpdater Architecture

The `FlowLiveUpdater` class orchestrates continuous updates by managing background threads in the Rust engine that monitor sources and apply updates.

```mermaid
graph TB
    subgraph "Python Layer"
        FlowLiveUpdater["FlowLiveUpdater"]
        Options["FlowLiveUpdaterOptions<br/>- live_mode<br/>- reexport_targets<br/>- print_stats"]
        StatusUpdates["FlowUpdaterStatusUpdates<br/>- active_sources<br/>- updated_sources"]
    end
    
    subgraph "PyO3 Bridge"
        EngineUpdater["_engine.FlowLiveUpdater"]
    end
    
    subgraph "Rust Engine"
        Monitor["Source Monitors"]
        Processor["Incremental Processor"]
        Targets["Target Writers"]
    end
    
    FlowLiveUpdater -->|"create()"| EngineUpdater
    Options -->|"dump_engine_object()"| EngineUpdater
    FlowLiveUpdater -->|"next_status_updates_async()"| StatusUpdates
    
    EngineUpdater --> Monitor
    Monitor -->|"detected changes"| Processor
    Processor --> Targets
    
    Monitor -.->|"refresh_interval"| Monitor
    Monitor -.->|"source-specific events"| Monitor
```

**Diagram: FlowLiveUpdater Component Architecture**

The `FlowLiveUpdater` provides a unified interface for both one-time and live updates. The distinction is controlled by the `live_mode` option—when true, sources with change capture mechanisms remain active after the initial update.

Sources: [python/cocoindex/flow.py:598-688](), [python/cocoindex/flow.py:571-597]()

---

## Configuration

### FlowLiveUpdaterOptions

The `FlowLiveUpdaterOptions` dataclass configures updater behavior:

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `live_mode` | `bool` | `True` | Enable continuous monitoring. If `False` or no sources have change capture, updater stops after one-time update |
| `reexport_targets` | `bool` | `False` | Force reexport even when no changes detected. Only applies to initial update in live mode |
| `print_stats` | `bool` | `False` | Print update statistics during processing |

**Example:**

```python
options = cocoindex.FlowLiveUpdaterOptions(
    live_mode=True,
    reexport_targets=False,
    print_stats=True
)
updater = cocoindex.FlowLiveUpdater(my_flow, options)
```

Sources: [python/cocoindex/flow.py:571-597](), [python/cocoindex/cli.py:469-484]()

### Source Refresh Options

The `refresh_interval` parameter is specified when adding a source to the flow:

```python
@cocoindex.flow_def(name="MyFlow")
def my_flow(flow_builder: cocoindex.FlowBuilder, data_scope: cocoindex.DataScope):
    data_scope["docs"] = flow_builder.add_source(
        cocoindex.sources.LocalFile(path="./documents"),
        refresh_interval=datetime.timedelta(minutes=1)
    )
```

**Implementation details:**

- Internally stored in `_SourceRefreshOptions` [python/cocoindex/flow.py:476-483]()
- Serialized to engine via `dump_engine_object()` [python/cocoindex/flow.py:532-540]()
- Only takes effect when `FlowLiveUpdaterOptions.live_mode=True`

Sources: [python/cocoindex/flow.py:476-543](), [docs/docs/core/flow_def.mdx:140-168]()

---

## Lifecycle Management

### Starting and Stopping

The `FlowLiveUpdater` follows a clear lifecycle:

```mermaid
stateDiagram-v2
    [*] --> Created: "FlowLiveUpdater(flow, options)"
    Created --> Running: "start() / start_async()"
    Running --> Running: "sources emit changes"
    Running --> Finished: "wait() / wait_async()<br/>(all sources done)"
    Running --> Aborted: "abort()"
    Aborted --> Finished: "wait() / wait_async()"
    Finished --> [*]
    
    note right of Running
        Background threads monitor
        sources and apply updates
    end note
```

**Diagram: FlowLiveUpdater State Machine**

**Key Methods:**

- **`start()`** / **`start_async()`**: Launches background monitoring. Returns immediately; processing continues asynchronously [python/cocoindex/flow.py:627-639]()
- **`abort()`**: Signals all monitoring threads to stop gracefully [python/cocoindex/flow.py:672-676]()
- **`wait()`** / **`wait_async()`**: Blocks until updater finishes (either naturally or after abort) [python/cocoindex/flow.py:641-651]()

**Context Manager Usage:**

```python
# Synchronous
with cocoindex.FlowLiveUpdater(my_flow, options) as updater:
    # updater.start() called automatically
    while True:
        updates = updater.next_status_updates()
        if not updates.active_sources:
            break
# updater.abort() and updater.wait() called automatically

# Asynchronous
async with cocoindex.FlowLiveUpdater(my_flow, options) as updater:
    while True:
        updates = await updater.next_status_updates_async()
        if not updates.active_sources:
            break
```

The context manager automatically handles start/abort/wait lifecycle [python/cocoindex/flow.py:611-625]().

Sources: [python/cocoindex/flow.py:598-688](), [docs/docs/core/flow_methods.mdx:232-365]()

### Status Updates

The `next_status_updates()` method provides visibility into ongoing live update activity:

```python
updates: FlowUpdaterStatusUpdates = updater.next_status_updates()

# Check which sources are still active
if updates.active_sources:
    print(f"Active: {', '.join(updates.active_sources)}")
else:
    print("All sources finished")

# React to sources that just updated
for source in updates.updated_sources:
    run_downstream_operations(source)
```

**`FlowUpdaterStatusUpdates` fields:**

- **`active_sources`**: `list[str]` - Names of sources still processing. Empty list indicates updater has stopped [python/cocoindex/flow.py:586-595]()
- **`updated_sources`**: `list[str]` - Names of sources with updates since last status check

The method blocks until:
- A batch of source updates completes processing
- The updater stops (naturally or via abort)

Multiple status updates are automatically combined if they arrive between calls, preventing queue buildup during slow downstream processing.

Sources: [python/cocoindex/flow.py:653-670](), [python/cocoindex/flow.py:586-595](), [docs/docs/core/flow_methods.mdx:268-318]()

### Synchronous vs Asynchronous APIs

All blocking operations provide both synchronous and asynchronous versions:

| Synchronous | Asynchronous | Notes |
|-------------|--------------|-------|
| `start()` | `start_async()` | Launch background monitoring |
| `wait()` | `wait_async()` | Block until completion |
| `next_status_updates()` | `next_status_updates_async()` | Get next status batch |
| Context manager | Async context manager | Lifecycle management |

Synchronous versions use `execution_context.run()` to bridge to async runtime [python/cocoindex/flow.py:631](), [python/cocoindex/flow.py:645](), [python/cocoindex/flow.py:660]().

Sources: [python/cocoindex/flow.py:627-670](), [docs/docs/core/flow_methods.mdx:330-364]()

---

## CLI Usage

### Update Command with Live Mode

The `cocoindex update` command supports live updates via the `-L` flag:

```bash
# One-time update
cocoindex update main.py

# Live update with continuous monitoring
cocoindex update main.py -L

# Live update with setup and forced reexport
cocoindex update main.py -L --setup --reexport -f
```

**CLI Flag Mapping:**

| CLI Flag | FlowLiveUpdaterOptions Field |
|----------|------------------------------|
| `-L` / `--live` | `live_mode=True` |
| `--reexport` | `reexport_targets=True` |
| `-q` / `--quiet` | `print_stats=False` (inverted) |

If no sources have change capture mechanisms enabled, live mode falls back to one-time update behavior with a warning message [python/cocoindex/cli.py:295-299]().

Sources: [python/cocoindex/cli.py:391-485](), [docs/docs/core/flow_methods.mdx:216-227]()

### Server Mode with Live Updates

The `cocoindex server` command can launch live updates alongside the HTTP server:

```bash
# Start server with live updates
cocoindex server main.py --live-update

# Server with setup and live updates
cocoindex server main.py --setup --live-update
```

When `--live-update` is enabled, the server starts background live updaters for all flows using `asyncio.run_coroutine_threadsafe()` [python/cocoindex/cli.py:772-774]():

```mermaid
sequenceDiagram
    participant CLI as "CLI Process"
    participant Server as "HTTP Server"
    participant Updater as "FlowLiveUpdater"
    participant Engine as "Rust Engine"
    
    CLI->>Server: "start_server()"
    CLI->>Updater: "asyncio.run_coroutine_threadsafe(update_all_flows_async())"
    Updater->>Engine: "create live updaters"
    
    par Server Operation
        Server->>Server: "handle HTTP requests"
    and Live Updates
        Engine->>Engine: "monitor sources"
        Engine->>Engine: "apply updates"
    end
    
    CLI->>CLI: "wait for SIGINT/SIGTERM"
    CLI->>CLI: "shutdown_event.wait()"
```

**Diagram: Server Mode with Live Updates**

Sources: [python/cocoindex/cli.py:616-784](), [python/cocoindex/cli.py:766-774]()

---

## Implementation Details

### Python-Rust Bridge

Live update orchestration spans Python and Rust layers:

**Python Layer** ([python/cocoindex/flow.py:598-688]()):
- `FlowLiveUpdater` class provides user-facing API
- Manages lifecycle (create, start, abort, wait)
- Handles status update polling
- Bridges to Rust via PyO3

**Rust Engine** (invoked via `_engine.FlowLiveUpdater`):
- Spawns background threads for source monitoring
- Implements change detection logic
- Drives incremental processing
- Manages internal state synchronization

**Key Flow:**

1. Python creates `FlowLiveUpdater` with options [python/cocoindex/flow.py:607-609]()
2. `start_async()` calls `_engine.FlowLiveUpdater.create()` [python/cocoindex/flow.py:637-639]()
3. Rust engine spawns monitoring threads
4. Python polls status via `next_status_updates_async()` [python/cocoindex/flow.py:662-670]()
5. Rust returns aggregated status when updates complete

Sources: [python/cocoindex/flow.py:598-688]()

### Source Refresh Configuration

The `_SourceRefreshOptions` dataclass encapsulates refresh settings:

```python
@dataclass
class _SourceRefreshOptions:
    refresh_interval: datetime.timedelta | None = None
```

This is serialized to the engine when adding a source [python/cocoindex/flow.py:532-540]():

```python
flow_builder_state.engine_flow_builder.add_source(
    kind, spec, target_scope, name,
    refresh_options=dump_engine_object(
        _SourceRefreshOptions(refresh_interval=refresh_interval)
    ),
    ...
)
```

The engine stores this configuration and activates the refresh timer when live mode starts.

Sources: [python/cocoindex/flow.py:476-543]()

### Execution Context Integration

All synchronous methods delegate to async implementations via `execution_context.run()`:

```python
def start(self) -> None:
    execution_context.run(self.start_async())

def wait(self) -> None:
    execution_context.run(self.wait_async())

def next_status_updates(self) -> FlowUpdaterStatusUpdates:
    return execution_context.run(self.next_status_updates_async())
```

The `execution_context` provides a shared event loop for async operations across the codebase [python/cocoindex/flow.py:631](), [python/cocoindex/flow.py:645](), [python/cocoindex/flow.py:660]().

Sources: [python/cocoindex/flow.py:627-670]()

---

## Relationship to One-Time Updates

Live updates extend the one-time update mechanism with continuous monitoring:

| Aspect | One-Time Update | Live Update |
|--------|----------------|-------------|
| **Trigger** | Explicit call to `flow.update()` | `FlowLiveUpdater` with `live_mode=True` |
| **Duration** | Completes when targets are fresh | Runs until aborted |
| **Change Detection** | Implicit (incremental processing) | Explicit (refresh interval, source events) |
| **Use Case** | Scheduled batch jobs | Real-time sync, long-running services |
| **Implementation** | `FlowLiveUpdater` with `live_mode=False` | `FlowLiveUpdater` with `live_mode=True` |

**Unified Interface:**

Both modes use `FlowLiveUpdater` internally. The `Flow.update()` method is a convenience wrapper:

```python
async def update_async(self, /, *, reexport_targets: bool = False):
    async with FlowLiveUpdater(
        self,
        FlowLiveUpdaterOptions(live_mode=False, reexport_targets=reexport_targets)
    ) as updater:
        await updater.wait_async()
    return updater.update_stats()
```

This demonstrates that one-time updates are essentially live updates with `live_mode=False` [python/cocoindex/flow.py:775-788]().

Sources: [python/cocoindex/flow.py:766-788](), [docs/docs/core/flow_methods.mdx:115-139]()

---

## Example Usage Patterns

### Continuous Monitoring with Status Updates

```python
import cocoindex

# Define flow with refresh interval
@cocoindex.flow_def(name="MonitoredFlow")
def monitored_flow(flow_builder, data_scope):
    data_scope["docs"] = flow_builder.add_source(
        cocoindex.sources.LocalFile(path="./data"),
        refresh_interval=datetime.timedelta(seconds=30)
    )
    # ... transformations ...

# Start live monitoring
with cocoindex.FlowLiveUpdater(
    monitored_flow,
    cocoindex.FlowLiveUpdaterOptions(print_stats=True)
) as updater:
    while True:
        updates = updater.next_status_updates()
        
        # Log which sources updated
        if updates.updated_sources:
            print(f"Updated: {updates.updated_sources}")
        
        # Break when all sources finish
        if not updates.active_sources:
            print("Monitoring complete")
            break
```

Sources: [docs/docs/core/flow_methods.mdx:296-318]()

### Background Live Updates with Query Server

```python
import asyncio
import cocoindex

async def run_with_live_updates():
    # Start live updater in background
    updater = cocoindex.FlowLiveUpdater(my_flow)
    await updater.start_async()
    
    # Run query server concurrently
    try:
        while True:
            # Handle queries...
            await handle_query_request()
    finally:
        updater.abort()
        await updater.wait_async()

asyncio.run(run_with_live_updates())
```

This pattern is used by `cocoindex server --live-update` [python/cocoindex/cli.py:766-774]().

Sources: [python/cocoindex/cli.py:766-784]()

### Multiple Sources with Different Mechanisms

```python
@cocoindex.flow_def(name="MultiSourceFlow")
def multi_source_flow(flow_builder, data_scope):
    # Local files with refresh interval
    data_scope["local"] = flow_builder.add_source(
        cocoindex.sources.LocalFile(path="./local"),
        refresh_interval=datetime.timedelta(minutes=5)
    )
    
    # S3 with event notifications (configured separately)
    data_scope["s3"] = flow_builder.add_source(
        cocoindex.sources.AmazonS3(
            bucket="my-bucket",
            prefix="data/",
            # Event notifications configured in AWS
        )
    )
    
    # Postgres with LISTEN/NOTIFY
    data_scope["pg"] = flow_builder.add_source(
        cocoindex.sources.Postgres(
            table="source_data",
            # LISTEN channel configured in spec
        )
    )
```

All change capture mechanisms activate simultaneously in live mode, with updates processed as they arrive from any source.

Sources: [docs/docs/core/flow_methods.mdx:205-212]()

---

# Page: Setup and Drop Operations

# Setup and Drop Operations

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [docs/docs/core/basics.md](docs/docs/core/basics.md)
- [docs/docs/core/flow_def.mdx](docs/docs/core/flow_def.mdx)
- [docs/docs/core/flow_methods.mdx](docs/docs/core/flow_methods.mdx)
- [python/cocoindex/__init__.py](python/cocoindex/__init__.py)
- [python/cocoindex/cli.py](python/cocoindex/cli.py)
- [python/cocoindex/flow.py](python/cocoindex/flow.py)
- [python/cocoindex/lib.py](python/cocoindex/lib.py)
- [python/cocoindex/setup.py](python/cocoindex/setup.py)

</details>



This document explains CocoIndex's backend setup and drop operations, which manage the lifecycle of persistent resources required by flows. Setup creates necessary backend infrastructure (internal state storage and target resources), while drop removes those resources. These operations are implemented through the `SetupChangeBundle` abstraction, which provides idempotent, transactional management of backend state changes.

For information about flow definition and target specifications, see [Flow Definition System](#3). For details about the internal state storage used during incremental processing, see [Incremental Processing and State Management](#9).

## Backend Resources Managed by Setup/Drop

CocoIndex flows require two categories of persistent backend resources:

**Internal State Storage**: PostgreSQL tables prefixed with the flow's full name (including app namespace) that store incremental processing metadata. These tables track source content fingerprints, transformation result caches, and dependency graphs to enable efficient change detection and recomputation.

**Target Resources**: External storage systems specified in flow export operations. Examples include PostgreSQL tables (for `Postgres` target), Qdrant collections (for `Qdrant` target), Neo4j nodes/relationships (for `Neo4j` target). The schema and indexes for these resources are derived from the collector's field types and index specifications.

The setup operation creates or updates these resources to match the current flow definition. The drop operation removes them entirely.

Sources: [python/cocoindex/flow.py:842-883](), [docs/docs/core/flow_methods.mdx:37-109](), [docs/docs/core/basics.md:98-105]()

## SetupChangeBundle: Core Abstraction

```mermaid
graph TB
    subgraph "Python API Layer"
        Flow["Flow<br/>(flow.py)"]
        Helper["make_setup_bundle()<br/>make_drop_bundle()"]
    end
    
    subgraph "Setup Module"
        Bundle["SetupChangeBundle<br/>(setup.py)"]
        EngineBundle["_engine.SetupChangeBundle"]
    end
    
    subgraph "Bundle Operations"
        Describe["describe() / describe_async()<br/>Returns (str, bool)"]
        Apply["apply() / apply_async()"]
        DescApply["describe_and_apply() / ..._async()"]
    end
    
    subgraph "Rust Engine"
        SetupLogic["Setup Logic<br/>Diff & Apply"]
        PgState["PostgreSQL<br/>Internal State Tables"]
        Targets["Target Backends<br/>(Postgres/Qdrant/Neo4j)"]
    end
    
    Flow -->|"setup() / drop()"| Helper
    Helper --> Bundle
    Bundle --> EngineBundle
    
    Bundle --> Describe
    Bundle --> Apply
    Bundle --> DescApply
    
    Describe -->|"Human-readable diff"| SetupLogic
    Apply --> SetupLogic
    DescApply --> SetupLogic
    
    SetupLogic --> PgState
    SetupLogic --> Targets
```

**Diagram: SetupChangeBundle Architecture**

The `SetupChangeBundle` class wraps an engine-level bundle object that represents a set of backend changes (setup or drop). It provides three key operations:

**describe() / describe_async()**: Returns a tuple `(description: str, is_up_to_date: bool)`. The description is a human-readable diff showing what changes would be applied. The boolean indicates whether backends are already in the desired state (true = no changes needed).

**apply() / apply_async()**: Executes the changes, creating/updating/dropping backend resources as needed. Accepts `report_to_stdout: bool` parameter to control progress reporting.

**describe_and_apply() / describe_and_apply_async()**: Convenience method that describes changes (optionally printing to stdout) and then applies them. Automatically skips application if `is_up_to_date` is true.

The class provides both synchronous and asynchronous versions of all methods. Synchronous methods use `execution_context.run()` to bridge to the async engine runtime.

Sources: [python/cocoindex/setup.py:10-73](), [python/cocoindex/flow.py:41]()

## Setup Operations

```mermaid
graph LR
    subgraph "Single Flow Setup"
        F1["Flow.setup()"]
        F2["Flow.setup_async()"]
    end
    
    subgraph "Batch Setup"
        B1["setup_all_flows()"]
        B2["setup_all_flows_async()"]
        MakeBundle["make_setup_bundle(flows)"]
        MakeBundleAsync["make_setup_bundle_async(flows)"]
    end
    
    subgraph "Bundle Creation"
        EngineFlows["await flow.internal_flow_async()<br/>for each flow"]
        CreateBundle["_engine.make_setup_bundle(flows)"]
    end
    
    F1 --> F2
    F2 --> MakeBundleAsync
    
    B1 --> B2
    B2 --> MakeBundleAsync
    
    MakeBundle --> MakeBundleAsync
    MakeBundleAsync --> EngineFlows
    EngineFlows --> CreateBundle
    CreateBundle --> Bundle["SetupChangeBundle"]
```

**Diagram: Setup Operation Flow**

### Flow Instance Methods

The `Flow` class provides instance methods to setup a single flow:

- **setup(report_to_stdout: bool = False)**: Synchronous setup
- **setup_async(report_to_stdout: bool = False)**: Asynchronous setup

These methods internally create a setup bundle for the single flow and call `describe_and_apply_async()` on it. The `report_to_stdout` flag controls whether changes are printed before application.

Implementation: The methods call `make_setup_bundle_async([self])` to create a bundle, then invoke its `describe_and_apply_async()` method.

Sources: [python/cocoindex/flow.py:842-853]()

### Batch Setup Functions

CocoIndex provides module-level functions to setup multiple flows at once:

- **make_setup_bundle(flows: Iterable[Flow]) -> SetupChangeBundle**: Creates a setup bundle for multiple flows
- **make_setup_bundle_async(flows: Iterable[Flow]) -> SetupChangeBundle**: Async version that awaits flow initialization
- **setup_all_flows(report_to_stdout: bool = False)**: Setup all registered flows
- **setup_all_flows_async(report_to_stdout: bool = False)**: Async version

The `make_setup_bundle_async()` function iterates through the provided flows, calls `internal_flow_async()` on each to ensure they're built, collects the engine flow objects, and passes them to `_engine.make_setup_bundle()` to create the bundle.

Sources: [python/cocoindex/flow.py:1208-1233](), [python/cocoindex/__init__.py:26]()

### Setup Execution Process

| Step | Operation | Description |
|------|-----------|-------------|
| 1 | Flow Building | Ensure all flows are built by calling `internal_flow_async()` |
| 2 | Bundle Creation | Engine analyzes current backend state vs. desired state |
| 3 | Diff Generation | `describe()` produces human-readable change summary |
| 4 | User Confirmation | CLI may prompt user to confirm changes (skipped if `--force`) |
| 5 | Backend Creation | Engine creates/updates PostgreSQL state tables |
| 6 | Target Setup | Engine creates/updates target resources with correct schemas |
| 7 | Index Creation | Vector indexes, FTS indexes created per index specifications |

Sources: [python/cocoindex/cli.py:281-303](), [python/cocoindex/setup.py:60-72]()

## Drop Operations

```mermaid
graph LR
    subgraph "Single Flow Drop"
        D1["Flow.drop()"]
        D2["Flow.drop_async()"]
    end
    
    subgraph "Batch Drop"
        BD1["drop_all_flows()"]
        BD2["drop_all_flows_async()"]
        MakeDrop["make_drop_bundle(flows)"]
        MakeDropAsync["make_drop_bundle_async(flows)"]
    end
    
    subgraph "Bundle Creation"
        EngineFlowsDrop["await flow.internal_flow_async()<br/>for each flow"]
        CreateDropBundle["_engine.make_drop_bundle(flows)"]
    end
    
    D1 --> D2
    D2 --> MakeDropAsync
    
    BD1 --> BD2
    BD2 --> MakeDropAsync
    
    MakeDrop --> MakeDropAsync
    MakeDropAsync --> EngineFlowsDrop
    EngineFlowsDrop --> CreateDropBundle
    CreateDropBundle --> DropBundle["SetupChangeBundle"]
```

**Diagram: Drop Operation Flow**

### Flow Instance Methods

The `Flow` class provides instance methods to drop a single flow's backends:

- **drop(report_to_stdout: bool = False)**: Synchronous drop
- **drop_async(report_to_stdout: bool = False)**: Asynchronous drop

Important: After calling `drop()`, the `Flow` instance remains valid in memory. You can call `setup()` again to recreate the backends. To remove the flow from the process entirely, call `close()` instead, which removes the flow from the registry but does not touch persistent backends.

Sources: [python/cocoindex/flow.py:855-871](), [python/cocoindex/flow.py:873-883]()

### Batch Drop Functions

Module-level functions for dropping multiple flows:

- **make_drop_bundle(flows: Iterable[Flow]) -> SetupChangeBundle**: Creates a drop bundle for multiple flows
- **make_drop_bundle_async(flows: Iterable[Flow]) -> SetupChangeBundle**: Async version
- **drop_all_flows(report_to_stdout: bool = False)**: Drop all registered flows
- **drop_all_flows_async(report_to_stdout: bool = False)**: Async version

The implementation mirrors setup functions but calls `_engine.make_drop_bundle()` instead.

Sources: [python/cocoindex/flow.py:1235-1260](), [python/cocoindex/__init__.py:26]()

### Drop Execution Process

| Step | Operation | Description |
|------|-----------|-------------|
| 1 | Flow Building | Ensure flows are built to access backend metadata |
| 2 | Bundle Creation | Engine identifies existing backends to remove |
| 3 | Diff Generation | `describe()` lists backends that will be dropped |
| 4 | User Confirmation | CLI may prompt for confirmation (skipped if `--force`) |
| 5 | Target Removal | Engine drops target resources (tables, collections, etc.) |
| 6 | State Cleanup | Engine drops PostgreSQL internal state tables |
| 7 | Registry Update | Flow remains in memory but backends are gone |

Sources: [python/cocoindex/cli.py:229-261](), [docs/docs/core/flow_methods.mdx:37-109]()

## CLI Commands

### cocoindex setup

```bash
cocoindex setup <APP_TARGET> [--force] [--reset]
```

Checks and applies backend setup changes for all flows defined in the application.

**Arguments:**
- `APP_TARGET`: Path to Python file or module name (e.g., `main.py` or `myapp.flows`)

**Options:**
- `--force`: Skip confirmation prompts and apply changes automatically
- `--reset`: Drop existing setup before running setup (equivalent to `cocoindex drop` followed by `cocoindex setup`)

**Behavior**: Loads the application, builds all flows, creates a setup bundle, describes changes, prompts for confirmation (unless `--force`), and applies changes. If `--reset` is specified, drops all flows first using `_drop_flows()` helper.

Sources: [python/cocoindex/cli.py:320-350]()

### cocoindex drop

```bash
cocoindex drop <APP_TARGET> [FLOW_NAME...] [--force]
```

Drops backend setup for flows.

**Arguments:**
- `APP_TARGET`: Path to Python file or module name
- `FLOW_NAME` (optional): Specific flow names to drop. If omitted, drops all flows in the app.

**Options:**
- `--force`: Skip confirmation prompts

**Behavior**: Loads the application, retrieves specified flows (or all flows if none specified), creates a drop bundle, describes changes, prompts for confirmation (unless `--force`), and applies changes. Invalid flow names are logged as warnings and ignored.

Sources: [python/cocoindex/cli.py:353-398]()

### Integration with update and server Commands

The `cocoindex update` and `cocoindex server` commands both support setup/drop integration:

**Common Options:**
- `--setup`: (Deprecated, now default) Automatically setup flows before updating/starting server
- `--reset`: Drop existing setup before proceeding, then setup
- `--force`: Skip confirmation prompts for setup/drop operations
- `--quiet`: Suppress setup output

**Behavior**: If `--reset` is specified, `_drop_flows()` is called first. Then if setup is needed (default behavior), `_setup_flows()` is called. Both helpers use `make_setup_bundle()` or `make_drop_bundle()` internally.

Sources: [python/cocoindex/cli.py:456-504](), [python/cocoindex/cli.py:642-710]()

## Idempotency and Safety Features

### Idempotent Operations

**Setup Idempotency**: `SetupChangeBundle.describe()` returns `is_up_to_date = true` when backends match the desired state. The `describe_and_apply()` method checks this flag and skips application if true, printing "No setup changes to apply."

Implementation check:
```python
if is_up_to_date:
    print("No setup changes to apply.")
    return
```

**Drop Idempotency**: Similarly, drop operations detect when no backends exist and return `is_up_to_date = true`, causing no destructive operations.

Sources: [python/cocoindex/setup.py:60-72](), [python/cocoindex/cli.py:289-295]()

### Change Preview and Confirmation

The setup/drop workflow includes safety checks:

| Feature | Implementation | Purpose |
|---------|----------------|---------|
| Change Description | `bundle.describe()` | Shows human-readable diff before applying |
| Up-to-date Check | `is_up_to_date` boolean | Skips unnecessary operations |
| User Confirmation | `click.confirm()` in CLI | Prevents accidental destructive changes |
| Force Override | `--force` flag | Allows automation without prompts |
| Report to Stdout | `report_to_stdout` parameter | Controls progress visibility |

The CLI helper function `_setup_flows()` demonstrates this pattern:
1. Create bundle
2. Call `describe()` to get diff
3. Print description
4. Check if up-to-date, exit early if so
5. Prompt for confirmation (unless `--force`)
6. Call `apply()`

Sources: [python/cocoindex/cli.py:281-303]()

### Flow State vs Backend State

Important distinction:

**In-Memory Flow State**: Managed by `Flow` instance lifecycle
- Created by `@flow_def` or `open_flow()`
- Removed by `Flow.close()`
- Does not affect persistent backends

**Backend State**: Managed by setup/drop operations
- Created by setup
- Removed by drop
- Persists across process restarts

After `drop()`, the flow instance remains valid and can be used to call `setup()` again. To remove both in-memory state and backends:
```python
flow.drop()  # Remove backends
flow.close()  # Remove in-memory state
```

Sources: [python/cocoindex/flow.py:873-883](), [docs/docs/core/flow_methods.mdx:102-109]()

## Flow Names and Setup Persistence

```mermaid
graph TB
    subgraph "Flow Naming"
        AppNS["COCOINDEX_APP_NAMESPACE"]
        FlowName["Flow.name"]
        FullName["Flow.full_name<br/>=get_flow_full_name(name)"]
    end
    
    subgraph "Setup Persistence Query"
        QueryFunc["flow_names_with_setup_async()"]
        EngineQuery["_engine.flow_names_with_setup_async()"]
        FilterNS["Filter by current app namespace"]
    end
    
    subgraph "PostgreSQL"
        StateTables["Internal State Tables<br/>Prefixed by full_name"]
    end
    
    AppNS --> FullName
    FlowName --> FullName
    
    FullName --> StateTables
    
    QueryFunc --> EngineQuery
    EngineQuery --> StateTables
    StateTables --> FilterNS
    FilterNS --> Result["List of flow names<br/>(without namespace prefix)"]
```

**Diagram: Flow Name Resolution for Setup**

The function `flow_names_with_setup()` / `flow_names_with_setup_async()` queries which flows have persisted backends. It calls `_engine.flow_names_with_setup_async()` to get all flows with setup (including full names with namespace prefix), then filters to only those matching the current app namespace and strips the prefix.

This enables the `cocoindex ls` command to show which flows are defined in the current process vs. which have persisted setup, marking flows with `[+]` if they're defined but not setup.

Full name construction: `get_flow_full_name(name)` prepends the app namespace with a `.` delimiter if the namespace is non-empty, e.g., `staging.myflow` when `COCOINDEX_APP_NAMESPACE=staging`.

Sources: [python/cocoindex/setup.py:75-92](), [python/cocoindex/flow.py:959-963](), [python/cocoindex/cli.py:141-185]()

## Example Usage Patterns

### Pattern 1: Manual Setup Before Update

```python
import cocoindex

@cocoindex.flow_def(name="MyFlow")
def my_flow(flow_builder, data_scope):
    # ... flow definition ...
    pass

# Explicit setup
my_flow.setup(report_to_stdout=True)

# Then update
my_flow.update()
```

### Pattern 2: Batch Setup with Confirmation

```python
# Setup all flows with preview
bundle = cocoindex.flow.make_setup_bundle(cocoindex.flow.flows().values())
desc, is_up_to_date = bundle.describe()
print(desc)

if not is_up_to_date:
    confirm = input("Apply changes? [y/N]: ")
    if confirm.lower() == 'y':
        bundle.apply(report_to_stdout=True)
```

### Pattern 3: Reset and Rebuild

```python
# Drop all backends
cocoindex.drop_all_flows(report_to_stdout=True)

# Setup fresh
cocoindex.setup_all_flows(report_to_stdout=True)

# Equivalent CLI:
# cocoindex setup main.py --reset --force
```

Sources: [docs/docs/core/flow_methods.mdx:60-100](), [python/cocoindex/cli.py:320-350]()

---

# Page: Configuration and Settings

# Configuration and Settings

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [docs/docs/core/settings.mdx](docs/docs/core/settings.mdx)
- [docs/docs/getting_started/overview.md](docs/docs/getting_started/overview.md)
- [python/cocoindex/setting.py](python/cocoindex/setting.py)

</details>



This page documents the configuration and settings system in CocoIndex, including how to provide database connections, set application namespaces, configure execution options, and manage authentication credentials. The settings system bridges Python and Rust, allowing configuration through environment variables, Python functions, or direct initialization.

For information about flow lifecycle management methods (`setup()`, `update()`, etc.), see [Flow Definition and Python API](#2.1). For CLI usage and commands, see [Command Line Interface](#8.1).

---

## Settings Architecture

CocoIndex uses a multi-layered settings system where configuration can be provided through environment variables, Python decorator functions, or direct initialization. Settings are serialized from Python and passed to the Rust core engine during initialization.

```mermaid
graph TB
    subgraph "Configuration Sources"
        ENV["Environment Variables<br/>COCOINDEX_DATABASE_URL<br/>COCOINDEX_APP_NAMESPACE<br/>etc."]
        DOTENV[".env File<br/>Loaded by CLI or dotenv"]
        DECORATOR["@cocoindex.settings<br/>Python Function"]
        DIRECT["cocoindex.init(settings)"]
    end
    
    subgraph "Python Settings Layer"
        SETTINGS_FROM_ENV["Settings.from_env()"]
        SETTINGS_CLASS["Settings<br/>- database<br/>- app_namespace<br/>- global_execution_options"]
        DB_SPEC["DatabaseConnectionSpec"]
        EXEC_OPT["GlobalExecutionOptions"]
        SERVER_SET["ServerSettings"]
    end
    
    subgraph "Settings Registration"
        GET_SETTINGS_FN["GET_SETTINGS_FN<br/>Mutex&lt;Box&lt;dyn Fn()&gt;&gt;"]
        SET_SETTINGS_FN["_engine.set_settings_fn()"]
    end
    
    subgraph "Rust Settings Layer"
        PY_MOD["py/mod.rs<br/>set_settings_fn()"]
        RUST_SETTINGS["settings::Settings<br/>Deserialized from Python"]
    end
    
    subgraph "Initialization"
        INIT_LIB["init_lib_context()"]
        CREATE_LIB["create_lib_context()"]
        LIB_CONTEXT["LibContext<br/>- db_pools<br/>- app_namespace<br/>- persistence_ctx<br/>- flows"]
    end
    
    subgraph "Runtime Usage"
        DB_POOLS["DbPools<br/>PgPool instances"]
        FLOW_CONTEXT["FlowContext<br/>Uses settings"]
        BUILTIN_DB["builtin_db_pool<br/>Internal tracking"]
    end
    
    ENV --> SETTINGS_FROM_ENV
    DOTENV --> ENV
    DECORATOR --> SETTINGS_CLASS
    DIRECT --> SETTINGS_CLASS
    SETTINGS_FROM_ENV --> SETTINGS_CLASS
    
    SETTINGS_CLASS --> DB_SPEC
    SETTINGS_CLASS --> EXEC_OPT
    SETTINGS_CLASS --> SERVER_SET
    
    DECORATOR --> SET_SETTINGS_FN
    SET_SETTINGS_FN --> GET_SETTINGS_FN
    DIRECT --> INIT_LIB
    
    GET_SETTINGS_FN --> PY_MOD
    PY_MOD --> RUST_SETTINGS
    
    RUST_SETTINGS --> INIT_LIB
    INIT_LIB --> CREATE_LIB
    CREATE_LIB --> LIB_CONTEXT
    
    LIB_CONTEXT --> DB_POOLS
    LIB_CONTEXT --> FLOW_CONTEXT
    LIB_CONTEXT --> BUILTIN_DB
```

**Sources:** [python/cocoindex/lib.py:1-76](), [python/cocoindex/setting.py:1-172](), [src/lib_context.rs:1-407](), [src/settings.rs:1-121]()

---

## Configuration Methods

CocoIndex supports three configuration methods, listed in order of precedence (later methods override earlier ones):

### Environment Variables

The simplest approach loads settings from environment variables when `Settings.from_env()` is called:

```python
# Automatic when no other method is used
settings = cocoindex.Settings.from_env()
```

**Sources:** [python/cocoindex/setting.py:82-128](), [python/cocoindex/lib.py:22]()

### Settings Function Decorator

Register a function that returns a `Settings` object using the `@cocoindex.settings` decorator:

```python
@cocoindex.settings
def cocoindex_settings() -> cocoindex.Settings:
    return cocoindex.Settings(
        database=cocoindex.DatabaseConnectionSpec(
            url="postgresql://localhost/mydb"
        ),
        app_namespace="dev"
    )
```

The decorated function is stored in `GET_SETTINGS_FN` and called when settings are needed. Only one settings function can be registered; subsequent registrations will emit a warning.

**Sources:** [python/cocoindex/lib.py:29-55]()

### Direct Initialization

Call `cocoindex.init()` with a `Settings` object:

```python
cocoindex.init(
    cocoindex.Settings(
        database=cocoindex.DatabaseConnectionSpec(
            url="postgresql://localhost/mydb"
        )
    )
)
```

This method takes precedence over the settings function and environment variables. Note that `cocoindex.init()` is optional—the library will auto-initialize using the settings function or environment variables when needed.

**Sources:** [python/cocoindex/lib.py:58-65](), [src/py/mod.rs:123-129]()

---

## Settings Structure

### Settings Class Hierarchy

```mermaid
graph LR
    SETTINGS["Settings"]
    DB_SPEC["DatabaseConnectionSpec"]
    EXEC_OPT["GlobalExecutionOptions"]
    SERVER_SET["ServerSettings"]
    
    SETTINGS -->|database| DB_SPEC
    SETTINGS -->|global_execution_options| EXEC_OPT
    
    DB_SPEC -->|url| URL["str"]
    DB_SPEC -->|user| USER["str | None"]
    DB_SPEC -->|password| PASS["str | None"]
    DB_SPEC -->|max_connections| MAX_CONN["int = 25"]
    DB_SPEC -->|min_connections| MIN_CONN["int = 5"]
    
    EXEC_OPT -->|source_max_inflight_rows| ROWS["int | None = 1024"]
    EXEC_OPT -->|source_max_inflight_bytes| BYTES["int | None = None"]
    
    SETTINGS -->|app_namespace| NS["str = ''"]
```

**Sources:** [python/cocoindex/setting.py:28-128](), [src/settings.rs:1-28]()

### Python Settings Classes

| Class | File | Purpose |
|-------|------|---------|
| `Settings` | [python/cocoindex/setting.py:74-128]() | Top-level settings container |
| `DatabaseConnectionSpec` | [python/cocoindex/setting.py:28-39]() | Database connection configuration |
| `GlobalExecutionOptions` | [python/cocoindex/setting.py:42-48]() | Global concurrency limits |
| `ServerSettings` | [python/cocoindex/setting.py:131-171]() | HTTP server configuration |

### Rust Settings Structs

The Rust engine uses corresponding structs that deserialize from Python via PyO3:

| Struct | File | Fields |
|--------|------|--------|
| `Settings` | [src/settings.rs:18-27]() | `database`, `app_namespace`, `global_execution_options` |
| `DatabaseConnectionSpec` | [src/settings.rs:3-10]() | `url`, `user`, `password`, `max_connections`, `min_connections` |
| `GlobalExecutionOptions` | [src/settings.rs:12-16]() | `source_max_inflight_rows`, `source_max_inflight_bytes` |
| `ServerSettings` | [src/server/mod.rs]() | `address`, `cors_origins` |

**Sources:** [src/settings.rs:1-121](), [src/py/mod.rs:100-116]()

---

## Environment Variables Reference

### Database Configuration

| Environment Variable | Corresponding Field | Type | Default | Required |
|---------------------|-------------------|------|---------|----------|
| `COCOINDEX_DATABASE_URL` | `database.url` | `str` | None | Yes (for persistence) |
| `COCOINDEX_DATABASE_USER` | `database.user` | `str` | None | No |
| `COCOINDEX_DATABASE_PASSWORD` | `database.password` | `str` | None | No |
| `COCOINDEX_DATABASE_MAX_CONNECTIONS` | `database.max_connections` | `int` | 25 | No |
| `COCOINDEX_DATABASE_MIN_CONNECTIONS` | `database.min_connections` | `int` | 5 | No |

### Application Configuration

| Environment Variable | Corresponding Field | Type | Default | Required |
|---------------------|-------------------|------|---------|----------|
| `COCOINDEX_APP_NAMESPACE` | `app_namespace` | `str` | `""` | No |

### Execution Configuration

| Environment Variable | Corresponding Field | Type | Default | Required |
|---------------------|-------------------|------|---------|----------|
| `COCOINDEX_SOURCE_MAX_INFLIGHT_ROWS` | `global_execution_options.source_max_inflight_rows` | `int` | 1024 | No |
| `COCOINDEX_SOURCE_MAX_INFLIGHT_BYTES` | `global_execution_options.source_max_inflight_bytes` | `int` | None | No |

### Server Configuration

| Environment Variable | Corresponding Field | Type | Default | Required |
|---------------------|-------------------|------|---------|----------|
| `COCOINDEX_SERVER_ADDRESS` | `address` | `str` | `127.0.0.1:49344` | No |
| `COCOINDEX_SERVER_CORS_ORIGINS` | `cors_origins` | `list[str]` | None | No |

**Sources:** [python/cocoindex/setting.py:51-71](), [python/cocoindex/setting.py:82-128](), [python/cocoindex/setting.py:141-152]()

---

## Database Connection Management

### Connection Pool Architecture

```mermaid
graph TB
    subgraph "Connection Request Flow"
        TARGET["Target Operation<br/>or Source Tracking"]
        FLOW_CTX["FlowContext<br/>use_execution_ctx()"]
        LIB_CTX["LibContext"]
        REQUIRE["require_builtin_db_pool()"]
    end
    
    subgraph "Pool Management"
        DB_POOLS["DbPools<br/>Mutex&lt;HashMap&lt;PoolKey, PoolValue&gt;&gt;"]
        POOL_KEY["PoolKey<br/>(url, user)"]
        POOL_VALUE["PoolValue<br/>Arc&lt;OnceCell&lt;PgPool&gt;&gt;"]
    end
    
    subgraph "Pool Creation"
        GET_POOL["get_pool(conn_spec)"]
        PG_OPTIONS["PgConnectOptions<br/>URL + credentials"]
        PG_POOL_OPTIONS["PgPoolOptions<br/>min/max connections<br/>timeouts"]
        PG_POOL["PgPool<br/>sqlx connection pool"]
    end
    
    TARGET --> FLOW_CTX
    FLOW_CTX --> LIB_CTX
    LIB_CTX --> REQUIRE
    REQUIRE --> DB_POOLS
    
    DB_POOLS --> POOL_KEY
    POOL_KEY --> POOL_VALUE
    POOL_VALUE --> GET_POOL
    
    GET_POOL --> PG_OPTIONS
    PG_OPTIONS --> PG_POOL_OPTIONS
    PG_POOL_OPTIONS --> PG_POOL
```

The `DbPools` struct maintains a cache of connection pools indexed by `(url, user)` tuples. Each pool is created lazily on first use and shared across all operations requiring that database connection.

**Implementation Details:**

1. **Pool Key**: `PoolKey = (String, Option<String>)` combines URL and optional username
2. **Pool Value**: `PoolValue = Arc<OnceCell<PgPool>>` ensures thread-safe lazy initialization
3. **Connection Timeout**: Initial connection attempt uses 30-second timeout
4. **Pool Configuration**:
   - Acquisition timeout: 5 minutes
   - Slow acquisition threshold: 10 seconds (logs warning)
   - Min/max connections from `DatabaseConnectionSpec`

**Sources:** [src/lib_context.rs:172-226](), [src/lib_context.rs:266-278]()

### Builtin Database Pool

The builtin database pool is used for internal CocoIndex operations:

- **Source tracking**: Records ordinals, fingerprints, and lineage in `source_tracking` table
- **Setup metadata**: Stores flow setup states in `setup_metadata` table
- **Change detection**: Tracks schema versions and compatibility

The pool is created when `PersistenceContext` is initialized if `Settings.database` is provided:

**Sources:** [src/lib_context.rs:282-316](), [src/lib_context.rs:232-235]()

### Persistence Context

```mermaid
graph TB
    LIB_CONTEXT["LibContext"]
    PERSISTENCE["persistence_ctx<br/>Option&lt;PersistenceContext&gt;"]
    BUILTIN_POOL["builtin_db_pool<br/>PgPool"]
    SETUP_CTX["setup_ctx<br/>RwLock&lt;LibSetupContext&gt;"]
    ALL_SETUP["all_setup_states<br/>AllSetupStates"]
    GLOBAL_CHANGE["global_setup_change<br/>GlobalSetupChange"]
    
    LIB_CONTEXT --> PERSISTENCE
    PERSISTENCE --> BUILTIN_POOL
    PERSISTENCE --> SETUP_CTX
    SETUP_CTX --> ALL_SETUP
    SETUP_CTX --> GLOBAL_CHANGE
```

The `PersistenceContext` is `None` if no database is configured, allowing CocoIndex to run in evaluation mode without persistent storage.

**Sources:** [src/lib_context.rs:228-235](), [src/lib_context.rs:288-302]()

---

## App Namespace

The `app_namespace` setting organizes flows across different environments, team members, or deployment contexts. When set, it prefixes all flow names.

### Namespace Resolution

```mermaid
graph LR
    USER_NAME["User-specified name<br/>'myflow'"]
    APP_NS["app_namespace<br/>'dev'"]
    FULL_NAME["Full flow name<br/>'dev.myflow'"]
    STORAGE["Backend storage<br/>setup_metadata<br/>source_tracking"]
    
    USER_NAME --> GET_FULL["get_flow_full_name()"]
    APP_NS --> GET_FULL
    GET_FULL --> FULL_NAME
    FULL_NAME --> STORAGE
```

**Python API:**

```python
# Get current namespace with optional trailing delimiter
namespace = cocoindex.get_app_namespace(trailing_delimiter='.')
# Returns "dev." if namespace is "dev", or "" if empty

# Split full name into namespace and local name
app_ns, local_name = cocoindex.setting.split_app_namespace('dev.myflow', '.')
# Returns ("dev", "myflow")
```

**Sources:** [python/cocoindex/setting.py:12-25](), [python/cocoindex/flow.py:945-949](), [src/lib_context.rs:307-308]()

---

## Global Execution Options

Control concurrency across all flows to prevent resource exhaustion:

| Option | Purpose | Default |
|--------|---------|---------|
| `source_max_inflight_rows` | Maximum concurrent source rows being processed globally | 1024 |
| `source_max_inflight_bytes` | Maximum concurrent data size in bytes globally | None |

These limits work alongside per-source limits (see [Flow Definition and Python API](#2.1)). Both limits must be satisfied to admit new source rows for processing.

### Concurrency Controller

```mermaid
graph TB
    GLOBAL_CTRL["global_concurrency_controller<br/>ConcurrencyController"]
    SOURCE1["Source 1<br/>max_inflight_rows=100"]
    SOURCE2["Source 2<br/>max_inflight_rows=50"]
    
    ADMIT["Admission Check<br/>Both global and local<br/>limits satisfied?"]
    PROCESS["Process Source Row"]
    
    SOURCE1 --> ADMIT
    SOURCE2 --> ADMIT
    GLOBAL_CTRL --> ADMIT
    ADMIT -->|Yes| PROCESS
    ADMIT -->|No| WAIT["Wait for capacity"]
```

The `ConcurrencyController` is created when `LibContext` is initialized:

**Sources:** [src/lib_context.rs:309-314](), [python/cocoindex/setting.py:42-48](), [src/settings.rs:12-16]()

---

## Server Settings

Configuration for the HTTP API server that exposes flow query handlers and management endpoints.

### ServerSettings Fields

| Field | Type | Default | Environment Variable |
|-------|------|---------|---------------------|
| `address` | `str` | `127.0.0.1:49344` | `COCOINDEX_SERVER_ADDRESS` |
| `cors_origins` | `list[str] \| None` | `None` | `COCOINDEX_SERVER_CORS_ORIGINS` |

### CORS Origins Parsing

The `cors_origins` field accepts a comma-separated list of origin URLs:

```python
# From environment variable
COCOINDEX_SERVER_CORS_ORIGINS="https://example.com,http://localhost:3000"

# In code
settings = ServerSettings(
    cors_origins=["https://example.com", "http://localhost:3000"]
)
```

**CLI Options:**

The `cocoindex server` command provides additional CORS configuration:
- `-c/--cors-origin`: Specify origins (comma-separated)
- `-ci/--cors-cocoindex`: Allow `https://cocoindex.io`
- `-cl/--cors-local <port>`: Allow `http://localhost:<port>`

**Sources:** [python/cocoindex/setting.py:131-171](), [python/cocoindex/cli.py:532-683]()

---

## Authentication Registry

CocoIndex provides a global registry for managing authentication credentials used by sources, targets, and LLM operations. Credentials are stored separately from flow definitions and can be referenced by key.

### Authentication API

```mermaid
graph TB
    subgraph "Python API"
        ADD_AUTH["add_auth_entry(key, value)"]
        ADD_TRANSIENT["add_transient_auth_entry(value)<br/>Returns: key"]
        REF_AUTH["ref_auth_entry(key)<br/>Returns: AuthEntryReference"]
    end
    
    subgraph "Rust Registry"
        AUTH_REG["AUTH_REGISTRY<br/>LazyLock&lt;Arc&lt;AuthRegistry&gt;&gt;"]
        STORAGE["HashMap&lt;String, Value&gt;<br/>+ Transient entries"]
    end
    
    subgraph "Usage in Operations"
        OP_SPEC["Operation Spec<br/>auth_ref: AuthEntryReference"]
        GET_AUTH["get_auth_entry(key)<br/>Retrieve credentials"]
        EXECUTOR["Executor<br/>Uses credentials"]
    end
    
    ADD_AUTH --> AUTH_REG
    ADD_TRANSIENT --> AUTH_REG
    REF_AUTH --> OP_SPEC
    
    AUTH_REG --> STORAGE
    
    OP_SPEC --> GET_AUTH
    GET_AUTH --> AUTH_REG
    GET_AUTH --> EXECUTOR
```

### Persistent vs Transient Entries

| Method | Lifetime | Key | Use Case |
|--------|----------|-----|----------|
| `add_auth_entry(key, value)` | Process lifetime | User-specified | Reusable credentials across flows |
| `add_transient_auth_entry(value)` | Process lifetime | Auto-generated UUID | One-time use, inline credentials |

**Example Usage:**

```python
# Add persistent credential
cocoindex.add_auth_entry("my_api_key", {"api_key": "sk-..."})

# Reference in operation
source = cocoindex.sources.CustomSource(
    auth_ref=cocoindex.ref_auth_entry("my_api_key")
)

# Add transient credential (auto-cleanup)
temp_key = cocoindex.add_transient_auth_entry({"token": "temp_token"})
```

**Sources:** [python/cocoindex/auth_registry.py](), [src/py/mod.rs:632-651](), [src/lib_context.rs:163-169]()

---

## Settings Lifecycle

### Initialization Flow

```mermaid
sequenceDiagram
    participant User as "User Code"
    participant Settings as "Settings.from_env()<br/>or @settings decorator"
    participant LibPy as "python/cocoindex/lib.py"
    participant PyMod as "src/py/mod.rs"
    participant LibCtx as "src/lib_context.rs"
    participant Pools as "DbPools"
    
    User->>Settings: Configure settings
    Settings->>LibPy: Settings object
    LibPy->>PyMod: set_settings_fn(closure)
    PyMod->>LibPy: Stored in GET_SETTINGS_FN
    
    User->>LibPy: First flow operation
    LibPy->>PyMod: init() or auto-init
    PyMod->>LibCtx: init_lib_context(settings)
    LibCtx->>LibCtx: create_lib_context(settings)
    LibCtx->>Pools: Create DbPools
    
    alt Database configured
        LibCtx->>Pools: get_pool(database_spec)
        Pools->>Pools: Create PgPool
        LibCtx->>LibCtx: Create PersistenceContext
        LibCtx->>LibCtx: Load AllSetupStates
    end
    
    LibCtx-->>PyMod: LibContext created
    PyMod-->>User: Ready for operations
```

**Key Functions:**

| Function | File | Purpose |
|----------|------|---------|
| `set_settings_fn()` | [src/py/mod.rs:100-116]() | Register Python settings closure |
| `get_settings()` | [src/lib_context.rs:320-328]() | Invoke registered settings function |
| `init_lib_context()` | [src/lib_context.rs:340-348]() | Initialize library with settings |
| `create_lib_context()` | [src/lib_context.rs:282-316]() | Create LibContext from settings |

**Sources:** [src/py/mod.rs:100-147](), [src/lib_context.rs:282-366](), [python/cocoindex/lib.py:22-65]()

### Settings Access Patterns

Once initialized, settings influence various system components:

| Component | Setting Used | Access Pattern |
|-----------|-------------|----------------|
| Database pools | `database` | `lib_context.db_pools.get_pool()` |
| Flow naming | `app_namespace` | `lib_context.app_namespace` |
| Source concurrency | `global_execution_options` | `lib_context.global_concurrency_controller` |
| Setup persistence | `database` | `lib_context.require_builtin_db_pool()` |

**Sources:** [src/lib_context.rs:237-279](), [src/lib_context.rs:304-315]()

---

## Environment File Loading

The CLI automatically loads environment variables from `.env` files:

```python
# CLI loads .env from current directory
dotenv_path = env_file or find_dotenv(usecwd=True)
if load_dotenv(dotenv_path=dotenv_path):
    click.echo(f"Loaded environment variables from: {loaded_env_path}")
```

For non-CLI usage, use `python-dotenv` or similar:

```python
from dotenv import load_dotenv
load_dotenv()  # Load .env from current directory
```

**Sources:** [python/cocoindex/cli.py:96-131]()

---

## Error Handling

### Missing Database Configuration

When database operations are attempted without `database` configured:

```
Error: Database is required for this operation. 
The easiest way is to set COCOINDEX_DATABASE_URL environment variable.
Please see https://cocoindex.io/docs/core/settings for more details.
```

**Sources:** [src/lib_context.rs:266-274]()

### Invalid Configuration

Environment variable parsing errors:

```python
# Setting.from_env() raises ValueError with context
raise ValueError(
    f"failed to parse environment variable {env_name}: {value}"
) from e
```

**Sources:** [python/cocoindex/setting.py:51-71]()

### Connection Failures

Database connection errors include URL context and suggest remediation:

```rust
.context(format!("Failed to connect to database {}", conn_spec.url))?
```

**Sources:** [src/lib_context.rs:198-221]()

---

# Page: Authentication and Secrets Management

# Authentication and Secrets Management

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [docs/docs/core/basics.md](docs/docs/core/basics.md)
- [docs/docs/core/flow_def.mdx](docs/docs/core/flow_def.mdx)
- [docs/docs/core/flow_methods.mdx](docs/docs/core/flow_methods.mdx)

</details>



## Overview

CocoIndex provides an **auth registry** system for managing authentication credentials and secrets when connecting to external services. The auth registry is an in-memory key-value store that solves two key problems:

1. **Secret isolation**: Operation specs are shown in various places (e.g., `cocoindex show` output) and should not contain sensitive credentials
2. **Backend lifecycle management**: When a target operation is removed from flow definition, its spec is gone, but CocoIndex still needs connection credentials to drop the associated persistent backend during cleanup

The auth registry complements other authentication mechanisms in CocoIndex: environment variables for LLM API keys, `DatabaseConnectionSpec` for database connections, and target-specific credential fields. For general configuration patterns, see [Configuration and Settings](#10). For specific target backends, see [Storage Backends and Export Targets](#8).

**Sources:** [docs/docs/core/flow_def.mdx:476-487]()

---

## Auth Registry Architecture

The auth registry stores credentials separately from operation specs, allowing specs to reference credentials by key rather than embedding them directly:

**Auth Registry System Architecture**

```mermaid
graph TB
    subgraph "Flow Definition Code"
        AddAuthEntry["cocoindex.add_auth_entry()<br/>Creates entry with explicit key"]
        AddTransientAuthEntry["cocoindex.add_transient_auth_entry()<br/>Creates entry with auto-generated key"]
        RefAuthEntry["cocoindex.ref_auth_entry()<br/>References entry by key string"]
    end
    
    subgraph "Auth Registry (In-Memory)"
        AuthRegistry["Auth Registry<br/>Key-Value Store"]
        ExplicitKeys["Explicit Keys<br/>'my_graph_conn'<br/>'prod_qdrant_key'"]
        AutoGenKeys["Auto-Generated Keys<br/>'transient_12345'<br/>'transient_67890'"]
    end
    
    subgraph "Operation Specs"
        TargetSpec["Target Specs<br/>Neo4j, Qdrant<br/>Reference auth entries"]
        SourceSpec["Source Specs<br/>AzureBlob, GoogleDrive<br/>Reference transient entries"]
        FunctionSpec["Function Specs<br/>May reference transient entries"]
    end
    
    subgraph "CocoIndex Engine"
        BackendManager["Backend Manager<br/>Resolves auth references<br/>at runtime"]
        SetupDrop["Setup/Drop Operations<br/>Uses auth keys to identify<br/>backends for cleanup"]
    end
    
    AddAuthEntry -->|"Stores"| ExplicitKeys
    AddTransientAuthEntry -->|"Stores"| AutoGenKeys
    RefAuthEntry -->|"Resolves from"| ExplicitKeys
    
    ExplicitKeys -->|"Part of"| AuthRegistry
    AutoGenKeys -->|"Part of"| AuthRegistry
    
    AuthRegistry -->|"Referenced by"| TargetSpec
    AuthRegistry -->|"Referenced by"| SourceSpec
    AuthRegistry -->|"Referenced by"| FunctionSpec
    
    TargetSpec --> BackendManager
    SourceSpec --> BackendManager
    FunctionSpec --> BackendManager
    
    BackendManager --> SetupDrop
    
    Note["Key stability important for targets:<br/>Same key = same backend across runs"]
```

**Entry Types:**

| Entry Type | Key Generation | Primary Use Case | Stability Required |
|------------|---------------|------------------|-------------------|
| `AuthEntryReference[T]` | Explicit key provided | Targets (Neo4j, Qdrant) | Yes - key identifies backend |
| `TransientAuthEntryReference[T]` | Auto-generated key | Sources and functions | No - key is ephemeral |

**Sources:** [docs/docs/core/flow_def.mdx:476-487]()

---

## Auth Entry: Explicit Key Management

Auth entries use explicit keys to identify backends across flow definition changes. This key stability is critical for target backends that require cleanup operations.

### Creating Auth Entries

The `cocoindex.add_auth_entry()` function creates an auth entry and returns an `AuthEntryReference[T]`:

**Function Signature:**
```python
def add_auth_entry(key: str, value: T) -> AuthEntryReference[T]
```

**Neo4j Target Example:**

```python
# Create auth entry for Neo4j connection
my_graph_conn = cocoindex.add_auth_entry(
    "my_graph_conn",
    cocoindex.targets.Neo4jConnectionSpec(
        uri="bolt://localhost:7687",
        user="neo4j",
        password="cocoindex",
    )
)

# Use in target spec
demo_collector.export(
    "MyGraph",
    cocoindex.targets.Neo4jRelationship(
        connection=my_graph_conn,
        from_label="Document",
        relationship_type="CONTAINS",
        to_label="Chunk"
    )
)
```

**Qdrant Target Example:**

```python
# Create auth entry for Qdrant API key
qdrant_auth = cocoindex.add_auth_entry(
    "prod_qdrant_key",
    "your-qdrant-api-key"
)

# Use in target spec
demo_collector.export(
    "embeddings",
    cocoindex.targets.Qdrant(
        url="https://xyz.cloud.qdrant.io:6333",
        api_key=qdrant_auth,
        collection_name="documents"
    )
)
```

**Sources:** [docs/docs/core/flow_def.mdx:499-509]()

### Referencing Auth Entries

Auth entries can be referenced in two ways:

**1. Direct Reference (Using the AuthEntryReference object):**

```python
my_graph_conn = cocoindex.add_auth_entry("my_graph_conn", spec)

demo_collector.export(
    "MyGraph",
    cocoindex.targets.Neo4jRelationship(connection=my_graph_conn, ...)
)
```

**2. String Key Reference (Using ref_auth_entry):**

```python
# Auth entry created elsewhere or in separate file
cocoindex.add_auth_entry("my_graph_conn", spec)

# Reference by key string
demo_collector.export(
    "MyGraph",
    cocoindex.targets.Neo4jRelationship(
        connection=cocoindex.ref_auth_entry("my_graph_conn"),
        ...
    )
)
```

The string key reference pattern is useful when:
- Auth entries are defined in a separate configuration module
- The same auth entry is used across multiple flows
- You want to decouple credential management from flow definition

**Sources:** [docs/docs/core/flow_def.mdx:511-529]()

### Key Stability Requirements

Auth entry keys must remain stable across flow definition changes. CocoIndex uses these keys to identify backends during setup and drop operations:

**Key Stability Flow:**

```mermaid
graph LR
    subgraph "Run 1: Initial Setup"
        Key1["Auth Key:<br/>'my_graph_conn'"]
        Backend1["Neo4j Database<br/>Created with key"]
    end
    
    subgraph "Run 2: Flow Update"
        Key2["Auth Key:<br/>'my_graph_conn'<br/>(same key)"]
        Backend2["Neo4j Database<br/>Recognized as same backend"]
    end
    
    subgraph "Run 3: Flow Drop"
        Key3["Auth Key:<br/>'my_graph_conn'<br/>(still same key)"]
        Backend3["Neo4j Database<br/>Dropped successfully"]
    end
    
    Key1 --> Backend1
    Backend1 -->|"Key unchanged"| Key2
    Key2 --> Backend2
    Backend2 -->|"Key unchanged"| Key3
    Key3 --> Backend3
    
    KeyChange["Different Key:<br/>'new_graph_conn'"]
    NewBackend["New Backend Created<br/>Old backend orphaned"]
    
    Backend2 -.->|"If key changes"| KeyChange
    KeyChange --> NewBackend
```

**Best Practices:**

1. **Keep keys stable**: Once chosen, do not change auth entry keys unless you intend to create a new backend
2. **Use descriptive keys**: `"prod_neo4j_conn"`, `"staging_qdrant_key"` are better than `"conn1"`, `"key2"`
3. **Keep orphaned keys until cleanup**: If a target is removed from flow definition, keep its auth entry until after running `cocoindex drop` to allow proper backend cleanup

**Sources:** [docs/docs/core/flow_def.mdx:533-540]()

---

## Transient Auth Entry: Auto-Generated Keys

Transient auth entries use automatically generated keys and are designed for sources and functions where backend key stability is not required.

### Creating Transient Auth Entries

The `cocoindex.add_transient_auth_entry()` function creates a transient entry and returns a `TransientAuthEntryReference[T]`:

**Function Signature:**
```python
def add_transient_auth_entry(value: T) -> TransientAuthEntryReference[T]
```

**Azure Blob Source Example:**

```python
flow_builder.add_source(
    cocoindex.sources.AzureBlob(
        account_name="mystorageaccount",
        container_name="documents",
        sas_token=cocoindex.add_transient_auth_entry(
            "sv=2021-06-08&ss=b&srt=sco&sp=rl&se=2024-12-31..."
        )
    )
)
```

**Google Drive Source Example:**

```python
flow_builder.add_source(
    cocoindex.sources.GoogleDrive(
        folder_id="1AbCdEfGhIjKlMnOpQrStUvWxYz",
        service_account_key=cocoindex.add_transient_auth_entry(
            {
                "type": "service_account",
                "project_id": "my-project",
                "private_key_id": "...",
                "private_key": "...",
                "client_email": "...",
                "client_id": "...",
                # ... rest of service account JSON
            }
        )
    )
)
```

**Sources:** [docs/docs/core/flow_def.mdx:542-561]()

### Type Compatibility

`TransientAuthEntryReference[T]` is a supertype of `AuthEntryReference[T]`, meaning you can pass an `AuthEntryReference` wherever a `TransientAuthEntryReference` is expected:

```python
# Auth entry can be used as transient auth entry
my_auth = cocoindex.add_auth_entry("stable_key", "my_secret")

# Valid: AuthEntryReference[str] → TransientAuthEntryReference[str]
flow_builder.add_source(
    cocoindex.sources.AzureBlob(
        account_name="mystorageaccount",
        container_name="documents",
        sas_token=my_auth  # Works even though parameter expects TransientAuthEntryReference
    )
)
```

This allows using stable keys for sources when desired, but transient entries are sufficient for most source and function use cases.

**Sources:** [docs/docs/core/flow_def.mdx:563-564]()

### Use Case Comparison

**Transient Auth Entry Use Cases:**

```mermaid
graph TB
    subgraph "Transient Auth Entry Pattern"
        TransientUse["Transient Auth Entry"]
        NoStability["No key stability needed"]
        EphemeralBackends["Ephemeral or no backend"]
        Examples1["Examples:<br/>• Source connectors<br/>• Function credentials<br/>• API keys for processing"]
    end
    
    subgraph "Auth Entry Pattern"
        AuthUse["Auth Entry (Explicit Key)"]
        RequiresStability["Key stability required"]
        PersistentBackends["Persistent backends"]
        Examples2["Examples:<br/>• Target databases<br/>• Graph databases<br/>• Vector stores"]
    end
    
    TransientUse --> NoStability
    NoStability --> EphemeralBackends
    EphemeralBackends --> Examples1
    
    AuthUse --> RequiresStability
    RequiresStability --> PersistentBackends
    PersistentBackends --> Examples2
```

| Pattern | Key Type | Backend Lifecycle | Setup/Drop Behavior |
|---------|----------|-------------------|-------------------|
| `add_auth_entry()` | Explicit, stable | Persistent across runs | Backend identified by key for cleanup |
| `add_transient_auth_entry()` | Auto-generated | N/A or ephemeral | No backend tracking needed |

**Sources:** [docs/docs/core/flow_def.mdx:476-564]()

---

## Integration with Other Authentication Mechanisms

The auth registry complements but does not replace other authentication mechanisms in CocoIndex. Understanding how these systems interact helps choose the right approach for each credential type.

### Auth Registry vs Environment Variables

**Environment Variables** are the primary mechanism for API keys and service credentials that are:
- Global to the application
- Not specific to individual operations
- Required by libraries and SDKs that read them directly

**Auth Registry** is used for credentials that are:
- Operation-specific (different targets may use different credentials)
- Need to be associated with backend lifecycle
- Should be kept out of operation specs shown in debugging output

**Authentication Mechanism Comparison:**

```mermaid
graph TB
    subgraph "Global Credentials (Environment Variables)"
        InternalDB["Internal Database<br/>COCOINDEX_DATABASE_URL<br/>COCOINDEX_DATABASE_USER<br/>COCOINDEX_DATABASE_PASSWORD"]
        LLMKeys["LLM API Keys<br/>OPENAI_API_KEY<br/>GEMINI_API_KEY<br/>ANTHROPIC_API_KEY"]
        CloudCreds["Cloud Provider Credentials<br/>AWS_ACCESS_KEY_ID<br/>AWS_SECRET_ACCESS_KEY<br/>GOOGLE_APPLICATION_CREDENTIALS"]
    end
    
    subgraph "Operation-Specific Credentials (Auth Registry)"
        TargetAuth["Target Backend Credentials<br/>add_auth_entry('neo4j_conn', ...)<br/>add_auth_entry('qdrant_key', ...)"]
        SourceAuth["Source Credentials<br/>add_transient_auth_entry(sas_token)<br/>add_transient_auth_entry(api_key)"]
    end
    
    subgraph "Direct Configuration (Not Secret)"
        ConfigParams["Configuration Parameters<br/>table_name<br/>collection_name<br/>model<br/>address"]
    end
    
    InternalDB -->|"Read by"| EngineInit["Engine Initialization"]
    LLMKeys -->|"Read by"| LLMProviders["LLM Provider Clients"]
    CloudCreds -->|"Read by"| CloudSDKs["Cloud SDKs"]
    
    TargetAuth -->|"Referenced by"| TargetSpecs["Target Specs"]
    SourceAuth -->|"Referenced by"| SourceSpecs["Source Specs"]
    ConfigParams -->|"Passed to"| OperationSpecs["Operation Specs"]
```

**Sources:** [docs/docs/core/flow_def.mdx:476-487](), [docs/docs/core/settings.mdx:108-150]()

### Common Authentication Patterns

| Credential Type | Storage Mechanism | Access Pattern | Example |
|----------------|-------------------|----------------|---------|
| Internal database password | Environment variable | `DatabaseConnectionSpec` reads `COCOINDEX_DATABASE_PASSWORD` | [Configuration and Settings](#10) |
| LLM API key | Environment variable | LLM client libraries read `OPENAI_API_KEY` | [LLM Integration](#7) |
| Target database connection | Auth registry | `add_auth_entry("neo4j_conn", Neo4jConnectionSpec(...))` | Neo4j target |
| Source API token | Auth registry (transient) | `add_transient_auth_entry(token)` | AzureBlob source |
| Cloud credentials (AWS/GCP) | Environment variable or file | SDKs read standard env vars | S3 source, Vertex AI |

**Sources:** [docs/docs/core/flow_def.mdx:476-564]()

### Example: Multi-Environment Setup

A typical production setup combines environment variables for global configuration with auth registry entries for environment-specific backends:

```python
# Environment variables in .env file
# COCOINDEX_APP_NAMESPACE=Production
# COCOINDEX_DATABASE_URL=postgres://internal-db/cocoindex
# COCOINDEX_DATABASE_PASSWORD=internal_secret
# OPENAI_API_KEY=sk-proj-...

import cocoindex

# Auth registry entries for production backends
prod_neo4j = cocoindex.add_auth_entry(
    "prod_neo4j_conn",
    cocoindex.targets.Neo4jConnectionSpec(
        uri="bolt://prod-neo4j:7687",
        user="neo4j",
        password="prod_neo4j_password"
    )
)

prod_qdrant = cocoindex.add_auth_entry(
    "prod_qdrant_key",
    "prod-qdrant-api-key-xyz"
)

@cocoindex.flow_def(name="ProductionFlow")
def production_flow(flow_builder, data_scope):
    # Internal database uses env vars automatically
    # LLM functions use OPENAI_API_KEY automatically
    
    data_scope["documents"] = flow_builder.add_source(...)
    
    # ... transformations ...
    
    # Targets use auth registry entries
    collector.export(
        "knowledge_graph",
        cocoindex.targets.Neo4jRelationship(connection=prod_neo4j, ...)
    )
    
    embeddings_collector.export(
        "embeddings",
        cocoindex.targets.Qdrant(
            url="https://prod-cluster.qdrant.io:6333",
            api_key=prod_qdrant,
            collection_name="documents"
        )
    )
```

**Sources:** [docs/docs/core/flow_def.mdx:476-564](), [docs/docs/core/settings.mdx:24-35]()

---

## Summary

CocoIndex implements a consistent authentication pattern across all integrations:

1. **Credentials via environment variables**: All secrets (API keys, passwords) use environment variables
2. **Configuration objects for structure**: Non-sensitive parameters (URLs, model names, table names) use typed configuration objects
3. **Three-tier precedence**: Environment variables < settings function < explicit init
4. **No credential storage**: Credentials never appear in code or internal state
5. **Provider-specific auth flows**: Each LLM provider follows its native authentication mechanism

This design enables secure credential management while maintaining flexibility for different deployment patterns (local development, containerized environments, cloud platforms).

---

# Page: App Namespaces and Multi-tenancy

# App Namespaces and Multi-tenancy

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [docs/docs/core/settings.mdx](docs/docs/core/settings.mdx)
- [docs/docs/getting_started/overview.md](docs/docs/getting_started/overview.md)
- [python/cocoindex/setting.py](python/cocoindex/setting.py)

</details>



This document explains the application namespace system in CocoIndex, which enables environment isolation and multi-tenant deployments. Namespaces allow multiple teams, environments (dev/staging/production), or applications to share the same CocoIndex database infrastructure while maintaining complete data isolation.

For general configuration options, see [Configuration and Settings](#10). For authentication and secrets management, see [Authentication and Secrets Management](#10.1).

## Overview

The application namespace is a string prefix that is automatically prepended to all flow names, internal state tables, and optionally to target tables. This prefixing creates logical boundaries within a shared database, preventing conflicts and enabling independent lifecycle management.

Key characteristics:
- Configured once at application startup via environment variable or `Settings` object
- Automatically applied to all flows registered in the process
- Affects internal state table names for complete isolation
- Optional but recommended for production deployments

**Namespace Configuration Flow**

```mermaid
graph TB
    ENV["COCOINDEX_APP_NAMESPACE<br/>environment variable"]
    SETTINGS["Settings dataclass"]
    INIT["cocoindex.init()"]
    ENGINE["_engine module<br/>(Rust)"]
    
    FLOW["@flow_def<br/>decorated function"]
    QUALIFIED["Fully qualified<br/>flow name"]
    
    ENV --> SETTINGS
    SETTINGS --> INIT
    INIT --> ENGINE
    
    FLOW --> ENGINE
    ENGINE --> QUALIFIED
    
    QUALIFIED --> STATE_TABLE["Internal state table:<br/>namespace__flowname__state"]
    QUALIFIED --> FLOW_REG["Flow registry entry:<br/>namespace.flowname"]
```

Sources: [python/cocoindex/setting.py:12-26](), [docs/docs/core/settings.mdx:93-106]()

## Namespace Configuration

The application namespace is specified through the `app_namespace` field in the `Settings` dataclass. There are three ways to configure it, listed in order of precedence (later methods override earlier ones):

| Method | Priority | Usage |
|--------|----------|-------|
| Environment variable `COCOINDEX_APP_NAMESPACE` | Lowest | Simple, suitable for development |
| `@cocoindex.settings` decorated function | Medium | Flexible, suitable for dynamic configuration |
| `cocoindex.init(Settings(...))` | Highest | Most explicit, suitable for complex applications |

### Environment Variable Configuration

The simplest approach for setting a namespace:

```bash
export COCOINDEX_APP_NAMESPACE=staging
```

When CocoIndex initializes, it reads this value from the environment via `Settings.from_env()` [python/cocoindex/setting.py:122]().

### Programmatic Configuration

Using the `@cocoindex.settings` decorator:

```python
@cocoindex.settings
def cocoindex_settings() -> cocoindex.Settings:
    return cocoindex.Settings(
        app_namespace="staging",
        database=cocoindex.DatabaseConnectionSpec(...)
    )
```

Or directly with `cocoindex.init()`:

```python
cocoindex.init(
    cocoindex.Settings(
        app_namespace="production",
        database=...
    )
)
```

Sources: [python/cocoindex/setting.py:75-128](), [docs/docs/core/settings.mdx:14-87]()

## Flow Name Qualification

When an application namespace is set, all flow names are automatically qualified with the namespace prefix using a period (`.`) as the delimiter.

**Flow Name Resolution**

```mermaid
graph LR
    CODE["Python code:<br/>@flow_def('MyFlow')"]
    
    NS_EMPTY["app_namespace = ''"]
    NS_SET["app_namespace = 'staging'"]
    
    QUALIFIED_EMPTY["Qualified name:<br/>'MyFlow'"]
    QUALIFIED_SET["Qualified name:<br/>'staging.MyFlow'"]
    
    CODE --> NS_EMPTY
    CODE --> NS_SET
    
    NS_EMPTY --> QUALIFIED_EMPTY
    NS_SET --> QUALIFIED_SET
```

The `get_app_namespace()` function retrieves the current namespace with optional delimiter appending:

```python
# Get namespace without delimiter
namespace = cocoindex.get_app_namespace()  # Returns "staging"

# Get namespace with trailing period
namespace = cocoindex.get_app_namespace(trailing_delimiter=".")  # Returns "staging."
```

The `split_app_namespace()` function parses qualified names back into their components:

```python
namespace, flow_name = cocoindex.split_app_namespace("staging.MyFlow", ".")
# namespace = "staging"
# flow_name = "MyFlow"
```

Sources: [python/cocoindex/setting.py:12-26]()

## Multi-Environment Isolation

The primary use case for namespaces is isolating different deployment environments that share the same database infrastructure.

**Environment Isolation Architecture**

```mermaid
graph TB
    subgraph "Shared PostgreSQL Database"
        subgraph "Development Namespace"
            DEV_FLOW1["Flow: dev.data_pipeline"]
            DEV_STATE1["Table: dev__data_pipeline__state"]
            DEV_TARGET1["Table: dev__embeddings"]
        end
        
        subgraph "Staging Namespace"
            STG_FLOW1["Flow: staging.data_pipeline"]
            STG_STATE1["Table: staging__data_pipeline__state"]
            STG_TARGET1["Table: staging__embeddings"]
        end
        
        subgraph "Production Namespace"
            PROD_FLOW1["Flow: production.data_pipeline"]
            PROD_STATE1["Table: production__data_pipeline__state"]
            PROD_TARGET1["Table: production__embeddings"]
        end
    end
    
    DEV_APP["Development App<br/>COCOINDEX_APP_NAMESPACE=dev"]
    STG_APP["Staging App<br/>COCOINDEX_APP_NAMESPACE=staging"]
    PROD_APP["Production App<br/>COCOINDEX_APP_NAMESPACE=production"]
    
    DEV_APP --> DEV_FLOW1
    STG_APP --> STG_FLOW1
    PROD_APP --> PROD_FLOW1
    
    DEV_FLOW1 --> DEV_STATE1
    DEV_FLOW1 --> DEV_TARGET1
    STG_FLOW1 --> STG_STATE1
    STG_FLOW1 --> STG_TARGET1
    PROD_FLOW1 --> PROD_STATE1
    PROD_FLOW1 --> PROD_TARGET1
```

This isolation provides several benefits:

- **Independent Development**: Developers can test flows without affecting staging or production
- **Safe Staging**: Staging environment can mirror production configuration without data conflicts
- **Cost Efficiency**: Single database instance serves multiple environments
- **Simplified Operations**: One connection string, multiple environments

Example deployment configuration:

```bash
# Development
COCOINDEX_APP_NAMESPACE=dev
COCOINDEX_DATABASE_URL=postgres://user:pass@localhost/cocoindex

# Staging
COCOINDEX_APP_NAMESPACE=staging
COCOINDEX_DATABASE_URL=postgres://user:pass@localhost/cocoindex

# Production
COCOINDEX_APP_NAMESPACE=production
COCOINDEX_DATABASE_URL=postgres://user:pass@localhost/cocoindex
```

Sources: [docs/docs/core/settings.mdx:93-106]()

## Multi-Tenant Deployments

Namespaces also enable multi-tenant architectures where multiple teams or customers share the same CocoIndex deployment.

**Multi-Tenant Architecture**

```mermaid
graph TB
    subgraph "CocoIndex Deployment"
        ROUTER["Application Router"]
        
        subgraph "Team A Resources"
            TEAM_A_FLOW["teamA.document_indexing"]
            TEAM_A_STATE["teamA__document_indexing__state"]
            TEAM_A_TARGETS["teamA__* target tables"]
        end
        
        subgraph "Team B Resources"
            TEAM_B_FLOW["teamB.document_indexing"]
            TEAM_B_STATE["teamB__document_indexing__state"]
            TEAM_B_TARGETS["teamB__* target tables"]
        end
        
        subgraph "Team C Resources"
            TEAM_C_FLOW["teamC.document_indexing"]
            TEAM_C_STATE["teamC__document_indexing__state"]
            TEAM_C_TARGETS["teamC__* target tables"]
        end
    end
    
    CLIENT_A["Team A Client<br/>namespace=teamA"]
    CLIENT_B["Team B Client<br/>namespace=teamB"]
    CLIENT_C["Team C Client<br/>namespace=teamC"]
    
    CLIENT_A --> ROUTER
    CLIENT_B --> ROUTER
    CLIENT_C --> ROUTER
    
    ROUTER --> TEAM_A_FLOW
    ROUTER --> TEAM_B_FLOW
    ROUTER --> TEAM_C_FLOW
    
    TEAM_A_FLOW --> TEAM_A_STATE
    TEAM_A_FLOW --> TEAM_A_TARGETS
    TEAM_B_FLOW --> TEAM_B_STATE
    TEAM_B_FLOW --> TEAM_B_TARGETS
    TEAM_C_FLOW --> TEAM_C_STATE
    TEAM_C_FLOW --> TEAM_C_TARGETS
```

In a multi-tenant deployment:
- Each team/customer is assigned a unique namespace
- All flow definitions and state are isolated
- Teams cannot access or modify other teams' flows
- Administrative operations (setup, drop, update) are scoped to the namespace

Sources: [docs/docs/core/settings.mdx:93-106]()

## Database Isolation Mechanism

Namespaces achieve isolation through systematic prefixing of database objects. The prefixing convention uses double underscores (`__`) as delimiters.

### Internal State Tables

CocoIndex stores incremental processing state in PostgreSQL tables. These tables are automatically prefixed with the namespace:

| Flow Name | Namespace | Internal State Table |
|-----------|-----------|---------------------|
| `MyFlow` | *(empty)* | `myflow__state` |
| `MyFlow` | `staging` | `staging__myflow__state` |
| `MyFlow` | `production` | `production__myflow__state` |

### Target Table Prefixing

When exporting to target databases (PostgreSQL, Qdrant, Neo4j), the namespace can optionally prefix target table/collection names. This is typically configured in the target specification:

```python
# Postgres target with namespace-aware table naming
postgres_target = PostgresTargetSpec(
    table_name=f"{cocoindex.get_app_namespace(trailing_delimiter='__')}embeddings",
    ...
)
```

**Database Object Hierarchy**

```mermaid
graph TB
    DB["PostgreSQL Database"]
    
    subgraph "No Namespace"
        NO_NS_STATE["flow1__state"]
        NO_NS_TARGET["embeddings"]
    end
    
    subgraph "Namespace: staging"
        STG_STATE["staging__flow1__state"]
        STG_TARGET["staging__embeddings"]
    end
    
    subgraph "Namespace: production"
        PROD_STATE["production__flow1__state"]
        PROD_TARGET["production__embeddings"]
    end
    
    DB --> NO_NS_STATE
    DB --> NO_NS_TARGET
    DB --> STG_STATE
    DB --> STG_TARGET
    DB --> PROD_STATE
    DB --> PROD_TARGET
```

Sources: [python/cocoindex/setting.py:12-26]()

## Namespace Lifecycle Operations

All flow lifecycle operations respect namespace boundaries:

### Flow Listing

```bash
# List all flows in the current namespace
cocoindex ls

# With namespace "staging", shows:
# - staging.flow1
# - staging.flow2
```

### Setup and Drop

```bash
# Setup creates namespace-prefixed state tables
cocoindex setup flow1
# Creates: staging__flow1__state

# Drop removes only namespace-scoped resources
cocoindex drop flow1
# Drops: staging__flow1__state and staging-prefixed targets
```

### Update Operations

```bash
# Update operates only on namespace-scoped flows
cocoindex update flow1
# Updates: staging.flow1
```

The namespace creates a complete isolation boundary - operations in one namespace cannot affect flows or data in another namespace, even when sharing the same database.

Sources: [docs/docs/core/settings.mdx:93-106]()

## Empty Namespace Behavior

When `app_namespace` is not set (empty string), flows operate in the default unnamed namespace:

- Flow names have no prefix: `MyFlow` (not `".MyFlow"`)
- State tables use flow name only: `myflow__state`
- No delimiter is prepended

The `get_app_namespace()` function handles this case explicitly [python/cocoindex/setting.py:14-17]():

```python
def get_app_namespace(*, trailing_delimiter: str | None = None) -> str:
    app_namespace: str = _engine.get_app_namespace()
    if app_namespace == "" or trailing_delimiter is None:
        return app_namespace
    return f"{app_namespace}{trailing_delimiter}"
```

This ensures that empty namespaces don't result in leading delimiters (e.g., `.MyFlow`).

Sources: [python/cocoindex/setting.py:12-17]()

## Best Practices

### Namespace Naming Conventions

- **Environment-based**: Use lowercase environment names (`dev`, `staging`, `production`)
- **Team-based**: Use team identifiers (`team_data`, `team_ml`, `research`)
- **Customer-based**: Use customer IDs or slugs (`customer_123`, `acme_corp`)
- **Avoid special characters**: Stick to alphanumeric characters and underscores

### When to Use Namespaces

| Scenario | Use Namespace? | Example |
|----------|---------------|---------|
| Single developer, local testing | Optional | No namespace or `dev` |
| Multiple environments on shared DB | **Yes** | `dev`, `staging`, `prod` |
| Multiple teams on shared DB | **Yes** | `teamA`, `teamB` |
| SaaS multi-tenant deployment | **Yes** | Customer identifiers |
| Dedicated database per environment | Optional | Not needed for isolation |

### Configuration Management

Store namespace configuration outside of code:

```bash
# .env.dev
COCOINDEX_APP_NAMESPACE=dev

# .env.staging
COCOINDEX_APP_NAMESPACE=staging

# .env.production
COCOINDEX_APP_NAMESPACE=production
```

### Migration Considerations

When migrating flows between namespaces:

1. **No automatic migration**: Namespaces create separate state - data doesn't automatically transfer
2. **Manual migration**: Use separate flow definitions or custom scripts to copy data
3. **Cleanup**: Drop old namespace resources after migration verification

Sources: [python/cocoindex/setting.py:1-172](), [docs/docs/core/settings.mdx:1-176]()

## Implementation Details

The namespace system is implemented at the FFI boundary between Python and Rust. The `_engine` module (Rust) maintains the canonical namespace value, accessed through `_engine.get_app_namespace()` [python/cocoindex/setting.py:14]().

**Namespace Resolution at Runtime**

```mermaid
sequenceDiagram
    participant App as "Python Application"
    participant Settings as "Settings.from_env()"
    participant Init as "cocoindex.init()"
    participant Engine as "_engine module<br/>(Rust)"
    participant Flow as "Flow Instance"
    
    App->>Settings: Load configuration
    Settings->>Settings: Read COCOINDEX_APP_NAMESPACE
    Settings->>Init: Pass Settings object
    Init->>Engine: Set app_namespace
    Note over Engine: Stores namespace in LibContext
    
    App->>Flow: Register @flow_def
    Flow->>Engine: get_app_namespace()
    Engine->>Flow: Return "staging"
    Flow->>Flow: Qualify name: staging.MyFlow
    Flow->>Engine: Register flow with qualified name
```

The Rust engine stores the namespace in `LibContext` (global state), ensuring consistent namespace application across all flows in the process. This prevents accidental namespace conflicts or mixing.

Sources: [python/cocoindex/setting.py:12-26](), [python/cocoindex/setting.py:75-128]()

---

# Page: Database Connection and Execution Options

# Database Connection and Execution Options

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [Cargo.lock](Cargo.lock)
- [Cargo.toml](Cargo.toml)
- [docs/docs/core/settings.mdx](docs/docs/core/settings.mdx)
- [docs/docs/getting_started/overview.md](docs/docs/getting_started/overview.md)
- [python/cocoindex/setting.py](python/cocoindex/setting.py)
- [rust/cocoindex/Cargo.toml](rust/cocoindex/Cargo.toml)
- [rust/cocoindex/src/lib_context.rs](rust/cocoindex/src/lib_context.rs)
- [rust/cocoindex/src/prelude.rs](rust/cocoindex/src/prelude.rs)
- [rust/py_utils/Cargo.toml](rust/py_utils/Cargo.toml)
- [rust/py_utils/src/future.rs](rust/py_utils/src/future.rs)
- [rust/utils/Cargo.toml](rust/utils/Cargo.toml)
- [rust/utils/src/error.rs](rust/utils/src/error.rs)
- [rust/utils/src/prelude.rs](rust/utils/src/prelude.rs)
- [rust/utils/src/retryable.rs](rust/utils/src/retryable.rs)

</details>



This page documents the database connection configuration and global execution options in CocoIndex. It covers `DatabaseConnectionSpec` for PostgreSQL connection pooling and `GlobalExecutionOptions` for concurrency control.

For general settings configuration (environment variables, setting functions, `cocoindex.init()`), see [Configuration and Settings](#10). For authentication management, see [Authentication and Secrets Management](#10.1). For namespace configuration, see [App Namespaces and Multi-tenancy](#10.2).

---

## DatabaseConnectionSpec

The `DatabaseConnectionSpec` class configures PostgreSQL database connections used for internal state storage and as target databases. It is defined in [python/cocoindex/setting.py:28-40]().

### Configuration Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `url` | `str` | Required | PostgreSQL connection URL |
| `user` | `str \| None` | `None` | Username (overrides URL) |
| `password` | `str \| None` | `None` | Password (overrides URL) |
| `max_connections` | `int` | `25` | Maximum pool size |
| `min_connections` | `int` | `5` | Minimum pool size |

The `user` and `password` fields override credentials in the URL, avoiding URL encoding issues with special characters.

**Environment Variables**: `COCOINDEX_DATABASE_URL`, `COCOINDEX_DATABASE_USER`, `COCOINDEX_DATABASE_PASSWORD`, `COCOINDEX_DATABASE_MAX_CONNECTIONS`, `COCOINDEX_DATABASE_MIN_CONNECTIONS`

Sources: [python/cocoindex/setting.py:28-40](), [docs/docs/core/settings.mdx:108-138]()

### Connection URL Format

```
postgres://[user[:password]@]host[:port]/database
```

If `user` or `password` contain special characters, provide them via separate fields rather than URL-encoding them in the connection string.

**Supabase Configuration**: For Supabase-hosted PostgreSQL:
- Use **Direct connection** URL if on IPv6
- Use **Session pooler** URL otherwise (adjust pool size limits)
- Transaction pooler is not supported

Sources: [docs/docs/core/settings.mdx:139-150]()

### Example Configuration

```python
import cocoindex

# Via Settings object
settings = cocoindex.Settings(
    database=cocoindex.DatabaseConnectionSpec(
        url="postgres://localhost/cocoindex",
        user="cocoindex_user",
        password="secret_password",
        max_connections=50,
        min_connections=10
    )
)
cocoindex.init(settings)

# Via environment variables
# COCOINDEX_DATABASE_URL=postgres://localhost/cocoindex
# COCOINDEX_DATABASE_USER=cocoindex_user
# COCOINDEX_DATABASE_PASSWORD=secret_password
# COCOINDEX_DATABASE_MAX_CONNECTIONS=50
# COCOINDEX_DATABASE_MIN_CONNECTIONS=10
```

Sources: [docs/docs/core/settings.mdx:42-70]()

---

## Connection Pool Management

CocoIndex maintains a pool of database connections for efficient reuse. Connection pooling is implemented in the Rust engine using SQLx's `PgPool`.

### DbPools Architecture

```mermaid
graph TB
    Settings["Settings<br/>(Python)"]
    LibContext["LibContext"]
    DbPools["DbPools"]
    PoolsMap["HashMap&lt;PoolKey, PoolValue&gt;"]
    PoolKey["PoolKey<br/>(url, user)"]
    OnceCell["OnceCell&lt;PgPool&gt;"]
    PgPool["PgPool<br/>(SQLx Pool)"]
    
    Settings -->|"create_lib_context"| LibContext
    LibContext -->|"contains"| DbPools
    DbPools -->|"pools: Mutex"| PoolsMap
    PoolsMap -->|"key"| PoolKey
    PoolsMap -->|"value"| OnceCell
    OnceCell -->|"get_or_try_init"| PgPool
    
    PgPool -->|"max_connections"| MaxConn["max_connections: u32"]
    PgPool -->|"min_connections"| MinConn["min_connections: u32"]
    PgPool -->|"acquire_timeout"| Timeout["acquire_timeout: 5 min"]
```

**DbPools Structure**: The `DbPools` struct [rust/cocoindex/src/lib_context.rs:177-180]() maintains a thread-safe map of connection pools, keyed by `(url, user)` tuples [rust/cocoindex/src/lib_context.rs:174](). This ensures the same database with the same user shares a single pool.

**Lazy Initialization**: Connection pools use `OnceCell` for lazy initialization [rust/cocoindex/src/lib_context.rs:189-228](). The first request to a database creates the pool; subsequent requests reuse it.

Sources: [rust/cocoindex/src/lib_context.rs:177-230]()

### Pool Creation Process

```mermaid
sequenceDiagram
    participant Caller
    participant DbPools
    participant OnceCell
    participant PgPoolOptions
    participant Database
    
    Caller->>DbPools: get_pool(conn_spec)
    DbPools->>DbPools: get or create OnceCell for key
    DbPools->>OnceCell: get_or_try_init()
    
    alt First call
        OnceCell->>PgPoolOptions: create with max_connections=1
        PgPoolOptions->>Database: connect (30s timeout)
        Database-->>PgPoolOptions: connection OK
        PgPoolOptions->>PgPoolOptions: create actual pool
        Note over PgPoolOptions: max_connections from spec<br/>acquire_timeout = 5 min<br/>acquire_slow_threshold = 10s
        PgPoolOptions->>Database: establish pool
        Database-->>OnceCell: PgPool
    else Already initialized
        OnceCell-->>DbPools: cached PgPool
    end
    
    DbPools-->>Caller: PgPool
```

**Two-Phase Connection**: Pool creation uses a two-phase approach [rust/cocoindex/src/lib_context.rs:199-224]():

1. **Validation Phase**: Create a 1-connection pool with 30-second timeout to verify connectivity
2. **Pool Creation**: If validation succeeds, create the actual pool with specified size

This prevents long hangs when database is unreachable.

**Timeouts**:
- Connection validation: 30 seconds [rust/cocoindex/src/lib_context.rs:204]()
- Connection acquisition: 5 minutes [rust/cocoindex/src/lib_context.rs:220]()
- Slow acquisition warning: 10 seconds [rust/cocoindex/src/lib_context.rs:218]()

Sources: [rust/cocoindex/src/lib_context.rs:189-228]()

### Pool Configuration Parameters

The `PgPoolOptions` are configured from `DatabaseConnectionSpec` [rust/cocoindex/src/lib_context.rs:215-224]():

```rust
let pool_options = PgPoolOptions::new()
    .max_connections(conn_spec.max_connections)
    .min_connections(conn_spec.min_connections)
    .acquire_slow_level(log::LevelFilter::Info)
    .acquire_slow_threshold(Duration::from_secs(10))
    .acquire_timeout(Duration::from_secs(5 * 60));
```

**Connection Lifecycle**:
- `max_connections`: Upper bound on concurrent connections
- `min_connections`: Connections kept warm for quick access
- Idle connections beyond `min_connections` are closed over time
- Slow acquisitions (>10s) are logged at INFO level

Sources: [rust/cocoindex/src/lib_context.rs:215-224](), [Cargo.toml:37-43]()

---

## GlobalExecutionOptions

The `GlobalExecutionOptions` class controls concurrency for source operations across all flows. It prevents memory exhaustion by limiting inflight data [python/cocoindex/setting.py:42-49]().

### Configuration Fields

| Field | Type | Default | Description |
|-------|------|---------|-------------|
| `source_max_inflight_rows` | `int \| None` | `1024` | Max concurrent rows across all sources |
| `source_max_inflight_bytes` | `int \| None` | `None` | Max concurrent bytes across all sources |

**Environment Variables**: `COCOINDEX_SOURCE_MAX_INFLIGHT_ROWS`, `COCOINDEX_SOURCE_MAX_INFLIGHT_BYTES`

Both limits are enforced simultaneously when specified. If a source operation would exceed either limit, it waits until previous operations complete.

Sources: [python/cocoindex/setting.py:42-49](), [docs/docs/core/settings.mdx:152-161]()

### Concurrency Control Flow

```mermaid
graph TB
    GlobalExecOpts["GlobalExecutionOptions"]
    Settings["Settings"]
    LibContext["LibContext"]
    ConcurController["ConcurrencyController"]
    Options["concur_control::Options"]
    
    GlobalExecOpts -->|"part of"| Settings
    Settings -->|"create_lib_context"| LibContext
    LibContext -->|"global_concurrency_controller"| ConcurController
    
    Options -->|"max_inflight_rows"| RowLimit["source_max_inflight_rows"]
    Options -->|"max_inflight_bytes"| ByteLimit["source_max_inflight_bytes"]
    
    GlobalExecOpts -->|"provides"| Options
    Options -->|"configure"| ConcurController
    
    SourceOp["Source Operations"]
    SourceOp -->|"check limits"| ConcurController
    ConcurController -->|"admit or wait"| SourceOp
```

The `ConcurrencyController` [rust/cocoindex/src/lib_context.rs:321-327]() is created when `LibContext` is initialized, using values from `GlobalExecutionOptions`.

Sources: [rust/cocoindex/src/lib_context.rs:321-327](), [python/cocoindex/setting.py:42-49]()

### Per-Source vs Global Limits

CocoIndex supports both global and per-source concurrency limits:

- **Global limits** (covered here): Apply across all sources in all flows
- **Per-source limits**: Configured per source operation (see flow definition documentation)

When both are specified, **both must be satisfied** to admit additional rows. This allows:
- Global backpressure to prevent memory exhaustion
- Per-source limits to prevent overwhelming specific data sources

**Example**: With `source_max_inflight_rows=1024` global and a source configured with `max_inflight_rows=100`, that source is limited to 100 rows, but all sources together cannot exceed 1024 rows.

Sources: [docs/docs/core/settings.mdx:159-161]()

### Concurrency Control Implementation

The concurrency controller uses semaphore-like permits:

```mermaid
stateDiagram-v2
    [*] --> CheckLimits: Source reads batch
    
    CheckLimits --> WaitForPermit: Limits exceeded
    CheckLimits --> AcquirePermit: Limits OK
    
    WaitForPermit --> AcquirePermit: Permits available
    
    AcquirePermit --> Processing: Permit acquired
    
    Processing --> ReleasePermit: Batch processed
    
    ReleasePermit --> [*]
    
    note right of CheckLimits
        Check both row and byte limits
        if configured
    end note
    
    note right of Processing
        Source data flows through
        transformation pipeline
    end note
```

**Permit Lifecycle**:
1. Source operation requests permits for batch size
2. If limits exceeded, operation waits until permits available
3. Operation acquires permits and processes batch
4. Operation releases permits when batch completes

This ensures bounded memory usage regardless of source parallelism.

Sources: [rust/cocoindex/src/lib_context.rs:321-327]()

---

## Settings Propagation

Settings flow from Python configuration through the FFI boundary into the Rust engine.

### Settings Flow Diagram

```mermaid
graph TB
    subgraph Python["Python Layer"]
        EnvVars["Environment Variables"]
        SettingsFunc["@cocoindex.settings function"]
        InitCall["cocoindex.init(Settings)"]
        SettingsObj["Settings dataclass"]
        
        EnvVars -.->|"Settings.from_env()"| SettingsObj
        SettingsFunc -.-> SettingsObj
        InitCall -.-> SettingsObj
    end
    
    subgraph FFI["FFI Boundary"]
        RegisterSettings["_engine.register_settings_fn"]
        InitEngine["_engine.init"]
    end
    
    subgraph Rust["Rust Engine"]
        GetSettings["get_settings()"]
        CreateLibContext["create_lib_context"]
        LibContextObj["LibContext"]
        DbPools2["DbPools"]
        PersistenceCtx["PersistenceContext"]
        ConcurControl["ConcurrencyController"]
    end
    
    SettingsFunc -->|"register"| RegisterSettings
    SettingsObj -->|"explicit init"| InitEngine
    
    RegisterSettings --> GetSettings
    InitEngine --> CreateLibContext
    GetSettings -.->|"fallback"| CreateLibContext
    
    CreateLibContext --> LibContextObj
    LibContextObj --> DbPools2
    LibContextObj --> PersistenceCtx
    LibContextObj --> ConcurControl
    
    PersistenceCtx -->|"builtin_db_pool"| Pool["PgPool"]
```

**Priority Order** (highest to lowest):
1. `cocoindex.init(Settings)` - explicit initialization [rust/cocoindex/src/lib_context.rs:353-361]()
2. `@cocoindex.settings` function - registered callback [rust/cocoindex/src/lib_context.rs:343-348]()
3. Environment variables - loaded via `Settings.from_env()` [python/cocoindex/setting.py:83-128]()

Sources: [rust/cocoindex/src/lib_context.rs:331-374](), [python/cocoindex/setting.py:74-128]()

### LibContext Initialization

```mermaid
sequenceDiagram
    participant Python
    participant FFI
    participant Rust
    participant DbPools
    participant Setup
    
    Python->>FFI: cocoindex.init(settings)
    FFI->>Rust: init_lib_context(settings)
    
    alt Settings provided
        Rust->>Rust: use provided settings
    else Settings not provided
        Rust->>Rust: call get_settings()
        Note over Rust: Uses registered function<br/>or Settings.from_env()
    end
    
    Rust->>Rust: create_lib_context(settings)
    
    alt Database configured
        Rust->>DbPools: get_pool(database_spec)
        DbPools->>DbPools: create PgPool
        Rust->>Setup: get_existing_setup_state(pool)
        Setup-->>Rust: AllSetupStates
        Rust->>Rust: create PersistenceContext
    else No database
        Rust->>Rust: persistence_ctx = None
    end
    
    Rust->>Rust: create ConcurrencyController
    Note over Rust: from global_execution_options
    
    Rust->>Rust: create LibContext
    Rust-->>Python: initialized
```

**Lazy Initialization**: If `cocoindex.init()` is not called explicitly, `LibContext` is initialized lazily when first needed [rust/cocoindex/src/lib_context.rs:363-374](). This occurs when any flow method is called.

**Thread Safety**: `LibContext` initialization uses a `tokio::sync::Mutex` to ensure only one instance is created [rust/cocoindex/src/lib_context.rs:350-351]().

Sources: [rust/cocoindex/src/lib_context.rs:286-329](), [rust/cocoindex/src/lib_context.rs:353-374]()

### PersistenceContext Structure

When a database is configured, `LibContext` contains a `PersistenceContext` [rust/cocoindex/src/lib_context.rs:236-239]():

```rust
pub struct PersistenceContext {
    pub builtin_db_pool: PgPool,
    pub setup_ctx: tokio::sync::RwLock<LibSetupContext>,
}
```

**Components**:
- `builtin_db_pool`: The primary connection pool for internal state storage
- `setup_ctx`: Setup state tracking for incremental processing

**Access**: Use `LibContext::require_persistence_ctx()` [rust/cocoindex/src/lib_context.rs:271-279]() to access, which returns an error if no database is configured.

Sources: [rust/cocoindex/src/lib_context.rs:236-239](), [rust/cocoindex/src/lib_context.rs:271-283]()

---

## Usage Patterns

### Checking Database Availability

```python
import cocoindex

lib_context = cocoindex._engine.get_lib_context()

# Check if database is configured
try:
    pool = lib_context.require_builtin_db_pool()
    print("Database available")
except Exception as e:
    print(f"No database configured: {e}")
```

Operations requiring persistence (setup, update, drop) automatically check for database availability and fail with clear error messages [rust/cocoindex/src/lib_context.rs:272-278]().

Sources: [rust/cocoindex/src/lib_context.rs:271-283]()

### Adjusting Connection Pool Size

For high-concurrency workloads:

```python
cocoindex.init(
    cocoindex.Settings(
        database=cocoindex.DatabaseConnectionSpec(
            url="postgres://localhost/cocoindex",
            max_connections=100,  # Increase for more parallelism
            min_connections=20     # Keep more warm connections
        )
    )
)
```

**Guidelines**:
- Default (`max_connections=25`) suits most workloads
- Increase for high parallelism or many flows
- Ensure PostgreSQL `max_connections` setting exceeds pool size
- Monitor slow acquisition warnings (>10s) to detect pool exhaustion

Sources: [python/cocoindex/setting.py:28-40](), [rust/cocoindex/src/lib_context.rs:215-224]()

### Tuning Concurrency Limits

Adjust global limits based on available memory:

```python
cocoindex.init(
    cocoindex.Settings(
        global_execution_options=cocoindex.GlobalExecutionOptions(
            source_max_inflight_rows=2048,      # Double default
            source_max_inflight_bytes=1024**3   # 1 GB
        )
    )
)
```

**Considerations**:
- Higher limits increase throughput but risk OOM
- Set `source_max_inflight_bytes` when processing large documents/images
- Combine with per-source limits for fine-grained control
- Monitor memory usage and adjust accordingly

Sources: [python/cocoindex/setting.py:42-49](), [rust/cocoindex/src/lib_context.rs:321-327]()

---

## Error Handling

### Connection Errors

Connection failures during pool creation are wrapped with context [rust/cocoindex/src/lib_context.rs:208-210]():

```
Failed to connect to database postgres://localhost/cocoindex
```

**Common Causes**:
- Database not running
- Incorrect credentials
- Network issues
- Firewall blocking connection

### Pool Exhaustion

When all connections are in use and acquisition times out (5 minutes):

```
Error: Failed to acquire connection from pool
```

**Resolution**:
- Increase `max_connections`
- Check for connection leaks (operations not releasing connections)
- Ensure database can handle connection count

### Permission Denied

If database user lacks required permissions:

```
Error: permission denied for schema public
```

**Required Permissions**:
- `CREATE` on schema (for setup)
- `SELECT`, `INSERT`, `UPDATE`, `DELETE` on tables
- For targets: appropriate permissions on target schema

Sources: [rust/cocoindex/src/lib_context.rs:189-228]()

---

## Advanced Configuration

### Multiple Database Pools

The `DbPools` structure [rust/cocoindex/src/lib_context.rs:177-230]() automatically manages pools for multiple databases. When exporting to target databases different from the internal storage database, separate pools are created.

**Pool Key**: Each pool is keyed by `(url, user)` [rust/cocoindex/src/lib_context.rs:174](), so:
- Same URL + different users = separate pools
- Same URL + same user = shared pool

This enables:
- Internal state in one database
- Multiple target databases
- Efficient connection reuse

Sources: [rust/cocoindex/src/lib_context.rs:174-230]()

### Connection Timeout Tuning

The acquire timeout is hardcoded to 5 minutes [rust/cocoindex/src/lib_context.rs:220](). For operations expected to take longer:

1. Increase `max_connections` to reduce contention
2. Add per-source concurrency limits to prevent pool exhaustion
3. Monitor slow acquisition warnings to detect bottlenecks

Sources: [rust/cocoindex/src/lib_context.rs:215-224]()

### Testing Without Database

For testing or evaluation, omit database configuration:

```python
# No database - in-memory only
cocoindex.init(cocoindex.Settings())
```

This allows flow evaluation without persistence. Operations requiring database (setup, update, drop) will fail with clear error messages.

Sources: [rust/cocoindex/src/lib_context.rs:271-279](), [rust/cocoindex/src/lib_context.rs:311-314]()

---

# Page: CLI and Development Tools

# CLI and Development Tools

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [dev/generate_cli_docs.py](dev/generate_cli_docs.py)
- [docs/docs/core/cli-commands.md](docs/docs/core/cli-commands.md)
- [docs/docs/core/cli.mdx](docs/docs/core/cli.mdx)
- [python/cocoindex/__init__.py](python/cocoindex/__init__.py)
- [python/cocoindex/cli.py](python/cocoindex/cli.py)
- [python/cocoindex/flow.py](python/cocoindex/flow.py)
- [python/cocoindex/lib.py](python/cocoindex/lib.py)
- [python/cocoindex/setup.py](python/cocoindex/setup.py)
- [python/cocoindex/user_app_loader.py](python/cocoindex/user_app_loader.py)

</details>



This page provides an overview of CocoIndex's command-line interface and development tooling. The CLI is the primary interface for managing flows, setting up backends, and running update operations. It also includes capabilities for starting an HTTP server to enable visualization tools like CocoInsight.

For detailed documentation on individual CLI commands, see [Command Line Interface](#11.1). For information on query handler APIs, see [Query Handlers and Search APIs](#11.2). For debugging and visualization capabilities, see [CocoInsight: Visualization and Debugging](#11.3).

## CLI Architecture

The CocoIndex CLI is implemented using the Click framework in [python/cocoindex/cli.py](). The entry point is the `cli` command group, which registers subcommands for flow management operations.

### CLI Entry Point and Initialization

```mermaid
graph TB
    Invocation["User runs 'cocoindex'"]
    CLIGroup["@cli group function"]
    EnvLoad["Load .env file"]
    PathSetup["Add app_dir to sys.path"]
    Init["_initialize_cocoindex_in_process()"]
    Atexit["Register lib.stop() at exit"]
    Subcommand["Execute subcommand"]
    
    Invocation --> CLIGroup
    CLIGroup --> EnvLoad
    EnvLoad --> PathSetup
    PathSetup --> Init
    Init --> Atexit
    Atexit --> Subcommand
```

Sources: [python/cocoindex/cli.py:96-139]()

The CLI initialization process:

1. **Environment Loading**: Searches for `.env` file using `find_dotenv()` or uses explicit path from `--env-file` option. Loaded variables do not override existing system environment variables [python/cocoindex/cli.py:126-130]()

2. **Path Configuration**: If `--app-dir` is specified, it's inserted at the front of `sys.path` to enable module imports [python/cocoindex/cli.py:132-133]()

3. **Library Initialization**: Calls `_initialize_cocoindex_in_process()` which registers `lib.stop()` cleanup handler via `atexit` [python/cocoindex/cli.py:92-93]()

### Global CLI Options

| Option | Description |
|--------|-------------|
| `-e, --env-file` | Path to `.env` file for environment variables |
| `-d, --app-dir` | Directory to load apps from (added to `PYTHONPATH`) |
| `-V, --version` | Show CocoIndex version |
| `--help` | Show help message |

Sources: [python/cocoindex/cli.py:104-121]()

## APP_TARGET and APP_FLOW_SPECIFIER Syntax

CocoIndex CLI commands use two related argument formats to identify applications and flows:

### APP_TARGET Format

Used by commands that operate on entire applications (`setup`, `drop`, `server`, `ls`):

```
<module_or_path>
```

Examples:
- `main.py` - Path to Python file
- `path/to/flows.py` - Path to Python file in subdirectory
- `my_package.flows` - Installed Python module

### APP_FLOW_SPECIFIER Format

Used by commands that can target specific flows (`show`, `update`, `evaluate`):

```
<module_or_path>[:<flow_name>]
```

Examples:
- `main.py:MyFlow` - Specific flow in file
- `my_package.flows:ProductionFlow` - Specific flow in module
- `main.py` - All flows (if only one flow exists)

### Parsing Implementation

```mermaid
graph TB
    Input["APP_FLOW_SPECIFIER input"]
    Parse["_parse_app_flow_specifier()"]
    Split["Split on first colon"]
    ValidateApp["Check app_ref not empty"]
    ValidateFlow["Check flow_ref is identifier"]
    Result["Return app_ref flow_ref tuple"]
    
    Input --> Parse
    Parse --> Split
    Split --> ValidateApp
    ValidateApp --> ValidateFlow
    ValidateFlow --> Result
```

Sources: [python/cocoindex/cli.py:29-56]()

The `_parse_app_flow_specifier()` function [python/cocoindex/cli.py:29-56]():

1. Splits input on first `:` character
2. Validates app reference is not empty
3. If flow name present, validates it's a valid Python identifier using `str.isidentifier()`
4. Returns tuple of `(app_ref, flow_name | None)`

The `_get_app_ref_from_specifier()` function [python/cocoindex/cli.py:59-77]() is used by app-level commands to extract just the app reference, warning if a flow name was unnecessarily provided.

## User Application Loading

### Loading Mechanism

```mermaid
graph TB
    Target["app_target string"]
    Load["load_user_app()"]
    CheckPath["Looks like path?"]
    LoadFile["Load as file"]
    LoadModule["Load as module"]
    AddUserApp["add_user_app()"]
    
    Target --> Load
    Load --> CheckPath
    CheckPath -->|"Contains / or .py"| LoadFile
    CheckPath -->|"Otherwise"| LoadModule
    LoadFile --> AddUserApp
    LoadModule --> AddUserApp
```

Sources: [python/cocoindex/user_app_loader.py:15-53](), [python/cocoindex/cli.py:80-89]()

The application loading process in `_load_user_app()` [python/cocoindex/cli.py:80-89]():

1. **Detection**: Checks if `app_target` contains path separator or ends with `.py` [python/cocoindex/user_app_loader.py:20]()

2. **File Loading**: If path-like:
   - Converts to absolute path
   - Adds directory to `sys.path`
   - Uses `importlib.util.spec_from_file_location()` to create module spec
   - Executes module with `spec.loader.exec_module()`
   - Removes directory from `sys.path` after loading
   [python/cocoindex/user_app_loader.py:22-45]()

3. **Module Loading**: If not path-like:
   - Uses `importlib.import_module()` directly
   [python/cocoindex/user_app_loader.py:48-53]()

4. **Registration**: Calls `add_user_app()` from subprocess execution system to track loaded apps [python/cocoindex/cli.py:89]()

## Flow Selection and Management

### Flow Name Resolution

When a flow name is not provided but required, the CLI implements interactive selection:

```mermaid
graph TB
    GetFlow["_flow_by_name(name)"]
    CheckName["name provided?"]
    GetNames["flow.flow_names()"]
    CheckCount["Count of flows"]
    ReturnSingle["Return single flow"]
    Interactive["Interactive selection"]
    Error["Raise UsageError"]
    
    GetFlow --> CheckName
    CheckName -->|"Yes"| Lookup["Lookup by name"]
    CheckName -->|"No"| GetNames
    GetNames --> CheckCount
    CheckCount -->|"0"| Error
    CheckCount -->|"1"| ReturnSingle
    CheckCount -->|"Multiple"| Interactive
```

Sources: [python/cocoindex/cli.py:816-856]()

The `_flow_name()` function [python/cocoindex/cli.py:816-853]() handles three cases:

1. **Named flow**: Validates the flow exists in `flow.flow_names()`
2. **Single flow**: Returns the only available flow automatically
3. **Multiple flows**: Presents interactive selection using arrow keys:
   - Up/Down arrows navigate
   - Enter selects
   - Uses Rich Console for rendering

## Query Handler System

Query handlers enable search and query capabilities for flows. They are registered using the `@query_handler` decorator or `add_query_handler()` method.

### Query Handler Registration

```mermaid
graph TB
    Decorator["@flow.query_handler()"]
    AddHandler["Flow.add_query_handler()"]
    AsyncWrap["to_async_call(handler)"]
    WrapResult["Wrap return in dict"]
    HandlerInfo["QueryHandlerInfo object"]
    LazyLock["Acquire _lazy_flow_lock"]
    CheckBuilt["Flow already built?"]
    AddNow["engine_flow.add_query_handler()"]
    AddLazy["Append to _lazy_query_handler_args"]
    
    Decorator --> AddHandler
    AddHandler --> AsyncWrap
    AddHandler --> HandlerInfo
    AsyncWrap --> WrapResult
    WrapResult --> LazyLock
    HandlerInfo --> LazyLock
    LazyLock --> CheckBuilt
    CheckBuilt -->|"Yes"| AddNow
    CheckBuilt -->|"No"| AddLazy
```

Sources: [python/cocoindex/flow.py:885-913]()

### Query Handler Components

**QueryHandlerInfo**: Configuration for result fields [python/cocoindex/query_handler.py]()
- `result_fields`: Optional specification of result field types

**QueryOutput**: Handler return value [python/cocoindex/query_handler.py]()
- `results`: List of result dictionaries
- `query_info`: Query metadata

**Registration Process** [python/cocoindex/flow.py:885-913]():
1. Handler function is wrapped with `to_async_call()` to ensure async execution
2. Results are serialized using `dump_engine_object()`
3. Handler and info are either immediately added to built flow or queued for lazy initialization

### Usage Pattern

```python
@flow_def()
def my_flow(fb: FlowBuilder, root: DataScope):
    # Define flow...
    pass

@my_flow.query_handler("search")
def search_handler(query: str) -> QueryOutput:
    # Implement search logic
    return QueryOutput(results=[...], query_info=...)
```

Sources: [python/cocoindex/flow.py:915-930]()

## Server Mode

The `server` command starts an HTTP server that exposes flow APIs for tools like CocoInsight.

### Server Startup Flow

```mermaid
graph TB
    ServerCmd["cocoindex server command"]
    LoadApp["_load_user_app()"]
    CheckFlows["Verify flows exist"]
    DropSetup["Optional: _drop_flows()"]
    Setup["Optional: _setup_flows()"]
    StartServer["lib.start_server()"]
    EnsureBuilt["flow.ensure_all_flows_built()"]
    EngineStart["_engine.start_server()"]
    LiveUpdate["Optional: Launch FlowLiveUpdater"]
    WaitSignal["Wait for SIGINT/SIGTERM"]
    
    ServerCmd --> LoadApp
    LoadApp --> CheckFlows
    CheckFlows --> DropSetup
    DropSetup --> Setup
    Setup --> StartServer
    StartServer --> EnsureBuilt
    EnsureBuilt --> EngineStart
    EngineStart --> LiveUpdate
    LiveUpdate --> WaitSignal
```

Sources: [python/cocoindex/cli.py:726-813]()

### Server Configuration Options

**Address Configuration**:
- `--address` / `-a`: Bind address in `IP:PORT` format
- Falls back to `COCOINDEX_SERVER_ADDRESS` environment variable
[python/cocoindex/cli.py:552-556]()

**CORS Configuration**:
- `--cors-origin` / `-c`: Comma-separated list of allowed origins
- `--cors-cocoindex` / `-ci`: Allow `https://cocoindex.io`
- `--cors-local` / `-cl`: Allow `http://localhost:<port>`
- Merged with `COCOINDEX_SERVER_CORS_ORIGINS` environment variable
[python/cocoindex/cli.py:558-580](), [python/cocoindex/cli.py:768-776]()

**Update Options**:
- `--live-update` / `-L`: Enable continuous source monitoring
- `--reexport`: Force reexport even without changes
- `--full-reprocess`: Invalidate all caches
[python/cocoindex/cli.py:582-617]()

**Auto-reload**:
- `--reload` / `-r`: Watch for code changes and restart
- Uses `watchfiles.run_process()` with `PythonFilter()`
- Watches current directory and app directory
[python/cocoindex/cli.py:635-710]()

### Server Implementation

Server startup in `_run_server()` [python/cocoindex/cli.py:726-813]():

1. **Flow Validation**: Ensures at least one flow is registered [python/cocoindex/cli.py:746-762]()
2. **Setup Management**: Optionally drops and/or sets up flows [python/cocoindex/cli.py:764-786]()
3. **Server Launch**: Calls `lib.start_server()` which:
   - Ensures all flows are built via `flow.ensure_all_flows_built()` [python/cocoindex/lib.py:69]()
   - Starts engine server via `_engine.start_server()` [python/cocoindex/lib.py:70]()
4. **Live Updates**: If enabled, launches `FlowLiveUpdater` in background thread [python/cocoindex/cli.py:795-804]()
5. **Signal Handling**: Registers handlers for SIGINT and SIGTERM [python/cocoindex/cli.py:806-813]()

## Common CLI Command Patterns

### Setup and Drop Operations

```mermaid
graph TB
    SetupCmd["cocoindex setup APP_TARGET"]
    DropCmd["cocoindex drop APP_TARGET"]
    LoadApp1["Load application"]
    LoadApp2["Load application"]
    GetFlows1["flow.flows().values()"]
    GetFlows2["flow.flows().values()"]
    SetupBundle["make_setup_bundle()"]
    DropBundle["make_drop_bundle()"]
    Describe1["bundle.describe()"]
    Describe2["bundle.describe()"]
    Confirm1["User confirmation"]
    Confirm2["User confirmation"]
    Apply1["bundle.apply()"]
    Apply2["bundle.apply()"]
    
    SetupCmd --> LoadApp1
    DropCmd --> LoadApp2
    LoadApp1 --> GetFlows1
    LoadApp2 --> GetFlows2
    GetFlows1 --> SetupBundle
    GetFlows2 --> DropBundle
    SetupBundle --> Describe1
    DropBundle --> Describe2
    Describe1 --> Confirm1
    Describe2 --> Confirm2
    Confirm1 -->|"--force or yes"| Apply1
    Confirm2 -->|"--force or yes"| Apply2
```

Sources: [python/cocoindex/cli.py:281-303](), [python/cocoindex/cli.py:229-261]()

### Update Command Flow

The `update` command orchestrates flow updates:

```mermaid
graph TB
    UpdateCmd["cocoindex update APP_FLOW_SPECIFIER"]
    ParseSpec["Parse specifier"]
    LoadApp["Load application"]
    GetFlows["Get target flows"]
    ResetCheck["--reset flag?"]
    DropFlows["_drop_flows()"]
    SetupCheck["--setup flag?"]
    SetupFlows["_setup_flows()"]
    SingleCheck["Single flow?"]
    UpdateSingle["FlowLiveUpdater context"]
    UpdateAll["update_all_flows_async()"]
    Wait["updater.wait()"]
    
    UpdateCmd --> ParseSpec
    ParseSpec --> LoadApp
    LoadApp --> GetFlows
    GetFlows --> ResetCheck
    ResetCheck -->|"Yes"| DropFlows
    ResetCheck -->|"No"| SetupCheck
    DropFlows --> SetupCheck
    SetupCheck -->|"Yes"| SetupFlows
    SetupCheck -->|"No"| SingleCheck
    SetupFlows --> SingleCheck
    SingleCheck -->|"Yes"| UpdateSingle
    SingleCheck -->|"No"| UpdateAll
    UpdateSingle --> Wait
```

Sources: [python/cocoindex/cli.py:456-504]()

**Key Options**:
- `--live` / `-L`: Continuous monitoring mode
- `--reexport`: Force target reexport
- `--full-reprocess`: Invalidate caches
- `--reset`: Drop before update (implies `--setup`)
- `--setup`: Auto-setup if needed (deprecated, now default)
- `--force` / `-f`: Skip confirmations
- `--quiet` / `-q`: Suppress output

Sources: [python/cocoindex/cli.py:401-504]()

## Flow Lifecycle Management

### Setup Operations

The `_setup_flows()` helper [python/cocoindex/cli.py:281-303]() implements the common setup pattern:

1. Creates `SetupChangeBundle` via `flow.make_setup_bundle()` [python/cocoindex/cli.py:288]()
2. Calls `bundle.describe()` to get change summary and up-to-date status [python/cocoindex/cli.py:289]()
3. If not up-to-date and not forced, prompts for confirmation [python/cocoindex/cli.py:292-300]()
4. Applies changes via `bundle.apply()` [python/cocoindex/cli.py:302]()

### Drop Operations

The `_drop_flows()` helper [python/cocoindex/cli.py:229-261]() follows similar pattern:

1. Creates `SetupChangeBundle` via `flow.make_drop_bundle()` [python/cocoindex/cli.py:248]()
2. Describes changes and checks if up-to-date [python/cocoindex/cli.py:249-252]()
3. Confirms with user unless `--force` specified [python/cocoindex/cli.py:254-260]()
4. Applies drop operation [python/cocoindex/cli.py:261]()

Both operations support the `report_to_stdout` parameter to control output verbosity.

## Development Workflow Tools

### CLI Documentation Generation

The CLI documentation is auto-generated using [dev/generate_cli_docs.py](). This script:

1. Uses Click's built-in help generation via `command.get_help()` [dev/generate_cli_docs.py:217]()
2. Parses help text to extract sections [dev/generate_cli_docs.py:62-128]()
3. Formats as markdown tables [dev/generate_cli_docs.py:124-126]()
4. Escapes HTML-like tags for MDX compatibility [dev/generate_cli_docs.py:31-58]()
5. Writes to [docs/docs/core/cli-commands.md]() [dev/generate_cli_docs.py:258-259]()

This ensures CLI documentation stays synchronized with implementation.

Sources: [dev/generate_cli_docs.py:1-291]()

### Evaluation Mode

The `evaluate` command [python/cocoindex/cli.py:506-546]() runs flows without persistent storage:

```python
cocoindex evaluate app.py:MyFlow --output-dir ./results
```

- Dumps flow outputs to files instead of updating indexes
- Uses `Flow.evaluate_and_dump()` method [python/cocoindex/flow.py:802-808]()
- Supports `--cache` / `--no-cache` for intermediate data reuse
- Useful for testing and validation before production deployment

### Live Reload

The `--reload` option [python/cocoindex/cli.py:635-710]() enables development mode:

1. Uses `watchfiles.run_process()` to monitor file changes [python/cocoindex/cli.py:694-704]()
2. Applies `PythonFilter()` to watch only Python files [python/cocoindex/cli.py:699]()
3. Watches current directory and app directory [python/cocoindex/cli.py:683-692]()
4. Restarts server process on change detection
5. Preserves setup state between reloads [python/cocoindex/cli.py:717-721]()

## Integration Points

The CLI integrates with other CocoIndex subsystems:

**Flow System**: Imports `flow` module for flow management [python/cocoindex/cli.py:20]()

**Settings**: Uses `setting` module for configuration [python/cocoindex/cli.py:20]()

**Library**: Calls `lib.start_server()` and `lib.stop()` [python/cocoindex/cli.py:20]()

**Setup System**: Uses `SetupChangeBundle` for backend management [python/cocoindex/cli.py:21]()

**Runtime Context**: Accesses `execution_context` for async operations [python/cocoindex/cli.py:22]()

**Subprocess Execution**: Registers user apps via `add_user_app()` [python/cocoindex/cli.py:23]()

Sources: [python/cocoindex/cli.py:19-24]()

---

# Page: Command Line Interface

# Command Line Interface

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [dev/generate_cli_docs.py](dev/generate_cli_docs.py)
- [docs/docs/core/cli-commands.md](docs/docs/core/cli-commands.md)
- [docs/docs/core/cli.mdx](docs/docs/core/cli.mdx)
- [python/cocoindex/user_app_loader.py](python/cocoindex/user_app_loader.py)

</details>



This document provides a comprehensive reference for the CocoIndex command-line interface (CLI). The CLI is the primary tool for managing flows, updating indexes, and starting servers. For information about defining flows programmatically, see [Flow Definition System](#3). For details about query handlers and search APIs, see [Query Handlers and Search APIs](#11.2).

## Overview

The CocoIndex CLI provides a unified interface for managing the complete lifecycle of data processing flows. All commands are invoked through the `cocoindex` executable and accept an application target specifying which Python module or file contains the flow definitions.

**Command Structure:**

```
cocoindex [GLOBAL_OPTIONS] COMMAND [COMMAND_OPTIONS] [ARGUMENTS]
```

The CLI implements seven primary commands:
- `ls` - List flows in an application or backend
- `show` - Display flow specifications and schemas
- `setup` - Initialize persistent backends
- `drop` - Remove persistent backends
- `update` - Process data and update indexes
- `evaluate` - Test flows without writing to targets
- `server` - Start HTTP server for query APIs

Sources: [python/cocoindex/cli.py:96-131]()

## Global Options

All CLI commands support the following global options, specified before the command name:

| Option | Type | Description |
|--------|------|-------------|
| `--version` | flag | Display CocoIndex version and exit |
| `--env-file PATH` | path | Load environment variables from specified file (default: `.env` in current directory) |
| `--app-dir PATH` | path | Add directory to Python path for module resolution (default: current directory) |

**Example:**
```bash
cocoindex --env-file production.env --app-dir /opt/myapp update app.py
```

Sources: [python/cocoindex/cli.py:96-131]()

## Application Specifier Format

Most commands require an application specifier that identifies which Python module or file contains flow definitions. The specifier follows this format:

```
APP_TARGET[:FLOW_NAME]
```

**Valid formats:**
- `path/to/app.py` - Python file path
- `installed.module` - Installed Python module
- `path/to/app.py:MyFlow` - Specific flow in file
- `installed.module:MyFlow` - Specific flow in module

**Parsing behavior:**
- If `:FLOW_NAME` is omitted, operations apply to all flows in the module
- Flow name must be a valid Python identifier
- For single-flow applications, flow name can be omitted in commands that require a specific flow

Sources: [python/cocoindex/cli.py:29-57](), [python/cocoindex/cli.py:59-78]()

## Command Reference

### ls - List Flows

Lists flows either from a loaded application or from persisted backend storage.

**Syntax:**
```bash
cocoindex ls [APP_TARGET]
```

**Behavior:**
- **With APP_TARGET**: Lists flows defined in the application, marking those without backend setup with `[+]`
- **Without APP_TARGET**: Lists only flows with persisted setup in the backend

**Example output:**
```
flow1
flow2 [+]
flow3

Notes:
  [+]: Flows present in the current process, but missing setup.
```

Sources: [python/cocoindex/cli.py:133-177]()

---

### show - Display Flow Specification

Displays the flow specification and schema in a formatted, hierarchical view.

**Syntax:**
```bash
cocoindex show APP_FLOW_SPECIFIER [OPTIONS]
```

**Options:**

| Option | Type | Default | Description |
|--------|------|---------|-------------|
| `--color/--no-color` | flag | `--color` | Enable/disable colored output |
| `--verbose` | flag | off | Show verbose output with full details |

**APP_FLOW_SPECIFIER**: Must include flow name if application defines multiple flows.

**Example:**
```bash
cocoindex show myapp.py:DocumentFlow --verbose
```

**Output includes:**
1. Hierarchical flow specification with sources, transforms, and exports
2. Schema table showing field names, types, and attributes

Sources: [python/cocoindex/cli.py:179-217]()

---

### setup - Initialize Backends

Creates or updates persistent backends for flows, including internal storage and export targets.

**Syntax:**
```bash
cocoindex setup APP_TARGET [OPTIONS]
```

**Options:**

| Option | Short | Default | Description |
|--------|-------|---------|-------------|
| `--force` | `-f` | off | Skip confirmation prompts |
| `--reset` | | off | Drop existing setup before running setup |

**Behavior:**
1. Loads application and analyzes flow definitions
2. Checks compatibility with existing backends
3. Displays proposed changes
4. Prompts for confirmation (unless `--force`)
5. Applies changes to backends

**Example:**
```bash
cocoindex setup myapp.py --force
```

Sources: [python/cocoindex/cli.py:310-341]()

---

### drop - Remove Backend Setup

Removes persistent backends for specified flows, cleaning up all stored data and indexes.

**Syntax:**
```bash
cocoindex drop APP_TARGET [FLOW_NAME...] [OPTIONS]
```

**Options:**

| Option | Short | Default | Description |
|--------|-------|---------|-------------|
| `--force` | `-f` | off | Skip confirmation prompts |

**Modes:**
- **Drop all flows**: `cocoindex drop myapp.py`
- **Drop specific flows**: `cocoindex drop myapp.py Flow1 Flow2`

**Warning:** This operation is destructive and removes all indexed data.

Sources: [python/cocoindex/cli.py:343-389]()

---

### update - Process Data and Update Index

Executes flow processing to update indexes with the latest data from sources.

**Syntax:**
```bash
cocoindex update APP_FLOW_SPECIFIER [OPTIONS]
```

**Options:**

| Option | Short | Default | Description |
|--------|-------|---------|-------------|
| `--live` | `-L` | off | Continuous monitoring with change capture |
| `--reexport` | | off | Force reexport even without changes |
| `--reset` | | off | Drop and recreate setup before updating |
| `--force` | `-f` | off | Skip confirmation prompts |
| `--quiet` | `-q` | off | Suppress output statistics |

**Execution modes:**

1. **One-time update** (default):
   - Processes all data sources
   - Applies incremental changes only
   - Returns after completion

2. **Live update** (`--live` flag):
   - Continuously monitors sources for changes
   - Automatically processes new/modified data
   - Runs until interrupted with Ctrl+C

**Example:**
```bash
# One-time update
cocoindex update myapp.py:DocumentFlow

# Live monitoring
cocoindex update myapp.py:DocumentFlow --live

# Force full reexport
cocoindex update myapp.py --reexport --force
```

**Setup integration:**
- Automatically runs setup if backends not initialized
- Use `--reset` to drop and recreate setup first

Sources: [python/cocoindex/cli.py:391-485]()

---

### evaluate - Test Flow Without Writing

Evaluates flow logic and dumps output to files without writing to target backends. Primarily used for testing and debugging.

**Syntax:**
```bash
cocoindex evaluate APP_FLOW_SPECIFIER [OPTIONS]
```

**Options:**

| Option | Short | Default | Description |
|--------|-------|---------|-------------|
| `--output-dir PATH` | `-o` | auto-generated | Directory for output files |
| `--cache/--no-cache` | | `--cache` | Use cached intermediate results |

**Default output directory format:**
```
eval_{app_namespace}_{flow_name}_{timestamp}
```

**Example:**
```bash
cocoindex evaluate myapp.py:DocumentFlow -o ./test_output
```

Sources: [python/cocoindex/cli.py:487-528]()

---

### server - Start HTTP Server

Starts an HTTP server providing REST APIs for query handlers and CocoInsight integration.

**Syntax:**
```bash
cocoindex server APP_TARGET [OPTIONS]
```

**Options:**

| Option | Short | Default | Description |
|--------|-------|---------|-------------|
| `--address IP:PORT` | `-a` | from env | Server bind address |
| `--cors-origin ORIGINS` | `-c` | from env | CORS allowed origins (comma-separated) |
| `--cors-cocoindex` | `-ci` | off | Allow https://cocoindex.io |
| `--cors-local PORT` | `-cl` | off | Allow http://localhost:PORT |
| `--live-update` | `-L` | off | Enable continuous monitoring |
| `--reexport` | | off | Force reexport on each update |
| `--reset` | | off | Drop and recreate setup on start |
| `--force` | `-f` | off | Skip confirmation prompts |
| `--quiet` | `-q` | off | Suppress output |
| `--reload` | `-r` | off | Auto-reload on code changes |

**Server features:**
1. Serves query handler endpoints
2. Provides CocoInsight visualization APIs
3. Supports CORS for web clients
4. Optional live update integration
5. Auto-reload for development

**Example:**
```bash
# Basic server
cocoindex server myapp.py

# Development server with auto-reload
cocoindex server myapp.py --reload -ci

# Production with CORS
cocoindex server myapp.py --address 0.0.0.0:8000 --cors-origin "https://example.com"
```

**Auto-reload behavior:**
- Watches Python files in app directory and current working directory
- Restarts server on file changes
- Preserves setup on reload (does not run `--reset`)

Sources: [python/cocoindex/cli.py:530-683](), [python/cocoindex/cli.py:685-695]()

## Command Workflow Diagrams

### CLI Command Architecture

```mermaid
graph TB
    User["User Terminal"]
    
    subgraph "CLI Entry Point"
        ClickGroup["@click.group()<br/>cli()"]
        GlobalOpts["Global Options:<br/>--env-file<br/>--app-dir"]
    end
    
    subgraph "Commands"
        LsCmd["@cli.command()<br/>ls()"]
        ShowCmd["@cli.command()<br/>show()"]
        SetupCmd["@cli.command()<br/>setup()"]
        DropCmd["@cli.command()<br/>drop()"]
        UpdateCmd["@cli.command()<br/>update()"]
        EvalCmd["@cli.command()<br/>evaluate()"]
        ServerCmd["@cli.command()<br/>server()"]
    end
    
    subgraph "Helper Functions"
        ParseSpec["_parse_app_flow_specifier()"]
        LoadApp["_load_user_app()"]
        InitLib["_initialize_cocoindex_in_process()"]
        SetupFlows["_setup_flows()"]
        DropFlows["_drop_flows()"]
    end
    
    subgraph "Flow Module"
        FlowAPI["flow.flows()<br/>flow.flow_by_name()<br/>flow.Flow.update()<br/>flow.FlowLiveUpdater"]
        SetupAPI["flow.make_setup_bundle()<br/>flow.make_drop_bundle()"]
    end
    
    User --> ClickGroup
    ClickGroup --> GlobalOpts
    GlobalOpts --> LsCmd
    GlobalOpts --> ShowCmd
    GlobalOpts --> SetupCmd
    GlobalOpts --> DropCmd
    GlobalOpts --> UpdateCmd
    GlobalOpts --> EvalCmd
    GlobalOpts --> ServerCmd
    
    LsCmd --> LoadApp
    ShowCmd --> ParseSpec
    SetupCmd --> LoadApp
    DropCmd --> LoadApp
    UpdateCmd --> ParseSpec
    EvalCmd --> ParseSpec
    ServerCmd --> LoadApp
    
    ParseSpec --> LoadApp
    LoadApp --> InitLib
    
    SetupCmd --> SetupFlows
    DropCmd --> DropFlows
    UpdateCmd --> SetupFlows
    ServerCmd --> SetupFlows
    
    SetupFlows --> SetupAPI
    DropFlows --> SetupAPI
    UpdateCmd --> FlowAPI
    ServerCmd --> FlowAPI
```

Sources: [python/cocoindex/cli.py:96-131](), [python/cocoindex/cli.py:29-90]()

### Update Command Flow

```mermaid
graph TB
    UpdateCmd["cocoindex update<br/>APP_FLOW_SPECIFIER"]
    
    ParseArgs["Parse Arguments:<br/>app_ref, flow_name"]
    LoadApp["load_user_app(app_ref)"]
    GetFlows["Get Flow instances:<br/>flow_by_name() or flows()"]
    
    CheckReset{"--reset<br/>flag?"}
    DropOp["_drop_flows()"]
    
    CheckSetup{"Setup<br/>needed?"}
    SetupOp["_setup_flows()"]
    
    CheckLive{"--live<br/>flag?"}
    
    OneTime["FlowLiveUpdater<br/>live_mode=False<br/>updater.wait()"]
    LiveMode["FlowLiveUpdater<br/>live_mode=True<br/>continuous monitoring"]
    
    Stats["Print update stats<br/>unless --quiet"]
    
    UpdateCmd --> ParseArgs
    ParseArgs --> LoadApp
    LoadApp --> GetFlows
    GetFlows --> CheckReset
    
    CheckReset -->|yes| DropOp
    CheckReset -->|no| CheckSetup
    DropOp --> CheckSetup
    
    CheckSetup -->|yes| SetupOp
    CheckSetup -->|no| CheckLive
    SetupOp --> CheckLive
    
    CheckLive -->|no| OneTime
    CheckLive -->|yes| LiveMode
    
    OneTime --> Stats
    LiveMode --> Stats
```

Sources: [python/cocoindex/cli.py:391-485]()

### Server Command Flow

```mermaid
graph TB
    ServerCmd["cocoindex server APP_TARGET"]
    
    ParseArgs["Parse Arguments"]
    
    CheckReload{"--reload<br/>flag?"}
    
    LoadApp["_load_user_app(app_ref)"]
    
    CheckReset{"--reset<br/>flag?"}
    DropOp["_drop_flows()"]
    
    SetupOp["_setup_flows()"]
    
    ConfigCORS["Configure CORS:<br/>--cors-origin<br/>--cors-cocoindex<br/>--cors-local"]
    
    StartServer["lib.start_server(<br/>server_settings)"]
    
    CheckLiveUpdate{"--live-update<br/>flag?"}
    
    StartUpdater["FlowLiveUpdater<br/>in background thread"]
    
    WaitShutdown["Wait for Ctrl+C<br/>shutdown_event.wait()"]
    
    WatchFiles["watchfiles.run_process()<br/>Monitor Python files"]
    Reload["Restart process<br/>on file changes"]
    
    ServerCmd --> ParseArgs
    ParseArgs --> CheckReload
    
    CheckReload -->|yes| WatchFiles
    CheckReload -->|no| LoadApp
    
    WatchFiles --> LoadApp
    LoadApp --> CheckReset
    
    CheckReset -->|yes| DropOp
    CheckReset -->|no| SetupOp
    DropOp --> SetupOp
    
    SetupOp --> ConfigCORS
    ConfigCORS --> StartServer
    StartServer --> CheckLiveUpdate
    
    CheckLiveUpdate -->|yes| StartUpdater
    CheckLiveUpdate -->|no| WaitShutdown
    StartUpdater --> WaitShutdown
    
    WaitShutdown --> Reload
    Reload --> LoadApp
```

Sources: [python/cocoindex/cli.py:616-683](), [python/cocoindex/cli.py:685-783]()

## Common Workflows

### Development Workflow

```bash
# 1. Create flow definition
# Edit myapp.py with @flow_def decorator

# 2. Validate flow structure
cocoindex show myapp.py:MyFlow --verbose

# 3. Test without writing to backends
cocoindex evaluate myapp.py:MyFlow -o ./test_output

# 4. Setup backends
cocoindex setup myapp.py

# 5. Run initial update
cocoindex update myapp.py:MyFlow

# 6. Start development server with auto-reload
cocoindex server myapp.py --reload -ci
```

### Production Deployment

```bash
# 1. Load environment variables
cocoindex --env-file production.env setup myapp

# 2. Run one-time update
cocoindex --env-file production.env update myapp --force --quiet

# 3. Start production server with CORS
cocoindex --env-file production.env server myapp \
    --address 0.0.0.0:8000 \
    --cors-origin "https://example.com" \
    --live-update
```

### Continuous Monitoring

```bash
# Start live updater in background
cocoindex update myapp.py --live &

# Monitor status in separate terminal
# (Use CocoInsight or log output)

# Stop updater with Ctrl+C or kill
```

### Clean Slate Reset

```bash
# Drop existing setup and recreate
cocoindex update myapp.py --reset --force

# Or use separate commands
cocoindex drop myapp.py --force
cocoindex setup myapp.py --force
cocoindex update myapp.py
```

Sources: [python/cocoindex/cli.py:310-485]()

## Environment Variable Integration

The CLI automatically loads environment variables from `.env` files for configuration:

**Precedence:**
1. Explicit `--env-file` path
2. `.env` in current directory
3. Already-set environment variables

**Key environment variables:**
- `COCOINDEX_DATABASE_URL` - Internal storage connection
- `COCOINDEX_SERVER_ADDRESS` - Default server bind address
- `COCOINDEX_SERVER_CORS_ORIGINS` - Default CORS origins
- `COCOINDEX_APP_NAMESPACE` - Application namespace for multi-tenancy
- LLM API keys (e.g., `OPENAI_API_KEY`)

Sources: [python/cocoindex/cli.py:114-126]()

## Error Handling

The CLI provides detailed error messages for common issues:

| Error Condition | Message Format |
|----------------|----------------|
| Missing APP_TARGET | "Application module/path part is missing or invalid" |
| Invalid flow name | "Flow 'X' not found. Available: ..." |
| No flows in application | "No flows are defined in 'X'" |
| Failed to load app | "Failed to load APP_TARGET 'X'" |
| Setup conflicts | Detailed description of incompatible changes |

**Exit codes:**
- `0` - Success
- `1` - General error
- `2` - Usage error (invalid arguments)

Sources: [python/cocoindex/cli.py:29-88](), [python/cocoindex/cli.py:786-826]()

## Interactive Features

### Flow Selection

When a command requires a flow name but multiple flows exist and no flow is specified, the CLI presents an interactive selector:

```
Select a Flow
> DocumentFlow
  ImageFlow
  VideoFlow
```

**Navigation:**
- Arrow keys: Move selection
- Enter: Confirm selection

This behavior applies to `show` and `evaluate` commands.

Sources: [python/cocoindex/cli.py:786-826]()

### Confirmation Prompts

Commands that modify backends prompt for confirmation unless `--force` is specified:

```
Changes need to be pushed. Continue? [yes/N]
```

**Affected commands:**
- `setup`
- `drop`
- `update` (when setup is needed)
- `server` (when setup is needed)

Sources: [python/cocoindex/cli.py:271-293](), [python/cocoindex/cli.py:219-252]()

---

# Page: Query Handlers and Search APIs

# Query Handlers and Search

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [examples/amazon_s3_embedding/README.md](examples/amazon_s3_embedding/README.md)
- [examples/amazon_s3_embedding/main.py](examples/amazon_s3_embedding/main.py)
- [examples/azure_blob_embedding/README.md](examples/azure_blob_embedding/README.md)
- [examples/azure_blob_embedding/main.py](examples/azure_blob_embedding/main.py)
- [examples/code_embedding/main.py](examples/code_embedding/main.py)
- [examples/gdrive_text_embedding/main.py](examples/gdrive_text_embedding/main.py)
- [examples/patient_intake_extraction/main.py](examples/patient_intake_extraction/main.py)
- [examples/text_embedding/main.py](examples/text_embedding/main.py)
- [examples/text_embedding_qdrant/main.py](examples/text_embedding_qdrant/main.py)
- [python/cocoindex/__init__.py](python/cocoindex/__init__.py)
- [python/cocoindex/cli.py](python/cocoindex/cli.py)
- [python/cocoindex/flow.py](python/cocoindex/flow.py)
- [python/cocoindex/lib.py](python/cocoindex/lib.py)
- [python/cocoindex/setup.py](python/cocoindex/setup.py)

</details>



Query handlers provide a standardized way to define custom query functions that retrieve data from the targets (databases, vector stores, etc.) populated by a CocoIndex flow. They enable semantic search, aggregations, filtering, and other retrieval operations tailored to your specific use case.

For information about building and updating the target data itself, see [Operate a Flow](#8.1).

## Purpose and Scope

After a CocoIndex flow transforms source data and exports it to target storage backends (Postgres tables, Qdrant collections, Neo4j graphs, etc.), you need a way to query that data. Query handlers serve this purpose by:

- Providing a standard interface for defining custom query logic
- Associating query functions with specific flows
- Enabling queries via CLI commands or HTTP API endpoints
- Supporting vector similarity search, keyword search, aggregations, and custom SQL/database queries

This document covers:
- Defining query handlers with the `@query_handler` decorator
- Implementing query functions and accessing target data
- Invoking queries via the `cocoindex query` CLI command
- Common query patterns including similarity search
- Integration with the CocoIndex server HTTP API

## Query Handler Registration

### The @query_handler Decorator

Query handlers are Python functions decorated with `@flow.query_handler()`. The decorator registers the function as a query handler for a specific flow, making it available for invocation via CLI or API.

```python
@flow_instance.query_handler()
def my_query_function(param1: str, param2: int = 10) -> cocoindex.QueryOutput:
    # Query implementation
    return cocoindex.QueryOutput(results=[...])
```

**Key characteristics:**
- The decorator is called on a `Flow` instance (returned by `@cocoindex.flow_def`)
- Query handler functions can accept any number of parameters with type annotations
- Parameters can have default values
- The function must return a `cocoindex.QueryOutput` object

Sources: [examples/hn_trending_topics/main.py:313-352](), [examples/hn_trending_topics/main.py:381-416]()

### Query Handler Registry

```mermaid
graph TB
    subgraph "Flow Definition"
        FLOW["Flow Instance<br/>(from @flow_def)"]
        DECORATOR["@flow.query_handler()"]
    end
    
    subgraph "Query Handler Registration"
        REGISTRY["Query Handler Registry<br/>Per-flow mapping:<br/>handler_name -> function"]
        FUNC1["query_function_1"]
        FUNC2["query_function_2"]
    end
    
    subgraph "Invocation Paths"
        CLI["cocoindex query CLI<br/>cocoindex query main.py handler_name"]
        API["HTTP API<br/>/flows/{name}/query/{handler}"]
    end
    
    FLOW --> DECORATOR
    DECORATOR --> REGISTRY
    REGISTRY --> FUNC1
    REGISTRY --> FUNC2
    
    CLI --> REGISTRY
    API --> REGISTRY
    
    REGISTRY --> EXECUTE["Execute Handler<br/>with parameters"]
```

**Query Handler Registry Architecture**

Each flow maintains its own registry of query handlers. When a query is invoked:
1. The flow name is used to locate the flow instance
2. The handler name is used to look up the registered function
3. Parameters are passed to the function
4. The function returns a `QueryOutput` object

Sources: Architecture Diagram 7 (CLI and Server Architecture)

## Implementing Query Functions

### Function Signature

Query handler functions follow standard Python function conventions:

| Component | Description | Example |
|-----------|-------------|---------|
| **Function Name** | Becomes the query handler identifier | `search_by_topic`, `get_trending_topics` |
| **Parameters** | Can be positional or keyword arguments with type hints | `topic: str`, `limit: int = 20` |
| **Return Type** | Must return `cocoindex.QueryOutput` | `-> cocoindex.QueryOutput` |

Sources: [examples/hn_trending_topics/main.py:313-316](), [examples/hn_trending_topics/main.py:382]()

### Accessing Target Data

Query handlers typically need to access the target tables/collections that were exported by the flow. CocoIndex provides utilities to get the actual table/collection names:

```python
import cocoindex

@my_flow.query_handler()
def search_data(query: str) -> cocoindex.QueryOutput:
    # Get the actual table name for the exported target
    table_name = cocoindex.utils.get_target_default_name(my_flow, "my_target")
    
    # Execute database query
    with connection_pool().connection() as conn:
        with conn.cursor() as cur:
            cur.execute(f"SELECT * FROM {table_name} WHERE text LIKE %s", (f"%{query}%",))
            results = [{"text": row[0]} for row in cur.fetchall()]
    
    return cocoindex.QueryOutput(results=results)
```

**`cocoindex.utils.get_target_default_name(flow, export_name)`**
- `flow`: The flow instance
- `export_name`: The name passed to `collector.export(name, ...)`
- Returns: The actual table/collection name, accounting for app namespace prefixes

Sources: [examples/hn_trending_topics/main.py:316-321](), [examples/hn_trending_topics/main.py:384-386]()

### QueryOutput Format

Query functions must return a `cocoindex.QueryOutput` object:

```python
cocoindex.QueryOutput(results=list_of_results)
```

The `results` field should be a list of dictionaries, where each dictionary represents one result record. The structure is flexible and depends on your query logic.

**Example return values:**

```python
# Simple list of matches
QueryOutput(results=[
    {"id": "123", "text": "...", "score": 0.95},
    {"id": "456", "text": "...", "score": 0.87}
])

# Nested aggregations
QueryOutput(results=[
    {
        "topic": "AI",
        "count": 150,
        "threads": [
            {"url": "...", "score": 10},
            {"url": "...", "score": 8}
        ]
    }
])
```

Sources: [examples/hn_trending_topics/main.py:351](), [examples/hn_trending_topics/main.py:416]()

## Query Handler Patterns

### Pattern 1: Direct Database Queries

The most common pattern is to execute SQL queries directly against the target databases:

```mermaid
graph LR
    HANDLER["Query Handler Function"]
    UTIL["get_target_default_name()"]
    CONN["Database Connection Pool"]
    QUERY["SQL Query Execution"]
    RESULT["Format Results"]
    OUTPUT["QueryOutput"]
    
    HANDLER --> UTIL
    UTIL --> CONN
    CONN --> QUERY
    QUERY --> RESULT
    RESULT --> OUTPUT
```

**Direct Database Query Pattern**

Sources: [examples/hn_trending_topics/main.py:313-351]()

### Pattern 2: Vector Similarity Search

For vector indexes, query handlers typically:
1. Embed the query text using the same embedding model as the flow
2. Execute a similarity search against the vector index
3. Return ranked results

```python
@flow.query_handler()
def semantic_search(query: str, limit: int = 10) -> cocoindex.QueryOutput:
    # Get target table name
    table_name = cocoindex.utils.get_target_default_name(flow, "embeddings_target")
    
    # Embed the query (using the same model as in the flow)
    query_embedding = embed_text(query)  # Your embedding function
    
    # Execute similarity search
    with connection_pool().connection() as conn:
        with conn.cursor() as cur:
            cur.execute(f"""
                SELECT text, location, 
                       1 - (embedding <=> %s::vector) AS similarity
                FROM {table_name}
                ORDER BY embedding <=> %s::vector
                LIMIT %s
            """, (query_embedding, query_embedding, limit))
            
            results = [
                {
                    "text": row[0],
                    "location": row[1],
                    "similarity": row[2]
                }
                for row in cur.fetchall()
            ]
    
    return cocoindex.QueryOutput(results=results)
```

**Key considerations for similarity search:**
- Use the same embedding model and parameters as defined in your flow
- The distance operator depends on your similarity metric:
  - `<=>` for L2 distance (use `1 - (embedding <=> query)` for similarity score)
  - `<#>` for inner product
  - `<->` for cosine distance (pgvector)

Sources: Architectural knowledge from Diagram 6 (Storage Backend Integration Architecture)

### Pattern 3: Aggregations and Analytics

Query handlers can perform aggregations across indexed data:

```python
@flow.query_handler()
def get_trending_topics(limit: int = 20) -> cocoindex.QueryOutput:
    topic_table = cocoindex.utils.get_target_default_name(flow, "topics")
    
    with connection_pool().connection() as conn:
        with conn.cursor() as cur:
            cur.execute(f"""
                SELECT 
                    topic,
                    COUNT(*) AS mention_count,
                    MAX(created_at) AS latest_mention
                FROM {topic_table}
                GROUP BY topic
                ORDER BY mention_count DESC, latest_mention DESC
                LIMIT %s
            """, (limit,))
            
            results = [
                {
                    "topic": row[0],
                    "mentions": row[1],
                    "latest": row[2].isoformat()
                }
                for row in cur.fetchall()
            ]
    
    return cocoindex.QueryOutput(results=results)
```

Sources: [examples/hn_trending_topics/main.py:381-416]()

### Pattern 4: Multi-Table Joins

Query handlers can join multiple exported targets:

```python
@flow.query_handler()
def search_with_metadata(topic: str) -> cocoindex.QueryOutput:
    topics_table = cocoindex.utils.get_target_default_name(flow, "topics")
    messages_table = cocoindex.utils.get_target_default_name(flow, "messages")
    
    with connection_pool().connection() as conn:
        with conn.cursor() as cur:
            cur.execute(f"""
                SELECT m.id, m.text, m.author, t.topic
                FROM {topics_table} t
                JOIN {messages_table} m ON t.message_id = m.id
                WHERE LOWER(t.topic) LIKE LOWER(%s)
                ORDER BY m.created_at DESC
            """, (f"%{topic}%",))
            
            results = [
                {
                    "id": row[0],
                    "text": row[1],
                    "author": row[2],
                    "topic": row[3]
                }
                for row in cur.fetchall()
            ]
    
    return cocoindex.QueryOutput(results=results)
```

Sources: [examples/hn_trending_topics/main.py:313-351]()

## Invoking Query Handlers

### CLI Invocation

Query handlers are invoked using the `cocoindex query` command:

```bash
cocoindex query <module_path> <handler_name> [arguments]
```

**Command structure:**

| Component | Description | Example |
|-----------|-------------|---------|
| `module_path` | Path to Python file containing the flow | `main.py` or `main` |
| `handler_name` | Name of the query handler function | `search_by_topic` |
| `arguments` | Handler parameters as `--name value` | `--topic "AI" --limit 10` |

**Examples:**

```bash
# Simple query with no parameters
cocoindex query main.py get_all_topics

# Query with string parameter
cocoindex query main.py search_by_topic --topic "Claude"

# Query with multiple parameters
cocoindex query main.py semantic_search --query "machine learning" --limit 20

# Query with default parameter overrides
cocoindex query main.py get_trending_topics --limit 50
```

**Parameter type handling:**
- String parameters: `--param "value"`
- Integer parameters: `--param 42`
- Boolean parameters: `--param true` or `--param false`
- Float parameters: `--param 3.14`

Sources: [docs/docs/core/flow_methods.mdx]() (references to CLI query command), Architecture Diagram 7

### HTTP API Invocation

When running `cocoindex server`, query handlers are exposed as HTTP endpoints:

```bash
# Start the server
cocoindex server main.py
```

Query handlers become available at:
```
POST /flows/{flow_name}/query/{handler_name}
```

**Request format:**
```json
{
  "param1": "value1",
  "param2": 42
}
```

**Response format:**
```json
{
  "results": [
    {"field1": "...", "field2": "..."},
    {"field1": "...", "field2": "..."}
  ]
}
```

Sources: Architecture Diagram 7 (CLI and Server Architecture)

## Query Handler Lifecycle

```mermaid
sequenceDiagram
    participant User as User/Client
    participant CLI as cocoindex query CLI
    participant Registry as Query Handler Registry
    participant Handler as Query Handler Function
    participant DB as Target Database/Storage
    
    User->>CLI: cocoindex query main.py handler_name --arg1 value
    CLI->>CLI: Import module, load flow
    CLI->>Registry: Lookup handler by name
    Registry-->>CLI: Handler function reference
    CLI->>Handler: Invoke with parsed arguments
    Handler->>Handler: get_target_default_name()
    Handler->>DB: Execute query (SQL, vector search, etc.)
    DB-->>Handler: Raw results
    Handler->>Handler: Format results
    Handler-->>CLI: QueryOutput(results=[...])
    CLI-->>User: JSON output to stdout
```

**Query Handler Execution Flow**

1. **Module Loading**: The CLI imports the Python module containing the flow definition
2. **Handler Lookup**: The query handler is retrieved from the flow's registry
3. **Parameter Parsing**: Command-line arguments are parsed and converted to appropriate types
4. **Execution**: The handler function is invoked with the parsed parameters
5. **Data Access**: The handler accesses target storage (typically via connection pools)
6. **Result Formatting**: Raw query results are transformed into the `QueryOutput` format
7. **Output**: Results are serialized to JSON and returned to the caller

Sources: Architecture Diagram 7, [examples/hn_trending_topics/main.py:313-416]()

## Connection Management

Query handlers typically need to connect to the target databases. Common patterns:

### Connection Pool Pattern

```python
import functools
from psycopg_pool import ConnectionPool

@functools.cache
def connection_pool() -> ConnectionPool:
    """Cached connection pool for database queries."""
    return ConnectionPool(os.environ["COCOINDEX_DATABASE_URL"])

@flow.query_handler()
def my_query() -> cocoindex.QueryOutput:
    with connection_pool().connection() as conn:
        with conn.cursor() as cur:
            # Execute query
            cur.execute("SELECT ...")
            results = cur.fetchall()
    return cocoindex.QueryOutput(results=[...])
```

**Benefits:**
- Connection pooling reduces overhead of creating new connections
- `@functools.cache` ensures only one pool is created per process
- Context managers handle connection lifecycle automatically

Sources: [examples/hn_trending_topics/main.py:308-310](), [examples/hn_trending_topics/main.py:323-327]()

### Client Initialization for Other Backends

For non-SQL backends (Qdrant, Neo4j, etc.), initialize clients similarly:

```python
@functools.cache
def qdrant_client() -> QdrantClient:
    return QdrantClient(url=os.environ["QDRANT_URL"])

@flow.query_handler()
def vector_search(query: str) -> cocoindex.QueryOutput:
    client = qdrant_client()
    results = client.search(
        collection_name=cocoindex.utils.get_target_default_name(flow, "vectors"),
        query_vector=embed_query(query),
        limit=10
    )
    return cocoindex.QueryOutput(results=[...])
```

Sources: Architectural knowledge from Diagram 6 (Storage Backend Integration)

## Integration with Flow Updates

Query handlers can be invoked while a flow is actively updating:

```mermaid
graph TB
    subgraph "Update Process"
        UPDATER["FlowLiveUpdater<br/>Continuous Updates"]
        SOURCE["Source Data Changes"]
        TARGETS["Target Storage<br/>Tables/Collections"]
    end
    
    subgraph "Query Process"
        QUERY["Query Handler"]
        READ["Read from Targets"]
        RESULTS["Query Results"]
    end
    
    SOURCE --> UPDATER
    UPDATER --> TARGETS
    QUERY --> READ
    READ --> TARGETS
    READ --> RESULTS
    
    UPDATER -.->|"Updates visible<br/>to queries"| TARGETS
```

**Concurrent Query and Update Pattern**

- Query handlers read from the same targets that the flow updates
- Updates are transactional and visible immediately after commit
- No special coordination is needed between queries and updates
- For live update mode, query results reflect the most recent committed data

**Best practices:**
- Use database transactions appropriately for consistency
- Consider read-only query connections for better isolation
- For analytics queries, consider periodic materialized views if needed

Sources: [docs/docs/core/flow_methods.mdx:232-369]() (live update documentation)

## Summary

Query handlers provide the retrieval layer for CocoIndex flows:

| Aspect | Details |
|--------|---------|
| **Definition** | Python functions decorated with `@flow.query_handler()` |
| **Registration** | Stored in per-flow query handler registry |
| **Parameters** | Flexible function parameters with type annotations |
| **Return Type** | Must return `cocoindex.QueryOutput(results=[...])` |
| **Data Access** | Use `cocoindex.utils.get_target_default_name()` to get table names |
| **Invocation** | Via `cocoindex query` CLI or HTTP API endpoints |
| **Common Patterns** | Similarity search, aggregations, joins, filtering |
| **Concurrency** | Can query while flow is updating in live mode |

Sources: [examples/hn_trending_topics/main.py:313-416](), [docs/docs/core/flow_methods.mdx](), Architecture Diagram 7

---

# Page: CocoInsight: Visualization and Debugging

# CocoInsight: Visualization and Debugging

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [docs/docs/cocoinsight_access.mdx](docs/docs/cocoinsight_access.mdx)
- [docs/docusaurus.config.ts](docs/docusaurus.config.ts)
- [docs/package.json](docs/package.json)
- [docs/sidebars.ts](docs/sidebars.ts)
- [docs/yarn.lock](docs/yarn.lock)

</details>



CocoInsight is a web-based interface for visualizing, inspecting, and debugging CocoIndex flows. It provides an interactive UI for exploring flow schemas, data lineage, and execution history without storing any user data. This page documents the HTTP server that exposes flow metadata and data to CocoInsight, including server setup, CORS configuration, and the debugging capabilities provided by the interface.

For information about running flows and managing them from the command line, see [Command Line Interface](#11.1). For query-specific functionality, see [Query Handlers and Search APIs](#11.2).

## System Architecture

CocoInsight operates as a client-server architecture where the web UI hosted at `https://cocoindex.io/cocoinsight` communicates with a local HTTP server embedded in the CocoIndex library. The server exposes flow metadata and data through REST endpoints while maintaining strict CORS policies for security.

```mermaid
graph TB
    subgraph "Browser"
        UI["CocoInsight Web UI<br/>(https://cocoindex.io/cocoinsight)"]
    end
    
    subgraph "Local Development Machine"
        CLI["cocoindex server<br/>CLI Command"]
        PY_API["start_server()<br/>Python API"]
        SERVER["HTTP Server<br/>Bind: 127.0.0.1:49344"]
        
        subgraph "CocoIndex Engine"
            FLOW_CTX["FlowContext<br/>Flow Registry"]
            LIB_CTX["LibContext<br/>Global State"]
            PG_STATE[("PostgreSQL<br/>Internal State")]
        end
    end
    
    CLI -->|"starts"| SERVER
    PY_API -->|"starts"| SERVER
    
    UI -->|"HTTP GET/POST<br/>/cocoindex/api/*"| SERVER
    
    SERVER -->|"queries"| FLOW_CTX
    SERVER -->|"accesses"| LIB_CTX
    FLOW_CTX -->|"reads"| PG_STATE
    
    SERVER -->|"CORS validation<br/>cors_origins check"| UI
```

**CocoInsight Communication Architecture**

Sources: [docs/docs/cocoinsight_access.mdx:1-97]()

The server acts as a bridge between the CocoInsight UI and the internal CocoIndex engine, exposing flow metadata through REST endpoints while enforcing CORS policies to control access.

## Starting the Server

### CLI-Based Server Launch

The `cocoindex server` command starts the HTTP server and loads flows from a Python module. By default, it binds to `127.0.0.1:49344` for local-only access.

```bash
# Basic usage - local access only
cocoindex server path/to/app.py

# Expose to all interfaces with CocoInsight access
cocoindex server path/to/app.py -a 0.0.0.0:49344 -ci

# Custom CORS configuration
cocoindex server path/to/app.py \
  -c https://example.com \
  -cl 3000

# Enable live updates while server runs
cocoindex server path/to/app.py -L
```

Sources: [docs/docs/cocoinsight_access.mdx:24-51]()

### Python API Server Launch

The server can also be started programmatically using the `start_server` function with `ServerSettings` configuration.

```python
from cocoindex import start_server, ServerSettings

# Local-only configuration (default)
server_settings = ServerSettings(
    address="127.0.0.1:49344",
    cors_origins=["https://cocoindex.io"],
)
start_server(server_settings)

# Expose to all network interfaces
server_settings = ServerSettings(
    address="0.0.0.0:49344",
    cors_origins=["https://cocoindex.io"],
)
start_server(server_settings)
```

Sources: [docs/docs/cocoinsight_access.mdx:52-76]()

## Server Configuration Options

### CLI Arguments

| Flag | Long Form | Description | Default |
|------|-----------|-------------|---------|
| `-a` | `--address` | Server bind address (IP:PORT) | `127.0.0.1:49344` |
| `-ci` | `--cors-cocoindex` | Allow `https://cocoindex.io` origin | Not set |
| `-c` | `--cors-origin` | Comma-separated allowed origins | None |
| `-cl` | `--cors-local` | Allow `http://localhost:<PORT>` | None |
| `-L` | `--live-update` | Enable continuous flow updates | Disabled |

Sources: [docs/docs/cocoinsight_access.mdx:44-50]()

### ServerSettings Class

The `ServerSettings` dataclass configures the HTTP server when using the Python API:

| Field | Type | Purpose |
|-------|------|---------|
| `address` | `str` | Bind address in `IP:PORT` format |
| `cors_origins` | `List[str]` | List of allowed CORS origins |

Sources: [docs/docs/cocoinsight_access.mdx:56-67]()

## CORS Security Model

CocoInsight's CORS configuration follows a security-first model where the server defaults to local-only access (`127.0.0.1:49344`) and requires explicit configuration to allow external origins.

```mermaid
graph TD
    REQUEST["HTTP Request<br/>from Origin"]
    
    VALIDATE["CORS Validation<br/>ServerSettings.cors_origins"]
    
    COCOINSIGHT["https://cocoindex.io<br/>CocoInsight UI"]
    LOCALHOST["http://localhost:PORT<br/>Local Development"]
    CUSTOM["Custom Origin<br/>User-specified"]
    
    ALLOW["Allow Request<br/>Process API call"]
    REJECT["Reject Request<br/>CORS Error"]
    
    REQUEST --> VALIDATE
    
    COCOINSIGHT -->|"-ci flag or<br/>explicit config"| VALIDATE
    LOCALHOST -->|"-cl PORT flag"| VALIDATE
    CUSTOM -->|"-c origin flag"| VALIDATE
    
    VALIDATE -->|"Origin in<br/>cors_origins"| ALLOW
    VALIDATE -->|"Origin not<br/>allowed"| REJECT
```

**CORS Origin Validation Flow**

Sources: [docs/docs/cocoinsight_access.mdx:38-49]()

The CORS configuration prevents unauthorized access to flow data while allowing the hosted CocoInsight UI to communicate with local servers when explicitly enabled.

## HTTP Endpoints

The server exposes endpoints under the `/cocoindex/api` prefix for CocoInsight communication, plus utility endpoints for health checks.

### Internal API Routes

| Endpoint | Method | Purpose | Stability |
|----------|--------|---------|-----------|
| `/cocoindex/api/*` | Various | Flow metadata and data access | Unstable - CocoInsight internal |
| `/healthz` | GET | Server health check | Stable |

Sources: [docs/docs/cocoinsight_access.mdx:78-96]()

### Health Check Endpoint

The `/healthz` endpoint returns server status and version information:

```http
GET /healthz
Content-Type: application/json

{
  "status": "ok",
  "version": "0.3.5"
}
```

Sources: [docs/docs/cocoinsight_access.mdx:87-96]()

### Internal API Structure

```mermaid
graph LR
    subgraph "CocoInsight UI Requests"
        FLOW_LIST["GET /cocoindex/api/flows<br/>List all flows"]
        FLOW_SCHEMA["GET /cocoindex/api/flows/{name}<br/>Flow schema"]
        FLOW_DATA["GET /cocoindex/api/flows/{name}/data<br/>Flow data samples"]
    end
    
    subgraph "Server Handler"
        ROUTER["HTTP Router<br/>/cocoindex/api/*"]
    end
    
    subgraph "Engine Layer"
        FLOW_REG["Flow Registry<br/>FlowContext"]
        STATE_DB[("Internal State<br/>PostgreSQL")]
    end
    
    FLOW_LIST --> ROUTER
    FLOW_SCHEMA --> ROUTER
    FLOW_DATA --> ROUTER
    
    ROUTER -->|"query metadata"| FLOW_REG
    ROUTER -->|"query data"| STATE_DB
    
    FLOW_REG -->|"schema info"| ROUTER
    STATE_DB -->|"row samples"| ROUTER
```

**Internal API Request Flow**

Sources: [docs/docs/cocoinsight_access.mdx:78-82]()

The internal API endpoints are designed specifically for CocoInsight consumption and are subject to change. They are not intended for direct programmatic access by user applications.

## CocoInsight Web Interface

### Accessing CocoInsight

Once the server is running with CocoInsight CORS enabled, access the interface at:

```
https://cocoindex.io/cocoinsight
```

Point it to your server address (e.g., `http://localhost:49344`) to establish the connection.

Sources: [docs/docs/cocoinsight_access.mdx:6-13](), [docs/docs/cocoinsight_access.mdx:80-81]()

### Video Tutorials

CocoIndex provides YouTube tutorials demonstrating CocoInsight capabilities:

- **Introducing CocoInsight**: Overview of the web UI features
  - URL: https://www.youtube.com/watch?v=MMrpUfUcZPk
  
- **Fast Iterate Your Indexing Strategy with CocoInsight**: Practical iteration workflows
  - URL: https://www.youtube.com/watch?v=crV7odEVYTE

Sources: [docs/docs/cocoinsight_access.mdx:15-18]()

### Data Privacy Model

CocoInsight implements a zero-retention policy:

- No flow schemas are stored on CocoInsight servers
- No user data is retained
- All queries are proxied in real-time to the user's local server
- Data remains entirely within the user's infrastructure

Sources: [docs/docs/cocoinsight_access.mdx:10-12]()

## Live Updates

The server can perform continuous flow updates while running, combining the functionality of `cocoindex update -L` with HTTP server capabilities.

```bash
# Enable live updates via CLI
cocoindex server path/to/app.py -L

# With CocoInsight access and live updates
cocoindex server path/to/app.py -L -ci -a 0.0.0.0:49344
```

Sources: [docs/docs/cocoinsight_access.mdx:50](), [docs/docs/cocoinsight_access.mdx:84-85]()

Live update mode monitors data sources for changes and automatically triggers incremental flow updates, similar to the standalone live update functionality described in [Live Updates and Change Capture](#9.1).

## Typical Usage Workflow

```mermaid
sequenceDiagram
    participant DEV as Developer
    participant CLI as cocoindex CLI
    participant SRV as HTTP Server<br/>127.0.0.1:49344
    participant UI as CocoInsight UI<br/>Browser
    participant ENG as CocoIndex Engine
    
    DEV->>CLI: cocoindex server app.py -ci
    CLI->>SRV: Start server<br/>Load flows
    SRV->>ENG: Initialize FlowContext
    
    Note over SRV: Server listening on<br/>127.0.0.1:49344
    
    DEV->>UI: Open https://cocoindex.io/cocoinsight
    UI->>DEV: Enter server address
    DEV->>UI: http://localhost:49344
    
    UI->>SRV: GET /cocoindex/api/flows
    SRV->>ENG: Query flow registry
    ENG-->>SRV: Flow list
    SRV-->>UI: JSON response
    
    UI->>SRV: GET /cocoindex/api/flows/my_flow
    SRV->>ENG: Query flow schema
    ENG-->>SRV: Schema metadata
    SRV-->>UI: JSON response
    
    Note over UI: Render flow<br/>visualization
    
    UI->>SRV: GET /cocoindex/api/flows/my_flow/data
    SRV->>ENG: Query sample data
    ENG-->>SRV: Row samples
    SRV-->>UI: JSON response
    
    Note over UI: Display data<br/>in tables
```

**CocoInsight Connection and Query Sequence**

Sources: [docs/docs/cocoinsight_access.mdx:1-97]()

## Debugging Capabilities

CocoInsight provides visualization and debugging features for:

- **Flow Schema Inspection**: View the complete dataflow graph with operation types, data types, and transformations
- **Data Lineage Tracking**: Trace how data flows through operations from sources to targets
- **Sample Data Preview**: Inspect actual data at each stage of the pipeline
- **Operation Metadata**: Examine operation configurations, arguments, and execution options

The interface enables rapid iteration on indexing strategies by providing immediate visibility into how data transformations affect pipeline outputs without requiring code modifications or redeployment.

Sources: [docs/docs/cocoinsight_access.mdx:6-18]()

## Security Considerations

### Default Local-Only Binding

The server binds to `127.0.0.1:49344` by default, preventing network access from external machines. This ensures that:

- Flow metadata is not exposed to the network unintentionally
- Sample data remains on the local machine
- Explicit configuration is required for external access

Sources: [docs/docs/cocoinsight_access.mdx:28-29]()

### Network Exposure

When exposing the server to all interfaces (`0.0.0.0`), consider:

- CORS still restricts which web origins can access the API
- No authentication is implemented in the current version
- Deploy behind a reverse proxy with authentication for production use
- Limit exposure to trusted networks only

Sources: [docs/docs/cocoinsight_access.mdx:35-36](), [docs/docs/cocoinsight_access.mdx:69-75]()

### CORS Origin Validation

The `cors_origins` list in `ServerSettings` acts as a whitelist:

- Only specified origins can make cross-origin requests
- The `-ci` flag adds `https://cocoindex.io` to the whitelist
- Custom origins require explicit `-c` flag or API configuration
- Misconfigured CORS will prevent CocoInsight from connecting

Sources: [docs/docs/cocoinsight_access.mdx:38-49](), [docs/docs/cocoinsight_access.mdx:59-64]()

---

**Navigation References:**
- For CLI flow management commands: [Command Line Interface](#11.1)
- For query endpoint implementation: [Query Handlers and Search APIs](#11.2)
- For flow lifecycle management: [Flow Lifecycle and Methods](#3.3)
- For live update mechanisms: [Live Updates and Change Capture](#9.1)

---

# Page: Rust Engine Architecture

# Rust Engine Architecture

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [Cargo.lock](Cargo.lock)
- [Cargo.toml](Cargo.toml)
- [rust/cocoindex/Cargo.toml](rust/cocoindex/Cargo.toml)
- [rust/cocoindex/src/lib_context.rs](rust/cocoindex/src/lib_context.rs)
- [rust/cocoindex/src/prelude.rs](rust/cocoindex/src/prelude.rs)
- [rust/py_utils/Cargo.toml](rust/py_utils/Cargo.toml)
- [rust/py_utils/src/future.rs](rust/py_utils/src/future.rs)
- [rust/utils/Cargo.toml](rust/utils/Cargo.toml)
- [rust/utils/src/error.rs](rust/utils/src/error.rs)
- [rust/utils/src/prelude.rs](rust/utils/src/prelude.rs)
- [rust/utils/src/retryable.rs](rust/utils/src/retryable.rs)

</details>



## Purpose and Scope

This document describes the core Rust engine that powers CocoIndex's execution layer. It covers the global and per-flow state management systems (`LibContext` and `FlowContext`), the error handling architecture with context chaining, retry mechanisms with exponential backoff, async runtime integration, and resource management including database connection pooling and concurrency control.

For information about the FFI boundary and PyO3 bindings, see [PyO3 Integration and FFI Layer](#12.1). For subprocess execution used by GPU operations, see [Subprocess Execution Architecture](#12.4).

---

## System Overview

The Rust engine provides the high-performance execution core for CocoIndex. It is compiled as a Python extension module (`cdylib`) and exposed through PyO3 bindings. The engine manages:

- **Global state**: Singleton `LibContext` holds all flows, database pools, and global execution settings
- **Per-flow state**: Each `FlowContext` maintains execution state, setup changes, and query handlers for a single flow
- **Execution contexts**: `FlowExecutionContext` tracks setup state, changes, and source indexing contexts
- **Error propagation**: Unified error system with variants for client errors, internal errors, host language errors, and context chaining
- **Retry logic**: Exponential backoff with timeout for retryable operations
- **Async runtime**: Single Tokio runtime shared across all flows with Python async interop

**Engine Architecture**

```mermaid
graph TB
    subgraph "Static Singletons"
        TOKIO_RUNTIME["TOKIO_RUNTIME<br/>(LazyLock&lt;Runtime&gt;)"]
        AUTH_REGISTRY["AUTH_REGISTRY<br/>(LazyLock&lt;AuthRegistry&gt;)"]
        LIB_CONTEXT["LIB_CONTEXT<br/>(tokio::Mutex&lt;Option&lt;LibContext&gt;&gt;)"]
    end
    
    subgraph "LibContext (Global State)"
        DB_POOLS["db_pools: DbPools"]
        PERSISTENCE["persistence_ctx: Option&lt;PersistenceContext&gt;"]
        FLOWS["flows: Mutex&lt;BTreeMap&lt;String, FlowContext&gt;&gt;"]
        CONCUR["global_concurrency_controller"]
        MULTI_PROG["multi_progress_bar"]
    end
    
    subgraph "PersistenceContext"
        BUILTIN_POOL["builtin_db_pool: PgPool"]
        SETUP_CTX["setup_ctx: RwLock&lt;LibSetupContext&gt;"]
    end
    
    subgraph "FlowContext (Per-Flow)"
        FLOW["flow: Arc&lt;AnalyzedFlow&gt;"]
        EXEC_CTX["execution_ctx: RwLock&lt;FlowExecutionContext&gt;"]
        QUERY_HANDLERS["query_handlers: RwLock&lt;HashMap&gt;"]
    end
    
    subgraph "FlowExecutionContext"
        SETUP_EXEC["setup_execution_context"]
        SETUP_CHANGE["setup_change: FlowSetupChange"]
        SOURCE_IDX["source_indexing_contexts: Vec&lt;OnceCell&gt;"]
    end
    
    LIB_CONTEXT --> TOKIO_RUNTIME
    LIB_CONTEXT --> AUTH_REGISTRY
    LIB_CONTEXT -.contains.-> DB_POOLS
    LIB_CONTEXT -.contains.-> PERSISTENCE
    LIB_CONTEXT -.contains.-> FLOWS
    LIB_CONTEXT -.contains.-> CONCUR
    LIB_CONTEXT -.contains.-> MULTI_PROG
    
    PERSISTENCE --> BUILTIN_POOL
    PERSISTENCE --> SETUP_CTX
    
    FLOWS --> FLOW
    FLOWS --> EXEC_CTX
    FLOWS --> QUERY_HANDLERS
    
    EXEC_CTX --> SETUP_EXEC
    EXEC_CTX --> SETUP_CHANGE
    EXEC_CTX --> SOURCE_IDX
```

Sources: [rust/cocoindex/src/lib_context.rs:1-420](), [Cargo.toml:1-124](), [rust/cocoindex/Cargo.toml:1-103]()

---

## Context Management Architecture

### LibContext: Global State

`LibContext` is the singleton that holds all global state for the engine. It is lazily initialized on first use and remains alive for the lifetime of the Python process.

**Key Fields:**

| Field | Type | Purpose |
|-------|------|---------|
| `db_pools` | `DbPools` | Connection pool manager for all databases |
| `persistence_ctx` | `Option<PersistenceContext>` | PostgreSQL state storage (None if no database configured) |
| `flows` | `Mutex<BTreeMap<String, Arc<FlowContext>>>` | Registry of all active flows |
| `app_namespace` | `String` | Application namespace for multi-tenancy |
| `global_concurrency_controller` | `Arc<ConcurrencyController>` | Global throttling for source reads |
| `multi_progress_bar` | `LazyLock<MultiProgress>` | CLI progress display |

The static singleton is accessed through `get_lib_context()` which performs lazy initialization:

```rust
// From rust/cocoindex/src/lib_context.rs:350-374
static LIB_CONTEXT: LazyLock<tokio::sync::Mutex<Option<Arc<LibContext>>>> = ...;

pub(crate) async fn get_lib_context() -> Result<Arc<LibContext>> {
    let mut lib_context_locked = LIB_CONTEXT.lock().await;
    if let Some(lib_context) = &*lib_context_locked {
        lib_context.clone()
    } else {
        let setting = get_settings()?;
        let lib_context = Arc::new(create_lib_context(setting).await?);
        *lib_context_locked = Some(lib_context.clone());
        lib_context
    }
}
```

**LibContext Initialization Flow**

```mermaid
sequenceDiagram
    participant Python
    participant get_lib_context
    participant create_lib_context
    participant DbPools
    participant PgPool
    participant LibContext
    
    Python->>get_lib_context: First access
    get_lib_context->>get_lib_context: Lock LIB_CONTEXT
    get_lib_context->>get_settings: Retrieve Settings
    get_lib_context->>create_lib_context: Initialize
    create_lib_context->>DbPools: Create DbPools
    create_lib_context->>PgPool: Connect to PostgreSQL
    PgPool-->>create_lib_context: builtin_db_pool
    create_lib_context->>setup: get_existing_setup_state
    create_lib_context->>LibContext: Construct
    LibContext-->>get_lib_context: Arc<LibContext>
    get_lib_context->>get_lib_context: Store in static
    get_lib_context-->>Python: Arc<LibContext>
```

Sources: [rust/cocoindex/src/lib_context.rs:164-374](), [rust/cocoindex/src/lib_context.rs:287-329]()

### FlowContext: Per-Flow State

Each registered flow has a dedicated `FlowContext` that maintains its execution state, setup information, and query handlers. Flow contexts are stored in `LibContext.flows` keyed by flow name.

**Key Fields:**

| Field | Type | Purpose |
|-------|------|---------|
| `flow` | `Arc<AnalyzedFlow>` | Immutable flow definition and data schema |
| `execution_ctx` | `Arc<RwLock<FlowExecutionContext>>` | Mutable execution state (setup, sources) |
| `query_handlers` | `RwLock<HashMap<String, QueryHandlerContext>>` | Registered search functions |

**FlowContext Creation:**

```rust
// From rust/cocoindex/src/lib_context.rs:119-131
pub async fn new(
    flow: Arc<AnalyzedFlow>,
    existing_flow_ss: Option<&setup::FlowSetupState<setup::ExistingMode>>,
) -> Result<Self> {
    let execution_ctx = Arc::new(tokio::sync::RwLock::new(
        FlowExecutionContext::new(&flow, existing_flow_ss).await?,
    ));
    Ok(Self {
        flow,
        execution_ctx,
        query_handlers: RwLock::new(HashMap::new()),
    })
}
```

Sources: [rust/cocoindex/src/lib_context.rs:108-162]()

### FlowExecutionContext: Setup and Indexing State

`FlowExecutionContext` holds the mutable execution state for a flow, including setup state comparisons and lazy-initialized source indexing contexts.

**Key Fields:**

| Field | Type | Purpose |
|-------|------|---------|
| `setup_execution_context` | `Arc<FlowSetupExecutionContext>` | Current setup configuration |
| `setup_change` | `FlowSetupChange` | Diff between desired and existing setup |
| `source_indexing_contexts` | `Vec<OnceCell<SourceIndexingContext>>` | Lazy-loaded source change trackers |

**Setup Change Validation:**

Before executing a flow, the engine checks that setup is up-to-date:

```rust
// From rust/cocoindex/src/lib_context.rs:133-144
pub async fn use_execution_ctx(&self) -> Result<RwLockReadGuard<'_, FlowExecutionContext>> {
    let execution_ctx = self.execution_ctx.read().await;
    if !execution_ctx.setup_change.is_up_to_date() {
        api_bail!(
            "Setup for flow `{}` is not up-to-date. Please run `cocoindex setup` to update the setup.",
            self.flow_name()
        );
    }
    Ok(execution_ctx)
}
```

Sources: [rust/cocoindex/src/lib_context.rs:18-101](), [rust/cocoindex/src/lib_context.rs:24-46]()

---

## Error System

CocoIndex uses a unified error type `utils::error::Error` with four variants to distinguish error sources and enable appropriate handling across the FFI boundary.

### Error Variants

**Error Type Hierarchy**

```mermaid
graph TB
    ERROR["Error (enum)"]
    
    CONTEXT["Context<br/>{msg: String, source: Box&lt;SError&gt;}"]
    HOST["HostLang<br/>(Box&lt;dyn HostError&gt;)"]
    CLIENT["Client<br/>{msg: String, bt: Backtrace}"]
    INTERNAL["Internal<br/>(anyhow::Error)"]
    
    ERROR --> CONTEXT
    ERROR --> HOST
    ERROR --> CLIENT
    ERROR --> INTERNAL
    
    CONTEXT -.recursive.-> ERROR
    HOST -.wraps.-> PY_ERROR["Python Exception"]
    CLIENT -.captures.-> BACKTRACE["Backtrace"]
    INTERNAL -.wraps.-> ANYHOW["Any std::error::Error"]
```

| Variant | Purpose | Creation | HTTP Status |
|---------|---------|----------|-------------|
| `Context` | Add context to existing error | `err.context("msg")` | Inherits from source |
| `HostLang` | Propagate Python exceptions | `Error::host(py_err)` | 500 Internal Server Error |
| `Client` | Invalid request/user input | `Error::client("msg")` | 400 Bad Request |
| `Internal` | Internal engine failures | `Error::internal(err)` | 500 Internal Server Error |

**Error Creation Examples:**

```rust
// From rust/utils/src/error.rs:54-73
impl Error {
    pub fn host(e: impl HostError) -> Self {
        Self::HostLang(Box::new(e))
    }

    pub fn client(msg: impl Into<String>) -> Self {
        Self::Client {
            msg: msg.into(),
            bt: Backtrace::capture(),
        }
    }

    pub fn internal(e: impl Into<anyhow::Error>) -> Self {
        Self::Internal(e.into())
    }
}
```

Sources: [rust/utils/src/error.rs:18-136](), [rust/utils/src/error.rs:54-73]()

### Context Chaining

Errors can be wrapped with context strings to build a breadcrumb trail of execution path. The `ContextExt` trait provides `.context()` and `.with_context()` methods:

```rust
// From rust/utils/src/error.rs:99-111
impl Error {
    pub fn context<C: Into<String>>(self, context: C) -> Self {
        Self::Context {
            msg: context.into(),
            source: Box::new(SError(self)),
        }
    }

    pub fn with_context<C: Into<String>, F: FnOnce() -> C>(self, f: F) -> Self {
        Self::Context {
            msg: f().into(),
            source: Box::new(SError(self)),
        }
    }
}
```

**Context Chain Display:**

When formatted, the error displays all context layers before the root cause:

```
Context:
  1: Failed to update flow
  2: Error executing source operation
  3: S3 bucket access denied
Invalid Request: bucket not found
```

Sources: [rust/utils/src/error.rs:99-161](), [rust/utils/src/error.rs:117-129]()

### SharedError: Thread-Safe Error Propagation

`SharedError` wraps `Error` in an `Arc<Mutex<>>` for sharing across async tasks. When extracted, it replaces itself with a `ResidualError` to prevent double-extraction while maintaining a readable error message.

**SharedError State Machine**

```mermaid
stateDiagram-v2
    [*] --> Error: new(err)
    Error --> ResidualErrorMessage: extract_error()
    ResidualErrorMessage --> ResidualErrorMessage: extract_error() again
    
    note right of Error
        Contains original Error
        with full backtrace
    end note
    
    note right of ResidualErrorMessage
        Residual error with
        message string only
    end note
```

```rust
// From rust/utils/src/error.rs:278-312
enum SharedErrorState {
    Error(Error),
    ResidualErrorMessage(ResidualError),
}

impl SharedError {
    fn extract_error(&self) -> Error {
        let mut state = self.0.lock().unwrap();
        let residual_err = match &mut *state {
            SharedErrorState::ResidualErrorMessage(err) => {
                return Error::internal(err.clone());
            }
            SharedErrorState::Error(err) => ResidualError::new(err),
        };
        
        let orig_state = std::mem::replace(state, SharedErrorState::ResidualErrorMessage(residual_err));
        match orig_state {
            SharedErrorState::Error(err) => err,
            _ => panic!("Expected shared error state to hold Error"),
        }
    }
}
```

Sources: [rust/utils/src/error.rs:247-357](), [rust/utils/src/error.rs:291-312]()

### Convenience Macros

The error system provides macros for quick error construction and early returns:

| Macro | Purpose | Returns |
|-------|---------|---------|
| `client_bail!(fmt, ...)` | Return client error immediately | `Err(Error::client(...))` |
| `client_error!(fmt, ...)` | Create client error | `Error::client(...)` |
| `internal_bail!(fmt, ...)` | Return internal error immediately | `Err(Error::internal_msg(...))` |
| `internal_error!(fmt, ...)` | Create internal error | `Error::internal_msg(...)` |
| `api_bail!(fmt, ...)` | Return API error (HTTP 400) | `Err(ApiError::new(...))` |
| `api_error!(fmt, ...)` | Create API error | `ApiError::new(...)` |

Sources: [rust/utils/src/error.rs:190-216](), [rust/utils/src/error.rs:443-455]()

---

## Retry System

The retry system in `utils::retryable` provides exponential backoff with jitter for transient failures. Operations can be marked as retryable through the `IsRetryable` trait.

### Retryable Trait

```rust
// From rust/utils/src/retryable.rs:7-9
pub trait IsRetryable {
    fn is_retryable(&self) -> bool;
}
```

**Built-in Retryable Implementations:**

| Error Type | Retryable Condition |
|------------|---------------------|
| `reqwest::Error` | HTTP 429 (Too Many Requests) |
| `async_openai::error::OpenAIError` | Underlying reqwest error is retryable |
| `neo4rs::Error` | Connection errors or transient Neo4j errors |
| `retryable::Error` | Explicit `is_retryable` flag |

Sources: [rust/utils/src/retryable.rs:7-104](), [rust/utils/src/retryable.rs:36-64]()

### Retry Options

```rust
// From rust/utils/src/retryable.rs:113-127
pub struct RetryOptions {
    pub retry_timeout: Option<Duration>,
    pub initial_backoff: Duration,
    pub max_backoff: Duration,
}

impl Default for RetryOptions {
    fn default() -> Self {
        Self {
            retry_timeout: Some(DEFAULT_RETRY_TIMEOUT), // 10 minutes
            initial_backoff: Duration::from_millis(100),
            max_backoff: Duration::from_secs(10),
        }
    }
}
```

**Retry Loop Algorithm**

```mermaid
graph TD
    START["Start Retry Loop"]
    EXEC["Execute Operation"]
    CHECK_OK{"Success?"}
    RETURN_OK["Return Result"]
    CHECK_RETRY{"Is Retryable?"}
    RETURN_ERR["Return Error"]
    CHECK_DEADLINE{"Deadline<br/>Exceeded?"}
    SLEEP["Sleep with Backoff"]
    CALC_BACKOFF["Calculate Next Backoff<br/>backoff = backoff * random(1.618-2.0)"]
    
    START --> EXEC
    EXEC --> CHECK_OK
    CHECK_OK -->|Yes| RETURN_OK
    CHECK_OK -->|No| CHECK_RETRY
    CHECK_RETRY -->|No| RETURN_ERR
    CHECK_RETRY -->|Yes| CHECK_DEADLINE
    CHECK_DEADLINE -->|Yes| RETURN_ERR
    CHECK_DEADLINE -->|No| SLEEP
    SLEEP --> CALC_BACKOFF
    CALC_BACKOFF --> EXEC
```

**Retry Execution:**

```rust
// From rust/utils/src/retryable.rs:135-182
pub async fn run<Ok, Err: IsRetryable, Fut: Future<Output = Result<Ok, Err>>, F: Fn() -> Fut>(
    f: F,
    options: &RetryOptions,
) -> Result<Ok, Err> {
    let deadline = options.retry_timeout.map(|timeout| Instant::now() + timeout);
    let mut backoff = options.initial_backoff;
    
    loop {
        match f().await {
            Result::Ok(result) => return Result::Ok(result),
            Result::Err(err) => {
                if !err.is_retryable() {
                    return Result::Err(err);
                }
                // Calculate sleep duration with jitter
                let sleep_duration = std::cmp::min(backoff, remaining_time);
                tokio::time::sleep(sleep_duration).await;
                
                // Exponential backoff with golden ratio jitter
                backoff = std::cmp::min(
                    Duration::from_micros((backoff.as_micros() * rand::random_range(1618..=2000) / 1000) as u64),
                    options.max_backoff,
                );
            }
        }
    }
}
```

Sources: [rust/utils/src/retryable.rs:113-182](), [rust/utils/src/retryable.rs:135-182]()

---

## Async Runtime Integration

CocoIndex uses a single Tokio runtime for all async operations, initialized lazily as a static singleton.

### Runtime Singleton

```rust
// From rust/cocoindex/src/lib_context.rs:164-169
static TOKIO_RUNTIME: LazyLock<Runtime> = LazyLock::new(|| Runtime::new().unwrap());

pub fn get_runtime() -> &'static Runtime {
    &TOKIO_RUNTIME
}
```

The runtime is configured with the `rt-multi-thread` feature and full Tokio capabilities including:
- Multi-threaded work-stealing executor
- File system operations
- Timers and intervals
- Signal handling
- Process spawning
- Tracing integration

Sources: [rust/cocoindex/src/lib_context.rs:164-169](), [Cargo.toml:44-51]()

### Python-Rust Async Bridge

The `CancelOnDropPy` future in `py_utils::future` bridges Python's `asyncio` with Tokio. It wraps a Python coroutine and ensures proper cancellation when the Rust future is dropped.

**CancelOnDropPy Structure:**

```rust
// From rust/py_utils/src/future.rs:14-33
struct CancelOnDropPy {
    inner: BoxFuture<'static, PyResult<Py<PyAny>>>,
    task: Py<PyAny>,
    event_loop: Py<PyAny>,
    ctx: Py<PyAny>,
    done: AtomicBool,
}

impl Drop for CancelOnDropPy {
    fn drop(&mut self) {
        if self.done.load(Ordering::SeqCst) {
            return;
        }
        Python::attach(|py| {
            self.event_loop.bind(py).call_method(
                "call_soon_threadsafe",
                (self.task.bind(py).getattr("cancel")?,),
                Some(&kwargs),
            )?;
        });
    }
}
```

**Async Bridge Mechanism**

```mermaid
sequenceDiagram
    participant Rust
    participant from_py_future
    participant asyncio
    participant CancelOnDropPy
    participant Tokio
    
    Rust->>from_py_future: Call with Python awaitable
    from_py_future->>asyncio: Get event_loop and context
    from_py_future->>asyncio: create_task(awaitable, context=ctx)
    asyncio-->>from_py_future: Python Task
    from_py_future->>CancelOnDropPy: Wrap Task + event loop
    CancelOnDropPy->>Tokio: Bridge to Rust Future
    Tokio->>Rust: Poll Future
    
    alt Future Completes
        Rust->>CancelOnDropPy: Poll Ready
        CancelOnDropPy->>CancelOnDropPy: Set done=true
    else Future Dropped
        Rust->>CancelOnDropPy: Drop
        CancelOnDropPy->>asyncio: call_soon_threadsafe(task.cancel)
        asyncio->>asyncio: Cancel Python Task
    end
```

**Creating Python-Rust Future:**

```rust
// From rust/py_utils/src/future.rs:60-86
pub fn from_py_future<'py, 'fut>(
    py: Python<'py>,
    locals: &TaskLocals,
    awaitable: Bound<'py, PyAny>,
) -> pyo3::PyResult<impl Future<Output = pyo3::PyResult<Py<PyAny>>> + Send> {
    // 1) Capture loop + context from TaskLocals for thread-safe cancellation
    let event_loop: Bound<'py, PyAny> = locals.event_loop(py).into();
    let ctx: Bound<'py, PyAny> = locals.context(py);

    // 2) Create a Task so we own a handle we can cancel later
    let task = event_loop.call_method("create_task", (awaitable,), Some(&kwarg))?;

    // 3) Bridge it to a Rust Future as usual
    let fut = pyo3_async_runtimes::into_future_with_locals(locals, task.clone())?.boxed();

    Ok(CancelOnDropPy {
        inner: fut,
        task: task.unbind(),
        event_loop: event_loop.unbind(),
        ctx: ctx.unbind(),
        done: AtomicBool::new(false),
    })
}
```

Sources: [rust/py_utils/src/future.rs:1-87](), [rust/py_utils/src/future.rs:14-58](), [rust/py_utils/src/future.rs:60-86]()

---

## Database Connection Pooling

The `DbPools` struct manages connection pools to multiple PostgreSQL databases, keyed by `(url, user)` tuples.

### DbPools Implementation

```rust
// From rust/cocoindex/src/lib_context.rs:174-180
type PoolKey = (String, Option<String>);
type PoolValue = Arc<tokio::sync::OnceCell<PgPool>>;

#[derive(Default)]
pub struct DbPools {
    pub pools: Mutex<HashMap<PoolKey, PoolValue>>,
}
```

**Connection Pool Strategy:**

| Setting | Value | Purpose |
|---------|-------|---------|
| Initial max connections | 1 | Fast connection test with 30s timeout |
| Production max connections | Configurable (default 10) | Long-lived pool for queries |
| Acquire timeout | 5 minutes | Prevent indefinite blocking |
| Slow acquisition threshold | 10 seconds | Log slow acquisitions at INFO level |

**Pool Initialization:**

```rust
// From rust/cocoindex/src/lib_context.rs:183-229
impl DbPools {
    pub async fn get_pool(&self, conn_spec: &settings::DatabaseConnectionSpec) -> Result<PgPool> {
        let db_pool_cell = {
            let key = (conn_spec.url.clone(), conn_spec.user.clone());
            let mut db_pools = self.pools.lock().unwrap();
            db_pools.entry(key).or_default().clone()
        };
        
        db_pool_cell.get_or_try_init(|| async move {
            // Build connection options
            let mut pg_options: PgConnectOptions = conn_spec.url.parse()?;
            if let Some(user) = &conn_spec.user {
                pg_options = pg_options.username(user);
            }
            if let Some(password) = &conn_spec.password {
                pg_options = pg_options.password(password);
            }

            // Fast connection test
            let test_pool = PgPoolOptions::new()
                .max_connections(1)
                .acquire_timeout(Duration::from_secs(30))
                .connect_with(pg_options.clone())
                .await?;
            let _ = test_pool.acquire().await?;

            // Create production pool
            let pool = PgPoolOptions::new()
                .max_connections(conn_spec.max_connections)
                .min_connections(conn_spec.min_connections)
                .acquire_slow_level(log::LevelFilter::Info)
                .acquire_slow_threshold(Duration::from_secs(10))
                .acquire_timeout(Duration::from_secs(5 * 60))
                .connect_with(pg_options)
                .await?;
            Ok(pool)
        }).await?;
        
        Ok(pool.clone())
    }
}
```

Sources: [rust/cocoindex/src/lib_context.rs:174-230](), [rust/cocoindex/src/lib_context.rs:183-229]()

---

## Concurrency Control

Global concurrency control prevents overwhelming data sources with too many concurrent requests. The `ConcurrencyController` tracks inflight rows and bytes.

**Concurrency Controller Options:**

```rust
// From cocoindex_utils/src/concur_control.rs
pub struct Options {
    pub max_inflight_rows: Option<usize>,
    pub max_inflight_bytes: Option<usize>,
}
```

The controller is initialized in `LibContext`:

```rust
// From rust/cocoindex/src/lib_context.rs:321-326
global_concurrency_controller: Arc::new(concur_control::ConcurrencyController::new(
    &concur_control::Options {
        max_inflight_rows: settings.global_execution_options.source_max_inflight_rows,
        max_inflight_bytes: settings.global_execution_options.source_max_inflight_bytes,
    },
))
```

**Resource Management Flow**

```mermaid
graph LR
    SOURCE["Source Reader"]
    ACQUIRE["Acquire Permit"]
    CONTROLLER["ConcurrencyController"]
    PROCESS["Process Batch"]
    RELEASE["Release Permit"]
    
    SOURCE -->|Request permit| ACQUIRE
    ACQUIRE -->|Check limits| CONTROLLER
    CONTROLLER -->|rows < max_rows<br/>bytes < max_bytes| PROCESS
    PROCESS -->|Complete| RELEASE
    RELEASE -->|Decrement counters| CONTROLLER
    CONTROLLER -.throttle.-> ACQUIRE
```

Sources: [rust/cocoindex/src/lib_context.rs:321-326](), [rust/cocoindex/Cargo.toml:17-25]()

---

## Key Code Entities Reference

This section maps high-level concepts to specific Rust types and functions for code navigation:

### State Management

| Concept | Code Entity | Location |
|---------|-------------|----------|
| Global singleton | `static LIB_CONTEXT: LazyLock<Mutex<Option<Arc<LibContext>>>>` | [rust/cocoindex/src/lib_context.rs:350]() |
| Global state | `struct LibContext` | [rust/cocoindex/src/lib_context.rs:241-249]() |
| Per-flow state | `struct FlowContext` | [rust/cocoindex/src/lib_context.rs:108-112]() |
| Execution state | `struct FlowExecutionContext` | [rust/cocoindex/src/lib_context.rs:18-22]() |
| Persistence layer | `struct PersistenceContext` | [rust/cocoindex/src/lib_context.rs:236-239]() |

### Error Handling

| Concept | Code Entity | Location |
|---------|-------------|----------|
| Error type | `enum Error` | [rust/utils/src/error.rs:18-23]() |
| Context chaining | `Error::context()` | [rust/utils/src/error.rs:99-104]() |
| Thread-safe errors | `struct SharedError` | [rust/utils/src/error.rs:284]() |
| API errors | `struct ApiError` | [rust/utils/src/error.rs:377-380]() |

### Retry Logic

| Concept | Code Entity | Location |
|---------|-------------|----------|
| Retry trait | `trait IsRetryable` | [rust/utils/src/retryable.rs:7-9]() |
| Retry configuration | `struct RetryOptions` | [rust/utils/src/retryable.rs:113-117]() |
| Retry executor | `async fn run()` | [rust/utils/src/retryable.rs:135-182]() |

### Async Runtime

| Concept | Code Entity | Location |
|---------|-------------|----------|
| Runtime singleton | `static TOKIO_RUNTIME: LazyLock<Runtime>` | [rust/cocoindex/src/lib_context.rs:164]() |
| Python-Rust bridge | `struct CancelOnDropPy` | [rust/py_utils/src/future.rs:14-20]() |
| Future conversion | `fn from_py_future()` | [rust/py_utils/src/future.rs:60-86]() |

### Database Management

| Concept | Code Entity | Location |
|---------|-------------|----------|
| Pool manager | `struct DbPools` | [rust/cocoindex/src/lib_context.rs:177-180]() |
| Pool acquisition | `async fn get_pool()` | [rust/cocoindex/src/lib_context.rs:183-229]() |
| Builtin database | `PersistenceContext.builtin_db_pool` | [rust/cocoindex/src/lib_context.rs:237]() |

Sources: All references above

---

# Page: PyO3 Integration and FFI Layer

# PyO3 Integration and FFI Layer

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [.github/workflows/CI.yml](.github/workflows/CI.yml)
- [.github/workflows/_docs_release.yml](.github/workflows/_docs_release.yml)
- [.github/workflows/_test.yml](.github/workflows/_test.yml)
- [.github/workflows/docs_release.yml](.github/workflows/docs_release.yml)
- [.github/workflows/docs_test.yml](.github/workflows/docs_test.yml)
- [.github/workflows/format.yml](.github/workflows/format.yml)
- [.github/workflows/release.yml](.github/workflows/release.yml)
- [Cargo.lock](Cargo.lock)
- [Cargo.toml](Cargo.toml)
- [pyproject.toml](pyproject.toml)
- [python/cocoindex/subprocess_exec.py](python/cocoindex/subprocess_exec.py)
- [rust/cocoindex/Cargo.toml](rust/cocoindex/Cargo.toml)
- [rust/cocoindex/src/lib_context.rs](rust/cocoindex/src/lib_context.rs)
- [rust/cocoindex/src/prelude.rs](rust/cocoindex/src/prelude.rs)
- [rust/py_utils/Cargo.toml](rust/py_utils/Cargo.toml)
- [rust/py_utils/src/future.rs](rust/py_utils/src/future.rs)
- [rust/utils/Cargo.toml](rust/utils/Cargo.toml)
- [rust/utils/src/error.rs](rust/utils/src/error.rs)
- [rust/utils/src/prelude.rs](rust/utils/src/prelude.rs)
- [rust/utils/src/retryable.rs](rust/utils/src/retryable.rs)
- [uv.lock](uv.lock)

</details>



## Purpose and Scope

This document explains the Python-Rust Foreign Function Interface (FFI) layer in CocoIndex, which enables seamless integration between the Python user-facing API and the high-performance Rust execution engine. The FFI layer is built using PyO3, a Rust library for creating Python bindings. This page covers the technical architecture of the bindings, data serialization across language boundaries, async runtime integration, and the build system that produces cross-platform Python wheels.

For information about the specific data type marshalling and encoding/decoding system, see [Operation System and Type Marshalling](#4). For the build system and maturin configuration details, see [Build System: Maturin and Cross-Platform Wheels](#12.2).

---

## PyO3 Configuration and Features

CocoIndex uses PyO3 version 0.25.1 with specific features enabled to support the required functionality across the Python-Rust boundary.

### Workspace Dependencies

The core PyO3 configuration is defined at the workspace level:

| Dependency | Version | Purpose |
|------------|---------|---------|
| `pyo3` | 0.25.1 | Core Python bindings library |
| `pythonize` | 0.25.0 | Serde-based Python object conversion |
| `pyo3-async-runtimes` | 0.25.0 | Async Python-Rust bridge with Tokio |

### PyO3 Features

The following PyO3 features are enabled:

| Feature | Purpose |
|---------|---------|
| `abi3-py311` | Stable Python ABI for Python 3.11+ (enables cross-version compatibility) |
| `auto-initialize` | Automatic Python interpreter initialization |
| `chrono` | Support for converting between `chrono` datetime types and Python datetime |
| `uuid` | Support for UUID type conversion between Rust and Python |

These features enable CocoIndex to ship a single wheel per platform that works across multiple Python versions (3.11, 3.12, 3.13, 3.14) without recompilation.

**Sources:** [Cargo.toml:12-19]()

---

## cdylib Architecture and Module Structure

CocoIndex's Rust engine is compiled as a C dynamic library (`cdylib`) that Python can load as an extension module.

### Library Configuration

```mermaid
graph LR
    subgraph "Rust Workspace"
        CocoindexCrate["cocoindex crate"]
        PyUtilsCrate["cocoindex_py_utils crate"]
        UtilsCrate["cocoindex_utils crate"]
    end
    
    subgraph "Build Output"
        Cdylib["cocoindex_engine.so/dll/dylib<br/>(cdylib)"]
    end
    
    subgraph "Python Package"
        EngineModule["cocoindex._engine<br/>(Python module)"]
        PythonAPI["cocoindex Python API"]
    end
    
    CocoindexCrate -->|"crate-type=['cdylib']"| Cdylib
    PyUtilsCrate -->|"dependency"| CocoindexCrate
    UtilsCrate -->|"dependency"| CocoindexCrate
    
    Cdylib -->|"loaded as"| EngineModule
    EngineModule -->|"imported by"| PythonAPI
```

### Module Naming

The Rust library is configured with two names:
- **Rust crate name**: `cocoindex` (package name in Cargo.toml)
- **Rust library name**: `cocoindex_engine` (library name, used for linking)
- **Python module name**: `cocoindex._engine` (configured in pyproject.toml via maturin)

This structure allows the Python package to import the Rust engine as `from cocoindex import _engine`, keeping it as a private implementation detail while the public API is in pure Python.

**Sources:** [rust/cocoindex/Cargo.toml:1-10](), [pyproject.toml:60]()

---

## Data Serialization with pythonize

The `pythonize` library provides bidirectional conversion between Rust's serde types and Python objects, enabling type-safe data exchange across the FFI boundary.

### pythonize Integration

```mermaid
graph TB
    subgraph "Python Space"
        PyDict["Python dict"]
        PyList["Python list"]
        PyTypes["Python types<br/>(str, int, float, etc)"]
    end
    
    subgraph "FFI Boundary"
        Pythonize["pythonize crate"]
        Depythonize["depythonize"]
    end
    
    subgraph "Rust Space"
        SerdeStruct["Rust structs<br/>(with Serialize/Deserialize)"]
        SerdeEnum["Rust enums"]
        SerdeTypes["Rust types"]
    end
    
    PyDict -->|"depythonize"| Pythonize
    PyList -->|"depythonize"| Pythonize
    PyTypes -->|"depythonize"| Pythonize
    
    Pythonize --> SerdeStruct
    Pythonize --> SerdeEnum
    Pythonize --> SerdeTypes
    
    SerdeStruct -->|"pythonize"| Depythonize
    SerdeEnum -->|"pythonize"| Depythonize
    SerdeTypes -->|"pythonize"| Depythonize
    
    Depythonize --> PyDict
    Depythonize --> PyList
    Depythonize --> PyTypes
```

### Conversion Pattern

The pythonize library enables a clean pattern for passing complex data structures:

1. **Python → Rust**: Python dictionaries are automatically converted to Rust structs that implement `serde::Deserialize`
2. **Rust → Python**: Rust structs implementing `serde::Serialize` are converted to Python dictionaries
3. **Type Safety**: Deserialization errors are caught at runtime with detailed error messages

This approach eliminates the need for manual field-by-field extraction from Python objects and provides compile-time guarantees on the Rust side while maintaining flexibility on the Python side.

**Sources:** [Cargo.toml:18](), [rust/cocoindex/Cargo.toml:28]()

---

## Async Runtime Bridge

CocoIndex uses `pyo3-async-runtimes` to bridge Python's asyncio event loop with Rust's Tokio runtime, enabling async operations to span both languages. The bridge includes automatic cancellation propagation when Rust futures are dropped.

### Dual Runtime Architecture

```mermaid
graph TB
    subgraph "Python Runtime"
        AsyncIO["asyncio.AbstractEventLoop"]
        PyTask["asyncio.Task"]
        Context["contextvars.Context"]
    end
    
    subgraph "FFI Bridge Layer"
        FromPyFuture["from_py_future()"]
        CancelOnDropPy["CancelOnDropPy"]
        IntoFuture["into_future_with_locals()"]
    end
    
    subgraph "Rust Runtime"
        TOKIO_RUNTIME["TOKIO_RUNTIME (LazyLock)"]
        RustFuture["BoxFuture<PyResult<Py<PyAny>>>"]
    end
    
    PyTask -->|"wrapped by"| FromPyFuture
    FromPyFuture -->|"creates"| CancelOnDropPy
    CancelOnDropPy -->|"bridges to"| RustFuture
    AsyncIO -->|"managed by"| CancelOnDropPy
    Context -->|"preserved in"| CancelOnDropPy
    RustFuture -->|"executes on"| TOKIO_RUNTIME
```

**Diagram: Async bridge architecture showing the CancelOnDropPy wrapper**

### CancelOnDropPy: Automatic Cancellation

The core of the async bridge is the `CancelOnDropPy` struct, which ensures Python task cancellation when Rust futures are dropped:

```rust
struct CancelOnDropPy {
    inner: BoxFuture<'static, PyResult<Py<PyAny>>>,
    task: Py<PyAny>,          // Python Task handle
    event_loop: Py<PyAny>,    // Python event loop
    ctx: Py<PyAny>,           // contextvars.Context
    done: AtomicBool,         // Completion flag
}
```

**Key behaviors:**

1. **Cancellation on Drop**: When the Rust future is dropped (e.g., due to timeout or early return), the `Drop` implementation calls `task.cancel()` on the Python side via `event_loop.call_soon_threadsafe()`
2. **Context Preservation**: The `contextvars.Context` is captured and passed to `call_soon_threadsafe()` to ensure cancellation runs with the correct context variables
3. **Thread Safety**: Uses `call_soon_threadsafe()` to safely cancel from any thread, as the Rust future may complete on a different Tokio worker thread
4. **Idempotency**: The `done` flag prevents cancellation if the task already completed naturally

### Runtime Integration Features

The `pyo3-async-runtimes` crate with the `tokio-runtime` feature provides:

1. **Automatic Runtime Detection**: Detects the Python asyncio loop and creates a bridge to Tokio
2. **Future Conversion**: The `from_py_future()` function [rust/py_utils/src/future.rs:60-86]() converts Python awaitables to Rust futures with automatic cancellation
3. **Task Spawning**: Python tasks are created via `event_loop.create_task()` with context preservation
4. **Shared Runtime**: The static `TOKIO_RUNTIME` [rust/cocoindex/src/lib_context.rs:164]() is shared across all FFI calls for efficient resource usage

This enables CocoIndex to expose async Rust functions directly to Python users as native async/await functions, with proper cleanup semantics when operations are cancelled.

**Sources:** [Cargo.toml:19](), [rust/cocoindex/Cargo.toml:30](), [rust/py_utils/src/future.rs:1-87](), [rust/cocoindex/src/lib_context.rs:164-169]()

---

## Build System Integration

The FFI layer is built using Maturin, a tool specifically designed for building PyO3-based Python packages.

### Maturin Configuration

The build is configured in `pyproject.toml`:

```mermaid
graph LR
    subgraph "Configuration"
        PyProjectToml["pyproject.toml"]
        CargoToml["rust/cocoindex/Cargo.toml"]
    end
    
    subgraph "Build Process"
        Maturin["maturin build"]
        Cargo["cargo build --release"]
    end
    
    subgraph "Output"
        Wheel["cocoindex-*.whl<br/>(Python wheel)"]
        Cdylib["_engine.so/dll/dylib<br/>(inside wheel)"]
    end
    
    PyProjectToml -->|"build-backend"| Maturin
    CargoToml -->|"manifest-path"| Maturin
    Maturin -->|"invokes"| Cargo
    Cargo -->|"produces"| Cdylib
    Cdylib -->|"packaged in"| Wheel
```

### Key Configuration Parameters

| Parameter | Value | Purpose |
|-----------|-------|---------|
| `bindings` | `pyo3` | Specifies PyO3 as the binding type |
| `python-source` | `python` | Location of Python source files |
| `module-name` | `cocoindex._engine` | Name of the Python module |
| `manifest-path` | `rust/cocoindex/Cargo.toml` | Path to the Rust crate |
| `features` | `["pyo3/extension-module"]` | Enables PyO3 extension module mode |

### Development vs Release Builds

The build system supports two modes:

1. **Development Mode** (`maturin develop`): Builds the extension in-place for local development
2. **Release Mode** (`maturin build --release`): Produces optimized wheels for distribution

**Sources:** [pyproject.toml:57-64](), [rust/cocoindex/Cargo.toml:8-10]()

---

## ABI3 and Cross-Version Compatibility

CocoIndex uses Python's stable ABI (Application Binary Interface) to support multiple Python versions with a single compiled binary.

### ABI3 Architecture

```mermaid
graph TB
    subgraph "Single Compiled Binary"
        Cdylib["_engine.abi3.so<br/>(Linux x86_64)"]
    end
    
    subgraph "Python Versions"
        Py311["Python 3.11"]
        Py312["Python 3.12"]
        Py313["Python 3.13"]
        Py314["Python 3.14"]
    end
    
    subgraph "PyO3 Features"
        ABI3["abi3-py311<br/>(stable ABI)"]
    end
    
    ABI3 -->|"enables"| Cdylib
    Cdylib -->|"compatible with"| Py311
    Cdylib -->|"compatible with"| Py312
    Cdylib -->|"compatible with"| Py313
    Cdylib -->|"compatible with"| Py314
```

### Benefits and Trade-offs

**Benefits:**
- **Reduced Build Matrix**: Only need to build once per platform instead of once per Python version
- **Distribution Efficiency**: Smaller total download size across all Python versions
- **Forward Compatibility**: New Python versions automatically work if they maintain stable ABI

**Trade-offs:**
- **Minimum Version**: Requires Python 3.11+ (the baseline for `abi3-py311`)
- **API Restrictions**: Limited to Python stable ABI functions only (no unstable/private APIs)

### Compatibility Testing

The CI/CD pipeline tests ABI3 compatibility by:

1. Building wheels on one Python version (3.13)
2. Testing the same wheel on multiple Python versions (3.11, 3.12, 3.13)
3. Verifying successful import and basic functionality

**Sources:** [Cargo.toml:13](), [.github/workflows/release.yml:76-92]()

---

## Multi-Platform Wheel Building

CocoIndex produces platform-specific wheels for five platform configurations, each containing the compiled cdylib for that platform.

### Build Matrix

The release workflow builds wheels for:

| Platform | Runner | Target | Container | Wheel Tag |
|----------|--------|--------|-----------|-----------|
| Linux x86_64 | ubuntu-latest | x86_64 | manylinux_2_28 | linux_x86_64 |
| Linux aarch64 | ubuntu-24.04-arm | aarch64 | manylinux_2_28 | linux_aarch64 |
| macOS Intel | macos-13 | x86_64 | - | macosx_x86_64 |
| macOS Apple Silicon | macos-latest | aarch64 | - | macosx_arm64 |
| Windows | windows-latest | x64 | - | win_amd64 |

### Build Pipeline

```mermaid
graph TB
    subgraph "Source"
        RustCode["Rust source<br/>(rust/cocoindex/)"]
        PyCode["Python source<br/>(python/cocoindex/)"]
    end
    
    subgraph "Build Stage"
        LinuxX64["ubuntu-latest<br/>manylinux_2_28"]
        LinuxArm["ubuntu-24.04-arm<br/>manylinux_2_28"]
        MacosArm["macos-latest"]
        MacosX64["macos-13"]
        Windows["windows-latest"]
    end
    
    subgraph "Artifacts"
        WheelLinuxX64["cocoindex-*-linux_x86_64.whl"]
        WheelLinuxArm["cocoindex-*-linux_aarch64.whl"]
        WheelMacArm["cocoindex-*-macosx_arm64.whl"]
        WheelMacX64["cocoindex-*-macosx_x86_64.whl"]
        WheelWin["cocoindex-*-win_amd64.whl"]
    end
    
    subgraph "Distribution"
        PyPI["PyPI"]
    end
    
    RustCode --> LinuxX64
    RustCode --> LinuxArm
    RustCode --> MacosArm
    RustCode --> MacosX64
    RustCode --> Windows
    
    PyCode --> LinuxX64
    PyCode --> LinuxArm
    PyCode --> MacosArm
    PyCode --> MacosX64
    PyCode --> Windows
    
    LinuxX64 --> WheelLinuxX64
    LinuxArm --> WheelLinuxArm
    MacosArm --> WheelMacArm
    MacosX64 --> WheelMacX64
    Windows --> WheelWin
    
    WheelLinuxX64 --> PyPI
    WheelLinuxArm --> PyPI
    WheelMacArm --> PyPI
    WheelMacX64 --> PyPI
    WheelWin --> PyPI
```

### manylinux Containers

Linux builds use `manylinux_2_28` containers to ensure broad compatibility:
- **GLIBC 2.28**: Compatible with most Linux distributions from 2018+
- **Cross-compilation**: Supports building for different architectures (x86_64, aarch64)
- **Static Linking**: Minimizes runtime dependencies

### Build Optimization

The build process uses several optimizations:
- **sccache**: Caches compiled Rust artifacts across builds
- **Release Profile**: Aggressive optimizations in `[profile.release]`
  - `codegen-units = 1`: Maximum optimization
  - `lto = true`: Link-time optimization
  - `strip = "symbols"`: Removes debug symbols

**Sources:** [.github/workflows/release.yml:40-74](), [Cargo.toml:118-121]()

---

## Error Handling Across FFI Boundary

CocoIndex uses a three-tier error system that spans the Python-Rust boundary with proper context preservation.

### Error Type Hierarchy

```mermaid
graph TB
    subgraph "Rust Error Types"
        Error["Error enum"]
        ErrorClient["Error::Client"]
        ErrorHostLang["Error::HostLang"]
        ErrorContext["Error::Context"]
        ErrorInternal["Error::Internal"]
    end
    
    subgraph "FFI Conversion"
        IntoPyResult["IntoPyResult trait"]
        PyErr["PyErr"]
    end
    
    subgraph "Python Exceptions"
        ValueError["ValueError"]
        RuntimeError["RuntimeError"]
        CustomException["Custom exceptions"]
    end
    
    ErrorClient -->|"user error"| IntoPyResult
    ErrorHostLang -->|"Python error"| IntoPyResult
    ErrorContext -->|"with context chain"| IntoPyResult
    ErrorInternal -->|"internal error"| IntoPyResult
    
    IntoPyResult --> PyErr
    PyErr --> ValueError
    PyErr --> RuntimeError
    PyErr --> CustomException
```

**Diagram: Error propagation from Rust to Python**

### Error Variants

The `Error` enum [rust/utils/src/error.rs:18-23]() provides structured error handling:

| Variant | Purpose | Python Exception |
|---------|---------|------------------|
| `Client { msg, bt }` | User input errors with backtrace | `ValueError` or `400 Bad Request` |
| `HostLang(Box<dyn HostError>)` | Python exceptions wrapped in Rust | Preserved original Python exception |
| `Context { msg, source }` | Error with contextual information | Chained exception with context |
| `Internal(anyhow::Error)` | Internal Rust errors | `RuntimeError` or `500 Internal Server Error` |

### Context Chaining

Errors support context chaining via the `context()` method:

```rust
result
    .context("while processing source")?
    .context("in flow execution")?
```

This creates a chain of `Error::Context` variants that preserve the full error path, similar to Python's exception chaining. The formatted error displays all contexts in order:

```
Context:
  1: in flow execution
  2: while processing source
Invalid Request: file not found
```

### Host Language Error Preservation

The `Error::HostLang` variant [rust/utils/src/error.rs:18-23]() preserves Python exceptions that cross into Rust:

1. **Python Exception → Rust**: When Python code called from Rust raises an exception, it's wrapped in `Error::HostLang`
2. **Rust Processing**: The Rust code can add context or handle the error
3. **Rust → Python**: When returning to Python, the original exception is unwrapped and re-raised

This ensures Python stack traces are preserved even when errors pass through Rust code.

### IntoPyResult Trait

The `cocoindex_py_utils` crate provides the `IntoPyResult` trait for converting Rust `Result` types to Python-compatible results:

```rust
pub trait IntoPyResult<T> {
    fn into_py_result(self, py: Python<'_>) -> PyResult<T>;
}
```

This trait is implemented for `Result<T, Error>` and automatically handles error variant conversion to appropriate Python exceptions.

**Sources:** [rust/utils/src/error.rs:1-456](), [rust/py_utils/Cargo.toml:9](), [rust/cocoindex/src/prelude.rs:38]()

---

## FFI Function Exposure Pattern

PyO3 functions exposed to Python follow a standard pattern for error handling and type conversion.

### Function Declaration Pattern

```mermaid
graph LR
    subgraph "Rust Implementation"
        PyModule["#[pymodule]"]
        PyFunction["#[pyfunction]"]
        PyClass["#[pyclass]"]
        RustLogic["Business logic"]
    end
    
    subgraph "Type Conversion"
        Pythonize["pythonize::depythonize()"]
        Depythonize["pythonize::pythonize()"]
        FromPyObject["FromPyObject trait"]
        IntoPy["IntoPy trait"]
    end
    
    subgraph "Python Module"
        EngineModule["cocoindex._engine"]
        PythonAPI["cocoindex.py"]
    end
    
    PyFunction -->|"uses"| Pythonize
    PyFunction -->|"uses"| Depythonize
    RustLogic -->|"returns Result"| PyFunction
    PyModule -->|"registers"| PyFunction
    PyModule -->|"registers"| PyClass
    
    PyModule -->|"exported as"| EngineModule
    EngineModule -->|"imported by"| PythonAPI
```

**Diagram: PyO3 module and function registration pattern**

### Module Registration

The `cocoindex_py_utils` crate [rust/py_utils/Cargo.toml:1-18]() provides utilities for exposing Rust types and functions to Python. This includes:

- **Type Conversion**: Helper functions for converting between Python and Rust types using `pythonize`
- **Error Handling**: The `IntoPyResult` trait maps Rust `Error` to Python exceptions (`PyErr`)
- **Async Support**: The `from_py_future()` function [rust/py_utils/src/future.rs:60-86]() bridges Python async to Rust async with automatic cancellation

The `cocoindex_utils` crate [rust/utils/Cargo.toml:1-42]() provides the core error types and utilities used throughout the engine, with optional features for different subsystems.

**Sources:** [rust/cocoindex/Cargo.toml:25-26](), [rust/py_utils/Cargo.toml:1-18](), [rust/utils/Cargo.toml:1-42]()

---

## Subprocess Execution Architecture

For GPU-intensive operations, CocoIndex uses a subprocess-based executor that maintains FFI boundaries while isolating heavy workloads.

### ProcessPoolExecutor Integration

```mermaid
graph TB
    subgraph "Main Process"
        ExecutorStub["_ExecutorStub"]
        GetPool["_get_pool()"]
        Pool["ProcessPoolExecutor<br/>(max_workers=1)"]
        UserApps["_user_apps list"]
    end
    
    subgraph "Subprocess"
        SubprocessInit["_subprocess_init()"]
        LoadUserApp["load_user_app()"]
        ExecutorRegistry["_SUBPROC_EXECUTORS<br/>dict[bytes, _ExecutorEntry]"]
        ExecutorEntry["_ExecutorEntry<br/>(executor, prepare, analyze)"]
        ParentWatchdog["parent watchdog thread"]
    end
    
    subgraph "Execution"
        SpAnalyze["_sp_analyze()"]
        SpPrepare["_sp_prepare()"]
        SpCall["_sp_call()"]
    end
    
    ExecutorStub -->|"submits to"| Pool
    Pool -->|"initializes with"| SubprocessInit
    SubprocessInit -->|"loads"| LoadUserApp
    SubprocessInit -->|"starts"| ParentWatchdog
    LoadUserApp -->|"populates"| UserApps
    
    Pool -->|"executes"| SpAnalyze
    Pool -->|"executes"| SpPrepare
    Pool -->|"executes"| SpCall
    
    SpCall -->|"retrieves from"| ExecutorRegistry
    ExecutorRegistry -->|"contains"| ExecutorEntry
```

**Diagram: Subprocess execution architecture for GPU operations**

### Key Components

| Component | Location | Purpose |
|-----------|----------|---------|
| `_ExecutorStub` | [python/cocoindex/subprocess_exec.py:239-267]() | Main process stub that submits work to subprocess |
| `_get_pool()` | [python/cocoindex/subprocess_exec.py:38-49]() | Lazily creates singleton ProcessPoolExecutor |
| `_subprocess_init()` | [python/cocoindex/subprocess_exec.py:137-162]() | Subprocess initializer that loads user apps |
| `_start_parent_watchdog()` | [python/cocoindex/subprocess_exec.py:103-134]() | Monitors parent process and terminates if parent dies |
| `_SUBPROC_EXECUTORS` | [python/cocoindex/subprocess_exec.py:184]() | Registry of executor instances keyed by pickled spec |

### Restart and Recovery

The system includes automatic restart on subprocess failure:

1. **BrokenProcessPool Detection**: `_submit_with_restart()` [python/cocoindex/subprocess_exec.py:82-96]() catches `BrokenProcessPool` exceptions
2. **Pool Restart**: `_restart_pool()` [python/cocoindex/subprocess_exec.py:57-79]() safely shuts down the old pool and creates a new one
3. **Retry Loop**: Automatically retries the operation on the new subprocess
4. **Parent Watchdog**: Uses `psutil` [python/cocoindex/subprocess_exec.py:111-133]() to detect parent process termination and exit subprocess to prevent orphans

### Pickle-Based Caching

Executors are cached using pickled (executor_factory, spec) tuples as keys:

```python
self._key_bytes = pickle.dumps(
    (executor_factory, spec), 
    protocol=pickle.HIGHEST_PROTOCOL
)
```

This enables reuse of expensive resources (models, connections) across multiple calls while maintaining isolation between different specs.

**Sources:** [python/cocoindex/subprocess_exec.py:1-278]()

---

## Development Workflow

The FFI layer integrates into the development workflow through standardized commands using `uv` for dependency management.

### Local Development

```mermaid
graph LR
    subgraph "Setup"
        UvSync["uv sync"]
        UvDeps["Install Python deps"]
    end
    
    subgraph "Build"
        MaturinDevelop["uv run maturin develop"]
        CompileRust["cargo build"]
        InstallExtension["Install _engine.so"]
    end
    
    subgraph "Test"
        RustTest["uv run cargo test"]
        PyTest["uv run pytest"]
        Mypy["uv run mypy"]
    end
    
    UvSync --> UvDeps
    UvDeps --> MaturinDevelop
    MaturinDevelop --> CompileRust
    CompileRust --> InstallExtension
    InstallExtension --> RustTest
    RustTest --> PyTest
    PyTest --> Mypy
```

**Diagram: Development workflow with uv and maturin**

### CI/CD Integration

The test workflow [.github/workflows/_test.yml:1-99]() validates the FFI layer on all supported platforms:

1. **Environment Setup**: Install Rust toolchain and Python via `actions/setup-python`
2. **Dependency Sync**: Run `uv sync --no-dev --group ci` [.github/workflows/_test.yml:59]()
3. **Rust Tests**: Run `uv run --no-sync cargo test --verbose` [.github/workflows/_test.yml:62-65]()
4. **Python Build**: Run `uv run --no-sync maturin develop --strip` [.github/workflows/_test.yml:68]()
5. **Type Checking**: Run `uv run --no-sync mypy` [.github/workflows/_test.yml:71]()
6. **Python Tests**: Run `uv run --no-sync pytest` [.github/workflows/_test.yml:74]()

The build matrix tests on:
- **Platforms**: ubuntu-latest, ubuntu-24.04-arm, macos-latest, macos-15-intel, windows-latest
- **Python Version**: 3.11 (with ABI3 compatibility for 3.11+)
- **Architecture**: x86_64, aarch64, arm64

**Sources:** [.github/workflows/_test.yml:1-99](), [pyproject.toml:81-91]()

---

## Performance Characteristics

The PyO3 FFI layer provides high-performance data exchange between Python and Rust.

### Zero-Copy Operations

PyO3 supports zero-copy operations for certain data types:
- **Bytes/bytearray**: Direct buffer access without copying
- **NumPy arrays**: Direct access to array data through `numpy` crate
- **String views**: Borrowed string references when possible

### Overhead Considerations

| Operation | Overhead | Notes |
|-----------|----------|-------|
| Function call | ~100-200ns | FFI boundary crossing |
| Small data conversion | ~50ns | Simple types (int, float, bool) |
| Dict/struct conversion | ~microseconds | Depends on structure complexity |
| Async bridge | ~1-2 microseconds | Event loop coordination |

The FFI overhead is negligible compared to the work done in the Rust engine (I/O, database operations, LLM calls), making the Python-Rust split highly effective.

**Sources:** [Cargo.toml:12-19](), [rust/cocoindex/Cargo.toml:27-29]()

---

# Page: Build System: Maturin and Cross-Platform Wheels

# Build System: Maturin and Cross-Platform Wheels

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [.github/workflows/CI.yml](.github/workflows/CI.yml)
- [.github/workflows/_docs_release.yml](.github/workflows/_docs_release.yml)
- [.github/workflows/_test.yml](.github/workflows/_test.yml)
- [.github/workflows/docs_release.yml](.github/workflows/docs_release.yml)
- [.github/workflows/docs_test.yml](.github/workflows/docs_test.yml)
- [.github/workflows/format.yml](.github/workflows/format.yml)
- [.github/workflows/release.yml](.github/workflows/release.yml)
- [pyproject.toml](pyproject.toml)
- [python/cocoindex/subprocess_exec.py](python/cocoindex/subprocess_exec.py)
- [uv.lock](uv.lock)

</details>



## Purpose and Scope

This document explains CocoIndex's build system, which uses Maturin to compile Rust code into Python extension modules and package them as cross-platform wheels. The build system bridges the Rust engine with the Python API layer, producing binary distributions for 5 platform/architecture combinations with ABI3 compatibility across Python 3.11-3.14.

For information about the Rust-Python FFI layer and PyO3 bindings, see [12.1](#12.1). For information about the CI/CD pipeline that executes these builds, see [12.3](#12.3).

---

## Build System Overview

CocoIndex uses **Maturin** as its build backend to compile the Rust workspace into a Python extension module. Maturin handles the complex task of compiling Rust code with PyO3 bindings, generating platform-specific wheels, and ensuring compatibility with multiple Python versions through the stable ABI (ABI3).

### Build Process Flow

```mermaid
graph TB
    subgraph "Source Code"
        PySource["Python Source<br/>python/cocoindex/"]
        RustWorkspace["Rust Workspace<br/>rust/cocoindex/<br/>rust/utils/<br/>rust/py_utils/"]
        PyProject["pyproject.toml<br/>Build Configuration"]
        CargoToml["Cargo.toml<br/>Workspace Configuration"]
    end
    
    subgraph "Maturin Build Process"
        Maturin["maturin build<br/>or maturin develop"]
        CargoCompile["Cargo Compilation<br/>cdylib target"]
        PyO3Bridge["PyO3 Bindings<br/>Python-Rust FFI"]
        BinaryGen["Binary Generation<br/>_engine.so/_engine.pyd"]
    end
    
    subgraph "Output Artifacts"
        Wheel["Platform Wheel<br/>cocoindex-*.whl"]
        SDist["Source Distribution<br/>cocoindex-*.tar.gz"]
    end
    
    PyProject --> Maturin
    CargoToml --> Maturin
    RustWorkspace --> CargoCompile
    Maturin --> CargoCompile
    CargoCompile --> PyO3Bridge
    PyO3Bridge --> BinaryGen
    BinaryGen --> Wheel
    PySource --> Wheel
    
    Maturin --> SDist
```

**Sources:** [pyproject.toml:1-94](), [Cargo.toml:1-122](), [rust/cocoindex/Cargo.toml:1-133]()

---

## Maturin Configuration

### pyproject.toml Build Settings

The build configuration is defined in `pyproject.toml` using the `[tool.maturin]` section:

```toml
[tool.maturin]
bindings = "pyo3"
python-source = "python"
module-name = "cocoindex._engine"
features = ["pyo3/extension-module"]
include = ["THIRD_PARTY_NOTICES.html"]
manifest-path = "rust/cocoindex/Cargo.toml"
```

**Key Configuration Options:**

| Setting | Value | Purpose |
|---------|-------|---------|
| `bindings` | `"pyo3"` | Specifies PyO3 as the FFI framework |
| `python-source` | `"python"` | Location of Python source code |
| `module-name` | `"cocoindex._engine"` | Python module name for the compiled extension |
| `features` | `["pyo3/extension-module"]` | Enables extension module features |
| `manifest-path` | `"rust/cocoindex/Cargo.toml"` | Points to the specific crate within workspace |

**Sources:** [pyproject.toml:57-64]()

### Workspace Cargo Configuration

The root `Cargo.toml` defines workspace-level settings and dependencies:

```toml
[workspace]
members = ["rust/*"]
resolver = "2"

[workspace.package]
version = "999.0.0"
edition = "2024"
rust-version = "1.89"
license = "Apache-2.0"
```

**PyO3 Workspace Dependency:**

```toml
[workspace.dependencies]
pyo3 = { version = "0.25.1", features = [
    "abi3-py311",
    "auto-initialize",
    "chrono",
    "uuid",
] }
pythonize = "0.25.0"
pyo3-async-runtimes = { version = "0.25.0", features = ["tokio-runtime"] }
```

The `abi3-py311` feature enables the stable ABI, allowing a single wheel to work across Python 3.11-3.14 without recompilation.

**Sources:** [Cargo.toml:1-19](), [Cargo.toml:11-19]()

---

## cdylib Architecture

### Library Type Configuration

The main CocoIndex crate is configured as a `cdylib` (C dynamic library) to produce a shared library that Python can import:

```toml
[lib]
name = "cocoindex_engine"
crate-type = ["cdylib"]
```

This generates:
- **Linux:** `cocoindex_engine.so`
- **macOS:** `cocoindex_engine.dylib` (renamed to `.so` by Maturin)
- **Windows:** `cocoindex_engine.pyd`

The library is imported in Python as `cocoindex._engine`, providing the Rust engine's functionality to the Python layer.

**Sources:** [rust/cocoindex/Cargo.toml:8-10]()

### Dependency Structure

```mermaid
graph TB
    CocoIndex["cocoindex<br/>[cdylib]<br/>Main extension module"]
    PyUtils["cocoindex_py_utils<br/>[lib]<br/>PyO3 utilities"]
    Utils["cocoindex_utils<br/>[lib]<br/>Core utilities"]
    
    CocoIndex --> PyUtils
    CocoIndex --> Utils
    PyUtils --> Utils
    
    PyO3["pyo3 = 0.25.1<br/>abi3-py311"]
    Pythonize["pythonize = 0.25.0<br/>serde conversion"]
    AsyncRuntime["pyo3-async-runtimes<br/>tokio-runtime"]
    
    CocoIndex --> PyO3
    CocoIndex --> Pythonize
    CocoIndex --> AsyncRuntime
    PyUtils --> PyO3
    PyUtils --> Pythonize
    PyUtils --> AsyncRuntime
```

**Sources:** [rust/cocoindex/Cargo.toml:16-29](), [Cargo.toml:11-19]()

---

## Cross-Platform Wheel Building

### Supported Platforms

CocoIndex builds wheels for 5 platform/architecture combinations:

| Platform | Runner | Target | Container |
|----------|--------|--------|-----------|
| Linux x86_64 | `ubuntu-latest` | `x86_64` | `manylinux_2_28-cross:x86_64` |
| Linux aarch64 | `ubuntu-24.04-arm` | `aarch64` | `manylinux_2_28-cross:aarch64` |
| macOS ARM64 | `macos-latest` | `aarch64` | - |
| macOS x86_64 | `macos-13` | `x86_64` | - |
| Windows x64 | `windows-latest` | `x64` | - |

**Sources:** [.github/workflows/release.yml:44-50]()

### Release Workflow

The release workflow builds wheels for all platforms in parallel:

```mermaid
graph TB
    Start["Release Trigger<br/>Tag push or manual"]
    
    Generate3P["generate-3p-notices<br/>Generate THIRD_PARTY_NOTICES.html"]
    
    subgraph "Parallel Builds"
        LinuxX64["Linux x86_64 Build<br/>manylinux_2_28"]
        LinuxARM["Linux aarch64 Build<br/>manylinux_2_28"]
        MacARM["macOS ARM64 Build"]
        MacX64["macOS x86_64 Build"]
        WinX64["Windows x64 Build"]
    end
    
    SDist["Build Source Distribution<br/>sdist"]
    TestABI3["Test ABI3 Compatibility<br/>Python 3.11, 3.12, 3.13"]
    
    Release["Release to PyPI<br/>Upload all wheels"]
    Attestation["Generate Attestation<br/>Build provenance"]
    
    Start --> Generate3P
    Generate3P --> LinuxX64
    Generate3P --> LinuxARM
    Generate3P --> MacARM
    Generate3P --> MacX64
    Generate3P --> WinX64
    Generate3P --> SDist
    
    LinuxX64 --> TestABI3
    LinuxARM --> TestABI3
    MacARM --> TestABI3
    MacX64 --> TestABI3
    WinX64 --> TestABI3
    SDist --> TestABI3
    
    TestABI3 --> Attestation
    Attestation --> Release
```

**Sources:** [.github/workflows/release.yml:1-155]()

### Maturin Action Configuration

The build step uses the `PyO3/maturin-action` with platform-specific settings:

```yaml
- name: Build wheels
  uses: PyO3/maturin-action@v1
  with:
    target: ${{ matrix.platform.target }}
    args: --release --out dist
    sccache: 'true'
    manylinux: auto
    container: ${{ matrix.platform.container }}
```

**Key Options:**
- `--release`: Enables release optimizations
- `sccache: 'true'`: Uses shared compilation cache for faster builds
- `manylinux: auto`: Automatically selects manylinux compatibility level
- `container`: Specifies Docker container for Linux builds (manylinux_2_28)

**Sources:** [.github/workflows/release.yml:62-69]()

---

## ABI3 Compatibility

### Stable Python ABI

CocoIndex uses PyO3's `abi3-py311` feature to target the stable Python ABI, introduced in PEP 384. This allows a single wheel to work across multiple Python versions without recompilation.

**Benefits:**
- **Reduced build matrix:** Build once per platform instead of once per platform-Python combination
- **Forward compatibility:** Wheels built for Python 3.11 work on 3.12, 3.13, and 3.14
- **Smaller release artifacts:** Fewer wheels to distribute and maintain

### ABI3 Testing

The release workflow includes ABI3 validation:

```yaml
test-abi3:
  runs-on: ubuntu-24.04
  needs: build
  strategy:
    matrix:
      py: ["3.11", "3.12", "3.13"]
  steps:
    - uses: actions/download-artifact@v4
      with:
        name: wheels-linux-x86_64
    - uses: actions/setup-python@v5
      with:
        python-version: ${{ matrix.py }}
    - run: pip install --find-links=./ cocoindex
    - run: python -c "import cocoindex, sys; print('import ok on', sys.version)"
```

This test ensures that the Linux x86_64 wheel (built for Python 3.11 ABI) successfully imports on Python 3.11, 3.12, and 3.13.

**Sources:** [.github/workflows/release.yml:76-92](), [Cargo.toml:12-17]()

---

## Build Optimization

### Release Profile

The workspace defines aggressive optimization settings for release builds:

```toml
[profile.release]
codegen-units = 1
strip = "symbols"
lto = true
```

**Optimization Settings:**

| Setting | Value | Effect |
|---------|-------|--------|
| `codegen-units = 1` | Single codegen unit | Enables maximum inter-procedural optimization at cost of longer compile time |
| `strip = "symbols"` | Strip debug symbols | Reduces binary size significantly |
| `lto = true` | Link-time optimization | Optimizes across crate boundaries |

These settings produce smaller, faster binaries at the expense of longer build times. Since wheels are built in CI and not locally, the longer build time is acceptable.

**Sources:** [Cargo.toml:118-121]()

### Development Build

For local development, Maturin supports a faster development build with `maturin develop`:

```bash
maturin develop --strip -E all-ci
```

**Development Options:**
- `--strip`: Strips debug symbols even in debug builds to reduce size
- `-E all-ci`: Installs extra dependencies for testing
- No `--release` flag: Uses faster debug builds

This produces a debug build with reduced optimization but much faster compile times, suitable for iterative development.

**Sources:** [.github/workflows/_test.yml:70]()

---

## Build System Integration

### Local Development Workflow

```mermaid
graph LR
    Edit["Edit Code<br/>Rust or Python"]
    Develop["maturin develop<br/>Debug build"]
    Test["Run Tests<br/>pytest"]
    
    Edit --> Develop
    Develop --> Test
    Test --> Edit
    
    style Develop fill:#f9f9f9
```

### CI/CD Workflow

```mermaid
graph LR
    Push["Git Push<br/>to main/PR"]
    
    subgraph "Test Matrix"
        LinuxTest["Linux x86_64<br/>Debug build + tests"]
        LinuxArmTest["Linux aarch64<br/>Debug build + tests"]
        MacTest["macOS ARM64<br/>Debug build + tests"]
        MacX64Test["macOS x86_64<br/>Debug build + tests"]
        WinTest["Windows x64<br/>Debug build + tests"]
    end
    
    Format["Format Check<br/>Rust + Python"]
    
    Push --> LinuxTest
    Push --> LinuxArmTest
    Push --> MacTest
    Push --> MacX64Test
    Push --> WinTest
    Push --> Format
    
    style Format fill:#f9f9f9
```

**Test Workflow Details:**
- Each platform builds with `maturin develop` (debug mode)
- Runs Rust tests with `cargo test`
- Runs Python tests with `pytest`
- Validates type hints with `mypy`

**Sources:** [.github/workflows/_test.yml:1-101](), [.github/workflows/CI.yml:1-32]()

---

## Environment Configuration

### Build Environment Variables

The test workflow uses environment variables to control Rust compilation:

```yaml
env:
  RUSTFLAGS: "-C debuginfo=0 ${{ matrix.platform.extra_rustflags }}"
  CARGO_INCREMENTAL: "0"
  SCCACHE_GHA_ENABLED: "true"
  RUSTC_WRAPPER: "sccache"
```

**Variable Purposes:**

| Variable | Value | Purpose |
|----------|-------|---------|
| `RUSTFLAGS` | `-C debuginfo=0` | Disables debug info to reduce build size |
| `CARGO_INCREMENTAL` | `"0"` | Disables incremental compilation for reproducible builds |
| `SCCACHE_GHA_ENABLED` | `"true"` | Enables shared compilation cache |
| `RUSTC_WRAPPER` | `"sccache"` | Uses sccache as compiler wrapper |

**Platform-Specific Flags:**
- **macOS:** `-C split-debuginfo=off` to avoid debug info issues

**Sources:** [.github/workflows/_test.yml:24-28]()

### Third-Party Notices

The build system generates `THIRD_PARTY_NOTICES.html` containing license information for all dependencies:

```yaml
- name: Generate THIRD_PARTY_NOTICES.html
  run: cargo about generate -o THIRD_PARTY_NOTICES.html about.hbs
```

This file is included in every wheel via the `include` directive in `pyproject.toml` and ensures license compliance.

**Sources:** [.github/workflows/release.yml:32-33](), [pyproject.toml:62]()

---

## Build Artifacts

### Wheel Naming Convention

Maturin generates wheels following PEP 427 naming:

```
cocoindex-{version}-{python_tag}-{abi_tag}-{platform_tag}.whl
```

**Example wheel names:**
- `cocoindex-0.1.0-cp311-abi3-manylinux_2_28_x86_64.whl`
- `cocoindex-0.1.0-cp311-abi3-macosx_11_0_arm64.whl`
- `cocoindex-0.1.0-cp311-abi3-win_amd64.whl`

**Tag Components:**
- `cp311`: CPython 3.11
- `abi3`: Stable ABI (forward compatible)
- `manylinux_2_28_x86_64`: Linux with glibc 2.28+ on x86_64

### Source Distribution

In addition to wheels, the build system generates a source distribution (sdist):

```yaml
- name: Build sdist
  uses: PyO3/maturin-action@v1
  with:
    command: sdist
    args: --out dist
```

The sdist contains:
- Python source code from `python/`
- Rust source code from `rust/`
- Build configuration (`pyproject.toml`, `Cargo.toml`)
- License and documentation files

Users can install from source with `pip install cocoindex --no-binary cocoindex`, which triggers a local Rust compilation.

**Sources:** [.github/workflows/release.yml:105-109]()

---

## Summary

The CocoIndex build system demonstrates a sophisticated Rust-Python hybrid architecture:

1. **Maturin** bridges Rust and Python, compiling the Rust workspace into Python extension modules
2. **ABI3 compatibility** reduces build matrix from 15 wheels (5 platforms × 3 Python versions) to 5 wheels
3. **Aggressive optimization** (LTO, single codegen unit) produces small, fast binaries
4. **Cross-platform CI/CD** ensures consistent builds across Linux, macOS, and Windows
5. **Cryptographic attestation** provides supply chain security for released wheels

This architecture balances **performance** (Rust for compute-intensive operations), **usability** (Python for user-facing APIs), and **distribution** (platform-specific wheels with broad compatibility).

**Sources:** [pyproject.toml:1-94](), [Cargo.toml:1-122](), [rust/cocoindex/Cargo.toml:1-133](), [.github/workflows/release.yml:1-155](), [.github/workflows/_test.yml:1-101]()

---

# Page: CI/CD Pipeline

# CI/CD Pipeline

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [.github/workflows/CI.yml](.github/workflows/CI.yml)
- [.github/workflows/_docs_release.yml](.github/workflows/_docs_release.yml)
- [.github/workflows/_test.yml](.github/workflows/_test.yml)
- [.github/workflows/docs_release.yml](.github/workflows/docs_release.yml)
- [.github/workflows/docs_test.yml](.github/workflows/docs_test.yml)
- [.github/workflows/format.yml](.github/workflows/format.yml)
- [.github/workflows/release.yml](.github/workflows/release.yml)
- [pyproject.toml](pyproject.toml)
- [python/cocoindex/subprocess_exec.py](python/cocoindex/subprocess_exec.py)
- [uv.lock](uv.lock)

</details>



## Purpose and Scope

This document describes the Continuous Integration and Continuous Deployment (CI/CD) infrastructure for CocoIndex. The CI/CD system is implemented using GitHub Actions and orchestrates testing across five platform architectures, format validation, multi-platform wheel building, PyPI publishing, and documentation deployment. For information about the build system configuration itself (maturin and PyO3), see [Build System: Maturin and Cross-Platform Wheels](#12.2).

The pipeline ensures code quality through comprehensive testing, enforces consistent formatting across Rust and Python codebases, and automates the release process with cryptographic attestations for supply chain security.

**Sources:** [.github/workflows/_test.yml:1-101](), [.github/workflows/release.yml:1-155](), [.github/workflows/CI.yml:1-32]()

---

## Workflow Architecture Overview

The CI/CD system consists of seven GitHub Actions workflows organized into a hierarchical structure with internal (reusable) and external (triggerable) workflows:

**Diagram: CI/CD Workflow Architecture**

```mermaid
graph TB
    subgraph "Entry Point Workflows"
        CI[".github/workflows/CI.yml<br/>Triggered on PR/push to main"]
        Format[".github/workflows/format.yml<br/>Format Check on PR/push"]
        Release[".github/workflows/release.yml<br/>Triggered on tag push or manual"]
        DocsTest[".github/workflows/docs_test.yml<br/>Docs test on PR"]
        DocsRelease[".github/workflows/docs_release.yml<br/>Manual docs release"]
    end
    
    subgraph "Internal Reusable Workflows"
        InternalTest[".github/workflows/_test.yml<br/>workflow_call: Run Tests"]
        InternalDocs[".github/workflows/_docs_release.yml<br/>workflow_call: Deploy Docs"]
    end
    
    subgraph "Jobs in _test.yml"
        BuildTest["build-test<br/>5 platform matrix"]
        Validate3P["validate-3p-notices<br/>cargo-about validation"]
    end
    
    subgraph "Jobs in release.yml"
        Gen3P["generate-3p-notices<br/>Create THIRD_PARTY_NOTICES.html"]
        Build["build<br/>5 platform wheel builds"]
        TestABI["test-abi3<br/>Test 3.11, 3.12, 3.13"]
        SDist["sdist<br/>Build source distribution"]
        ReleaseJob["release<br/>PyPI publish + attestations"]
        ReleaseDocs["release-docs<br/>Calls _docs_release.yml"]
    end
    
    CI --> InternalTest
    InternalTest --> BuildTest
    InternalTest --> Validate3P
    
    Release --> Gen3P
    Gen3P --> Build
    Gen3P --> SDist
    Build --> TestABI
    Build --> ReleaseJob
    TestABI --> ReleaseJob
    SDist --> ReleaseJob
    ReleaseJob --> ReleaseDocs
    ReleaseDocs --> InternalDocs
    
    DocsRelease --> InternalDocs
```

**Sources:** [.github/workflows/CI.yml:1-32](), [.github/workflows/format.yml:1-56](), [.github/workflows/release.yml:1-155](), [.github/workflows/docs_test.yml:1-27](), [.github/workflows/_test.yml:1-101](), [.github/workflows/_docs_release.yml:1-37]()

---

## Test Workflow Matrix

The `_test.yml` internal workflow executes comprehensive tests across five platform/architecture combinations and multiple Python versions. This workflow is called by the main `CI.yml` workflow.

### Platform Matrix Configuration

**Diagram: Test Matrix Platform Coverage**

```mermaid
graph LR
    subgraph "Platform Matrix"
        UbuntuX64["ubuntu-latest<br/>x86_64<br/>python_exec: .venv/bin/python"]
        UbuntuARM["ubuntu-24.04-arm<br/>aarch64<br/>python_exec: .venv/bin/python"]
        MacOSARM["macos-latest<br/>ARM64<br/>extra_rustflags: -C split-debuginfo=off"]
        MacOSX64["macos-13<br/>x86_64<br/>extra_rustflags: -C split-debuginfo=off"]
        Windows["windows-latest<br/>x64<br/>python_exec: .venv\\Scripts\\python"]
    end
    
    subgraph "Test Steps"
        RustNoDefault["Rust tests<br/>--no-default-features"]
        RustDefault["Rust tests<br/>with default features"]
        MaturinDev["maturin develop<br/>--strip -E all-ci"]
        Mypy["mypy type check<br/>python/"]
        Pytest["pytest<br/>python/cocoindex/tests"]
    end
    
    UbuntuX64 --> RustNoDefault
    UbuntuARM --> RustNoDefault
    MacOSARM --> RustNoDefault
    MacOSX64 --> RustNoDefault
    Windows --> RustNoDefault
    
    RustNoDefault --> RustDefault
    RustDefault --> MaturinDev
    MaturinDev --> Mypy
    Mypy --> Pytest
```

The matrix is defined at [.github/workflows/_test.yml:14-22]():

| Platform | Runner | Python Exec Path | Extra Rust Flags |
|----------|--------|------------------|------------------|
| Ubuntu x86_64 | `ubuntu-latest` | `.venv/bin/python` | None |
| Ubuntu ARM | `ubuntu-24.04-arm` | `.venv/bin/python` | None |
| macOS ARM | `macos-latest` | `.venv/bin/python` | `-C split-debuginfo=off` |
| macOS x86_64 | `macos-13` | `.venv/bin/python` | `-C split-debuginfo=off` |
| Windows | `windows-latest` | `.venv\Scripts\python` | None |

**Sources:** [.github/workflows/_test.yml:14-22]()

### Build Optimization with sccache

The test workflow uses `sccache` for Rust compilation caching to reduce build times. Environment variables are configured at [.github/workflows/_test.yml:24-28]():

```
RUSTFLAGS: "-C debuginfo=0 ${{ matrix.platform.extra_rustflags }}"
CARGO_INCREMENTAL: "0"
SCCACHE_GHA_ENABLED: "true"
RUSTC_WRAPPER: "sccache"
```

The `sccache-action` is initialized at [.github/workflows/_test.yml:35-36](), and debug symbols are disabled (`-C debuginfo=0`) to reduce artifact sizes. Incremental compilation is disabled because sccache provides better cross-build caching.

**Sources:** [.github/workflows/_test.yml:24-36]()

### Test Execution Sequence

Each platform executes the following test sequence:

1. **Rust Tests (no default features)** - [.github/workflows/_test.yml:56-57]()
   ```
   cargo test --no-default-features --verbose
   ```

2. **Rust Tests (with features)** - [.github/workflows/_test.yml:59-60]()
   ```
   cargo test --verbose
   ```

3. **Python Development Build** - [.github/workflows/_test.yml:68-70]()
   ```
   maturin develop --strip -E all-ci
   ```
   Uses the `all-ci` feature set defined in [pyproject.toml:83]() which includes `pytest`, `pytest-asyncio`, `mypy`, and `pydantic>=2.11.9`.

4. **Type Checking** - [.github/workflows/_test.yml:71-73]()
   ```
   mypy python
   ```

5. **Python Unit Tests** - [.github/workflows/_test.yml:74-76]()
   ```
   pytest --capture=no python/cocoindex/tests
   ```

**Sources:** [.github/workflows/_test.yml:56-76](), [pyproject.toml:83]()

### Third-Party License Validation

The `validate-3p-notices` job verifies that third-party license notices are up-to-date by running `cargo-about` in dry-run mode at [.github/workflows/_test.yml:78-101]():

```
cargo about generate about.hbs > /dev/null
```

If validation fails, contributors must update `/about.toml` and regenerate notices.

**Sources:** [.github/workflows/_test.yml:78-101]()

---

## Format Checking Workflow

The `format.yml` workflow enforces consistent code style across Rust and Python codebases without running tests. It is triggered on pull requests and pushes to `main` when files in `rust/`, `python/`, or `examples/` directories change.

**Diagram: Format Check Jobs**

```mermaid
graph TB
    FormatWorkflow[".github/workflows/format.yml"]
    
    subgraph "Rust Format Check"
        RustCheckout["actions/checkout@v4"]
        RustToolchain["dtolnay/rust-toolchain@stable<br/>components: rustfmt"]
        RustFmt["cargo fmt --check"]
    end
    
    subgraph "Python Format Check"
        PyCheckout["actions/checkout@v4"]
        PySetup["actions/setup-python@v5<br/>python-version: 3.11"]
        RuffInstall["pip install ruff"]
        RuffCheck["ruff format --check ."]
    end
    
    FormatWorkflow --> RustCheckout
    RustCheckout --> RustToolchain
    RustToolchain --> RustFmt
    
    FormatWorkflow --> PyCheckout
    PyCheckout --> PySetup
    PySetup --> RuffInstall
    RuffInstall --> RuffCheck
```

### Rust Format Checking

Defined at [.github/workflows/format.yml:27-39](), this job:
- Uses `rustfmt` component from stable Rust toolchain
- Runs `cargo fmt --check` which exits with error if formatting issues are found
- Does not modify files (check-only mode)

### Python Format Checking

Defined at [.github/workflows/format.yml:41-56](), this job:
- Uses Python 3.11 as the base version
- Installs `ruff` formatter
- Runs `ruff format --check .` to verify formatting without modifications

**Sources:** [.github/workflows/format.yml:1-56]()

---

## Release Workflow Pipeline

The `release.yml` workflow orchestrates a multi-stage release process that builds platform-specific wheels, tests ABI compatibility, generates cryptographic attestations, and publishes to PyPI. The workflow is triggered automatically on tag pushes matching `v*` or can be manually triggered for dry-run testing.

### Release Pipeline Stages

**Diagram: Release Job Dependencies**

```mermaid
graph TB
    Trigger["Tag push 'v*' or<br/>workflow_dispatch"]
    
    Gen3P["generate-3p-notices<br/>cargo-about generate"]
    Artifact3P["Upload THIRD_PARTY_NOTICES.html<br/>artifact"]
    
    Build["build job<br/>5 platform matrix"]
    SDist["sdist job<br/>Source distribution"]
    
    TestABI["test-abi3 job<br/>Test Python 3.11, 3.12, 3.13"]
    
    Release["release job<br/>Attestation + PyPI publish"]
    
    ReleaseDocs["release-docs job<br/>Call _docs_release.yml"]
    
    Trigger --> Gen3P
    Gen3P --> Artifact3P
    Artifact3P --> Build
    Artifact3P --> SDist
    Build --> TestABI
    Build --> Release
    SDist --> Release
    TestABI --> Release
    Release --> ReleaseDocs
```

**Sources:** [.github/workflows/release.yml:1-155]()

### Third-Party Notices Generation

The first stage ([.github/workflows/release.yml:18-38]()) generates `THIRD_PARTY_NOTICES.html` containing all open-source license attributions:

1. Checks out repository
2. Runs `.github/scripts/update_version.sh` to update version strings
3. Installs `cargo-about` via `cargo-binstall`
4. Generates notices: `cargo about generate -o THIRD_PARTY_NOTICES.html about.hbs`
5. Uploads as artifact for use by downstream jobs

This artifact is downloaded by all build jobs and included in the final wheel packages as specified in [pyproject.toml:62]().

**Sources:** [.github/workflows/release.yml:18-38](), [pyproject.toml:62]()

### Multi-Platform Wheel Building

The `build` job ([.github/workflows/release.yml:40-74]()) creates wheels for five platform targets:

| OS | Runner | Target | Container |
|----|--------|--------|-----------|
| Linux x86_64 | `ubuntu-latest` | `x86_64` | `manylinux_2_28-cross:x86_64` |
| Linux ARM | `ubuntu-24.04-arm` | `aarch64` | `manylinux_2_28-cross:aarch64` |
| macOS ARM | `macos-latest` | `aarch64` | None |
| macOS x86_64 | `macos-13` | `x86_64` | None |
| Windows | `windows-latest` | `x64` | None |

The build process uses the `PyO3/maturin-action@v1` ([.github/workflows/release.yml:62-69]()):

```yaml
- name: Build wheels
  uses: PyO3/maturin-action@v1
  with:
    target: ${{ matrix.platform.target }}
    args: --release --out dist
    sccache: 'true'
    manylinux: auto
    container: ${{ matrix.platform.container }}
```

Linux builds use `manylinux_2_28` containers for broad compatibility. The action automatically configures cross-compilation toolchains and handles platform-specific build requirements.

**Sources:** [.github/workflows/release.yml:40-74]()

### ABI3 Compatibility Testing

The `test-abi3` job ([.github/workflows/release.yml:76-92]()) verifies that the built wheels work across multiple Python versions using the stable ABI:

```yaml
strategy:
  matrix:
    py: ["3.11", "3.12", "3.13"]
```

For each Python version, the job:
1. Downloads the Linux x86_64 wheel
2. Sets up the specific Python version
3. Installs cocoindex from local wheel: `pip install --find-links=./ cocoindex`
4. Tests import: `python -c "import cocoindex, sys; print('import ok on', sys.version)"`

This ensures the single wheel built with ABI3 support works across Python 3.11, 3.12, and 3.13 as declared in [pyproject.toml:31-34]().

**Sources:** [.github/workflows/release.yml:76-92](), [pyproject.toml:31-34]()

### Source Distribution Build

The `sdist` job ([.github/workflows/release.yml:94-114]()) creates the source distribution tarball:

```yaml
- name: Build sdist
  uses: PyO3/maturin-action@v1
  with:
    command: sdist
    args: --out dist
```

The sdist includes Rust source code and can be used for platforms not covered by pre-built wheels.

**Sources:** [.github/workflows/release.yml:94-114]()

### Release with Cryptographic Attestations

The `release` job ([.github/workflows/release.yml:116-149]()) performs the final publication with supply chain security measures:

**Permissions Required:**
- `id-token: write` - Sign release artifacts
- `contents: write` - Upload release artifacts  
- `attestations: write` - Generate artifact attestation

**Key steps:**

1. **Download all artifacts** ([.github/workflows/release.yml:133-142]()):
   - Third-party notices
   - All platform wheels (5 platforms)
   - Source distribution

2. **Generate artifact attestation** ([.github/workflows/release.yml:139-142]()): Uses `actions/attest-build-provenance@v1` to create cryptographic attestations for all wheels with subject path `'wheels-*/*'`. This creates a verifiable chain of custody proving the artifacts were built by this GitHub Actions workflow.

3. **Publish to PyPI** ([.github/workflows/release.yml:143-148]()): Only executes on actual tag pushes (not manual triggers):
   ```yaml
   if: ${{ startsWith(github.ref, 'refs/tags/') }}
   uses: PyO3/maturin-action@v1
   with:
     command: upload
     args: --non-interactive --skip-existing wheels-*/*
   ```

**Sources:** [.github/workflows/release.yml:116-149]()

---

## Documentation Deployment

CocoIndex documentation is deployed to GitHub Pages through two workflows: `_docs_release.yml` (internal reusable) and `docs_release.yml` (manual trigger).

### Documentation Build and Deploy Process

**Diagram: Documentation Deployment Flow**

```mermaid
graph TB
    subgraph "Trigger Sources"
        Manual["docs_release.yml<br/>workflow_dispatch"]
        FromRelease["release.yml<br/>release-docs job"]
    end
    
    Internal["_docs_release.yml<br/>workflow_call"]
    
    subgraph "Deploy Steps"
        Checkout["actions/checkout@v4"]
        NodeSetup["actions/setup-node@v4<br/>cache: yarn"]
        SSHAgent["webfactory/ssh-agent<br/>ssh-private-key: GH_PAGES_DEPLOY"]
        SetEnv["Set environment variables<br/>POSTHOG, MIXPANEL, ALGOLIA"]
        GitConfig["git config user.email/name"]
        YarnInstall["yarn --cwd docs install"]
        YarnDeploy["yarn --cwd docs deploy"]
    end
    
    Manual --> Internal
    FromRelease --> Internal
    Internal --> Checkout
    Checkout --> NodeSetup
    NodeSetup --> SSHAgent
    SSHAgent --> SetEnv
    SetEnv --> GitConfig
    GitConfig --> YarnInstall
    YarnInstall --> YarnDeploy
```

### Documentation Release Workflow

The internal `_docs_release.yml` workflow ([.github/workflows/_docs_release.yml:1-37]()) performs the deployment:

1. **SSH Authentication**: Uses `webfactory/ssh-agent@v0.5.0` with the `GH_PAGES_DEPLOY` secret to authenticate for pushing to the `gh-pages` branch.

2. **Environment Configuration** ([.github/workflows/_docs_release.yml:26-32]()): Sets analytics and search configuration:
   ```bash
   export COCOINDEX_DOCS_POSTHOG_API_KEY=${{ vars.COCOINDEX_DOCS_POSTHOG_API_KEY }}
   export COCOINDEX_DOCS_MIXPANEL_API_KEY=${{ vars.COCOINDEX_DOCS_MIXPANEL_API_KEY }}
   export COCOINDEX_DOCS_ALGOLIA_APP_ID=${{ vars.COCOINDEX_DOCS_ALGOLIA_APP_ID }}
   export COCOINDEX_DOCS_ALGOLIA_API_KEY=${{ vars.COCOINDEX_DOCS_ALGOLIA_API_KEY }}
   ```

3. **Git Configuration** ([.github/workflows/_docs_release.yml:33-34]()): Configures commit author for documentation updates.

4. **Deployment**: Executes `yarn --cwd docs deploy` which builds and pushes the documentation site to GitHub Pages.

**Permissions**: The workflow requires `contents: read` and `pages: write` permissions ([.github/workflows/_docs_release.yml:7-8]()).

**Sources:** [.github/workflows/_docs_release.yml:1-37]()

### Documentation Testing

The `docs_test.yml` workflow ([.github/workflows/docs_test.yml:1-27]()) validates documentation builds on pull requests:

```yaml
- name: Test build website
  run: yarn --cwd docs build
```

This ensures documentation changes don't break the build before merging.

**Sources:** [.github/workflows/docs_test.yml:1-27]()

---

## Workflow Triggers and Path Filters

Each workflow is configured with specific trigger conditions and path filters to optimize CI/CD resource usage:

### CI Workflow Triggers

**Diagram: Workflow Trigger Conditions**

```mermaid
graph TB
    subgraph "CI.yml Triggers"
        CIPullRequest["pull_request<br/>branches: main"]
        CIPush["push<br/>branches: main"]
        CIManual["workflow_dispatch"]
    end
    
    subgraph "Watched Paths"
        RustPath["rust/**"]
        PythonPath["python/**"]
        TomlPath["*.toml"]
        WorkflowPath[".github/workflows/*.yml"]
    end
    
    CIPullRequest --> RustPath
    CIPullRequest --> PythonPath
    CIPullRequest --> TomlPath
    CIPullRequest --> WorkflowPath
    
    CIPush --> RustPath
    CIPush --> PythonPath
    CIPush --> TomlPath
    CIPush --> WorkflowPath
```

### Workflow Trigger Matrix

| Workflow | Trigger Events | Path Filters | Purpose |
|----------|---------------|--------------|---------|
| `CI.yml` | PR to main, push to main, manual | `rust/**`, `python/**`, `*.toml`, `.github/workflows/*.yml` | Run full test suite |
| `format.yml` | PR to main, push to main, manual | `rust/**`, `python/**`, `examples/**` | Check code formatting |
| `release.yml` | Tag push `v*`, manual | None | Build and publish release |
| `docs_test.yml` | PR to main | `docs/**`, `.github/workflows/docs_test.yml` | Test documentation build |
| `docs_release.yml` | Manual only | None | Deploy documentation |

**Sources:** [.github/workflows/CI.yml:8-23](), [.github/workflows/format.yml:8-21](), [.github/workflows/release.yml:7-11](), [.github/workflows/docs_test.yml:3-8](), [.github/workflows/docs_release.yml:3-4]()

### Path Filter Optimization

Path filters prevent unnecessary workflow runs. For example:
- Changes to `docs/` only trigger `docs_test.yml`, not the full test suite
- Changes to `README.md` don't trigger any workflows
- Changes to workflow files themselves trigger `CI.yml` to test the CI changes

This optimization reduces compute costs and speeds up feedback for contributors.

**Sources:** [.github/workflows/CI.yml:11-22](), [.github/workflows/format.yml:11-20]()

---

## Security and Permissions Model

The CI/CD pipeline implements least-privilege permissions and security best practices.

### Workflow Permissions Summary

| Workflow | Permissions | Purpose |
|----------|-------------|---------|
| `_test.yml` | `contents: read` | Checkout code only |
| `release.yml` (release job) | `id-token: write`, `contents: write`, `attestations: write` | Sign artifacts, upload releases, generate attestations |
| `release.yml` (other jobs) | `contents: read` | Checkout code only |
| `_docs_release.yml` | `contents: read`, `pages: write` | Deploy to GitHub Pages |
| `format.yml` | `contents: read` | Checkout code only |
| `CI.yml` | `contents: read` | Checkout code only |

**Sources:** [.github/workflows/_test.yml:6-7](), [.github/workflows/release.yml:13-15](), [.github/workflows/release.yml:120-126](), [.github/workflows/_docs_release.yml:6-8](), [.github/workflows/format.yml:23-24](), [.github/workflows/CI.yml:25-26]()

### Cryptographic Attestations

The release workflow generates build provenance attestations using `actions/attest-build-provenance@v1` ([.github/workflows/release.yml:139-142]()):

```yaml
- name: Generate artifact attestation
  uses: actions/attest-build-provenance@v1
  with:
    subject-path: 'wheels-*/*'
```

This creates a Sigstore-compatible attestation that:
- Links the built wheels to the exact GitHub Actions workflow run that produced them
- Records the commit SHA, repository, workflow file, and runner environment
- Can be verified by users to ensure authenticity

These attestations enhance supply chain security by providing cryptographic proof of build provenance.

**Sources:** [.github/workflows/release.yml:139-142]()

### Environment Protection

The release workflow uses GitHub environment protection ([.github/workflows/release.yml:127]()) with the `release` environment. This allows administrators to:
- Require manual approval before publishing to PyPI
- Restrict who can trigger releases
- Configure environment-specific secrets

**Sources:** [.github/workflows/release.yml:127]()

---

## Build Caching and Performance Optimization

The CI/CD pipeline uses multiple caching strategies to reduce build times and improve performance.

### sccache Configuration

All build and test workflows use `sccache` for Rust compilation caching. Configuration at [.github/workflows/_test.yml:24-28]():

```yaml
env:
  RUSTFLAGS: "-C debuginfo=0 ${{ matrix.platform.extra_rustflags }}"
  CARGO_INCREMENTAL: "0"
  SCCACHE_GHA_ENABLED: "true"
  RUSTC_WRAPPER: "sccache"
```

**Key optimizations:**
- `SCCACHE_GHA_ENABLED: "true"` - Uses GitHub Actions cache backend
- `CARGO_INCREMENTAL: "0"` - Disables incremental compilation since sccache provides better cross-build caching
- `RUSTFLAGS: "-C debuginfo=0"` - Disables debug symbols to reduce artifact size
- macOS builds add `-C split-debuginfo=off` to avoid platform-specific debug info issues

The `mozilla-actions/sccache-action@v0.0.9` action initializes sccache at [.github/workflows/_test.yml:35-36]().

**Sources:** [.github/workflows/_test.yml:24-36]()

### Yarn Cache

Documentation workflows cache Yarn dependencies using `actions/setup-node@v4` with cache configuration ([.github/workflows/docs_test.yml:20-23](), [.github/workflows/_docs_release.yml:18-21]()):

```yaml
- uses: actions/setup-node@v4
  with:
    cache: yarn
    cache-dependency-path: docs/yarn.lock
```

This caches `node_modules` based on the `docs/yarn.lock` file hash.

**Sources:** [.github/workflows/docs_test.yml:20-23](), [.github/workflows/_docs_release.yml:18-21]()

### Disabled Caching

Python and Rust workspace caching are currently disabled due to GitHub Actions cache size limits:

- Python pip cache: Commented at [.github/workflows/_test.yml:42-43]()
- Rust target cache: Commented at [.github/workflows/_test.yml:47-54]()

The comments indicate these may be re-enabled when cache size limits increase.

**Sources:** [.github/workflows/_test.yml:42-43](), [.github/workflows/_test.yml:47-54]()

---

## Version Management and Release Process

### Version Update Script

The release workflow calls `.github/scripts/update_version.sh` at multiple stages ([.github/workflows/release.yml:24](), [.github/workflows/release.yml:55](), [.github/workflows/release.yml:101](), [.github/workflows/release.yml:132]()) to synchronize version strings across:
- `pyproject.toml` - Python package version
- `Cargo.toml` files - Rust crate versions
- Documentation configuration

This ensures version consistency across all package components before building and publishing.

**Sources:** [.github/workflows/release.yml:24](), [.github/workflows/release.yml:55](), [.github/workflows/release.yml:101](), [.github/workflows/release.yml:132]()

### Maturin Build Configuration

The build system is configured in [pyproject.toml:57-64]():

```toml
[tool.maturin]
bindings = "pyo3"
python-source = "python"
module-name = "cocoindex._engine"
features = ["pyo3/extension-module"]
include = ["THIRD_PARTY_NOTICES.html"]
manifest-path = "rust/cocoindex/Cargo.toml"
```

- `bindings = "pyo3"` - Uses PyO3 for Rust-Python bindings
- `module-name = "cocoindex._engine"` - Generates the internal `_engine` module
- `manifest-path` - Points to the Rust crate within the workspace
- `include` - Bundles third-party notices in the wheel

**Sources:** [pyproject.toml:57-64]()

### Release Dry-Run Mode

The release workflow supports dry-run mode via manual `workflow_dispatch` trigger. In this mode:
- All jobs execute (building, testing, attestation generation)
- PyPI publishing is skipped (conditional on `startsWith(github.ref, 'refs/tags/')`)
- Documentation deployment is skipped (conditional on tag push)

This allows testing the release pipeline without publishing artifacts.

**Sources:** [.github/workflows/release.yml:7-11](), [.github/workflows/release.yml:144](), [.github/workflows/release.yml:153]()

---

# Page: Subprocess Execution Architecture

# Subprocess Execution Architecture

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [.github/workflows/CI.yml](.github/workflows/CI.yml)
- [.github/workflows/_docs_release.yml](.github/workflows/_docs_release.yml)
- [.github/workflows/_test.yml](.github/workflows/_test.yml)
- [.github/workflows/docs_release.yml](.github/workflows/docs_release.yml)
- [.github/workflows/docs_test.yml](.github/workflows/docs_test.yml)
- [.github/workflows/format.yml](.github/workflows/format.yml)
- [.github/workflows/release.yml](.github/workflows/release.yml)
- [pyproject.toml](pyproject.toml)
- [python/cocoindex/subprocess_exec.py](python/cocoindex/subprocess_exec.py)
- [uv.lock](uv.lock)

</details>



## Purpose and Scope

This document describes CocoIndex's subprocess execution system, which enables running operations (particularly GPU-intensive operations) in isolated worker processes. The architecture provides automatic recovery from process failures, parent-child lifecycle management, and efficient executor reuse through caching.

This system is primarily used for operations decorated with `@executor_class` that require GPU resources or process isolation. For information about operation registration and decorators, see [Operation Registration and Decorators](#4.1). For the broader Rust engine integration, see [Rust Engine Architecture](#12).

**Sources:** [python/cocoindex/subprocess_exec.py:1-278]()

## Architecture Overview

The subprocess execution architecture consists of three main components:

1. **Parent Process**: Maintains a singleton `ProcessPoolExecutor` with a single worker and provides `_ExecutorStub` proxies
2. **Worker Process**: Hosts an executor registry, executes operations, and runs a parent watchdog thread
3. **Communication Layer**: Uses Python's multiprocessing with pickle serialization for cross-process data transfer

```mermaid
graph TB
    subgraph "Parent Process"
        stub["_ExecutorStub"]
        pool["ProcessPoolExecutor<br/>(max_workers=1)"]
        restart["_restart_pool()"]
    end
    
    subgraph "Worker Process"
        init["_subprocess_init()"]
        watchdog["Parent Watchdog Thread"]
        registry["_SUBPROC_EXECUTORS<br/>dict[bytes, _ExecutorEntry]"]
        analyze["_sp_analyze()"]
        prepare["_sp_prepare()"]
        call["_sp_call()"]
    end
    
    stub -->|"submit()"| pool
    pool -->|"pickle & spawn"| init
    init -->|"starts"| watchdog
    init -->|"loads user apps"| registry
    
    pool -->|"future"| analyze
    pool -->|"future"| prepare
    pool -->|"future"| call
    
    analyze --> registry
    prepare --> registry
    call --> registry
    
    watchdog -->|"monitors parent<br/>exits if dead"| stub
    
    restart -->|"on BrokenProcessPool"| pool
```

The system ensures that GPU operations run in a dedicated worker process to avoid device conflicts, while providing transparent proxy semantics to the parent process.

**Sources:** [python/cocoindex/subprocess_exec.py:1-49](), [python/cocoindex/subprocess_exec.py:163-232]()

## Process Pool Management

### Singleton Pool Initialization

The parent process maintains a single global `ProcessPoolExecutor` instance, created lazily on first access:

```mermaid
graph LR
    A["_pool = None<br/>_pool_lock = threading.Lock()"]
    B["_get_pool()"]
    C{"pool exists?"}
    D["Create ProcessPoolExecutor<br/>max_workers=1<br/>mp_context=spawn"]
    E["Return pool"]
    
    B --> C
    C -->|"No"| D
    D --> E
    C -->|"Yes"| E
```

**Key characteristics:**
- **Single worker**: `max_workers=1` ensures sequential execution and avoids GPU contention
- **Spawn context**: Uses `mp.get_context("spawn")` for clean process creation on all platforms
- **Initializer**: Passes `_subprocess_init` with user app targets and parent PID
- **Thread-safe**: Protected by `_pool_lock` for concurrent access

| Component | Location | Purpose |
|-----------|----------|---------|
| `_pool` | [python/cocoindex/subprocess_exec.py:33]() | Global pool singleton |
| `_pool_lock` | [python/cocoindex/subprocess_exec.py:32]() | Thread synchronization |
| `_user_apps` | [python/cocoindex/subprocess_exec.py:34]() | User app targets to load |
| `_get_pool()` | [python/cocoindex/subprocess_exec.py:38-49]() | Pool accessor with lazy init |
| `add_user_app()` | [python/cocoindex/subprocess_exec.py:52-54]() | Register user app for loading |

**Sources:** [python/cocoindex/subprocess_exec.py:29-54]()

### Pool Restart and Recovery

When the worker process crashes or becomes unresponsive, the pool is automatically restarted:

```mermaid
stateDiagram-v2
    [*] --> PoolRunning: _get_pool()
    PoolRunning --> DetectFailure: BrokenProcessPool exception
    DetectFailure --> Restarting: _restart_pool()
    Restarting --> ShutdownOld: Acquire _pool_lock
    ShutdownOld --> CreateNew: prev_pool.shutdown(cancel_futures=True)
    CreateNew --> PoolRunning: New ProcessPoolExecutor
    PoolRunning --> WorkSubmitted: _submit_with_restart()
    WorkSubmitted --> PoolRunning: Success
    WorkSubmitted --> DetectFailure: Failure
```

The `_restart_pool()` function performs thread-safe pool replacement:

- **Lock-protected**: Ensures only one thread performs restart
- **Identity check**: Skips restart if another thread already swapped the pool
- **Clean shutdown**: Cancels pending futures in the old pool
- **Retry loop**: `_submit_with_restart()` retries indefinitely until success

**Sources:** [python/cocoindex/subprocess_exec.py:57-96]()

## Executor Stub Pattern

### _ExecutorStub Implementation

The `_ExecutorStub` class provides a proxy interface that delegates execution to the subprocess:

```mermaid
classDiagram
    class _ExecutorStub {
        -bytes _key_bytes
        +__init__(executor_factory, spec)
        +__call__(*args, **kwargs) async
        +analyze() [optional]
        +prepare() [optional] async
    }
    
    class ExecutorFactory {
        <<interface>>
        +analyze()
        +prepare()
        +__call__(*args, **kwargs)
    }
    
    _ExecutorStub --> ExecutorFactory: wraps via pickle
    
    note for _ExecutorStub "Key = pickle(executor_factory, spec)\nConditionally exposes analyze/prepare\nbased on factory's methods"
```

**Key features:**
- **Pickle-based key**: `(executor_factory, spec)` tuple pickled to identify executor instances
- **Conditional methods**: Only exposes `analyze()` and `prepare()` if the underlying factory has them
- **Async interface**: `__call__` and `prepare` are async, `analyze` is sync (wrapped with `execution_context.run()`)
- **Transparent proxying**: Callers interact with stub as if it were the actual executor

**Sources:** [python/cocoindex/subprocess_exec.py:239-277]()

### Method Resolution

The stub dynamically attaches methods based on the executor factory's capabilities:

| Method | Presence Check | Execution Context | Return Path |
|--------|---------------|------------------|-------------|
| `analyze()` | `hasattr(executor_factory, "analyze")` | Sync via `execution_context.run()` | `_sp_analyze()` in subprocess |
| `prepare()` | `hasattr(executor_factory, "prepare")` | Async via `_submit_with_restart()` | `_sp_prepare()` in subprocess |
| `__call__()` | Always present | Async via `_submit_with_restart()` | `_sp_call()` in subprocess |

**Sources:** [python/cocoindex/subprocess_exec.py:247-266]()

## Subprocess Initialization and Lifecycle

### Initialization Sequence

When the worker process spawns, `_subprocess_init()` performs setup:

```mermaid
sequenceDiagram
    participant Parent as Parent Process
    participant Worker as Worker Process
    participant Watchdog as Watchdog Thread
    
    Parent->>Worker: spawn with initargs=(user_apps, parent_pid)
    Worker->>Worker: Enable faulthandler
    Worker->>Worker: Signal handling: SIGINT → SIG_IGN
    Worker->>Watchdog: Start parent watchdog thread
    Worker->>Worker: Check already loaded apps
    Worker->>Worker: Load each user_app via load_user_app()
    Worker->>Worker: Update _user_apps registry
    Note over Worker: Ready to accept work
```

**Initialization steps:**

1. **Fault handling**: `faulthandler.enable()` for crash diagnostics
2. **Signal configuration**: Ignores `SIGINT` to avoid KeyboardInterrupt propagation
3. **Watchdog startup**: Launches background thread monitoring parent process
4. **User app loading**: Imports user-defined operations and flows via `load_user_app()`
5. **Duplicate prevention**: Skips apps already loaded if process was forked

| Function | Location | Purpose |
|----------|----------|---------|
| `_subprocess_init()` | [python/cocoindex/subprocess_exec.py:137-162]() | Worker process entry point |
| `load_user_app()` | [python/cocoindex/subprocess_exec.py:22]() | Imports user application modules |

**Sources:** [python/cocoindex/subprocess_exec.py:137-162](), [pyproject.toml:19]()

## Parent Watchdog Mechanism

### Watchdog Thread Design

The parent watchdog ensures the worker process terminates if the parent dies or crashes:

```mermaid
graph TB
    subgraph "Worker Process"
        init["_subprocess_init()"]
        watchdog["_start_parent_watchdog()"]
        thread["Daemon Thread"]
        check["Check Loop"]
    end
    
    subgraph "Checks"
        pid["Get parent psutil.Process"]
        cache["Cache create_time"]
        alive{"Parent alive?"}
        same{"Same PID?"}
        exit["os._exit(1)"]
    end
    
    init --> watchdog
    watchdog --> pid
    pid --> cache
    cache --> thread
    thread --> check
    check --> alive
    alive -->|"No"| exit
    alive -->|"Yes"| same
    same -->|"No (PID reuse)"| exit
    same -->|"Yes"| check
    
    check -.->|"sleep(WATCHDOG_INTERVAL_SECONDS)"| check
```

**Implementation details:**

- **PID reuse protection**: Caches `create_time()` to detect when OS reuses the parent PID
- **Daemon thread**: Runs in background without blocking worker operations
- **Polling interval**: `WATCHDOG_INTERVAL_SECONDS = 10.0` balances responsiveness and overhead
- **Hard exit**: Uses `os._exit(1)` for immediate termination without cleanup
- **psutil dependency**: Requires `psutil>=7.2.1` for cross-platform process monitoring

**Sources:** [python/cocoindex/subprocess_exec.py:103-135](), [python/cocoindex/subprocess_exec.py:27](), [pyproject.toml:19]()

### Watchdog Failure Modes

| Scenario | Detection | Action |
|----------|-----------|--------|
| Parent exits normally | `is_running() == False` | Immediate exit |
| Parent crashes | `NoSuchProcess` exception | Immediate exit |
| Parent PID reused | `create_time() != cached` | Immediate exit |
| Parent inaccessible | `psutil.Error` on init | Exit before entering loop |

**Sources:** [python/cocoindex/subprocess_exec.py:116-134]()

## Executor Registry and Caching

### Registry Structure

The subprocess maintains a dictionary mapping pickled executor keys to `_ExecutorEntry` instances:

```mermaid
classDiagram
    class _SUBPROC_EXECUTORS {
        <<dict~bytes, _ExecutorEntry~>>
        +get(key_bytes)
        +__setitem__(key_bytes, entry)
    }
    
    class _ExecutorEntry {
        +executor: Any
        +prepare: _OnceResult
        +analyze: _OnceResult
        +ready_to_call: bool
    }
    
    class _OnceResult {
        -_result: Any
        -_done: bool
        +run_once(method, *args, **kwargs)
    }
    
    _SUBPROC_EXECUTORS --> _ExecutorEntry: values
    _ExecutorEntry --> _OnceResult: contains 2x
    
    note for _OnceResult "Memoizes first call result\nSubsequent calls return cached"
```

**Key behaviors:**

- **Lazy creation**: Executors only created on first use via `_get_or_create_entry()`
- **Persistent caching**: Entry survives across multiple calls for same (executor_factory, spec)
- **Result memoization**: `_OnceResult` ensures `analyze()` and `prepare()` run at most once per entry
- **Spec assignment**: `inst.spec = spec` after creating `executor_factory()` instance

| Component | Location | Purpose |
|-----------|----------|---------|
| `_SUBPROC_EXECUTORS` | [python/cocoindex/subprocess_exec.py:184]() | Global registry in subprocess |
| `_ExecutorEntry` | [python/cocoindex/subprocess_exec.py:176-182]() | Per-executor state holder |
| `_OnceResult` | [python/cocoindex/subprocess_exec.py:164-173]() | Memoization helper |
| `_get_or_create_entry()` | [python/cocoindex/subprocess_exec.py:200-208]() | Registry accessor |

**Sources:** [python/cocoindex/subprocess_exec.py:164-208]()

### Caching Strategy

The caching system has three levels:

1. **Executor instance**: Created once per (executor_factory, spec) key
2. **analyze() result**: Cached in `_ExecutorEntry.analyze._OnceResult`
3. **prepare() result**: Cached in `_ExecutorEntry.prepare._OnceResult`

```mermaid
sequenceDiagram
    participant Stub as _ExecutorStub
    participant Pool as ProcessPoolExecutor
    participant Registry as _SUBPROC_EXECUTORS
    participant Entry as _ExecutorEntry
    participant Executor as Actual Executor
    
    Note over Stub: First call to analyze()
    Stub->>Pool: submit(_sp_analyze, key_bytes)
    Pool->>Registry: _get_or_create_entry(key_bytes)
    Registry->>Entry: Create new entry
    Entry->>Executor: executor_factory()
    Entry->>Executor: inst.spec = spec
    Entry->>Executor: analyze()
    Executor-->>Entry: result
    Entry->>Entry: Cache in _OnceResult
    Entry-->>Stub: return result
    
    Note over Stub: Second call to analyze()
    Stub->>Pool: submit(_sp_analyze, key_bytes)
    Pool->>Registry: _get_or_create_entry(key_bytes)
    Registry-->>Pool: Return existing entry
    Pool->>Entry: run_once(analyze)
    Entry-->>Pool: Return cached result
    Pool-->>Stub: return result
```

**Sources:** [python/cocoindex/subprocess_exec.py:210-231]()

## Error Recovery and Restart Logic

### Recovery Mechanism

The `_submit_with_restart()` function implements automatic recovery from subprocess failures:

```mermaid
graph TD
    A["_submit_with_restart(fn, *args)"]
    B["Get current pool: _get_pool()"]
    C["Submit to pool: pool.submit(fn, *args)"]
    D["Await future: asyncio.wrap_future(fut)"]
    E{"Success?"}
    F["Return result"]
    G{"BrokenProcessPool?"}
    H["_restart_pool(old_pool)"]
    I["Retry submit"]
    
    A --> B
    B --> C
    C --> D
    D --> E
    E -->|"Yes"| F
    E -->|"No"| G
    G -->|"Yes"| H
    H --> I
    I --> B
    G -->|"Other exception"| F
```

**Recovery guarantees:**

- **Infinite retry**: Loops indefinitely on `BrokenProcessPool` until success
- **Exception propagation**: Non-pool-related exceptions bubble up to caller
- **State reset**: New pool starts fresh with empty executor registry
- **Idempotent operations**: Relies on operations being safe to retry

| Failure Type | Detection | Recovery Action |
|-------------|-----------|----------------|
| Process crash | `BrokenProcessPool` | Restart pool, retry |
| Parent death | Watchdog triggers `os._exit(1)` | Parent detects broken pool, restarts |
| Serialization error | Pickle exception | Propagates to caller |
| Timeout (OS-level) | Future never completes | No explicit timeout; relies on OS |

**Sources:** [python/cocoindex/subprocess_exec.py:82-96]()

### Call Execution with Preparation

The `_sp_call()` function ensures executors are properly prepared before each invocation:

```mermaid
sequenceDiagram
    participant Stub as _ExecutorStub.__call__()
    participant Call as _sp_call()
    participant Entry as _ExecutorEntry
    participant Exec as Executor
    
    Stub->>Call: key_bytes, args, kwargs
    Call->>Entry: Get or create entry
    
    alt Not ready_to_call
        Call->>Entry: Check if analyze() exists
        Entry->>Exec: run_once(analyze)
        Call->>Entry: Check if prepare() exists
        Entry->>Exec: run_once(prepare)
        Entry->>Entry: ready_to_call = True
    end
    
    Call->>Exec: _call_method(__call__, args, kwargs)
    
    alt Executor is async
        Exec->>Exec: asyncio.run(method(...))
    else Executor is sync
        Exec->>Exec: method(...)
    end
    
    Exec-->>Call: Result
    Call-->>Stub: Result
```

**Preparation logic:**

- **Lazy preparation**: `analyze()` and `prepare()` only called before first `__call__`
- **Per-crash reset**: `ready_to_call` flag ensures re-preparation after subprocess restart
- **Optional methods**: Skips `analyze()`/`prepare()` if executor doesn't have them
- **Async handling**: `_call_method()` detects coroutines and runs them with `asyncio.run()`

**Sources:** [python/cocoindex/subprocess_exec.py:187-232]()

## Call Execution Flow

### End-to-End Execution Path

Complete flow from stub creation to result return:

```mermaid
flowchart TD
    A["User creates executor_stub(factory, spec)"]
    B["_ExecutorStub.__init__"]
    C["Pickle (factory, spec) to key_bytes"]
    D["User calls stub(*args, **kwargs)"]
    E["_submit_with_restart(_sp_call, key_bytes, args, kwargs)"]
    F["ProcessPoolExecutor.submit()"]
    G["Worker: _sp_call(key_bytes, args, kwargs)"]
    H["_get_or_create_entry(key_bytes)"]
    I{"Entry exists?"}
    J["Create: executor_factory()"]
    K["Assign: inst.spec = spec"]
    L["Create _ExecutorEntry"]
    M{"ready_to_call?"}
    N["Run analyze() if exists"]
    O["Run prepare() if exists"]
    P["Set ready_to_call = True"]
    Q["Call executor(*args, **kwargs)"]
    R["Return result to parent"]
    S["asyncio.wrap_future() awaits"]
    T["Return to user"]
    
    A --> B
    B --> C
    C --> D
    D --> E
    E --> F
    F --> G
    G --> H
    H --> I
    I -->|"No"| J
    J --> K
    K --> L
    I -->|"Yes"| M
    L --> M
    M -->|"No"| N
    N --> O
    O --> P
    M -->|"Yes"| Q
    P --> Q
    Q --> R
    R --> S
    S --> T
```

**Sources:** [python/cocoindex/subprocess_exec.py:239-278](), [python/cocoindex/subprocess_exec.py:82-96](), [python/cocoindex/subprocess_exec.py:221-232]()

### Async/Sync Bridge

The `_call_method()` helper bridges async and sync executors:

| Executor Type | Detection | Execution |
|--------------|-----------|-----------|
| Async (coroutine function) | `asyncio.iscoroutinefunction(method)` | `asyncio.run(method(*args, **kwargs))` |
| Sync (regular function) | Not a coroutine function | `method(*args, **kwargs)` |
| Error during call | Any exception | Wrapped in `RuntimeError` with context |

**Sources:** [python/cocoindex/subprocess_exec.py:187-198]()

## Integration with Operation System

The subprocess architecture integrates with CocoIndex's operation system (see [Operation Registration and Decorators](#4.1)):

- **GPU operations**: Operations decorated with `@executor_class` that set `gpu=True` in `OpArgs`
- **Isolation requirement**: Separates GPU device contexts between parent and worker processes
- **Automatic wrapping**: Engine automatically wraps GPU executors with `executor_stub()`
- **Transparent execution**: User code calls operations normally; subprocess execution is hidden

**Sources:** [python/cocoindex/subprocess_exec.py:1-10]()

---

# Page: Error Handling and Retry Logic

# Error Handling and Retry Logic

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [Cargo.lock](Cargo.lock)
- [Cargo.toml](Cargo.toml)
- [rust/cocoindex/Cargo.toml](rust/cocoindex/Cargo.toml)
- [rust/cocoindex/src/lib_context.rs](rust/cocoindex/src/lib_context.rs)
- [rust/cocoindex/src/prelude.rs](rust/cocoindex/src/prelude.rs)
- [rust/py_utils/Cargo.toml](rust/py_utils/Cargo.toml)
- [rust/py_utils/src/future.rs](rust/py_utils/src/future.rs)
- [rust/utils/Cargo.toml](rust/utils/Cargo.toml)
- [rust/utils/src/error.rs](rust/utils/src/error.rs)
- [rust/utils/src/prelude.rs](rust/utils/src/prelude.rs)
- [rust/utils/src/retryable.rs](rust/utils/src/retryable.rs)

</details>



This document describes CocoIndex's error handling architecture and retry mechanism. The system provides a unified error model that bridges Rust and Python across the FFI boundary, with support for context chaining, automatic backtraces, and configurable retry logic with exponential backoff.

For information about the FFI boundary and PyO3 integration, see [PyO3 Integration and FFI Layer](#12.1). For subprocess error handling, see [Subprocess Execution Architecture](#12.4).

---

## Purpose and Scope

The error handling system serves three primary functions:

1. **Unified Error Type**: A single `Error` enum that handles client errors, internal errors, host language (Python) errors, and contextual information
2. **Retry Logic**: Automatic retry with exponential backoff for transient failures in external systems (LLM APIs, databases, cloud storage)
3. **FFI Safety**: Type-safe error propagation across the Rust-Python boundary via PyO3

The system distinguishes between recoverable errors (which should be retried) and permanent failures (which should fail fast), enabling robust operation against unreliable external services.

---

## Error Type Hierarchy

### Core Error Variants

CocoIndex uses a unified `Error` enum with four variants to represent all error conditions:

```mermaid
graph TB
    Error["Error enum"]
    
    subgraph "Error Variants"
        Client["Client<br/>{msg: String, bt: Backtrace}"]
        Internal["Internal<br/>(anyhow::Error)"]
        HostLang["HostLang<br/>(Box&lt;dyn HostError&gt;)"]
        Context["Context<br/>{msg: String, source: Box&lt;SError&gt;}"]
    end
    
    Error --> Client
    Error --> Internal
    Error --> HostLang
    Error --> Context
    
    Context --> Client
    Context --> Internal
    Context --> HostLang
    Context --> Context
    
    Client -.captures.-> Backtrace["Backtrace<br/>(for debugging)"]
    Internal -.wraps.-> AnyhowError["anyhow::Error<br/>(with chain)"]
    HostLang -.wraps.-> PyError["Python Exception<br/>(via PyO3)"]
    
    note1["Invalid user input<br/>HTTP 400 responses"]
    note2["System/library errors<br/>HTTP 500 responses"]
    note3["Python exceptions<br/>across FFI boundary"]
    note4["Adds context without<br/>losing original error"]
    
    Client -.-> note1
    Internal -.-> note2
    HostLang -.-> note3
    Context -.-> note4
```

**Sources:** [rust/utils/src/error.rs:18-23]()

### Error Construction Methods

| Method | Variant | Use Case | Backtrace |
|--------|---------|----------|-----------|
| `Error::client(msg)` | `Client` | Invalid user input, malformed requests | Captured |
| `Error::internal(e)` | `Internal` | System errors, library errors | From source |
| `Error::internal_msg(msg)` | `Internal` | Internal logic errors | Captured |
| `Error::host(e)` | `HostLang` | Python exceptions crossing FFI | None |
| `error.context(msg)` | `Context` | Adding contextual information | Preserved |

**Sources:** [rust/utils/src/error.rs:54-111]()

---

## Context Chaining System

Errors support layered context to provide detailed failure information without losing the original error:

```mermaid
graph TB
    subgraph "Context Chain Example"
        Top["Error::Context<br/>{msg: 'Failed to process flow'}"]
        Mid["Error::Context<br/>{msg: 'Failed to read source'}"]
        Bottom["Error::Internal<br/>(reqwest::Error: connection timeout)"]
        
        Top --> Mid
        Mid --> Bottom
    end
    
    subgraph "Display Format"
        Display["Context:<br/>  1: Failed to process flow<br/>  2: Failed to read source<br/>connection timeout"]
    end
    
    subgraph "API Response"
        HTTP["HTTP 500<br/>{error: 'Context:\\n  1: Failed...'}"]
    end
    
    Top -.formats to.-> Display
    Display -.serializes to.-> HTTP
```

**Sources:** [rust/utils/src/error.rs:99-111](), [rust/utils/src/error.rs:117-129]()

### ContextExt Trait

The `ContextExt` trait provides ergonomic methods for adding context to `Result` types:

```rust
// From error.rs
pub trait ContextExt<T> {
    fn context<C: Into<String>>(self, context: C) -> Result<T>;
    fn with_context<C: Into<String>, F: FnOnce() -> C>(self, f: F) -> Result<T>;
}
```

This trait is implemented for:
- `Result<T, Error>` - Direct context addition
- `Result<T, E>` where `E: StdError` - Converts to `Error::Internal` then adds context
- `Option<T>` - Converts `None` to `Error::Client` with context

**Usage pattern:**
```rust
// Eager context
let result = operation().context("Failed to connect to database")?;

// Lazy context (only evaluated on error)
let result = operation().with_context(|| format!("Failed to process item {}", id))?;
```

**Sources:** [rust/utils/src/error.rs:138-171]()

---

## Retry Mechanism

### IsRetryable Trait

The retry system is based on the `IsRetryable` trait, which determines whether an error warrants a retry:

```mermaid
graph LR
    subgraph "IsRetryable Implementations"
        RetryableError["retryable::Error"]
        ReqwestError["reqwest::Error"]
        OpenAIError["OpenAIError"]
        Neo4jError["neo4rs::Error"]
    end
    
    IsRetryable["IsRetryable trait<br/>fn is_retryable(&self) -> bool"]
    
    RetryableError -.implements.-> IsRetryable
    ReqwestError -.implements.-> IsRetryable
    OpenAIError -.implements.-> IsRetryable
    Neo4jError -.implements.-> IsRetryable
    
    ReqwestError --> Check429["Check status == 429<br/>(Too Many Requests)"]
    OpenAIError --> DelegateReqwest["Delegate to<br/>inner reqwest::Error"]
    Neo4jError --> CheckTransient["ConnectionError or<br/>Neo4jErrorKind::Transient"]
```

**Sources:** [rust/utils/src/retryable.rs:7-64]()

### Retry Configuration

The `RetryOptions` struct controls retry behavior:

| Field | Default | Description |
|-------|---------|-------------|
| `retry_timeout` | `Some(10 minutes)` | Maximum time to retry before giving up |
| `initial_backoff` | `100ms` | Starting backoff duration |
| `max_backoff` | `10s` | Maximum backoff duration |

A specialized configuration exists for heavily loaded systems:

```rust
pub static HEAVY_LOADED_OPTIONS: RetryOptions = RetryOptions {
    retry_timeout: Some(DEFAULT_RETRY_TIMEOUT),
    initial_backoff: Duration::from_secs(1),
    max_backoff: Duration::from_secs(60),
};
```

**Sources:** [rust/utils/src/retryable.rs:113-133]()

---

## Exponential Backoff Algorithm

The retry logic implements exponential backoff with jitter to avoid thundering herd problems:

```mermaid
graph TB
    Start["Start Operation"]
    Execute["Execute async function"]
    Success["Return success"]
    CheckRetry{"Is retryable?"}
    CheckDeadline{"Past deadline?"}
    Sleep["Sleep backoff duration"]
    UpdateBackoff["backoff = min(<br/>backoff * random(1.618-2.0),<br/>max_backoff)"]
    
    Start --> Execute
    Execute --> Success
    Execute --> CheckRetry
    
    CheckRetry -->|No| Return["Return error"]
    CheckRetry -->|Yes| CheckDeadline
    
    CheckDeadline -->|Yes| Return
    CheckDeadline -->|No| Sleep
    
    Sleep --> UpdateBackoff
    UpdateBackoff --> Execute
    
    Note1["Jitter factor:<br/>1.618-2.0<br/>(golden ratio based)"]
    UpdateBackoff -.-> Note1
```

**Key algorithm details:**

1. **Jitter calculation**: `backoff * random_range(1618..=2000) / 1000` applies a 61.8%-200% multiplier
2. **Deadline enforcement**: Remaining time is capped to deadline
3. **Backoff cap**: Never exceeds `max_backoff` regardless of iterations
4. **Immediate retry**: First retry happens after `initial_backoff`, not 0

**Sources:** [rust/utils/src/retryable.rs:135-182]()

---

## Error Construction Macros

CocoIndex provides convenience macros for common error patterns:

### Client Error Macros

```rust
// Immediate return with formatted client error
client_bail!("Invalid flow name: {}", name);

// Create client error without returning
let err = client_error!("Invalid configuration");
```

### Internal Error Macros

```rust
// Immediate return with formatted internal error
internal_bail!("Database inconsistency detected");

// Create internal error without returning
let err = internal_error!("Unexpected state");
```

### API Error Macros

```rust
// For HTTP API endpoints - returns 400 status
api_bail!("Missing required parameter: {}", param_name);

let err = api_error!("Validation failed");
```

**Sources:** [rust/utils/src/error.rs:190-455]()

---

## FFI Boundary Error Handling

### HostError Type

The `HostError` trait represents errors originating from Python that cross into Rust:

```mermaid
graph TB
    subgraph "Python Side"
        PyException["Python Exception"]
    end
    
    subgraph "FFI Boundary (PyO3)"
        PyErr["pyo3::PyErr"]
        Conversion["Convert to HostError"]
    end
    
    subgraph "Rust Side"
        HostError["Box&lt;dyn HostError&gt;"]
        ErrorEnum["Error::HostLang"]
    end
    
    PyException --> PyErr
    PyErr --> Conversion
    Conversion --> HostError
    HostError --> ErrorEnum
    
    Note["HostError trait:<br/>Any + StdError + Send + Sync + 'static"]
    HostError -.-> Note
```

**Type definition:**
```rust
pub trait HostError: Any + StdError + Send + Sync + 'static {}
impl<T: Any + StdError + Send + Sync + 'static> HostError for T {}
```

This allows Rust to store and propagate Python exceptions while maintaining type safety and error chaining.

**Sources:** [rust/utils/src/error.rs:15-16]()

### Cancellation Handling

For async operations crossing the FFI boundary, `CancelOnDropPy` ensures Python tasks are cancelled when Rust futures are dropped:

```mermaid
sequenceDiagram
    participant Rust as Rust Future
    participant Wrapper as CancelOnDropPy
    participant PyTask as Python Task
    participant EventLoop as Python Event Loop
    
    Rust->>Wrapper: Create from awaitable
    Wrapper->>PyTask: create_task(awaitable)
    Wrapper->>Wrapper: Store task handle
    
    alt Future completes normally
        PyTask->>Wrapper: Result ready
        Wrapper->>Rust: Poll::Ready(result)
    else Future dropped (cancelled)
        Rust->>Wrapper: Drop
        Wrapper->>EventLoop: call_soon_threadsafe
        EventLoop->>PyTask: task.cancel()
    end
```

This prevents resource leaks when Rust cancels operations that involve Python coroutines.

**Sources:** [rust/py_utils/src/future.rs:14-86]()

---

## Integration with LibContext and FlowContext

The error system integrates with CocoIndex's context management:

```mermaid
graph TB
    subgraph "LibContext Operations"
        GetPool["db_pools.get_pool()"]
        GetFlow["get_flow_context()"]
        RequireDB["require_builtin_db_pool()"]
    end
    
    subgraph "FlowContext Operations"
        UseExecCtx["use_execution_ctx()"]
        GetSourceCtx["get_source_indexing_context()"]
    end
    
    subgraph "Error Types Returned"
        ConnError["Connection errors<br/>(Internal)"]
        NotFound["Flow not found<br/>(ApiError 404)"]
        SetupRequired["Setup not up-to-date<br/>(ApiError 400)"]
        NoDatabase["Database required<br/>(Client)"]
    end
    
    GetPool -->|fails| ConnError
    GetFlow -->|fails| NotFound
    RequireDB -->|fails| NoDatabase
    UseExecCtx -->|fails| SetupRequired
    GetSourceCtx -->|fails| ConnError
```

**Key patterns:**

1. **Database connection failures** → `Error::Internal` with context
2. **Missing resources** → `ApiError` with appropriate HTTP status
3. **Configuration issues** → `Error::Client` with actionable message
4. **Setup state mismatches** → `ApiError` with setup instructions

**Sources:** [rust/cocoindex/src/lib_context.rs:183-229](), [rust/cocoindex/src/lib_context.rs:252-283](), [rust/cocoindex/src/lib_context.rs:133-157]()

---

## HTTP Response Mapping

The error system automatically converts to HTTP responses via Axum:

```mermaid
graph LR
    subgraph "Error Variants"
        Client["Error::Client"]
        Internal["Error::Internal"]
        HostLang["Error::HostLang"]
        Context["Error::Context"]
    end
    
    subgraph "HTTP Status Codes"
        Status400["400 Bad Request"]
        Status500["500 Internal Server Error"]
    end
    
    subgraph "Response Body"
        JSON["JSON: {error: string}"]
    end
    
    Client --> Status400
    Internal --> Status500
    HostLang --> Status500
    Context --> Status500
    
    Status400 --> JSON
    Status500 --> JSON
```

**Implementation:** The `IntoResponse` trait converts errors to JSON responses with appropriate status codes. Debug information is logged but not exposed to clients for `Internal` errors.

**Sources:** [rust/utils/src/error.rs:173-188]()

---

## ApiError Type

For HTTP API endpoints, `ApiError` provides explicit status code control:

```rust
pub struct ApiError {
    pub err: anyhow::Error,
    pub status_code: StatusCode,
}
```

**Conversion rules:**
- `Error::Client` → `ApiError` with `400 Bad Request`
- Other `Error` variants → `ApiError` with `500 Internal Server Error`
- Existing `ApiError` in `anyhow::Error` → Preserved status code

This allows API handlers to return domain-specific status codes (404, 401, etc.) while maintaining the error chain.

**Sources:** [rust/utils/src/error.rs:376-441]()

---

## Retry Usage Patterns

### Basic Retry

```rust
use cocoindex_utils::retryable::{self, IsRetryable};

let result = retryable::run(
    || async {
        api_client.call().await
    },
    &retryable::RetryOptions::default()
).await?;
```

### Custom Configuration

```rust
let options = retryable::RetryOptions {
    retry_timeout: Some(Duration::from_secs(30 * 60)), // 30 minutes
    initial_backoff: Duration::from_millis(500),
    max_backoff: Duration::from_secs(30),
};

let result = retryable::run(|| async { ... }, &options).await?;
```

### Heavy Load Configuration

```rust
// For systems under heavy load (longer backoffs)
let result = retryable::run(
    || async { ... },
    &retryable::HEAVY_LOADED_OPTIONS
).await?;
```

**Sources:** [rust/utils/src/retryable.rs:135-182]()

---

## Type Safety Guarantees

The error system provides several type safety guarantees:

1. **No unchecked errors**: All operations return `Result<T, Error>` - no panic on error paths
2. **Backtrace preservation**: Context chaining never loses the original backtrace
3. **FFI boundary safety**: Python exceptions cannot escape as invalid Rust types
4. **Thread safety**: `SharedError` allows error sharing across threads (legacy support)

### Error Downcasting

For `HostError`, the original Python error can be recovered:

```rust
if let Error::HostLang(host_err) = error.without_contexts() {
    let any: &dyn Any = host_err.as_ref();
    if let Some(py_err) = any.downcast_ref::<PyErr>() {
        // Handle Python-specific error
    }
}
```

**Sources:** [rust/utils/src/error.rs:83-88]()

---

## Summary of Key Types

| Type | Purpose | Location |
|------|---------|----------|
| `Error` | Unified error enum | [rust/utils/src/error.rs:18-23]() |
| `ContextExt` | Trait for adding context | [rust/utils/src/error.rs:138-171]() |
| `ApiError` | HTTP status-aware errors | [rust/utils/src/error.rs:376-390]() |
| `IsRetryable` | Trait for retry decisions | [rust/utils/src/retryable.rs:7-9]() |
| `RetryOptions` | Retry configuration | [rust/utils/src/retryable.rs:113-127]() |
| `retryable::Error` | Retryable error wrapper | [rust/utils/src/retryable.rs:11-104]() |
| `HostError` | FFI boundary errors | [rust/utils/src/error.rs:15-16]() |
| `CancelOnDropPy` | Async cancellation | [rust/py_utils/src/future.rs:14-58]() |

---

# Page: Examples and Use Cases

# Examples and Use Cases

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [examples/amazon_s3_embedding/pyproject.toml](examples/amazon_s3_embedding/pyproject.toml)
- [examples/code_embedding/README.md](examples/code_embedding/README.md)
- [examples/code_embedding/pyproject.toml](examples/code_embedding/pyproject.toml)
- [examples/docs_to_knowledge_graph/README.md](examples/docs_to_knowledge_graph/README.md)
- [examples/docs_to_knowledge_graph/main.py](examples/docs_to_knowledge_graph/main.py)
- [examples/docs_to_knowledge_graph/pyproject.toml](examples/docs_to_knowledge_graph/pyproject.toml)
- [examples/fastapi_server_docker/requirements.txt](examples/fastapi_server_docker/requirements.txt)
- [examples/gdrive_text_embedding/README.md](examples/gdrive_text_embedding/README.md)
- [examples/gdrive_text_embedding/pyproject.toml](examples/gdrive_text_embedding/pyproject.toml)
- [examples/image_search/pyproject.toml](examples/image_search/pyproject.toml)
- [examples/manuals_llm_extraction/README.md](examples/manuals_llm_extraction/README.md)
- [examples/manuals_llm_extraction/pyproject.toml](examples/manuals_llm_extraction/pyproject.toml)
- [examples/paper_metadata/pyproject.toml](examples/paper_metadata/pyproject.toml)
- [examples/patient_intake_extraction/pyproject.toml](examples/patient_intake_extraction/pyproject.toml)
- [examples/pdf_embedding/README.md](examples/pdf_embedding/README.md)
- [examples/pdf_embedding/pyproject.toml](examples/pdf_embedding/pyproject.toml)
- [examples/product_recommendation/README.md](examples/product_recommendation/README.md)
- [examples/product_recommendation/main.py](examples/product_recommendation/main.py)
- [examples/product_recommendation/pyproject.toml](examples/product_recommendation/pyproject.toml)
- [examples/text_embedding/README.md](examples/text_embedding/README.md)
- [examples/text_embedding/pyproject.toml](examples/text_embedding/pyproject.toml)
- [examples/text_embedding_qdrant/README.md](examples/text_embedding_qdrant/README.md)
- [examples/text_embedding_qdrant/pyproject.toml](examples/text_embedding_qdrant/pyproject.toml)

</details>



This document catalogs the example implementations available in the CocoIndex repository, demonstrating capabilities across different domains and data types. Examples are located in the `examples/` directory and serve as reference implementations for common patterns.

For detailed implementation guides of specific example types, see the sub-pages: text/code embedding ([13.1](#13.1)), PDF processing ([13.2](#13.2)), knowledge graphs ([13.3](#13.3)), cloud storage ([13.4](#13.4)), image search ([13.5](#13.5)), production deployment ([13.6](#13.6)), and custom operations ([13.7](#13.7)).

For information about the flow definition system used in examples, see [Flow Definition System](#3). For operation decorators and registration, see [Operation System and Type Marshalling](#4).

## Example Structure and Organization

All examples follow a consistent structure:

| Component | Purpose | Location |
|-----------|---------|----------|
| `main.py` | Flow definition with `@flow_def` decorator | Root of example directory |
| `pyproject.toml` | Dependencies and package metadata | Root of example directory |
| `README.md` | Setup instructions and documentation | Root of example directory |
| `.env.example` | Template for required credentials | Root of example directory (if needed) |

Each example is a self-contained project with its own dependency specification in `pyproject.toml`, allowing independent installation via `pip install -e .`.

Sources: [examples/text_embedding/pyproject.toml:1-16](), [examples/code_embedding/pyproject.toml:1-15](), [examples/text_embedding/README.md:1-60]()

## Example Categories Overview

The following table categorizes all available examples by domain:

| Category | Example | Description | Key Technologies | Location |
|----------|---------|-------------|------------------|----------|
| **Text Processing** | Text Embedding | Semantic search over markdown files | `SentenceTransformerEmbed`, `SplitRecursively`, PostgreSQL | `examples/text_embedding/` |
| | Text Embedding (Qdrant) | Semantic search with Qdrant vector DB | `SentenceTransformerEmbed`, Qdrant | `examples/text_embedding_qdrant/` |
| **Code Processing** | Code Embedding | Syntax-aware code chunking with Tree-sitter | `SplitRecursively`, Tree-sitter, PostgreSQL | `examples/code_embedding/` |
| **Document Processing** | PDF Embedding | PDF to Markdown conversion and chunking | `marker-pdf`, `SplitRecursively`, `SentenceTransformerEmbed` | `examples/pdf_embedding/` |
| | LLM Extraction | Structured data extraction from PDFs | `ExtractByLlm`, Ollama/OpenAI | `examples/manuals_llm_extraction/` |
| | Paper Metadata | Academic paper processing with metadata | `pypdf`, `marker-pdf`, embeddings | `examples/paper_metadata/` |
| **Knowledge Graphs** | Docs to Knowledge Graph | Entity/relationship extraction and Neo4j export | `ExtractByLlm`, Neo4j, LLM APIs | `examples/docs_to_knowledge_graph/` |
| | Product Recommendation | Product taxonomy and complementary product KG | `ExtractByLlm`, Neo4j, Jinja2 | `examples/product_recommendation/` |
| **Cloud Sources** | Google Drive Embedding | Real-time sync from Google Drive folders | `GoogleDrive` source, live updates | `examples/gdrive_text_embedding/` |
| | Amazon S3 Embedding | S3 file ingestion and processing | `AmazonS3` source | `examples/amazon_s3_embedding/` |
| **Multimodal** | Image Search | CLIP and ColPali image embeddings | `ColPaliEmbedImage`, Qdrant, FastAPI | `examples/image_search/` |
| **Healthcare** | Patient Intake | Medical form extraction | `ExtractByLlm`, `markitdown` | `examples/patient_intake_extraction/` |
| **Production** | FastAPI Docker | Production deployment with REST API | FastAPI, Docker, uvicorn | `examples/fastapi_server_docker/` |

Sources: [examples/text_embedding/pyproject.toml:1-16](), [examples/code_embedding/pyproject.toml:1-15](), [examples/pdf_embedding/pyproject.toml:1-16](), [examples/manuals_llm_extraction/pyproject.toml:1-10](), [examples/docs_to_knowledge_graph/pyproject.toml:1-10](), [examples/product_recommendation/pyproject.toml:1-10](), [examples/gdrive_text_embedding/pyproject.toml:1-13](), [examples/amazon_s3_embedding/pyproject.toml:1-14](), [examples/image_search/pyproject.toml:1-18](), [examples/patient_intake_extraction/pyproject.toml:1-12](), [examples/paper_metadata/pyproject.toml:1-13](), [examples/fastapi_server_docker/requirements.txt:1-8]()

## Typical Example Flow Pattern

The following diagram illustrates the common pattern used across examples, mapping to actual code entities:

```mermaid
graph TB
    subgraph "Flow Definition (main.py)"
        flow_def["@flow_def decorator"]
        flow_builder["FlowBuilder parameter"]
        data_scope["DataScope parameter"]
        add_source["flow_builder.add_source()"]
        add_collector["data_scope.add_collector()"]
        transform["slice.transform()"]
        collect["collector.collect()"]
        export["collector.export()"]
    end
    
    subgraph "Source Configuration"
        LocalFile["cocoindex.sources.LocalFile"]
        GoogleDrive["cocoindex.sources.GoogleDrive"]
        AmazonS3["cocoindex.sources.AmazonS3"]
    end
    
    subgraph "Processing Functions"
        SplitRecursively["cocoindex.functions.SplitRecursively"]
        ExtractByLlm["cocoindex.functions.ExtractByLlm"]
        SentenceTransformerEmbed["cocoindex.functions.SentenceTransformerEmbed"]
        ParseJson["cocoindex.functions.ParseJson"]
    end
    
    subgraph "Target Configuration"
        Postgres["cocoindex.targets.Postgres"]
        Qdrant["cocoindex.targets.Qdrant"]
        Neo4j["cocoindex.targets.Neo4j"]
    end
    
    subgraph "CLI Execution"
        setup["cocoindex setup main"]
        update["cocoindex update main"]
        server["cocoindex server -ci main"]
    end
    
    flow_def --> flow_builder
    flow_def --> data_scope
    flow_builder --> add_source
    data_scope --> add_collector
    
    add_source --> LocalFile
    add_source --> GoogleDrive
    add_source --> AmazonS3
    
    add_source --> transform
    transform --> SplitRecursively
    transform --> ExtractByLlm
    transform --> SentenceTransformerEmbed
    transform --> ParseJson
    
    transform --> collect
    collect --> add_collector
    add_collector --> export
    
    export --> Postgres
    export --> Qdrant
    export --> Neo4j
    
    setup --> update
    update --> server
```

Sources: [examples/text_embedding/README.md:14-48](), [examples/code_embedding/README.md:20-56](), [examples/docs_to_knowledge_graph/main.py:44-192](), [examples/product_recommendation/main.py:91-204]()

## Example File Path Organization

The examples directory is organized by use case, with each example as a self-contained project:

```mermaid
graph TB
    examples["examples/"]
    
    text_emb["text_embedding/"]
    text_emb_q["text_embedding_qdrant/"]
    code_emb["code_embedding/"]
    
    pdf_emb["pdf_embedding/"]
    manual_llm["manuals_llm_extraction/"]
    paper_meta["paper_metadata/"]
    patient["patient_intake_extraction/"]
    
    docs_kg["docs_to_knowledge_graph/"]
    product_rec["product_recommendation/"]
    
    gdrive["gdrive_text_embedding/"]
    s3["amazon_s3_embedding/"]
    
    image_search["image_search/"]
    
    fastapi_docker["fastapi_server_docker/"]
    
    examples --> text_emb
    examples --> text_emb_q
    examples --> code_emb
    examples --> pdf_emb
    examples --> manual_llm
    examples --> paper_meta
    examples --> patient
    examples --> docs_kg
    examples --> product_rec
    examples --> gdrive
    examples --> s3
    examples --> image_search
    examples --> fastapi_docker
    
    text_emb --> text_emb_main["main.py"]
    text_emb --> text_emb_pyproject["pyproject.toml"]
    text_emb --> text_emb_readme["README.md"]
    
    code_emb --> code_emb_main["main.py"]
    code_emb --> code_emb_pyproject["pyproject.toml"]
    code_emb --> code_emb_readme["README.md"]
    
    docs_kg --> docs_kg_main["main.py"]
    docs_kg --> docs_kg_pyproject["pyproject.toml"]
    docs_kg --> docs_kg_readme["README.md"]
```

Sources: [examples/text_embedding/pyproject.toml:1-16](), [examples/code_embedding/pyproject.toml:1-15](), [examples/pdf_embedding/pyproject.toml:1-16](), [examples/docs_to_knowledge_graph/pyproject.toml:1-10](), [examples/product_recommendation/pyproject.toml:1-10](), [examples/gdrive_text_embedding/pyproject.toml:1-13](), [examples/image_search/pyproject.toml:1-18]()

## Common Patterns Across Examples

### Flow Definition Pattern

All examples follow the same flow definition pattern:

```python
@cocoindex.flow_def(name="FlowName")
def flow_function(
    flow_builder: cocoindex.FlowBuilder, 
    data_scope: cocoindex.DataScope
) -> None:
    # 1. Add source
    data_scope["source_name"] = flow_builder.add_source(...)
    
    # 2. Create collectors
    collector = data_scope.add_collector()
    
    # 3. Process data
    with data_scope["source_name"].row() as row:
        row["field"] = row["input"].transform(...)
        collector.collect(...)
    
    # 4. Export to target
    collector.export("target_name", target_spec, primary_key_fields=[...])
```

Sources: [examples/docs_to_knowledge_graph/main.py:44-192](), [examples/product_recommendation/main.py:91-204]()

### Dependency Installation Pattern

Examples use `cocoindex` with optional feature flags:

| Feature Flag | Purpose | Examples Using |
|--------------|---------|----------------|
| `[embeddings]` | SentenceTransformer embedding support | text_embedding, code_embedding, pdf_embedding |
| `[colpali]` | ColPali multimodal embeddings | image_search |
| (no flags) | Core CocoIndex functionality | manuals_llm_extraction, docs_to_knowledge_graph |

Sources: [examples/text_embedding/pyproject.toml:7](), [examples/code_embedding/pyproject.toml:7](), [examples/pdf_embedding/pyproject.toml:7](), [examples/image_search/pyproject.toml:7](), [examples/manuals_llm_extraction/pyproject.toml:6]()

### CLI Execution Pattern

All examples follow standard CLI workflow:

```bash
# 1. Install dependencies
pip install -e .

# 2. Setup backend (first run only)
cocoindex setup main

# 3. Update/run flow
cocoindex update main

# 4. Optional: Run with live updates
cocoindex update -L main

# 5. Optional: Start server with CocoInsight
cocoindex server -ci main
```

Sources: [examples/text_embedding/README.md:32-57](), [examples/code_embedding/README.md:40-68](), [examples/pdf_embedding/README.md:30-57]()

### Authentication Pattern

Examples requiring credentials use the auth registry:

```python
# Add auth entry for reuse
conn_spec = cocoindex.add_auth_entry(
    "ServiceName",
    cocoindex.targets.ServiceConnection(
        uri="connection_string",
        user="username",
        password="password",
    ),
)

# Reference in target specs
collector.export(
    "target_name",
    cocoindex.targets.Service(connection=conn_spec, ...)
)
```

Sources: [examples/docs_to_knowledge_graph/main.py:9-16](), [examples/product_recommendation/main.py:10-17]()

## CocoInsight Integration

All examples support CocoInsight for visualization and debugging. The standard pattern is:

```bash
cocoindex server -ci main
```

Then navigate to `https://cocoindex.io/cocoinsight` to connect to the local server. CocoInsight provides:

- Flow visualization with data lineage
- Data preview at each pipeline stage
- Incremental processing statistics
- Zero data retention (connects to local server only)

Sources: [examples/text_embedding/README.md:50-59](), [examples/code_embedding/README.md:58-69](), [examples/pdf_embedding/README.md:49-57](), [examples/docs_to_knowledge_graph/README.md:54-65](), [examples/gdrive_text_embedding/README.md:67-83]()

## Running Examples

### Prerequisites

Most examples require:
- PostgreSQL with pgvector extension (for internal state management)
- Python 3.10 or higher
- Optional: LLM API keys (OpenAI, Gemini) or Ollama for local LLM execution

Specific examples may require:
- Neo4j or Kuzu for knowledge graph examples
- Qdrant for vector database examples
- Cloud service credentials (Google Drive, AWS S3)
- Docker for production deployment examples

Sources: [examples/text_embedding/README.md:26-28](), [examples/code_embedding/README.md:34-36](), [examples/docs_to_knowledge_graph/README.md:15-19](), [examples/product_recommendation/README.md:9-13]()

### Environment Configuration

Examples using credentials provide `.env.example` files. Copy and configure:

```bash
cp .env.example .env
# Edit .env with your credentials
```

Configuration is loaded via `python-dotenv` in example code. CocoIndex automatically reads environment variables prefixed with `COCOINDEX_`.

Sources: [examples/gdrive_text_embedding/README.md:35-41]()

## Example Data Sources

Examples demonstrate various data source types:

| Source Type | Example Usage | Configuration Class |
|-------------|---------------|---------------------|
| Local Files | Text, code, PDF files | `cocoindex.sources.LocalFile` |
| Google Drive | Cloud documents | `cocoindex.sources.GoogleDrive` |
| Amazon S3 | Object storage | `cocoindex.sources.AmazonS3` |
| JSON Data | Structured product data | `cocoindex.sources.LocalFile` + `ParseJson` |

Sources: [examples/docs_to_knowledge_graph/main.py:51-56](), [examples/product_recommendation/main.py:98-100]()

## Target Systems Demonstrated

Examples export to various target systems:

| Target Type | Examples Using | Purpose |
|-------------|----------------|---------|
| PostgreSQL + pgvector | text_embedding, code_embedding, pdf_embedding, gdrive_text_embedding | Vector similarity search |
| Qdrant | text_embedding_qdrant, image_search | Dedicated vector database |
| Neo4j | docs_to_knowledge_graph, product_recommendation | Property graph storage |
| Kuzu | Alternative KG examples | Embedded graph database |

Sources: [examples/text_embedding/pyproject.toml:9-10](), [examples/text_embedding_qdrant/pyproject.toml:9](), [examples/docs_to_knowledge_graph/main.py:18-22](), [examples/image_search/pyproject.toml:12]()

---

# Page: Text and Code Embedding Pipelines

# Text and Code Embedding Pipelines

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [examples/amazon_s3_embedding/README.md](examples/amazon_s3_embedding/README.md)
- [examples/amazon_s3_embedding/main.py](examples/amazon_s3_embedding/main.py)
- [examples/azure_blob_embedding/README.md](examples/azure_blob_embedding/README.md)
- [examples/azure_blob_embedding/main.py](examples/azure_blob_embedding/main.py)
- [examples/code_embedding/README.md](examples/code_embedding/README.md)
- [examples/code_embedding/main.py](examples/code_embedding/main.py)
- [examples/docs_to_knowledge_graph/README.md](examples/docs_to_knowledge_graph/README.md)
- [examples/docs_to_knowledge_graph/main.py](examples/docs_to_knowledge_graph/main.py)
- [examples/gdrive_text_embedding/README.md](examples/gdrive_text_embedding/README.md)
- [examples/gdrive_text_embedding/main.py](examples/gdrive_text_embedding/main.py)
- [examples/manuals_llm_extraction/README.md](examples/manuals_llm_extraction/README.md)
- [examples/patient_intake_extraction/main.py](examples/patient_intake_extraction/main.py)
- [examples/pdf_embedding/README.md](examples/pdf_embedding/README.md)
- [examples/product_recommendation/README.md](examples/product_recommendation/README.md)
- [examples/product_recommendation/main.py](examples/product_recommendation/main.py)
- [examples/text_embedding/README.md](examples/text_embedding/README.md)
- [examples/text_embedding/main.py](examples/text_embedding/main.py)
- [examples/text_embedding_qdrant/README.md](examples/text_embedding_qdrant/README.md)
- [examples/text_embedding_qdrant/main.py](examples/text_embedding_qdrant/main.py)

</details>



This document covers the example applications that demonstrate text and code embedding pipelines using CocoIndex. These examples illustrate the common pattern for building semantic search indexes from various sources (local files, Google Drive, Amazon S3) and storing embeddings in vector databases (Postgres+pgvector, Qdrant).

For document processing with PDF and multi-format support, see [PDF and Document Processing](#13.2). For LLM-based extraction and knowledge graph construction, see [Knowledge Graph and Extraction Pipelines](#13.3).

## Overview of Embedding Examples

CocoIndex provides five primary embedding examples that follow a consistent architectural pattern:

| Example | Source Type | Chunking Strategy | Target Database | Live Updates |
|---------|-------------|-------------------|-----------------|--------------|
| `text_embedding` | LocalFile (markdown) | `SplitRecursively` (markdown) | Postgres+pgvector | Supported via `refresh_interval` |
| `code_embedding` | LocalFile (code files) | `SplitRecursively` (tree-sitter) | Postgres+pgvector | Not configured |
| `gdrive_text_embedding` | GoogleDrive | `SplitRecursively` (markdown) | Postgres+pgvector | Real-time via Drive API |
| `amazon_s3_embedding` | AmazonS3 | `SplitRecursively` (markdown) | Postgres+pgvector | Real-time via SQS |
| `text_embedding_qdrant` | LocalFile (markdown) | `SplitRecursively` (markdown) | Qdrant | Not configured |

Sources: [examples/text_embedding/main.py:1-146](), [examples/code_embedding/main.py:1-163](), [examples/gdrive_text_embedding/main.py:1-119](), [examples/amazon_s3_embedding/main.py:1-124](), [examples/text_embedding_qdrant/main.py:1-125]()

## Common Pipeline Architecture

All embedding examples follow a three-component architecture: a reusable transform flow for embeddings, a main flow definition for indexing, and a query handler for semantic search.

### Pipeline Flow Diagram

```mermaid
graph TB
    Source["Source Connector<br/>(LocalFile/GoogleDrive/AmazonS3)"]
    DetectLang["DetectProgrammingLanguage<br/>(code only)"]
    Split["SplitRecursively<br/>(chunking)"]
    Transform["text_to_embedding<br/>@transform_flow"]
    Embed["SentenceTransformerEmbed<br/>or EmbedText"]
    Collect["add_collector<br/>(collect results)"]
    Export["export<br/>(Postgres/Qdrant)"]
    Query["search<br/>@query_handler"]
    
    Source --> DetectLang
    Source --> Split
    DetectLang --> Split
    Split --> Transform
    Transform --> Embed
    Embed --> Collect
    Collect --> Export
    Export --> Query
    
    Query -.->|"reuses"| Transform
```

**Pipeline Architecture**

Sources: [examples/text_embedding/main.py:33-74](), [examples/code_embedding/main.py:32-81]()

### Transform Flow Pattern

Each example defines a reusable `text_to_embedding` transform flow decorated with `@cocoindex.transform_flow()`. This pattern enables the same embedding logic to be used for both indexing and querying.

```mermaid
graph LR
    IndexPipeline["Flow Definition<br/>@flow_def"]
    QueryHandler["Query Handler<br/>@query_handler or search()"]
    TransformFlow["text_to_embedding<br/>@transform_flow"]
    
    IndexPipeline -->|"calls"| TransformFlow
    QueryHandler -->|"text_to_embedding.eval()"| TransformFlow
```

**Transform Flow Reuse Pattern**

The transform flow signature is consistent across examples:

```python
@cocoindex.transform_flow()
def text_to_embedding(
    text: cocoindex.DataSlice[str],
) -> cocoindex.DataSlice[NDArray[np.float32]]:  # or list[float]
```

Sources: [examples/text_embedding/main.py:12-30](), [examples/code_embedding/main.py:11-29](), [examples/gdrive_text_embedding/main.py:8-20]()

## Text Embedding Pipeline

The text embedding example at [examples/text_embedding/main.py]() demonstrates the canonical pattern for markdown file indexing.

### Source Configuration

The flow adds a `LocalFile` source that monitors markdown files with a 5-second refresh interval for automatic updates:

[examples/text_embedding/main.py:40-43]()

The `refresh_interval` parameter enables live updates. When combined with `cocoindex update -L`, the system continuously monitors source changes. For more on live updates, see [Live Updates and Change Capture](#9.1).

### Chunking and Embedding

The pipeline processes each document through language-aware chunking:

[examples/text_embedding/main.py:47-62]()

The nested `row()` context managers enable hierarchical processing where each document is split into chunks, and each chunk is embedded independently. For details on `DataScope` and `row()`, see [DataScope and DataSlice](#3.1).

### Primary Key and Vector Index Configuration

The export operation specifies primary keys for deduplication and vector indexes for similarity search:

[examples/text_embedding/main.py:64-74]()

The composite primary key `["filename", "location"]` ensures unique identification of each chunk, enabling incremental updates when source files change. For vector index configuration options, see [Vector Databases: Postgres and Qdrant](#8.1).

Sources: [examples/text_embedding/main.py:33-74](), [examples/text_embedding/README.md:1-60]()

### Query Implementation

The query handler demonstrates the standard pattern for semantic search using the pgvector distance operator:

[examples/text_embedding/main.py:88-123]()

The `@query_handler` decorator with `result_fields` configuration enables CocoInsight visualization. The `text_to_embedding.eval(query)` call reuses the indexing transform for query embedding.

Sources: [examples/text_embedding/main.py:88-123]()

## Code Embedding Pipeline with Tree-sitter

The code embedding example at [examples/code_embedding/main.py]() demonstrates syntax-aware chunking using Tree-sitter parsers for 40+ programming languages.

### Code-Specific Source Configuration

The source configuration targets code files across the repository:

[examples/code_embedding/main.py:39-45]()

The patterns include Python, Rust, TOML, and Markdown files while excluding hidden directories, build artifacts, and `node_modules`.

### Language Detection and Syntax-Aware Chunking

The pipeline detects programming language before chunking to enable Tree-sitter parsing:

[examples/code_embedding/main.py:48-68]()

The `DetectProgrammingLanguage()` function analyzes file extensions and content to determine the language, which is then passed to `SplitRecursively()` for syntax-aware chunking. For supported languages, see [Text Processing and Chunking](#6.1).

### Chunk Location Tracking

The chunking operation provides `start` and `end` location information:

[examples/code_embedding/main.py:61-68]()

The `location` field contains hierarchical path information (e.g., "chunk_0/chunk_1"), while `start` and `end` contain line and character positions:

```python
{
    "start": {"line": 10, "character": 0},
    "end": {"line": 25, "character": 45}
}
```

This enables precise source location references in search results.

Sources: [examples/code_embedding/main.py:32-81](), [examples/code_embedding/README.md:1-70]()

### Code Search Results

The search function formats results with line number information:

[examples/code_embedding/main.py:100-134]()

The `QueryOutput` structure with `query_info` enables CocoInsight to visualize query embeddings and similarity metrics. For query handler patterns, see [Query Handlers and Search APIs](#11.2).

Sources: [examples/code_embedding/main.py:94-134]()

## Source Variations

The embedding pipeline pattern extends to different source connectors with minimal code changes.

### Google Drive Source with Real-Time Updates

The `gdrive_text_embedding` example demonstrates continuous synchronization with Google Drive:

[examples/gdrive_text_embedding/main.py:30-40]()

The `recent_changes_poll_interval` parameter configures how frequently the connector polls the Drive API for changes. Combined with `refresh_interval`, this enables near-real-time indexing of Google Drive content. For Google Drive setup, see [Built-in Data Sources](#5).

Sources: [examples/gdrive_text_embedding/main.py:23-71](), [examples/gdrive_text_embedding/README.md:1-84]()

### Amazon S3 Source with Event-Driven Updates

The S3 embedding example supports event-driven updates via SQS:

[examples/amazon_s3_embedding/main.py:30-42]()

When `sqs_queue_url` is configured, the connector receives S3 event notifications for object creation/deletion, enabling immediate index updates without polling. For S3 configuration details, see [Built-in Data Sources](#5).

### Live Update Context Manager

The S3 example uses `FlowLiveUpdater` for continuous monitoring:

[examples/amazon_s3_embedding/main.py:103-118]()

The context manager runs a background thread that processes source updates while the main thread handles queries. For live update mechanics, see [Live Updates and Change Capture](#9.1).

Sources: [examples/amazon_s3_embedding/main.py:23-124]()

## Target Database Variations

### Qdrant Vector Database

The `text_embedding_qdrant` example demonstrates using Qdrant instead of Postgres:

[examples/text_embedding_qdrant/main.py:26-62]()

Key differences from Postgres:
- Uses `GeneratedField.UUID` for primary key instead of composite keys
- Vector field name must match Qdrant collection configuration
- No vector index specification needed (handled by Qdrant)

### Qdrant Query Implementation

Querying Qdrant uses the official client library:

[examples/text_embedding_qdrant/main.py:70-101]()

The `query_vector` parameter is a tuple `("text_embedding", embedding)` where the first element is the named vector field. For Qdrant configuration details, see [Vector Databases: Postgres and Qdrant](#8.1).

Sources: [examples/text_embedding_qdrant/main.py:26-101](), [examples/text_embedding_qdrant/README.md:1-68]()

## Embedding Model Configuration

### Local SentenceTransformer Models

Most examples use local SentenceTransformer models:

[examples/text_embedding/main.py:26-29]()

The `SentenceTransformerEmbed` function downloads and caches models locally, enabling offline operation after initial download. For available models, see [Embedding and AI Functions](#6.2).

### Remote API-Based Embeddings

Examples include commented alternatives for remote APIs:

[examples/text_embedding/main.py:19-25]()

Remote embeddings support OpenAI, Gemini, Voyage, and other providers via `EmbedText()`. For LLM configuration, see [LLM Integration](#7).

Sources: [examples/text_embedding/main.py:12-30](), [examples/code_embedding/main.py:11-29]()

## Common Execution Patterns

### Setup and Update Flow

All examples follow this execution sequence:

```mermaid
stateDiagram-v2
    [*] --> Setup: cocoindex setup main
    Setup --> Update: cocoindex update main
    Update --> Query: python main.py
    Query --> Update: cocoindex update main
    
    note right of Setup
        Creates target tables/collections
        One-time operation
    end note
    
    note right of Update
        Processes all sources
        Incremental: only changed data
    end note
    
    note right of Query
        Interactive query loop
        Reuses embedding transform
    end note
```

**Execution Lifecycle**

Sources: [examples/text_embedding/README.md:38-48](), [examples/code_embedding/README.md:38-56]()

### Main Function Pattern

The standard main function pattern runs an interactive query loop:

[examples/text_embedding/main.py:126-140]()

The initialization sequence:
1. `load_dotenv()` - loads environment variables from `.env`
2. `cocoindex.init()` - initializes settings from environment
3. Query loop - runs search on already-indexed data

For one-time updates before querying, examples may call:

[examples/code_embedding/main.py:137-140]()

Sources: [examples/text_embedding/main.py:126-145](), [examples/code_embedding/main.py:137-162]()

## CocoInsight Integration

All examples support CocoInsight visualization for pipeline debugging:

```bash
cocoindex server -ci main
```

The `-ci` flag starts a server that CocoInsight connects to at `https://cocoindex.io/cocoinsight`. For live monitoring with continuous updates:

```bash
cocoindex server -ci main -L
```

The `-L` flag combines CocoInsight visualization with live update monitoring. For CocoInsight details, see [CocoInsight: Visualization and Debugging](#11.3).

Sources: [examples/text_embedding/README.md:50-60](), [examples/code_embedding/README.md:58-68](), [examples/gdrive_text_embedding/README.md:67-84]()

## Example Directory Structure

Each example follows a consistent directory structure:

```
examples/<example_name>/
├── main.py              # Flow definition and query handler
├── README.md            # Setup instructions and description
├── pyproject.toml       # Dependencies
├── .env.example         # Environment variable template (if needed)
└── <source_files>/      # Sample data (e.g., markdown_files/, pdf_files/)
```

The `pyproject.toml` specifies example-specific dependencies beyond core CocoIndex:

- `text_embedding`: `psycopg[binary]`, `psycopg-pool`, `pgvector`
- `code_embedding`: Same as text_embedding
- `gdrive_text_embedding`: Adds Google Drive dependencies
- `amazon_s3_embedding`: Adds boto3 for S3
- `text_embedding_qdrant`: Adds `qdrant-client`

Sources: [examples/text_embedding/README.md:32-48](), [examples/code_embedding/README.md:40-56]()

---

# Page: PDF and Document Processing

# PDF and Document Processing

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [examples/code_embedding/README.md](examples/code_embedding/README.md)
- [examples/custom_output_files/main.py](examples/custom_output_files/main.py)
- [examples/docs_to_knowledge_graph/README.md](examples/docs_to_knowledge_graph/README.md)
- [examples/docs_to_knowledge_graph/main.py](examples/docs_to_knowledge_graph/main.py)
- [examples/face_recognition/main.py](examples/face_recognition/main.py)
- [examples/fastapi_server_docker/main.py](examples/fastapi_server_docker/main.py)
- [examples/gdrive_text_embedding/README.md](examples/gdrive_text_embedding/README.md)
- [examples/image_search/colpali_main.py](examples/image_search/colpali_main.py)
- [examples/image_search/main.py](examples/image_search/main.py)
- [examples/manuals_llm_extraction/README.md](examples/manuals_llm_extraction/README.md)
- [examples/manuals_llm_extraction/main.py](examples/manuals_llm_extraction/main.py)
- [examples/paper_metadata/main.py](examples/paper_metadata/main.py)
- [examples/pdf_elements_embedding/main.py](examples/pdf_elements_embedding/main.py)
- [examples/pdf_embedding/README.md](examples/pdf_embedding/README.md)
- [examples/pdf_embedding/main.py](examples/pdf_embedding/main.py)
- [examples/postgres_source/main.py](examples/postgres_source/main.py)
- [examples/postgres_source/pyproject.toml](examples/postgres_source/pyproject.toml)
- [examples/product_recommendation/README.md](examples/product_recommendation/README.md)
- [examples/product_recommendation/main.py](examples/product_recommendation/main.py)
- [examples/text_embedding/README.md](examples/text_embedding/README.md)
- [examples/text_embedding_qdrant/README.md](examples/text_embedding_qdrant/README.md)

</details>



This document covers PDF and multi-format document processing in CocoIndex, including text extraction, visual document embedding, and structured data extraction workflows. CocoIndex provides two primary approaches for document processing: traditional text extraction with OCR/parsers followed by text embedding, and direct visual embedding using ColPali for layout-aware document search.

For text-only processing workflows without PDF conversion, see [Text and Code Embedding Pipelines](#13.1). For LLM-based extraction patterns applicable to documents, see [Knowledge Graph and Extraction Pipelines](#13.3). For details on specific embedding and chunking functions, see [Embedding and AI Functions](#6.2) and [Text Processing and Chunking](#6.1).

---

## PDF Processing Approaches

CocoIndex supports two distinct approaches for processing PDF and document files, each optimized for different use cases.

### Text Extraction Approach

The text extraction approach converts documents to markdown or plain text, then applies standard text processing workflows. This approach:

- Extracts textual content from PDFs using conversion libraries
- Applies text chunking strategies (e.g., `SplitRecursively`)
- Generates text embeddings for semantic search
- Suitable for text-heavy documents where layout is less critical

**Primary Example**: [examples/pdf_embedding/main.py:1-150]()

### Visual Embedding Approach

The visual embedding approach treats documents as images and generates embeddings that preserve layout, tables, and visual structure. This approach:

- Converts PDF pages to images or processes image files directly
- Uses ColPali model for vision-language understanding
- Enables natural language queries against visual content without OCR
- Ideal for documents with complex layouts, tables, charts, or mixed content

**Primary Example**: [examples/multi_format_indexing/README.md:1-88]()

```mermaid
graph TB
    subgraph "Text Extraction Approach"
        PDF1["PDF Files"]
        Convert1["PdfToMarkdown or<br/>marker library"]
        Markdown["Markdown Text"]
        Chunk1["SplitRecursively"]
        Embed1["SentenceTransformerEmbed<br/>or EmbedText"]
        Store1["Postgres + pgvector"]
        
        PDF1 --> Convert1
        Convert1 --> Markdown
        Markdown --> Chunk1
        Chunk1 --> Embed1
        Embed1 --> Store1
    end
    
    subgraph "Visual Embedding Approach"
        PDF2["PDF Files / Images"]
        Convert2["ConvertPdfPageToImage<br/>or direct images"]
        Images["Page Images"]
        Embed2["ColPaliEmbed"]
        Store2["Qdrant Vector DB"]
        
        PDF2 --> Convert2
        Convert2 --> Images
        Images --> Embed2
        Embed2 --> Store2
    end
    
    Query1["Text Query"]
    Query2["Text Query"]
    
    Query1 --> Embed1
    Query2 --> Embed2
    
    Store1 --> Results1["Text-based Results"]
    Store2 --> Results2["Visual Results"]
```

**Diagram: Document Processing Approaches**

Sources: [examples/pdf_embedding/main.py:1-150](), [examples/multi_format_indexing/README.md:1-88]()

| Approach | Best For | Libraries Used | Output Storage |
|----------|----------|----------------|----------------|
| Text Extraction | Text-heavy documents, legal docs, manuals | marker, MarkItDown | Postgres + pgvector |
| Visual Embedding | Forms, tables, charts, mixed layouts | ColPali, pdf2image | Qdrant, Postgres |

---

## PDF-to-Text Pipeline Example

The `pdf_embedding` example demonstrates a complete pipeline for converting PDFs to searchable text embeddings using the marker library.

### Custom PDF Conversion Function

The example implements a custom function `PdfToMarkdown` using CocoIndex's operation system:

```mermaid
graph LR
    FunctionSpec["PdfToMarkdown<br/>cocoindex.op.FunctionSpec"]
    ExecutorClass["PdfToMarkdownExecutor<br/>@executor_class decorator"]
    Converter["PdfConverter<br/>marker.converters.pdf"]
    
    FunctionSpec -->|"defines spec"| ExecutorClass
    ExecutorClass -->|"uses"| Converter
    ExecutorClass -->|"gpu=True<br/>cache=True"| OpArgs["OpArgs Config"]
```

**Diagram: Custom PDF Conversion Function Structure**

The function specification and executor are defined in [examples/pdf_embedding/main.py:14-36]():

- `PdfToMarkdown` extends `cocoindex.op.FunctionSpec` to define the operation
- `PdfToMarkdownExecutor` implements the conversion logic
- Decorator `@cocoindex.op.executor_class(gpu=True, cache=True)` configures execution
- The `prepare()` method initializes the marker `PdfConverter` once
- The `__call__()` method processes individual PDF byte content

Sources: [examples/pdf_embedding/main.py:14-36]()

### Complete Flow Definition

```mermaid
graph TB
    Source["LocalFile Source<br/>path='pdf_files'<br/>binary=True"]
    
    Transform1["PdfToMarkdown()<br/>Custom Function"]
    Transform2["SplitRecursively<br/>language='markdown'<br/>chunk_size=2000"]
    Transform3["text_to_embedding<br/>SentenceTransformerEmbed"]
    
    Collector["pdf_embeddings<br/>Collector"]
    Export["Postgres Target<br/>table: pdf_embeddings<br/>VectorIndexDef"]
    
    Source -->|"content (bytes)"| Transform1
    Transform1 -->|"markdown (str)"| Transform2
    Transform2 -->|"chunks"| Transform3
    Transform3 -->|"embedding"| Collector
    Collector --> Export
```

**Diagram: PDF Embedding Flow Pipeline**

The flow definition in [examples/pdf_embedding/main.py:54-96]() follows this structure:

1. **Source**: `LocalFile` with `binary=True` reads PDF files as bytes
2. **PDF Conversion**: `doc["markdown"] = doc["content"].transform(PdfToMarkdown())`
3. **Chunking**: `doc["chunks"] = doc["markdown"].transform(cocoindex.functions.SplitRecursively())`
4. **Embedding**: `chunk["embedding"] = text_to_embedding(chunk["text"])`
5. **Export**: Postgres target with vector index on embedding field

Sources: [examples/pdf_embedding/main.py:54-96]()

### Query Implementation

The search function reuses the embedding transform flow:

```python
query_vector = text_to_embedding.eval(query)
```

This ensures query embeddings use the same model and configuration as indexed documents. The SQL query performs vector similarity search using pgvector's `<=>` operator:

```sql
SELECT filename, text, embedding <=> %s::vector AS distance
FROM {table_name} ORDER BY distance LIMIT %s
```

Sources: [examples/pdf_embedding/main.py:99-119]()

---

## Visual Document Indexing with ColPali

The `multi_format_indexing` example demonstrates visual document search using ColPali, which generates embeddings directly from document images without text extraction.

### ColPali Processing Functions

CocoIndex provides built-in ColPali functions for visual document processing:

| Function | Purpose | Input | Output |
|----------|---------|-------|--------|
| `ConvertPdfPageToImage` | Converts PDF pages to images | PDF bytes | Image per page |
| `ColPaliEmbed` | Generates visual embeddings | Images | Multi-vector embeddings |
| `ColPaliQueryEmbed` | Embeds text queries | Text string | Query vector |

These functions are documented in [Built-in Processing Functions](#6) and [Embedding and AI Functions](#6.2).

### Multi-Format Flow Architecture

```mermaid
graph TB
    subgraph "Source Ingestion"
        LocalFiles["LocalFile Source<br/>path='source_files'"]
        PDFs["PDF Files<br/>*.pdf"]
        Images["Image Files<br/>*.png, *.jpg"]
    end
    
    subgraph "Format Detection"
        IsPdf{"Is PDF?"}
    end
    
    subgraph "PDF Processing"
        PdfConvert["ConvertPdfPageToImage<br/>resolution=300 DPI"]
        PdfPages["Page Images"]
    end
    
    subgraph "Image Processing"
        DirectImage["Direct Image"]
    end
    
    subgraph "Visual Embedding"
        ColPali["ColPaliEmbed<br/>model='vidore/colpali-v1.2'"]
        Embeddings["Multi-vector Embeddings"]
    end
    
    subgraph "Storage"
        Qdrant["Qdrant Collection<br/>multi-vector storage"]
    end
    
    LocalFiles --> IsPdf
    IsPdf -->|"Yes"| PdfConvert
    IsPdf -->|"No"| DirectImage
    
    PdfConvert --> PdfPages
    PdfPages --> ColPali
    DirectImage --> ColPali
    ColPali --> Embeddings
    Embeddings --> Qdrant
```

**Diagram: Multi-Format Visual Document Pipeline**

The pipeline handles both PDF and image files with conditional processing based on file extension. PDF files are converted to high-resolution images (300 DPI), while image files are processed directly. All images flow through the same ColPali embedding function.

Sources: [examples/multi_format_indexing/README.md:1-88]()

### ColPali Query Pattern

Visual document search queries follow this pattern:

1. User provides a natural language query string
2. `ColPaliQueryEmbed` converts the text to a visual query embedding
3. Qdrant performs multi-vector similarity search
4. Results include the visual content and relevance scores

This approach enables queries like "find documents with pie charts" or "show invoices with amounts over $10,000" without requiring OCR or text extraction.

Sources: [examples/multi_format_indexing/README.md:20-22]()

---

## Multi-Format Document Processing with MarkItDown

For scenarios requiring structured data extraction from diverse document formats (PDF, DOCX, PPTX, etc.), CocoIndex integrates with Microsoft's MarkItDown library.

### Supported Formats

The `patient_intake_extraction` example demonstrates processing multiple document formats:

- **PDF**: Medical forms, patient intake documents
- **DOCX**: Word documents with structured forms
- **PPTX**: Presentation slides with tabular data
- **XLSX**: Spreadsheets with patient information

All formats are converted to markdown for unified processing.

Sources: [examples/patient_intake_extraction/README.md:1-57]()

### Integration Pattern

```mermaid
graph TB
    subgraph "Document Sources"
        LocalDocs["LocalFile<br/>path='patient_forms'<br/>binary=True"]
        Formats["*.pdf, *.docx,<br/>*.pptx, *.xlsx"]
    end
    
    subgraph "Conversion Layer"
        MarkItDown["MarkItDown Library<br/>External Tool"]
        CustomFunc["Custom Function<br/>ConvertToMarkdown"]
    end
    
    subgraph "Processing Pipeline"
        Markdown["Unified Markdown"]
        Extract["ExtractByLlm<br/>Structured Extraction"]
        Schema["Pydantic Schema<br/>PatientInfo"]
    end
    
    subgraph "Storage"
        Postgres["Postgres Table<br/>patient_data"]
    end
    
    LocalDocs --> Formats
    Formats --> MarkItDown
    MarkItDown --> CustomFunc
    CustomFunc --> Markdown
    Markdown --> Extract
    Extract --> Schema
    Schema --> Postgres
```

**Diagram: Multi-Format Document Extraction Pipeline**

The integration requires:

1. Installing MarkItDown with `pip install 'markitdown[all]'`
2. Implementing a custom function that wraps MarkItDown's conversion
3. Using `LocalFile` source with `binary=True` to read raw file content
4. Applying LLM extraction with Pydantic schemas for structured output

Sources: [examples/patient_intake_extraction/README.md:1-57]()

### Manual PDF Processing Example

The `manuals_llm_extraction` example shows a similar pattern for processing technical documentation PDFs:

```mermaid
graph LR
    PDFs["PDF Manuals<br/>Python Docs"]
    Convert["PDF to Markdown<br/>marker or MarkItDown"]
    Extract["ExtractByLlm<br/>module_info schema"]
    Custom["Custom Function<br/>extract_summary"]
    Store["Postgres Table<br/>modules_info"]
    
    PDFs --> Convert
    Convert --> Extract
    Extract --> Custom
    Custom --> Store
```

**Diagram: Technical Documentation Extraction Flow**

This pattern combines:
- PDF-to-markdown conversion
- LLM-based structured extraction with custom schemas
- Post-processing with custom functions for derived fields
- Storage in Postgres for querying

Sources: [examples/manuals_llm_extraction/README.md:1-68]()

---

## Custom PDF Processing Functions

### Implementing a PDF Converter

To create a custom PDF processing function, implement the function specification and executor pattern:

```mermaid
classDiagram
    class FunctionSpec {
        <<interface>>
    }
    
    class PdfToMarkdown {
        +spec definition
    }
    
    class ExecutorClass {
        +prepare()
        +__call__(content: bytes) str
    }
    
    class PdfToMarkdownExecutor {
        -PdfConverter _converter
        +prepare() void
        +__call__(content: bytes) str
    }
    
    FunctionSpec <|-- PdfToMarkdown
    PdfToMarkdown --> ExecutorClass
    ExecutorClass <|-- PdfToMarkdownExecutor
```

**Diagram: Custom Function Implementation Pattern**

Key implementation details from [examples/pdf_embedding/main.py:14-36]():

1. **Spec Class**: Inherits from `cocoindex.op.FunctionSpec`
2. **Executor Decorator**: `@cocoindex.op.executor_class(gpu=True, cache=True, behavior_version=1)`
3. **Prepare Method**: Initializes heavy resources (models, converters) once
4. **Call Method**: Processes individual inputs efficiently

The `gpu=True` parameter requests GPU allocation for the executor. The `cache=True` parameter enables result caching based on input checksums. The `behavior_version` parameter ensures cache invalidation when implementation changes.

Sources: [examples/pdf_embedding/main.py:14-36]()

### PDF Processing Libraries

Common libraries for PDF conversion:

| Library | Purpose | Installation | Use Case |
|---------|---------|--------------|----------|
| **marker** | PDF to markdown with layout preservation | `pip install marker-pdf` | High-quality conversion, GPU-accelerated |
| **MarkItDown** | Multi-format to markdown | `pip install markitdown[all]` | Unified processing of PDF/DOCX/PPTX |
| **pdf2image** | PDF to images | `pip install pdf2image` + poppler | Visual embedding with ColPali |
| **PyPDF2** | Basic PDF text extraction | `pip install PyPDF2` | Simple text-only PDFs |

### Tempfile Pattern for Binary Processing

When processing binary content (PDFs, images), use temporary files to interface with external libraries:

```python
with tempfile.NamedTemporaryFile(delete=True, suffix=".pdf") as temp_file:
    temp_file.write(content)
    temp_file.flush()
    result = converter(temp_file.name)
    return result
```

This pattern appears in [examples/pdf_embedding/main.py:32-36]() and ensures proper cleanup of temporary files.

Sources: [examples/pdf_embedding/main.py:32-36]()

---

## Document Processing Patterns

### Pattern: PDF Pipeline with Chunking

The most common pattern for text-based PDF processing:

```mermaid
sequenceDiagram
    participant Source as LocalFile Source
    participant Convert as PDF Converter
    participant Chunk as SplitRecursively
    participant Embed as Embedding Function
    participant Collect as Collector
    participant Export as Export Target
    
    Source->>Convert: Binary PDF content
    Convert->>Chunk: Markdown text
    Chunk->>Embed: Text chunks
    Embed->>Collect: Embeddings
    Collect->>Export: Batch write
```

**Diagram: Standard PDF Processing Sequence**

This pattern is implemented in:
- [examples/pdf_embedding/main.py:54-96]()
- [examples/manuals_llm_extraction/README.md:1-68]()
- [examples/patient_intake_extraction/README.md:1-57]()

Sources: [examples/pdf_embedding/main.py:54-96]()

### Pattern: Visual Document with ColPali

For layout-aware document search without text extraction:

```mermaid
sequenceDiagram
    participant Source as LocalFile Source
    participant Detect as Format Detection
    participant Convert as ConvertPdfPageToImage
    participant Embed as ColPaliEmbed
    participant Store as Qdrant Storage
    
    Source->>Detect: PDF or Image file
    alt is PDF
        Detect->>Convert: PDF bytes
        Convert->>Embed: Page images
    else is Image
        Detect->>Embed: Image directly
    end
    Embed->>Store: Multi-vector embeddings
```

**Diagram: Visual Document Processing Sequence**

This pattern preserves document layout and enables visual search capabilities.

Sources: [examples/multi_format_indexing/README.md:1-88]()

### Pattern: LLM Extraction from Documents

For structured data extraction from documents:

1. **Convert**: Transform document to markdown (text-based representation)
2. **Extract**: Use `ExtractByLlm` with Pydantic schema to extract structured data
3. **Post-process**: Apply custom functions for derived fields or validation
4. **Store**: Export structured records to Postgres or other targets

This pattern appears in:
- [examples/patient_intake_extraction/README.md:1-57]() (medical forms)
- [examples/manuals_llm_extraction/README.md:1-68]() (technical docs)
- [examples/docs_to_knowledge_graph/README.md:1-66]() (relationship extraction)

Sources: [examples/patient_intake_extraction/README.md:1-57](), [examples/manuals_llm_extraction/README.md:1-68]()

---

## Configuration and Dependencies

### Binary Source Configuration

PDF and document processing requires binary file reading:

```python
flow_builder.add_source(
    cocoindex.sources.LocalFile(
        path="pdf_files",
        binary=True  # Required for PDF processing
    )
)
```

The `binary=True` parameter ensures files are read as bytes rather than decoded as text strings.

Sources: [examples/pdf_embedding/main.py:62]()

### GPU and Caching Configuration

PDF processing functions benefit from GPU acceleration and result caching:

| Parameter | Purpose | Recommendation |
|-----------|---------|----------------|
| `gpu=True` | Enable GPU for PDF conversion | Use for marker-based conversion |
| `cache=True` | Cache conversion results | Always enable for expensive operations |
| `behavior_version=1` | Track implementation changes | Increment when logic changes |

These parameters are configured via the `@executor_class` decorator in [examples/pdf_embedding/main.py:18]().

Sources: [examples/pdf_embedding/main.py:18]()

### External Dependencies

Document processing examples require additional installations:

**For marker-based PDF conversion**:
```bash
pip install marker-pdf
```

**For MarkItDown multi-format conversion**:
```bash
pip install 'markitdown[all]'
```

**For ColPali visual embedding**:
```bash
pip install pdf2image pillow
# Plus system dependency: poppler
```

See [examples/multi_format_indexing/README.md:42]() for platform-specific poppler installation instructions.

Sources: [examples/pdf_embedding/main.py:6-8](), [examples/patient_intake_extraction/README.md:26-30](), [examples/multi_format_indexing/README.md:42]()

---

## Query and Search Patterns

### Text-Based PDF Search

After indexing PDF-converted text with embeddings, queries reuse the embedding transform:

```python
@cocoindex.transform_flow()
def text_to_embedding(text: cocoindex.DataSlice[str]) -> cocoindex.DataSlice[list[float]]:
    return text.transform(
        cocoindex.functions.SentenceTransformerEmbed(
            model="sentence-transformers/all-MiniLM-L6-v2"
        )
    )

# Query time
query_vector = text_to_embedding.eval(query)
```

This ensures consistency between indexing and querying. The `.eval()` method executes the transform flow on a single input value.

Sources: [examples/pdf_embedding/main.py:39-51](), [examples/pdf_embedding/main.py:105]()

### Visual Document Search

ColPali queries use a separate query embedding function:

```python
query_embedding = ColPaliQueryEmbed.eval(query_text)
search_results = client.search(
    collection_name=COLLECTION_NAME,
    query_vector=("visual_embedding", query_embedding),
    limit=10
)
```

The visual embedding space supports natural language queries against document images without requiring text extraction.

Sources: [examples/multi_format_indexing/README.md:20-22]()

### Structured Data Queries

After LLM extraction creates structured records, standard SQL queries access the data:

```sql
SELECT filename, module_info->'title' AS title, module_summary 
FROM modules_info;
```

The extracted structured data is stored as JSONB or individual columns, enabling rich queries.

Sources: [examples/manuals_llm_extraction/README.md:44-48]()

---

## Integration with CocoInsight

All PDF and document processing examples support CocoInsight visualization for debugging and monitoring:

```bash
cocoindex server -ci main
```

Then open [https://cocoindex.io/cocoinsight](https://cocoindex.io/cocoinsight).

CocoInsight provides:

- **Pipeline Visualization**: View the document processing flow and transformations
- **Data Lineage**: Track how PDF content flows through conversion, chunking, and embedding
- **Chunking Visualization**: Inspect chunk boundaries and overlap (especially useful for code)
- **Zero Data Retention**: CocoInsight connects to your local server without storing pipeline data

This is particularly useful for:
- Debugging PDF conversion quality
- Tuning chunk sizes and overlap for different document types
- Verifying LLM extraction results
- Monitoring incremental updates as documents change

Sources: [examples/pdf_embedding/README.md:49-58](), [examples/multi_format_indexing/README.md:79-88](), [examples/patient_intake_extraction/README.md:48-57]()

---

## Best Practices

### Choosing a PDF Processing Approach

| Document Type | Recommended Approach | Reason |
|--------------|---------------------|--------|
| Text-heavy manuals, articles | Text extraction + chunking | Better for semantic search of content |
| Forms, invoices, receipts | Visual embedding (ColPali) | Preserves layout and table structure |
| Mixed content (text + charts) | Visual embedding (ColPali) | Captures both text and visual elements |
| Structured data extraction | Text extraction + LLM | LLMs work better with text than images |
| Large document volumes | Text extraction | More storage-efficient than images |

### Chunking Strategy for PDFs

When using text extraction, configure `SplitRecursively` based on content:

```python
doc["chunks"] = doc["markdown"].transform(
    cocoindex.functions.SplitRecursively(),
    language="markdown",
    chunk_size=2000,      # Larger for technical docs
    chunk_overlap=500,    # Overlap to preserve context
)
```

Adjust `chunk_size` and `chunk_overlap` based on:
- Average paragraph length in documents
- Embedding model's context window
- Balance between granularity and context

Sources: [examples/pdf_embedding/main.py:69-74]()

### Caching and Performance

Enable caching for expensive PDF conversions:

```python
@cocoindex.op.executor_class(gpu=True, cache=True, behavior_version=1)
class PdfToMarkdownExecutor:
    # ...
```

CocoIndex caches based on input checksums, so unchanged PDFs skip reconversion during incremental updates.

Sources: [examples/pdf_embedding/main.py:18]()

### Error Handling

PDF processing can fail due to corrupted files, unsupported formats, or conversion errors. Implement robust error handling in custom functions and monitor failures through CocoInsight.

---

This page covered PDF and document processing in CocoIndex, including text extraction pipelines, visual document embedding with ColPali, multi-format processing, and custom function implementation patterns. For related topics, see [Text and Code Embedding Pipelines](#13.1) for non-PDF text processing and [Knowledge Graph and Extraction Pipelines](#13.3) for advanced LLM extraction patterns.

---

# Page: Knowledge Graph and Extraction Pipelines

# Knowledge Graph and Extraction Pipelines

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [examples/code_embedding/README.md](examples/code_embedding/README.md)
- [examples/custom_output_files/main.py](examples/custom_output_files/main.py)
- [examples/docs_to_knowledge_graph/README.md](examples/docs_to_knowledge_graph/README.md)
- [examples/docs_to_knowledge_graph/main.py](examples/docs_to_knowledge_graph/main.py)
- [examples/face_recognition/main.py](examples/face_recognition/main.py)
- [examples/fastapi_server_docker/main.py](examples/fastapi_server_docker/main.py)
- [examples/gdrive_text_embedding/README.md](examples/gdrive_text_embedding/README.md)
- [examples/image_search/colpali_main.py](examples/image_search/colpali_main.py)
- [examples/image_search/main.py](examples/image_search/main.py)
- [examples/manuals_llm_extraction/README.md](examples/manuals_llm_extraction/README.md)
- [examples/manuals_llm_extraction/main.py](examples/manuals_llm_extraction/main.py)
- [examples/paper_metadata/main.py](examples/paper_metadata/main.py)
- [examples/pdf_elements_embedding/main.py](examples/pdf_elements_embedding/main.py)
- [examples/pdf_embedding/README.md](examples/pdf_embedding/README.md)
- [examples/pdf_embedding/main.py](examples/pdf_embedding/main.py)
- [examples/postgres_source/main.py](examples/postgres_source/main.py)
- [examples/postgres_source/pyproject.toml](examples/postgres_source/pyproject.toml)
- [examples/product_recommendation/README.md](examples/product_recommendation/README.md)
- [examples/product_recommendation/main.py](examples/product_recommendation/main.py)
- [examples/text_embedding/README.md](examples/text_embedding/README.md)
- [examples/text_embedding_qdrant/README.md](examples/text_embedding_qdrant/README.md)

</details>



This page demonstrates how to build knowledge graph construction pipelines in CocoIndex, focusing on LLM-based entity and relationship extraction from documents and exporting results to graph databases. These examples show end-to-end flows from unstructured text to structured graph representations.

For general information about graph database targets, see [8.2 Knowledge Graphs: Neo4j and Kuzu](#8.2). For LLM integration details, see [7 LLM Integration](#7).

---

## Overview

Knowledge graph extraction pipelines in CocoIndex follow a common pattern:

```mermaid
graph LR
    Documents["Document Source<br/>(LocalFile, etc.)"]
    Extract["ExtractByLlm<br/>with structured output"]
    Relationships["Relationship Data<br/>(DataCollector)"]
    Nodes["Entity/Node Data<br/>(DataCollector)"]
    GraphDB["Graph Database<br/>(Neo4j/Kuzu)"]
    
    Documents --> Extract
    Extract --> Relationships
    Extract --> Nodes
    Relationships --> GraphDB
    Nodes --> GraphDB
```

**Key components:**
- **Source documents**: Text files (Markdown, JSON, etc.) containing unstructured information
- **LLM extraction**: Using `ExtractByLlm` function to extract structured entities and relationships
- **Multiple collectors**: Separate collectors for nodes, relationships, and mention links
- **Graph export**: Using `Neo4j` or `Kuzu` targets with node/relationship mappings

Sources: [examples/docs_to_knowledge_graph/main.py:1-192]()

---

## Entity and Relationship Extraction

### Defining Extraction Schemas

Knowledge graph extraction begins with defining dataclass schemas that the LLM will populate:

```mermaid
classDiagram
    class Relationship {
        +str subject
        +str predicate
        +str object
    }
    
    class DocumentSummary {
        +str title
        +str summary
    }
    
    class ProductTaxonomy {
        +str name
    }
    
    class ProductTaxonomyInfo {
        +list[ProductTaxonomy] taxonomies
        +list[ProductTaxonomy] complementary_taxonomies
    }
```

**Relationship extraction example:**

[examples/docs_to_knowledge_graph/main.py:32-42]() defines a `Relationship` dataclass with `subject`, `predicate`, and `object` fields. The docstring provides context to the LLM about what constitutes valid entities (e.g., "should be nouns" like "CocoIndex", "Incremental Processing").

**Product taxonomy example:**

[examples/product_recommendation/main.py:44-77]() defines `ProductTaxonomy` and `ProductTaxonomyInfo` classes using Pydantic's `Field` with detailed descriptions. These descriptions guide the LLM to extract appropriate taxonomies and complementary products.

Sources: [examples/docs_to_knowledge_graph/main.py:24-42](), [examples/product_recommendation/main.py:44-77]()

### Extracting with ExtractByLlm

The `ExtractByLlm` function transforms text into structured data:

```mermaid
graph TB
    Input["Document Content<br/>(DataSlice[str])"]
    Transform["doc['content'].transform(ExtractByLlm)"]
    LlmSpec["LlmSpec<br/>(api_type, model)"]
    OutputType["output_type parameter<br/>(dataclass or list[dataclass])"]
    Instruction["instruction parameter<br/>(prompt guidance)"]
    Result["Structured Output<br/>(DataSlice[T])"]
    
    Input --> Transform
    LlmSpec --> Transform
    OutputType --> Transform
    Instruction --> Transform
    Transform --> Result
```

**Extracting relationships from documents:**

[examples/docs_to_knowledge_graph/main.py:87-105]() shows extracting a `list[Relationship]` from document content:

```python
doc["relationships"] = doc["content"].transform(
    cocoindex.functions.ExtractByLlm(
        llm_spec=cocoindex.LlmSpec(
            api_type=cocoindex.LlmApiType.OLLAMA,
            model="llama3.2",
        ),
        output_type=list[Relationship],
        instruction="Please extract relationships from CocoIndex documents...",
    )
)
```

The `output_type=list[Relationship]` tells CocoIndex to extract a list of relationship triples. The LLM parses the document and returns structured instances.

**Extracting nested structures:**

[examples/product_recommendation/main.py:113-120]() demonstrates extracting complex nested structures with `ProductTaxonomyInfo` containing lists of taxonomies.

Sources: [examples/docs_to_knowledge_graph/main.py:87-105](), [examples/product_recommendation/main.py:113-120](), [examples/manuals_llm_extraction/main.py:105-127]()

---

## Graph Database Export Patterns

### Node Export

Nodes represent entities in the graph. The basic pattern uses `Nodes` mapping:

```mermaid
graph TB
    Collector["DataCollector<br/>document_node.collect(...)"]
    Export["collector.export()"]
    Target["targets.Neo4j or targets.Kuzu"]
    Mapping["targets.Nodes(label='Document')"]
    Database["Graph Database<br/>Nodes created"]
    
    Collector --> Export
    Export --> Target
    Target --> Mapping
    Mapping --> Database
```

**Document nodes example:**

[examples/docs_to_knowledge_graph/main.py:129-135]() exports document summaries as nodes:

```python
document_node.export(
    "document_node",
    GraphDbSpec(
        connection=conn_spec, 
        mapping=cocoindex.targets.Nodes(label="Document")
    ),
    primary_key_fields=["filename"],
)
```

This creates `Document` nodes with properties `filename`, `title`, and `summary` (from the collector).

Sources: [examples/docs_to_knowledge_graph/main.py:129-135](), [examples/product_recommendation/main.py:136-142]()

### Relationship Export

Relationships connect nodes. They use `Relationships` mapping with source and target node specifications:

```mermaid
graph TB
    Collector["DataCollector<br/>entity_relationship.collect(...)"]
    Export["collector.export()"]
    RelMapping["targets.Relationships(rel_type='...')"]
    SourceNode["targets.NodeFromFields<br/>(source entity)"]
    TargetNode["targets.NodeFromFields<br/>(target entity)"]
    Database["Graph Database<br/>Relationships created"]
    
    Collector --> Export
    Export --> RelMapping
    RelMapping --> SourceNode
    RelMapping --> TargetNode
    SourceNode --> Database
    TargetNode --> Database
```

**Entity relationship example:**

[examples/docs_to_knowledge_graph/main.py:144-169]() shows creating relationships between entities:

```python
entity_relationship.export(
    "entity_relationship",
    GraphDbSpec(
        connection=conn_spec,
        mapping=cocoindex.targets.Relationships(
            rel_type="RELATIONSHIP",
            source=cocoindex.targets.NodeFromFields(
                label="Entity",
                fields=[
                    cocoindex.targets.TargetFieldMapping(
                        source="subject", target="value"
                    ),
                ],
            ),
            target=cocoindex.targets.NodeFromFields(
                label="Entity",
                fields=[
                    cocoindex.targets.TargetFieldMapping(
                        source="object", target="value"
                    ),
                ],
            ),
        ),
    ),
    primary_key_fields=["id"],
)
```

**Key components:**
- `rel_type`: The relationship type in the graph (e.g., "RELATIONSHIP", "MENTION")
- `source`: Specification for the source node, using `NodeFromFields`
- `target`: Specification for the target node
- `TargetFieldMapping`: Maps collector field names to node property names

Sources: [examples/docs_to_knowledge_graph/main.py:144-169](), [examples/product_recommendation/main.py:152-177]()

### Node Declarations

When relationships reference nodes that aren't directly exported from a collector, use `GraphDbDeclaration`:

```mermaid
graph LR
    Declaration["flow_builder.declare()"]
    GraphDbDeclaration["targets.Neo4jDeclaration or KuzuDeclaration"]
    NodeSpec["nodes_label='Entity'<br/>primary_key_fields=['value']"]
    Database["Graph Database<br/>Node schema declared"]
    
    Declaration --> GraphDbDeclaration
    GraphDbDeclaration --> NodeSpec
    NodeSpec --> Database
```

**Entity node declaration:**

[examples/docs_to_knowledge_graph/main.py:137-143]() declares `Entity` nodes that will be referenced by relationships:

```python
flow_builder.declare(
    GraphDbDeclaration(
        connection=conn_spec,
        nodes_label="Entity",
        primary_key_fields=["value"],
    )
)
```

This tells the graph database that `Entity` nodes exist with `value` as their primary key. When relationships reference entities (via `NodeFromFields`), the database ensures these nodes are created or merged.

Sources: [examples/docs_to_knowledge_graph/main.py:137-143](), [examples/product_recommendation/main.py:144-150]()

---

## Complete Example: Document to Knowledge Graph

### Flow Architecture

The `DocsToKG` example ([examples/docs_to_knowledge_graph/main.py:44-191]()) demonstrates a complete pipeline:

```mermaid
graph TB
    subgraph "Source"
        LocalFiles["LocalFile source<br/>path='docs/docs/core'<br/>patterns=['*.md', '*.mdx']"]
    end
    
    subgraph "Extraction"
        DocSummary["ExtractByLlm<br/>output_type=DocumentSummary"]
        DocRels["ExtractByLlm<br/>output_type=list[Relationship]"]
    end
    
    subgraph "Collection"
        DocNode["document_node collector<br/>filename, title, summary"]
        EntityRel["entity_relationship collector<br/>id, subject, predicate, object"]
        EntityMention["entity_mention collector<br/>id, entity, filename"]
    end
    
    subgraph "Export"
        DocExport["Document nodes<br/>label='Document'"]
        RelExport["RELATIONSHIP edges<br/>Entity -> Entity"]
        MentionExport["MENTION edges<br/>Document -> Entity"]
    end
    
    LocalFiles --> DocSummary
    LocalFiles --> DocRels
    
    DocSummary --> DocNode
    DocRels --> EntityRel
    DocRels --> EntityMention
    
    DocNode --> DocExport
    EntityRel --> RelExport
    EntityMention --> MentionExport
```

Sources: [examples/docs_to_knowledge_graph/main.py:44-191]()

### Implementation Details

**1. Source configuration:**

[examples/docs_to_knowledge_graph/main.py:51-56]() ingests Markdown documentation:

```python
data_scope["documents"] = flow_builder.add_source(
    cocoindex.sources.LocalFile(
        path=os.path.join("..", "..", "docs", "docs", "core"),
        included_patterns=["*.md", "*.mdx"],
    )
)
```

**2. Dual extraction:**

[examples/docs_to_knowledge_graph/main.py:64-79]() extracts document summaries, while [examples/docs_to_knowledge_graph/main.py:87-105]() extracts relationships. Both use the same `doc["content"]` but with different output types.

**3. Triple generation:**

The flow iterates over extracted relationships and generates three types of data:

| Collector | Purpose | Fields |
|-----------|---------|--------|
| `entity_relationship` | Subject-predicate-object triples | `id`, `subject`, `predicate`, `object` |
| `entity_mention` (subject) | Links documents to subject entities | `id`, `entity`, `filename` |
| `entity_mention` (object) | Links documents to object entities | `id`, `entity`, `filename` |

[examples/docs_to_knowledge_graph/main.py:107-126]() implements this pattern using nested row iteration over the relationships list.

**4. Graph structure:**

The resulting graph has three node types and two relationship types:

```mermaid
graph LR
    Document["Document node<br/>filename, title, summary"]
    EntityA["Entity node<br/>value"]
    EntityB["Entity node<br/>value"]
    
    Document -->|"MENTION"| EntityA
    Document -->|"MENTION"| EntityB
    EntityA -->|"RELATIONSHIP<br/>(predicate)"| EntityB
```

Sources: [examples/docs_to_knowledge_graph/main.py:44-191]()

---

## Advanced Patterns

### Multi-Collector Patterns

Complex knowledge graphs often require multiple collectors to capture different aspects:

**Product recommendation example:**

[examples/product_recommendation/main.py:103-135]() uses three collectors:

| Collector | Purpose | Export Type |
|-----------|---------|-------------|
| `product_node` | Product entities | Nodes (label="Product") |
| `product_taxonomy` | Product → Taxonomy links | Relationships (rel_type="PRODUCT_TAXONOMY") |
| `product_complementary_taxonomy` | Product → Complementary links | Relationships (rel_type="PRODUCT_COMPLEMENTARY_TAXONOMY") |

This creates a graph where products connect to their taxonomies and complementary taxonomies through different relationship types.

Sources: [examples/product_recommendation/main.py:91-203]()

### Nested List Extraction

When extracting lists from LLMs, iterate over results to generate multiple graph elements:

```mermaid
graph TB
    DocContent["doc['content']"]
    ExtractList["transform(ExtractByLlm)<br/>output_type=list[Relationship]"]
    Relationships["doc['relationships']<br/>(list of relationships)"]
    RowIteration["with doc['relationships'].row()"]
    Individual["relationship['subject']<br/>relationship['predicate']<br/>relationship['object']"]
    Collect["Collect each as separate row"]
    
    DocContent --> ExtractList
    ExtractList --> Relationships
    Relationships --> RowIteration
    RowIteration --> Individual
    Individual --> Collect
```

[examples/docs_to_knowledge_graph/main.py:107-126]() demonstrates this pattern. The key is using `.row()` to iterate over the extracted list, allowing each relationship to be collected separately and potentially create multiple graph elements from a single document.

Sources: [examples/docs_to_knowledge_graph/main.py:107-126](), [examples/product_recommendation/main.py:123-134]()

### Field Mapping

Use `TargetFieldMapping` to rename fields when exporting to graphs:

```python
cocoindex.targets.TargetFieldMapping(
    source="subject",  # Field name in collector
    target="value"     # Property name in graph node
)
```

This is essential when:
- Multiple collectors export to the same node type with different field names
- Graph schema has different naming conventions than the collector
- Joining relationships to existing nodes with specific property names

[examples/docs_to_knowledge_graph/main.py:152-156]() and [examples/docs_to_knowledge_graph/main.py:160-165]() show mapping both `subject` and `object` fields to `value` properties on `Entity` nodes.

Sources: [examples/docs_to_knowledge_graph/main.py:150-167](), [examples/product_recommendation/main.py:161-172]()

### Connection Management

Graph database connections are managed through auth entries:

```python
neo4j_conn_spec = cocoindex.add_auth_entry(
    "Neo4jConnection",
    cocoindex.targets.Neo4jConnection(
        uri="bolt://localhost:7687",
        user="neo4j",
        password="cocoindex",
    ),
)
```

The connection spec is then referenced in all export operations: [examples/docs_to_knowledge_graph/main.py:132](), [examples/docs_to_knowledge_graph/main.py:147](), [examples/docs_to_knowledge_graph/main.py:173]().

For Kuzu, the same pattern applies with `KuzuConnection` and a database path instead of URI.

Sources: [examples/docs_to_knowledge_graph/main.py:9-16](), [examples/product_recommendation/main.py:10-22]()

---

## Example Variations

### Document Relationship Extraction

The `DocsToKG` example focuses on extracting conceptual relationships from documentation:

- **Input**: Markdown/MDX documentation files
- **Extraction**: Core concepts and their relationships (e.g., "CocoIndex uses Incremental Processing")
- **Output**: Concept graph with entity nodes and relationship edges
- **Use case**: Documentation knowledge base, concept exploration

Sources: [examples/docs_to_knowledge_graph/main.py:1-192](), [examples/docs_to_knowledge_graph/README.md:1-66]()

### Product Taxonomy Graph

The `StoreProduct` example builds product recommendation graphs:

- **Input**: JSON product descriptions
- **Extraction**: Product taxonomies and complementary product categories
- **Output**: Product-Taxonomy-Complementary graph structure
- **Use case**: Product recommendations, category management

Key differences:
- Uses Pydantic models with detailed field descriptions ([examples/product_recommendation/main.py:44-77]())
- Employs a template system to format product data ([examples/product_recommendation/main.py:24-41]())
- Creates bidirectional taxonomy relationships

Sources: [examples/product_recommendation/main.py:1-204](), [examples/product_recommendation/README.md:1-61]()

### Structured Manual Extraction

The `ManualExtraction` example ([examples/manuals_llm_extraction/main.py:90-140]()) shows extracting hierarchical structures:

- **Input**: PDF manuals converted to Markdown
- **Extraction**: Module info with nested classes, methods, and arguments
- **Output**: Relational database (Postgres) rather than graph
- **Pattern**: Demonstrates complex nested dataclass extraction

This example uses the same `ExtractByLlm` pattern but exports to Postgres instead of a graph database, showing the flexibility of the extraction approach.

Sources: [examples/manuals_llm_extraction/main.py:37-140](), [examples/manuals_llm_extraction/README.md:1-68]()

---

## Incremental Updates

All knowledge graph examples support incremental processing:

1. **Source change detection**: LocalFile sources track file modifications
2. **Cached LLM calls**: `ExtractByLlm` caches results by content hash
3. **Incremental graph updates**: Only modified documents trigger re-extraction
4. **Merge semantics**: Graph databases merge nodes by primary key

Example refresh configuration:

```python
data_scope["products"] = flow_builder.add_source(
    cocoindex.sources.LocalFile(path="products"),
    refresh_interval=datetime.timedelta(seconds=5),
)
```

[examples/product_recommendation/main.py:98-101]() shows setting a 5-second refresh interval for continuous monitoring.

When a document changes:
1. CocoIndex detects the change via content fingerprinting
2. LLM extraction runs only for changed documents
3. Graph database receives updated triples
4. Existing nodes are merged (updated), orphaned relationships are removed

Sources: [examples/product_recommendation/main.py:98-101](), [examples/docs_to_knowledge_graph/main.py:51-56]()

---

# Page: Cloud Storage Integration

# Cloud Storage Integration

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [examples/amazon_s3_embedding/README.md](examples/amazon_s3_embedding/README.md)
- [examples/amazon_s3_embedding/main.py](examples/amazon_s3_embedding/main.py)
- [examples/azure_blob_embedding/README.md](examples/azure_blob_embedding/README.md)
- [examples/azure_blob_embedding/main.py](examples/azure_blob_embedding/main.py)
- [examples/code_embedding/README.md](examples/code_embedding/README.md)
- [examples/code_embedding/main.py](examples/code_embedding/main.py)
- [examples/docs_to_knowledge_graph/README.md](examples/docs_to_knowledge_graph/README.md)
- [examples/docs_to_knowledge_graph/main.py](examples/docs_to_knowledge_graph/main.py)
- [examples/gdrive_text_embedding/README.md](examples/gdrive_text_embedding/README.md)
- [examples/gdrive_text_embedding/main.py](examples/gdrive_text_embedding/main.py)
- [examples/manuals_llm_extraction/README.md](examples/manuals_llm_extraction/README.md)
- [examples/patient_intake_extraction/main.py](examples/patient_intake_extraction/main.py)
- [examples/pdf_embedding/README.md](examples/pdf_embedding/README.md)
- [examples/product_recommendation/README.md](examples/product_recommendation/README.md)
- [examples/product_recommendation/main.py](examples/product_recommendation/main.py)
- [examples/text_embedding/README.md](examples/text_embedding/README.md)
- [examples/text_embedding/main.py](examples/text_embedding/main.py)
- [examples/text_embedding_qdrant/README.md](examples/text_embedding_qdrant/README.md)
- [examples/text_embedding_qdrant/main.py](examples/text_embedding_qdrant/main.py)

</details>



This page demonstrates how to integrate CocoIndex with cloud storage providers: Google Drive, Amazon S3, and Azure Blob Storage. Each integration example shows authentication setup, source connector configuration, and incremental synchronization patterns.

For general information about source connectors, see [Built-in Data Sources](#5). For authentication and secrets management, see [Authentication and Secrets Management](#10.1). For live update mechanisms, see [Live Updates and Change Capture](#9.1).

## Overview

CocoIndex provides built-in source connectors for three major cloud storage providers, each with support for:

- **Authentication**: Service accounts (Google Drive), IAM roles/access keys (AWS), managed identities/connection strings (Azure)
- **Change Detection**: Polling APIs, SQS notifications, or periodic refresh intervals
- **Incremental Sync**: Automatic detection of added, modified, or deleted files
- **Pattern Filtering**: Include/exclude files based on glob patterns
- **Binary/Text Mode**: Read files as bytes or decoded text

| Provider | Source Connector | Authentication Method | Change Detection |
|----------|-----------------|----------------------|------------------|
| Google Drive | `cocoindex.sources.GoogleDrive` | Service account credential file | Recent changes API polling |
| Amazon S3 | `cocoindex.sources.AmazonS3` | AWS credentials (env vars/IAM) | SQS notifications or polling |
| Azure Blob Storage | `cocoindex.sources.AzureBlob` | Connection string or managed identity | Periodic refresh |

## Google Drive Integration

### Authentication Setup

Google Drive integration requires a service account with credentials stored in a JSON file. The service account must have read access to the folders being indexed.

```mermaid
graph TB
    ServiceAccount["Service Account<br/>(JSON credential file)"]
    EnvVar["GOOGLE_SERVICE_ACCOUNT_CREDENTIAL<br/>environment variable"]
    FolderIds["GOOGLE_DRIVE_ROOT_FOLDER_IDS<br/>comma-separated folder IDs"]
    
    ServiceAccount --> EnvVar
    FolderIds --> GoogleDriveSource["cocoindex.sources.GoogleDrive"]
    EnvVar --> GoogleDriveSource
    
    GoogleDriveSource --> ChangesPoll["recent_changes_poll_interval<br/>polling mechanism"]
    GoogleDriveSource --> RefreshInterval["refresh_interval<br/>full scan interval"]
    
    ChangesPoll --> IncrementalUpdate["Incremental file detection"]
    RefreshInterval --> FullScan["Periodic full scan"]
```

**Google Drive Flow with Live Updates**

Sources: [examples/gdrive_text_embedding/main.py:1-124]()

### Source Configuration

The Google Drive source connector configuration in [examples/gdrive_text_embedding/main.py:31-41]() demonstrates:

```python
cocoindex.sources.GoogleDrive(
    service_account_credential_path=credential_path,
    root_folder_ids=root_folder_ids,
    recent_changes_poll_interval=datetime.timedelta(seconds=10),
)
```

**Configuration Parameters:**

| Parameter | Type | Purpose |
|-----------|------|---------|
| `service_account_credential_path` | `str` | Path to Google service account JSON credential file |
| `root_folder_ids` | `list[str]` | List of Google Drive folder IDs to index |
| `recent_changes_poll_interval` | `timedelta` | Frequency to poll the Drive Changes API for updates |

The `refresh_interval` parameter on `add_source()` [examples/gdrive_text_embedding/main.py:40]() controls periodic full scans as a fallback.

Sources: [examples/gdrive_text_embedding/main.py:31-41]()

### Complete Example

```mermaid
graph LR
    GDrive["Google Drive<br/>Folders"]
    GDriveSource["GoogleDrive Source"]
    Chunks["SplitRecursively<br/>chunking"]
    Embed["SentenceTransformerEmbed"]
    Export["Postgres Target<br/>doc_embeddings table"]
    
    GDrive -->|"recent_changes_poll_interval"| GDriveSource
    GDriveSource -->|"documents"| Chunks
    Chunks -->|"chunks"| Embed
    Embed -->|"embedding"| Export
```

**Google Drive Text Embedding Pipeline**

The complete flow [examples/gdrive_text_embedding/main.py:24-72]() demonstrates:

1. **Source ingestion** with Google Drive credentials from environment variables [examples/gdrive_text_embedding/main.py:31-32]()
2. **Change detection** via `recent_changes_poll_interval` parameter [examples/gdrive_text_embedding/main.py:38]()
3. **Text processing** with `SplitRecursively` chunking [examples/gdrive_text_embedding/main.py:46-51]()
4. **Embedding generation** using `SentenceTransformerEmbed` [examples/gdrive_text_embedding/main.py:54]()
5. **Export** to Postgres with vector indexes [examples/gdrive_text_embedding/main.py:62-72]()

Sources: [examples/gdrive_text_embedding/main.py:24-72](), [examples/gdrive_text_embedding/README.md:1-84]()

## Amazon S3 Integration

### Authentication Setup

Amazon S3 integration uses AWS credentials from environment variables or IAM roles. CocoIndex automatically discovers credentials through the standard AWS credential chain.

```mermaid
graph TB
    AWSCreds["AWS Credentials"]
    EnvVars["Environment Variables:<br/>AWS_ACCESS_KEY_ID<br/>AWS_SECRET_ACCESS_KEY<br/>AWS_REGION"]
    IAMRole["IAM Role<br/>(EC2/ECS/Lambda)"]
    
    AWSCreds --> EnvVars
    AWSCreds --> IAMRole
    
    EnvVars --> S3Source["cocoindex.sources.AmazonS3"]
    IAMRole --> S3Source
    
    BucketEnv["AMAZON_S3_BUCKET_NAME"]
    PrefixEnv["AMAZON_S3_PREFIX<br/>(optional)"]
    SQSEnv["AMAZON_S3_SQS_QUEUE_URL<br/>(optional)"]
    
    BucketEnv --> S3Source
    PrefixEnv --> S3Source
    SQSEnv --> S3Source
    
    S3Source --> SQSNotifications["SQS-based change notifications"]
    S3Source --> PeriodicRefresh["Periodic full bucket scan"]
```

**S3 Authentication and Change Detection**

Sources: [examples/amazon_s3_embedding/main.py:1-127](), [examples/amazon_s3_embedding/README.md:1-66]()

### Source Configuration with SQS Notifications

The S3 source connector [examples/amazon_s3_embedding/main.py:30-42]() supports optional SQS queue integration for real-time change notifications:

```python
cocoindex.sources.AmazonS3(
    bucket_name=bucket_name,
    prefix=prefix,
    included_patterns=["*.md", "*.mdx", "*.txt", "*.docx"],
    binary=False,
    sqs_queue_url=sqs_queue_url,
)
```

**Configuration Parameters:**

| Parameter | Type | Purpose |
|-----------|------|---------|
| `bucket_name` | `str` | S3 bucket name to index |
| `prefix` | `str \| None` | Optional prefix path within bucket |
| `included_patterns` | `list[str]` | Glob patterns for files to include |
| `binary` | `bool` | Read files as bytes (`True`) or text (`False`) |
| `sqs_queue_url` | `str \| None` | SQS queue URL for S3 event notifications |

When `sqs_queue_url` is provided, CocoIndex receives near-real-time notifications of S3 object changes. Otherwise, it performs periodic bucket scans based on `refresh_interval`.

Sources: [examples/amazon_s3_embedding/main.py:30-42]()

### Live Update Pattern

The S3 example demonstrates using `FlowLiveUpdater` for continuous synchronization [examples/amazon_s3_embedding/main.py:106-121]():

```python
with cocoindex.FlowLiveUpdater(amazon_s3_text_embedding_flow) as updater:
    while True:
        query = input("Enter search query (or Enter to quit): ")
        # Process queries while automatic updates run in background
```

This pattern enables:
- **Background updates**: Flow automatically syncs when S3 changes are detected
- **Interactive queries**: Application remains responsive for user queries
- **Resource cleanup**: Context manager ensures proper shutdown

Sources: [examples/amazon_s3_embedding/main.py:106-121]()

### Complete Example

```mermaid
graph LR
    S3Bucket["S3 Bucket"]
    SQS["SQS Queue<br/>(optional)"]
    S3Source["AmazonS3 Source"]
    Chunks["SplitRecursively"]
    Embed["SentenceTransformerEmbed"]
    Export["Postgres Target"]
    
    S3Bucket -->|"Object events"| SQS
    SQS -->|"Notifications"| S3Source
    S3Bucket -->|"Periodic scan"| S3Source
    
    S3Source -->|"filtered by patterns"| Chunks
    Chunks --> Embed
    Embed --> Export
```

**S3 Incremental Sync Pipeline**

The flow [examples/amazon_s3_embedding/main.py:23-73]() demonstrates pattern filtering and incremental processing from S3 buckets.

Sources: [examples/amazon_s3_embedding/main.py:23-73](), [examples/amazon_s3_embedding/README.md:1-66]()

## Azure Blob Storage Integration

### Authentication Setup

Azure Blob Storage integration supports connection strings or managed identities for authentication.

```mermaid
graph TB
    AzureAuth["Azure Authentication"]
    ConnString["Connection String<br/>(AZURE_STORAGE_CONNECTION_STRING)"]
    ManagedId["Managed Identity<br/>(for Azure-hosted apps)"]
    
    AzureAuth --> ConnString
    AzureAuth --> ManagedId
    
    AcctName["AZURE_STORAGE_ACCOUNT_NAME"]
    ContainerName["AZURE_BLOB_CONTAINER_NAME"]
    Prefix["AZURE_BLOB_PREFIX<br/>(optional)"]
    
    ConnString --> AzureBlobSource["cocoindex.sources.AzureBlob"]
    ManagedId --> AzureBlobSource
    AcctName --> AzureBlobSource
    ContainerName --> AzureBlobSource
    Prefix --> AzureBlobSource
    
    AzureBlobSource --> PeriodicRefresh["Periodic container scan"]
```

**Azure Blob Authentication Configuration**

Sources: [examples/azure_blob_embedding/main.py:1-129](), [examples/azure_blob_embedding/README.md:1-66]()

### Source Configuration

The Azure Blob source connector [examples/azure_blob_embedding/main.py:30-42]() configuration:

```python
cocoindex.sources.AzureBlob(
    account_name=account_name,
    container_name=container_name,
    prefix=prefix,
    included_patterns=["*.md", "*.mdx", "*.txt", "*.docx"],
    binary=False,
)
```

**Configuration Parameters:**

| Parameter | Type | Purpose |
|-----------|------|---------|
| `account_name` | `str` | Azure storage account name |
| `container_name` | `str` | Blob container name to index |
| `prefix` | `str \| None` | Optional prefix path within container |
| `included_patterns` | `list[str]` | Glob patterns for files to include |
| `binary` | `bool` | Read blobs as bytes (`True`) or text (`False`) |

Azure Blob Storage integration relies on periodic refresh intervals since Azure does not provide a native change notification mechanism like S3's SQS.

Sources: [examples/azure_blob_embedding/main.py:30-42]()

### Complete Example

The Azure Blob example [examples/azure_blob_embedding/main.py:23-73]() follows the same pattern as S3 but with Azure-specific authentication:

1. **Environment variables** loaded from `.env` file [examples/azure_blob_embedding/main.py:126-127]()
2. **Source configuration** with account and container names [examples/azure_blob_embedding/main.py:30-42]()
3. **Setup and update** operations [examples/azure_blob_embedding/main.py:106-108]()
4. **Query loop** for interactive search [examples/azure_blob_embedding/main.py:111-122]()

Sources: [examples/azure_blob_embedding/main.py:23-129]()

## Common Patterns Across Providers

### Pattern Filtering

All three cloud storage connectors support glob pattern filtering for selective file ingestion:

```python
included_patterns=["*.md", "*.mdx", "*.txt", "*.docx"]
```

This filtering happens at the source level, reducing unnecessary data transfer and processing. Patterns follow standard glob syntax with wildcards (`*`, `**`, `?`).

Sources: [examples/amazon_s3_embedding/main.py:38](), [examples/azure_blob_embedding/main.py:39]()

### Binary vs. Text Mode

The `binary` parameter controls how file content is read:

- **`binary=False`** (default): Content is decoded as UTF-8 text and provided as `str` to the flow
- **`binary=True`**: Content is provided as raw `bytes`, useful for PDFs, images, or mixed-format processing

Example with binary mode for PDF processing:

```python
cocoindex.sources.LocalFile(
    path=os.path.join("data", "patient_forms"), 
    binary=True
)
```

Sources: [examples/patient_intake_extraction/main.py:120-123](), [examples/amazon_s3_embedding/main.py:39]()

### Refresh Intervals

All cloud sources support `refresh_interval` on `add_source()` to control full scan frequency:

```python
flow_builder.add_source(
    cocoindex.sources.GoogleDrive(...),
    refresh_interval=datetime.timedelta(minutes=1)
)
```

This provides a fallback mechanism for change detection and ensures eventual consistency even if notification mechanisms fail.

Sources: [examples/gdrive_text_embedding/main.py:40](), [examples/text_embedding/main.py:42]()

### Reusable Transform Flows

All examples extract the embedding logic into a reusable `@transform_flow()`:

```mermaid
graph TB
    TransformFlow["@transform_flow()<br/>text_to_embedding"]
    
    IndexPath["Indexing Path:<br/>chunk['text'].call(text_to_embedding)"]
    QueryPath["Query Path:<br/>text_to_embedding.eval(query)"]
    
    TransformFlow --> IndexPath
    TransformFlow --> QueryPath
    
    IndexPath --> EmbedStore["Store embeddings<br/>in database"]
    QueryPath --> SearchLogic["Compare with stored<br/>embeddings"]
```

**Transform Flow Reuse Pattern**

This pattern [examples/gdrive_text_embedding/main.py:9-21]() enables:
- **Code reuse**: Same embedding logic for indexing and querying
- **Consistency**: Identical transformations guarantee comparable results
- **Flexibility**: Easy to switch embedding models in one place

Sources: [examples/gdrive_text_embedding/main.py:9-21](), [examples/amazon_s3_embedding/main.py:8-20](), [examples/azure_blob_embedding/main.py:8-20]()

## Authentication and Credentials Management

### Environment Variable Pattern

All examples use environment variables for credential management:

```python
from dotenv import load_dotenv
load_dotenv()

credential_path = os.environ["GOOGLE_SERVICE_ACCOUNT_CREDENTIAL"]
bucket_name = os.environ["AMAZON_S3_BUCKET_NAME"]
account_name = os.environ["AZURE_STORAGE_ACCOUNT_NAME"]
```

The `.env` file pattern keeps credentials out of source code and supports different configurations per environment.

Sources: [examples/gdrive_text_embedding/main.py:120-122](), [examples/amazon_s3_embedding/main.py:123-125](), [examples/azure_blob_embedding/main.py:125-127]()

### CocoIndex Auth Registry

For credentials that need to be shared across flows or stored securely, use the auth registry system (see [Authentication and Secrets Management](#10.1)):

```python
neo4j_conn = cocoindex.add_auth_entry(
    "Neo4jConnection",
    cocoindex.targets.Neo4jConnection(
        uri="bolt://localhost:7687",
        user="neo4j",
        password="cocoindex",
    ),
)
```

Sources: [examples/docs_to_knowledge_graph/main.py:9-16]()

## Incremental Synchronization

### Change Detection Mechanisms

```mermaid
graph TB
    CloudStorage["Cloud Storage<br/>Source"]
    
    ChangeDetection["Change Detection<br/>Mechanism"]
    
    GoogleAPI["Google Drive:<br/>Changes API polling"]
    S3SQS["S3:<br/>SQS notifications"]
    PeriodicScan["All Providers:<br/>Periodic full scan"]
    
    CloudStorage --> ChangeDetection
    ChangeDetection --> GoogleAPI
    ChangeDetection --> S3SQS
    ChangeDetection --> PeriodicScan
    
    GoogleAPI --> IncrementalEngine["CocoIndex Incremental<br/>Processing Engine"]
    S3SQS --> IncrementalEngine
    PeriodicScan --> IncrementalEngine
    
    IncrementalEngine --> ContentFingerprint["Content fingerprinting"]
    IncrementalEngine --> CachedResults["Cached transformation<br/>results"]
    
    ContentFingerprint --> MinimalRecompute["Minimal recomputation<br/>of changed files only"]
    CachedResults --> MinimalRecompute
```

**Cloud Storage Incremental Sync Architecture**

CocoIndex's incremental processing (see [Incremental Processing and State Management](#9)) combines cloud provider change detection with content-based fingerprinting:

1. **Provider-level detection**: SQS notifications, API polling, or periodic scans identify potential changes
2. **Content fingerprinting**: CocoIndex compares file content hashes against cached state
3. **Dependency tracking**: Only affected transformations are recomputed
4. **Result memoization**: Expensive operations (embeddings, LLM calls) are cached

This two-tier approach minimizes both network traffic and computational costs.

Sources: [examples/gdrive_text_embedding/main.py:34-41](), [examples/amazon_s3_embedding/main.py:34-42]()

### Live Update Integration

The `FlowLiveUpdater` pattern enables continuous background synchronization:

```python
amazon_s3_text_embedding_flow.setup()
with cocoindex.FlowLiveUpdater(amazon_s3_text_embedding_flow) as updater:
    # Application continues running while updates happen automatically
    while True:
        query = input("Enter search query: ")
        results = search(pool, query)
```

This creates a long-running process that:
- Monitors the cloud storage source for changes
- Automatically triggers incremental updates when changes are detected
- Allows the application to remain responsive for queries
- Properly cleans up resources when exiting the context manager

Sources: [examples/amazon_s3_embedding/main.py:106-121](), [examples/gdrive_text_embedding/main.py:98-118]()

## Setup and Deployment

### Prerequisites Checklist

| Provider | Authentication Setup | Additional Requirements |
|----------|---------------------|------------------------|
| Google Drive | Service account JSON file; Share folders with service account email | None |
| Amazon S3 | AWS credentials (env vars or IAM role); Optional SQS queue for notifications | S3 bucket access permissions |
| Azure Blob | Connection string or managed identity; Storage account name | Blob container access permissions |
| All | PostgreSQL database with pgvector extension | `COCOINDEX_DATABASE_URL` environment variable |

### Typical Workflow

```mermaid
sequenceDiagram
    participant User
    participant Env["Environment<br/>(.env file)"]
    participant CocoIndex["CocoIndex<br/>Python API"]
    participant CloudProvider["Cloud Storage<br/>Provider"]
    participant Postgres["PostgreSQL<br/>State DB"]
    
    User->>Env: Configure credentials
    User->>CocoIndex: cocoindex.init()
    CocoIndex->>Postgres: Connect to state database
    
    User->>CocoIndex: flow.setup()
    CocoIndex->>Postgres: Create state tables
    CocoIndex->>Postgres: Create target tables
    
    User->>CocoIndex: flow.update() or FlowLiveUpdater
    CocoIndex->>CloudProvider: List files with credentials
    CloudProvider-->>CocoIndex: File metadata
    CocoIndex->>CloudProvider: Download changed files
    CloudProvider-->>CocoIndex: File contents
    CocoIndex->>Postgres: Store fingerprints and results
    
    User->>CocoIndex: search(query)
    CocoIndex->>Postgres: Query vector embeddings
    Postgres-->>CocoIndex: Top K results
    CocoIndex-->>User: Search results
```

**Cloud Storage Integration Lifecycle**

Steps:
1. **Configuration**: Set up credentials in `.env` file [examples/gdrive_text_embedding/README.md:36-41]()
2. **Initialization**: Call `cocoindex.init()` to load settings [examples/gdrive_text_embedding/main.py:122]()
3. **Setup**: Run `flow.setup()` to create backend resources [examples/amazon_s3_embedding/main.py:106]()
4. **Update**: Run `flow.update()` or use `FlowLiveUpdater` for continuous sync [examples/amazon_s3_embedding/main.py:107]()
5. **Query**: Execute search operations against indexed data [examples/gdrive_text_embedding/main.py:75-96]()

Sources: [examples/gdrive_text_embedding/README.md:44-64](), [examples/amazon_s3_embedding/main.py:106-121]()

### CocoInsight Visualization

All examples support CocoInsight for flow visualization and debugging:

```bash
cocoindex server -ci main
```

Then open the web UI at https://cocoindex.io/cocoinsight to:
- Visualize the complete data flow from cloud source to target
- Inspect data at each transformation step
- Monitor incremental update statistics
- Debug issues with cloud authentication or data processing

Sources: [examples/gdrive_text_embedding/README.md:67-83](), [examples/amazon_s3_embedding/README.md:50-66]()

---

# Page: Image Search and Multimodal Embeddings

# Image Search and Multimodal Embeddings

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [examples/custom_output_files/main.py](examples/custom_output_files/main.py)
- [examples/face_recognition/main.py](examples/face_recognition/main.py)
- [examples/fastapi_server_docker/main.py](examples/fastapi_server_docker/main.py)
- [examples/image_search/colpali_main.py](examples/image_search/colpali_main.py)
- [examples/image_search/main.py](examples/image_search/main.py)
- [examples/manuals_llm_extraction/main.py](examples/manuals_llm_extraction/main.py)
- [examples/paper_metadata/main.py](examples/paper_metadata/main.py)
- [examples/pdf_elements_embedding/main.py](examples/pdf_elements_embedding/main.py)
- [examples/pdf_embedding/main.py](examples/pdf_embedding/main.py)
- [examples/postgres_source/main.py](examples/postgres_source/main.py)
- [examples/postgres_source/pyproject.toml](examples/postgres_source/pyproject.toml)

</details>



## Purpose and Scope

This document demonstrates image search and multimodal embedding implementations in CocoIndex. It covers:
- **CLIP-based image search**: Embedding images for visual similarity search
- **ColPali multimodal embeddings**: Multi-vector document understanding
- **Face recognition**: Spatial data handling with face embeddings
- **Mixed modality indexing**: Processing both text and images from documents
- **FastAPI integration**: Production-ready search APIs with live updates

For general embedding concepts and text embeddings, see [Text and Code Embedding Pipelines](#13.1). For LLM integration used in captioning, see [LLM Integration](#7). For production deployment patterns, see [Production Deployment: FastAPI and Docker](#13.6).

---

## CLIP-Based Image Search Architecture

The CLIP (Contrastive Language-Image Pre-training) model enables cross-modal search where text queries retrieve visually similar images. CocoIndex provides a complete pipeline from image ingestion to FastAPI-served search endpoints.

### System Architecture

```mermaid
graph TB
    subgraph "Source Layer"
        SOURCE["LocalFile<br/>path='img'<br/>patterns=['*.jpg', '*.png']"]
    end
    
    subgraph "Processing Layer"
        EMBED_IMG["embed_image()<br/>Custom Function<br/>CLIP Image Encoder"]
        CAPTION["ExtractByLlm<br/>Ollama Vision Model<br/>Optional"]
    end
    
    subgraph "Storage Layer"
        QDRANT["Qdrant Collection<br/>collection='ImageSearch'<br/>768-dim vectors"]
    end
    
    subgraph "Query Layer"
        EMBED_QUERY["embed_query()<br/>CLIP Text Encoder"]
        FASTAPI["FastAPI /search<br/>QdrantClient.search()"]
        LIVE["FlowLiveUpdater<br/>Auto-sync on changes"]
    end
    
    SOURCE --> EMBED_IMG
    SOURCE -.optional.-> CAPTION
    EMBED_IMG --> QDRANT
    CAPTION --> QDRANT
    
    EMBED_QUERY --> FASTAPI
    FASTAPI --> QDRANT
    LIVE -.monitors.-> SOURCE
    
    style EMBED_IMG fill:#f9f9f9
    style QDRANT fill:#f0f0f0
    style FASTAPI fill:#f9f9f9
```

**Sources**: [examples/image_search/main.py:1-171]()

### Custom Image Embedding Function

The `embed_image` function uses the CLIP model to convert images to 768-dimensional embeddings:

```python
@cocoindex.op.function(cache=True, behavior_version=1, gpu=True)
def embed_image(
    img_bytes: bytes,
) -> cocoindex.Vector[cocoindex.Float32, Literal[768]]:
    """Convert image to embedding using CLIP model."""
```

Key characteristics:
- **GPU enabled**: `gpu=True` routes execution to subprocess with GPU access
- **Cached**: `cache=True` stores results in internal state DB
- **Type safety**: Return type specifies exact vector dimensions (768)
- **Model loading**: Cached via `@functools.cache` on `get_clip_model()` to avoid reloading

The function loads images with PIL, processes them through the CLIP vision encoder, and returns normalized feature vectors suitable for cosine similarity search.

**Sources**: [examples/image_search/main.py:42-54](), [examples/image_search/main.py:24-28]()

### Flow Definition with Optional Captioning

The `ImageObjectEmbedding` flow demonstrates conditional field collection based on environment configuration:

```python
@cocoindex.flow_def(name="ImageObjectEmbedding")
def image_object_embedding_flow(
    flow_builder: cocoindex.FlowBuilder, data_scope: cocoindex.DataScope
) -> None:
    data_scope["images"] = flow_builder.add_source(
        cocoindex.sources.LocalFile(
            path="img", included_patterns=["*.jpg", "*.jpeg", "*.png"], binary=True
        ),
        refresh_interval=datetime.timedelta(minutes=1),
    )
```

**Conditional captioning logic**:
- Checks for `OLLAMA_MODEL` environment variable
- If present, uses `ExtractByLlm` with Ollama vision model to generate captions
- Detailed prompt engineering: "Describe the image in one detailed sentence. Name all visible animal species, objects, and the main scene..."
- Caption field only collected when model is available

This pattern allows the same flow to work with or without captioning capabilities, adapting to available resources.

**Sources**: [examples/image_search/main.py:58-110]()

### Query-Time Text Embedding

The `embed_query` function provides the text encoder counterpart for cross-modal search:

```python
def embed_query(text: str) -> list[float]:
    """Embed the caption using CLIP model."""
    model, processor = get_clip_model()
    inputs = processor(text=[text], return_tensors="pt", padding=True)
    with torch.no_grad():
        features = model.get_text_features(**inputs)
    return features[0].tolist()
```

This function uses the same CLIP model but processes text through the text encoder. The shared embedding space allows text queries to retrieve visually similar images via cosine similarity.

**Sources**: [examples/image_search/main.py:31-39]()

---

## ColPali Multimodal Document Embeddings

ColPali extends traditional embeddings with **multi-vector representations**, where each image/document produces multiple token-level embeddings. This enables fine-grained understanding of document layouts, diagrams, and mixed text-image content.

### Multi-Vector Architecture

```mermaid
graph LR
    subgraph "Single-Vector (CLIP)"
        IMG1["Image"] --> CLIP["CLIP Encoder"] --> VEC1["[768 floats]<br/>Single embedding"]
    end
    
    subgraph "Multi-Vector (ColPali)"
        IMG2["Image"] --> COLPALI["ColPali Encoder"] --> MVEC["[[128 floats],<br/>[128 floats],<br/>...<br/>[128 floats]]<br/>~1024 tokens"]
    end
    
    subgraph "Retrieval"
        QUERY["Query Text"] --> QENC["ColPali Query Encoder"] --> QVEC["[[128 floats],<br/>...]"]
        QVEC --> MAXSIM["MaxSim Scoring<br/>max(cosine(qi, dj))"]
        MVEC --> MAXSIM
        MAXSIM --> SCORE["Similarity Score"]
    end
    
    style COLPALI fill:#f9f9f9
    style MAXSIM fill:#f0f0f0
```

**MaxSim scoring**: For each query token, finds the maximum similarity with any document token, then averages across all query tokens. This captures fine-grained semantic matches within documents.

**Sources**: [examples/image_search/colpali_main.py:1-135]()

### Built-in ColPali Functions

CocoIndex provides two built-in functions for ColPali embeddings:

| Function | Purpose | Input | Output | Usage |
|----------|---------|-------|--------|-------|
| `ColPaliEmbedImage` | Embed images/documents | `bytes` | `list[list[float]]` | Indexing pipeline |
| `ColPaliEmbedQuery` | Embed text queries | `str` | `list[list[float]]` | Query-time encoding |

Both functions support model selection via the `model` parameter (default: `"vidore/colpali-v1.2"`).

**Example usage in flow**:

```python
img["embedding"] = img["content"].transform(
    cocoindex.functions.ColPaliEmbedImage(model=COLPALI_MODEL_NAME)
)
```

**Sources**: [examples/image_search/colpali_main.py:43-70]()

### Transform Flow for Query Consistency

The `text_to_colpali_embedding` transform flow ensures identical encoding logic for both indexing and querying:

```python
@cocoindex.transform_flow()
def text_to_colpali_embedding(
    text: cocoindex.DataSlice[str],
) -> cocoindex.DataSlice[list[list[float]]]:
    """
    Embed text using a ColPali model, returning multi-vector format.
    This is shared logic between indexing and querying.
    """
    return text.transform(
        cocoindex.functions.ColPaliEmbedQuery(model=COLPALI_MODEL_NAME)
    )
```

At query time, this flow is evaluated in-memory:

```python
query_embedding = text_to_colpali_embedding.eval(q)
```

This pattern guarantees that queries use the exact same model and processing steps as indexed documents, preventing embedding drift.

**Sources**: [examples/image_search/colpali_main.py:30-40](), [examples/image_search/colpali_main.py:109]()

### Qdrant Multi-Vector Search

Qdrant's `query_points` API handles multi-vector queries with MaxSim scoring:

```python
search_results = app.state.qdrant_client.query_points(
    collection_name=QDRANT_COLLECTION,
    query=query_embedding,  # list[list[float]]
    using="embedding",
    limit=limit,
    with_payload=True,
)
```

Key differences from single-vector search:
- Uses `query_points` instead of `search`
- Query parameter accepts multi-vector format directly
- `using` parameter specifies the vector field name
- Qdrant automatically applies MaxSim scoring

**Sources**: [examples/image_search/colpali_main.py:115-121]()

---

## FastAPI Integration with Live Updates

The image search examples demonstrate production-ready FastAPI integration with automatic synchronization.

### Application Lifecycle

```mermaid
graph TB
    START["FastAPI Startup"] --> LIFESPAN["lifespan() Context Manager"]
    
    LIFESPAN --> INIT["cocoindex.init()"]
    INIT --> SETUP["flow.setup()"]
    SETUP --> CLIENT["QdrantClient<br/>Connection Pool"]
    CLIENT --> UPDATER["FlowLiveUpdater<br/>Start Monitoring"]
    
    UPDATER --> SERVE["Application Ready<br/>/search Endpoint"]
    
    subgraph "Background"
        UPDATER -.watches.-> SOURCE["LocalFile Source"]
        SOURCE -.changes.-> UPDATE["Incremental Update"]
        UPDATE -.writes.-> QDRANT["Qdrant"]
    end
    
    SERVE --> SHUTDOWN["Shutdown"]
    
    style UPDATER fill:#f9f9f9
    style QDRANT fill:#f0f0f0
```

**Sources**: [examples/image_search/main.py:113-126]()

### Live Updater Configuration

The `FlowLiveUpdater` enables continuous synchronization:

```python
app.state.live_updater = cocoindex.FlowLiveUpdater(image_object_embedding_flow)
app.state.live_updater.start()
```

Behavior:
- Monitors source locations based on `refresh_interval` (e.g., `timedelta(minutes=1)`)
- Detects file additions, modifications, and deletions
- Triggers incremental processing only for changed data
- Automatically updates Qdrant collection with mutations

The updater runs in a background thread, allowing the FastAPI server to handle requests while keeping the index synchronized.

**Sources**: [examples/image_search/main.py:122-123](), [examples/image_search/colpali_main.py:82-83]()

### Search Endpoint Implementation

The `/search` endpoint demonstrates text-to-image retrieval:

```python
@app.get("/search")
def search(
    q: str = Query(..., description="Search query"),
    limit: int = Query(5, description="Number of results"),
) -> dict[str, Any]:
    query_embedding = embed_query(q)
    
    search_results = app.state.qdrant_client.search(
        collection_name=QDRANT_COLLECTION,
        query_vector=("embedding", query_embedding),
        limit=limit,
        with_payload=True,
    )
    
    return {
        "results": [
            {
                "filename": result.payload["filename"],
                "score": result.score,
                "caption": result.payload.get("caption"),
            }
            for result in search_results
        ]
    }
```

The endpoint uses the application state to access the Qdrant client, encodes the query text, performs vector search, and returns results with scores and optional captions.

**Sources**: [examples/image_search/main.py:143-170]()

---

## Face Recognition and Spatial Data Handling

Face recognition requires special handling for spatial relationships between extracted faces and source images.

### Face Extraction with Arg Relationship

```mermaid
graph TB
    IMG["Source Image<br/>image.jpg"] --> EXTRACT["extract_faces()<br/>arg_relationship=<br/>RECTS_BASE_IMAGE"]
    
    EXTRACT --> FACE1["FaceBase<br/>rect=(10,20,110,120)<br/>image=cropped_bytes"]
    EXTRACT --> FACE2["FaceBase<br/>rect=(200,50,300,150)<br/>image=cropped_bytes"]
    
    FACE1 --> EMBED1["extract_face_embedding()<br/>128-dim vector"]
    FACE2 --> EMBED2["extract_face_embedding()<br/>128-dim vector"]
    
    EMBED1 --> QDRANT["Qdrant Collection"]
    EMBED2 --> QDRANT
    
    style EXTRACT fill:#f9f9f9
    style QDRANT fill:#f0f0f0
```

**Sources**: [examples/face_recognition/main.py:1-125]()

### Arg Relationship Specification

The `extract_faces` function uses a special argument relationship to optimize caching:

```python
@cocoindex.op.function(
    cache=True,
    behavior_version=1,
    gpu=True,
    arg_relationship=(cocoindex.op.ArgRelationship.RECTS_BASE_IMAGE, "content"),
)
def extract_faces(content: bytes) -> list[FaceBase]:
```

**`ArgRelationship.RECTS_BASE_IMAGE`**:
- Indicates that results contain cropped regions from the input image
- Enables intelligent cache invalidation: if the base image changes, all face crops are invalidated
- Optimizes storage: CocoIndex can track spatial dependencies without re-extracting unchanged regions
- The second parameter `"content"` specifies which argument contains the base image

This pattern is essential for operations that produce spatially-dependent outputs like face detection, object detection, or image segmentation.

**Sources**: [examples/face_recognition/main.py:34-75]()

### Face Embedding Pipeline

The face recognition flow demonstrates nested processing:

```python
with data_scope["images"].row() as image:
    image["faces"] = image["content"].transform(extract_faces)
    
    with image["faces"].row() as face:
        face["embedding"] = face["image"].transform(extract_face_embedding)
        
        face_embeddings.collect(
            id=cocoindex.GeneratedField.UUID,
            filename=image["filename"],
            rect=face["rect"],
            embedding=face["embedding"],
        )
```

This pattern:
1. Extracts all faces from each image
2. Embeds each face independently
3. Collects each face embedding with spatial metadata (rect) and source filename
4. Enables face-level search while maintaining provenance to source images

**Sources**: [examples/face_recognition/main.py:105-118]()

---

## Mixed Modality PDF Indexing

The PDF elements example shows how to process documents with both text and images, creating separate vector collections optimized for each modality.

### Dual-Stream Architecture

```mermaid
graph TB
    PDF["PDF Source<br/>source_files/*.pdf"] --> EXTRACT["extract_pdf_elements()"]
    
    EXTRACT --> PAGES["PdfPage Objects<br/>page_number, text, images[]"]
    
    PAGES --> TEXT_STREAM["Text Processing Stream"]
    PAGES --> IMAGE_STREAM["Image Processing Stream"]
    
    subgraph "Text Stream"
        TEXT_STREAM --> CHUNK["SplitRecursively<br/>custom separators"]
        CHUNK --> TEMBED["SentenceTransformer<br/>Embed"]
        TEMBED --> TQDRANT["Qdrant Collection<br/>PdfElementsEmbeddingText"]
    end
    
    subgraph "Image Stream"
        IMAGE_STREAM --> IEXTRACT["PdfImage Objects<br/>thumbnail, name"]
        IEXTRACT --> IEMBED["clip_embed_image()<br/>CLIP embeddings"]
        IEMBED --> IQDRANT["Qdrant Collection<br/>PdfElementsEmbeddingImage"]
    end
    
    style EXTRACT fill:#f9f9f9
    style TQDRANT fill:#f0f0f0
    style IQDRANT fill:#f0f0f0
```

**Sources**: [examples/pdf_elements_embedding/main.py:1-181]()

### PDF Element Extraction

The `extract_pdf_elements` function returns structured data containing both text and images:

```python
@dataclass
class PdfImage:
    name: str
    data: bytes

@dataclass
class PdfPage:
    page_number: int
    text: str
    images: list[PdfImage]

@cocoindex.op.function()
def extract_pdf_elements(content: bytes) -> list[PdfPage]:
```

Processing pipeline:
1. Uses `pypdf.PdfReader` to iterate pages
2. Extracts text with `page.extract_text()`
3. Extracts images with `page.images`, filtering by size (≥16×16 pixels)
4. Creates thumbnails (512×512) to reduce storage and improve embedding speed
5. Returns structured `PdfPage` objects with both modalities

**Sources**: [examples/pdf_elements_embedding/main.py:66-101]()

### Separate Collection Exports

The flow exports to two distinct Qdrant collections:

```python
text_output.export(
    "text_embeddings",
    cocoindex.targets.Qdrant(
        connection=qdrant_connection,
        collection_name=QDRANT_COLLECTION_TEXT,
    ),
    primary_key_fields=["id"],
)

image_output.export(
    "image_embeddings",
    cocoindex.targets.Qdrant(
        connection=qdrant_connection,
        collection_name=QDRANT_COLLECTION_IMAGE,
    ),
    primary_key_fields=["id"],
)
```

**Rationale for separate collections**:
- Different embedding models (SentenceTransformer vs CLIP) produce different vector dimensions
- Different similarity metrics may be optimal (cosine vs dot product)
- Independent scaling: text vs image workloads may have different performance characteristics
- Query flexibility: search text-only, image-only, or combine results from both collections

**Sources**: [examples/pdf_elements_embedding/main.py:165-180]()

### Custom Text Chunking Configuration

The example demonstrates custom language specifications for domain-specific text splitting:

```python
cocoindex.functions.SplitRecursively(
    custom_languages=[
        cocoindex.functions.CustomLanguageSpec(
            language_name="text",
            separators_regex=[
                r"\n(\s*\n)+",    # Paragraph breaks
                r"[\.!\?]\s+",    # Sentence boundaries
                r"\n",            # Line breaks
                r"\s+",           # Whitespace
            ],
        )
    ]
),
language="text",
chunk_size=600,
chunk_overlap=100,
```

This configuration prioritizes preserving paragraph structure over generic markdown formatting, which is more appropriate for extracted PDF text.

**Sources**: [examples/pdf_elements_embedding/main.py:129-145]()

---

## Common Patterns and Best Practices

### GPU Function Configuration

All image processing functions follow this pattern:

```python
@cocoindex.op.function(cache=True, behavior_version=1, gpu=True)
def process_image(img_bytes: bytes) -> OutputType:
    # Load model once via @functools.cache
    model = get_cached_model()
    # Process with torch.no_grad()
    with torch.no_grad():
        result = model(preprocess(img_bytes))
    return result
```

Key elements:
- **`gpu=True`**: Routes to subprocess with GPU access (see [Subprocess Execution Architecture](#12.4))
- **`cache=True`**: Memoizes results in PostgreSQL state DB
- **`behavior_version`**: Enables cache invalidation when function logic changes
- **Model caching**: Use `@functools.cache` to avoid reloading models per invocation
- **No gradient**: Always use `torch.no_grad()` for inference

**Sources**: [examples/image_search/main.py:42-54](), [examples/face_recognition/main.py:78-88](), [examples/pdf_elements_embedding/main.py:28-38]()

### Qdrant Connection Management

Two patterns for Qdrant configuration:

**1. Direct connection in target spec:**
```python
cocoindex.targets.Qdrant(collection_name=QDRANT_COLLECTION)
```

**2. Shared connection via auth registry:**
```python
qdrant_connection = cocoindex.add_auth_entry(
    "qdrant_connection",
    cocoindex.targets.QdrantConnection(grpc_url=QDRANT_GRPC_URL),
)

cocoindex.targets.Qdrant(
    connection=qdrant_connection,
    collection_name=collection_name,
)
```

The second pattern enables connection reuse across multiple exports and supports authentication management (see [Authentication and Secrets Management](#10.1)).

**Sources**: [examples/image_search/main.py:106-110](), [examples/pdf_elements_embedding/main.py:104-180]()

### Static File Serving with FastAPI

All FastAPI examples serve images via StaticFiles middleware:

```python
app.mount("/img", StaticFiles(directory="img"), name="img")
```

This enables:
- Direct image access via URLs like `http://localhost:8080/img/photo.jpg`
- Frontend applications can display search results by constructing URLs from filenames
- No need for separate image serving infrastructure
- Works seamlessly with live updates (new files appear immediately)

Combined with CORS middleware, this creates a complete API for image search applications.

**Sources**: [examples/image_search/main.py:138-139](), [examples/image_search/colpali_main.py:98-99]()

---

## Summary: Multimodal Embedding Capabilities

| Capability | Implementation | Use Case |
|------------|---------------|----------|
| **Image-Text Search** | CLIP embeddings with `embed_image()` and `embed_query()` | Find images by text description |
| **Document Understanding** | ColPali multi-vector embeddings | Search within PDFs, diagrams, layouts |
| **Face Similarity** | Face Recognition + custom embeddings | Identity matching, face clustering |
| **Mixed Content** | Dual-stream processing with separate collections | PDFs with text and images |
| **Live Sync** | `FlowLiveUpdater` with `refresh_interval` | Real-time index updates |
| **Production API** | FastAPI with lifespan management | Scalable search services |
| **Transform Flows** | `@transform_flow()` for query/index consistency | Guaranteed embedding compatibility |

All patterns leverage CocoIndex's incremental processing (see [Incremental Processing and State Management](#9)) to efficiently handle large image collections without reprocessing unchanged data.

---

# Page: Production Deployment: FastAPI and Docker

# Production Deployment: FastAPI and Docker

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [examples/amazon_s3_embedding/pyproject.toml](examples/amazon_s3_embedding/pyproject.toml)
- [examples/code_embedding/pyproject.toml](examples/code_embedding/pyproject.toml)
- [examples/custom_output_files/main.py](examples/custom_output_files/main.py)
- [examples/docs_to_knowledge_graph/pyproject.toml](examples/docs_to_knowledge_graph/pyproject.toml)
- [examples/face_recognition/main.py](examples/face_recognition/main.py)
- [examples/fastapi_server_docker/main.py](examples/fastapi_server_docker/main.py)
- [examples/fastapi_server_docker/requirements.txt](examples/fastapi_server_docker/requirements.txt)
- [examples/gdrive_text_embedding/pyproject.toml](examples/gdrive_text_embedding/pyproject.toml)
- [examples/image_search/colpali_main.py](examples/image_search/colpali_main.py)
- [examples/image_search/main.py](examples/image_search/main.py)
- [examples/image_search/pyproject.toml](examples/image_search/pyproject.toml)
- [examples/manuals_llm_extraction/main.py](examples/manuals_llm_extraction/main.py)
- [examples/manuals_llm_extraction/pyproject.toml](examples/manuals_llm_extraction/pyproject.toml)
- [examples/paper_metadata/main.py](examples/paper_metadata/main.py)
- [examples/paper_metadata/pyproject.toml](examples/paper_metadata/pyproject.toml)
- [examples/patient_intake_extraction/pyproject.toml](examples/patient_intake_extraction/pyproject.toml)
- [examples/pdf_elements_embedding/main.py](examples/pdf_elements_embedding/main.py)
- [examples/pdf_embedding/main.py](examples/pdf_embedding/main.py)
- [examples/pdf_embedding/pyproject.toml](examples/pdf_embedding/pyproject.toml)
- [examples/postgres_source/main.py](examples/postgres_source/main.py)
- [examples/postgres_source/pyproject.toml](examples/postgres_source/pyproject.toml)
- [examples/product_recommendation/pyproject.toml](examples/product_recommendation/pyproject.toml)
- [examples/text_embedding/pyproject.toml](examples/text_embedding/pyproject.toml)
- [examples/text_embedding_qdrant/pyproject.toml](examples/text_embedding_qdrant/pyproject.toml)

</details>



## Purpose and Scope

This page documents how to build production-ready query APIs with CocoIndex using FastAPI and Docker containerization. It covers the `fastapi_server_docker` example, which demonstrates deploying a semantic search API with proper containerization, database configuration, and API endpoint patterns.

For information about the Flow definition and query handlers, see [Query Handlers and Search APIs](#11.2). For backend configuration and authentication, see [Configuration and Settings](#10).

---

## Architecture Overview

The FastAPI Docker deployment follows a multi-container architecture that separates the database layer from the application layer. The system demonstrates two deployment patterns: local development and containerized production.

### System Components

```mermaid
graph TB
    subgraph "Client Layer"
        Client["HTTP Client<br/>(curl, browser, app)"]
    end
    
    subgraph "Application Container"
        FastAPI["FastAPI Application<br/>main:fastapi_app<br/>uvicorn server"]
        QueryHandler["@query_handler<br/>search function"]
        FlowDef["@flow_def<br/>Flow Definition"]
        CocoIndex["CocoIndex Engine<br/>Postgres state mgmt"]
    end
    
    subgraph "Database Container"
        PGVector["PostgreSQL 17<br/>with pgvector extension<br/>coco_db:5436"]
    end
    
    subgraph "Environment Config"
        EnvFile[".env file<br/>COCOINDEX_DATABASE_URL"]
    end
    
    Client -->|"GET /search?q=query&limit=3"| FastAPI
    FastAPI --> QueryHandler
    QueryHandler --> FlowDef
    FlowDef --> CocoIndex
    CocoIndex --> PGVector
    EnvFile -.->|"configures"| FastAPI
    EnvFile -.->|"configures"| CocoIndex
```

**Diagram: FastAPI Docker Deployment Architecture**

The architecture consists of:
- **FastAPI Application**: Serves HTTP requests on port 8000 (local) or 8080 (Docker)
- **Query Handler**: Decorated function that handles semantic search queries
- **CocoIndex Engine**: Manages flow execution and state tracking
- **PostgreSQL with pgvector**: Stores embeddings and provides vector similarity search
- **Environment Configuration**: Controls database connection strings for different environments

Sources: [examples/fastapi_server_docker/README.md:1-60]()

---

## Deployment Patterns

The example supports two deployment patterns with different database connection configurations.

### Local Development Pattern

For local development without Docker, the application connects to a locally-running Postgres instance:

| Configuration | Value |
|--------------|-------|
| Database URL | `postgres://cocoindex:cocoindex@localhost/cocoindex` |
| Application Port | 8000 |
| Server Command | `uvicorn main:fastapi_app --reload --host 0.0.0.0 --port 8000` |
| Database Location | Host machine |

The local pattern uses the `--reload` flag for automatic reloading during development.

### Docker Containerized Pattern

For containerized deployment, the application runs in a Docker container and connects to a separate database container:

| Configuration | Value |
|--------------|-------|
| Database URL | `postgres://cocoindex:cocoindex@coco_db:5436/cocoindex` |
| Application Port | 8080 |
| Database Container | `coco_db` on internal network |
| Database Port | 5436 (mapped from 5432) |
| Orchestration | Docker Compose |

The containerized pattern uses Docker Compose to manage both the application and database containers with proper networking.

Sources: [examples/fastapi_server_docker/README.md:8-53]()

---

## Configuration Management

### Environment Variables

The deployment uses a `.env` file to configure the database connection. The `COCOINDEX_DATABASE_URL` environment variable determines how the application connects to Postgres.

```mermaid
graph LR
    subgraph "Development"
        DevEnv[".env file<br/>localhost URL"]
        DevApp["Application<br/>(local Python)"]
        DevDB["PostgreSQL<br/>(localhost:5432)"]
        
        DevEnv -->|"COCOINDEX_DATABASE_URL"| DevApp
        DevApp --> DevDB
    end
    
    subgraph "Production"
        ProdEnv[".env file<br/>coco_db URL"]
        ProdApp["Application<br/>(Docker container)"]
        ProdDB["PostgreSQL<br/>(coco_db:5436)"]
        
        ProdEnv -->|"COCOINDEX_DATABASE_URL"| ProdApp
        ProdApp --> ProdDB
    end
```

**Diagram: Configuration Pattern Differences**

The configuration follows the three-tier precedence model documented in [Configuration and Settings](#10), where environment variables provide the base configuration layer.

Sources: [examples/fastapi_server_docker/README.md:10-47]()

---

## FastAPI Application Structure

### Dependencies and Requirements

The application requires specific dependencies for production deployment:

| Package | Version | Purpose |
|---------|---------|---------|
| `cocoindex[embeddings]` | >=0.3.9 | Core CocoIndex framework with embedding support |
| `fastapi` | 0.115.12 | Web framework for API endpoints |
| `fastapi-cli` | 0.0.7 | CLI tools for FastAPI |
| `uvicorn` | 0.34.2 | ASGI server for serving FastAPI |
| `psycopg[binary]` | 3.2.6 | PostgreSQL adapter with binary support |
| `psycopg_pool` | 3.2.6 | Connection pooling for Postgres |
| `python-dotenv` | >=1.0.1 | Environment variable loading |

Sources: [examples/fastapi_server_docker/requirements.txt:1-8]()

### API Endpoint Pattern

The query API follows a standard pattern where a FastAPI application exposes query handlers decorated with `@query_handler`:

```mermaid
sequenceDiagram
    participant Client
    participant FastAPI as "FastAPI App"
    participant Handler as "@query_handler<br/>search()"
    participant Embedding as "SentenceTransformer<br/>Embed"
    participant DB as "PostgreSQL<br/>pgvector"
    
    Client->>FastAPI: GET /search?q="query text"&limit=3
    FastAPI->>Handler: Invoke query handler
    Handler->>Embedding: Embed query text
    Embedding-->>Handler: Query vector
    Handler->>DB: Vector similarity search<br/>ORDER BY embedding <=> vector
    DB-->>Handler: Top 3 results
    Handler-->>FastAPI: JSON response
    FastAPI-->>Client: Search results
```

**Diagram: Query Execution Flow**

The query handler:
1. Receives HTTP request parameters (query text and limit)
2. Embeds the query text using the same embedding function from the flow
3. Performs vector similarity search using pgvector's `<=>` operator
4. Returns JSON-formatted results

For detailed information on implementing query handlers, see [Query Handlers and Search APIs](#11.2).

Sources: [examples/fastapi_server_docker/README.md:35-39]()

---

## Docker Containerization

### Container Architecture

The Docker deployment uses Docker Compose to orchestrate multiple services:

```mermaid
graph TB
    subgraph "Docker Compose Network"
        subgraph "Application Service"
            AppContainer["Python Container<br/>Built from Dockerfile"]
            UvicornServer["Uvicorn Server<br/>Port 8000"]
            MainModule["main.py<br/>fastapi_app"]
            
            AppContainer --> UvicornServer
            UvicornServer --> MainModule
        end
        
        subgraph "Database Service"
            DBContainer["pgvector17 Container<br/>coco_db"]
            PostgresDB["PostgreSQL 17<br/>pgvector extension"]
            DataVolume["Persistent Volume"]
            
            DBContainer --> PostgresDB
            PostgresDB --> DataVolume
        end
        
        MainModule -.->|"postgres://...@coco_db:5436"| PostgresDB
    end
    
    HostPort8080["Host Port 8080"] --> AppContainer
    HostPort5436["Host Port 5436"] --> DBContainer
```

**Diagram: Docker Compose Service Architecture**

The Docker Compose configuration defines:
- **Application Service**: Runs the FastAPI application with CocoIndex
- **Database Service**: Runs PostgreSQL 17 with pgvector extension (container name `coco_db`)
- **Network**: Internal Docker network for service communication
- **Port Mappings**: 8080 for API, 5436 for database access

Sources: [examples/fastapi_server_docker/README.md:42-59]()

### Build and Deployment Process

The deployment process follows these steps:

| Phase | Command | Purpose |
|-------|---------|---------|
| Configuration | Edit `.env` file | Set `COCOINDEX_DATABASE_URL` with `coco_db` hostname |
| Index Setup | `cocoindex update main` | Build the vector index (optional pre-step) |
| Container Build | `docker compose up --build` | Build images and start containers |
| Service Start | Docker Compose orchestration | Start both database and application |
| Health Check | `curl "http://0.0.0.0:8080/search?q=model&limit=3"` | Verify API endpoint |

The `--build` flag ensures images are rebuilt with latest code changes.

Sources: [examples/fastapi_server_docker/README.md:42-59]()

---

## Production Considerations

### Port Configuration

The example demonstrates clear port separation between environments:

| Environment | API Port | Database Port | Access Pattern |
|------------|----------|---------------|----------------|
| Local Development | 8000 | 5432 (default) | `localhost:8000` |
| Docker Production | 8080 | 5436 | `0.0.0.0:8080` |

The production configuration uses `0.0.0.0` binding to allow external access, while the database uses a non-standard port (5436) to avoid conflicts.

### Database Connection Pooling

The requirements include `psycopg_pool` for connection pooling, which is critical for production deployments to:
- Manage concurrent request handling
- Reduce connection overhead
- Prevent connection exhaustion
- Improve overall performance

For database connection configuration, see [Configuration and Settings](#10) for `DatabaseConnectionSpec` with `max_connections` and `min_connections` settings.

Sources: [examples/fastapi_server_docker/requirements.txt:6-7](), [examples/fastapi_server_docker/README.md:29-59]()

---

## API Endpoint Specification

### Search Endpoint

The deployment provides a semantic search endpoint with the following specification:

**Endpoint**: `GET /search`

**Query Parameters**:
| Parameter | Type | Required | Description |
|-----------|------|----------|-------------|
| `q` | string | Yes | Search query text to embed and match |
| `limit` | integer | No | Maximum number of results to return (default varies) |

**Response Format**: JSON array of search results

**Example Request**:
```
curl "http://localhost:8000/search?q=model&limit=3"
```

The endpoint internally:
1. Extracts query parameters from the HTTP request
2. Invokes the `@query_handler` decorated function
3. Executes the embedding and similarity search
4. Returns formatted JSON response

Sources: [examples/fastapi_server_docker/README.md:35-59]()

---

## Integration with CocoIndex Flow

### Flow Definition Integration

The FastAPI deployment integrates with a standard CocoIndex flow definition that:

```mermaid
graph LR
    Source["add_source<br/>LocalFile<br/>markdown files"]
    Transform1["transform<br/>SplitRecursively<br/>chunking"]
    Transform2["transform<br/>SentenceTransformer<br/>embedding"]
    Export["export<br/>Postgres + pgvector"]
    Query["@query_handler<br/>search endpoint"]
    
    Source --> Transform1
    Transform1 --> Transform2
    Transform2 --> Export
    Export -.->|"queries"| Query
    Query -.->|"reuses embedding"| Transform2
```

**Diagram: Flow Integration with Query API**

The flow definition (referenced as `main` in commands):
- Defines the indexing pipeline with `@flow_def` decorator
- Specifies source, transformation, and export operations
- Is referenced by `cocoindex update main` command
- Provides embedding functions that query handlers can reuse

The query handler can access the flow's embedding functions to ensure query embeddings match the indexed embeddings, as documented in [Query Handlers and Search APIs](#11.2).

Sources: [examples/fastapi_server_docker/README.md:1-27]()

---

## Operational Workflow

### Complete Deployment Sequence

```mermaid
stateDiagram-v2
    [*] --> Configure: Create .env file
    
    Configure --> InstallDeps: pip install -e .
    note right of InstallDeps
        Install dependencies from
        requirements.txt
    end note
    
    InstallDeps --> BuildIndex: cocoindex update main
    note right of BuildIndex
        Build vector index
        (can be done in container)
    end note
    
    BuildIndex --> StartDocker: docker compose up --build
    note right of StartDocker
        Builds containers
        Starts services
    end note
    
    StartDocker --> Running: Services operational
    
    Running --> Query: curl /search endpoint
    note right of Query
        Test with:
        curl "http://0.0.0.0:8080/search?q=model&limit=3"
    end note
    
    Query --> Running: Continue serving
    Running --> Shutdown: docker compose down
    Shutdown --> [*]
```

**Diagram: Production Deployment State Machine**

This workflow represents the complete lifecycle from initial configuration through deployment and testing. The index can be built either before containerization (for faster startup) or within the container during first run.

Sources: [examples/fastapi_server_docker/README.md:8-59]()

---

## Summary

The FastAPI Docker example demonstrates production-ready deployment patterns for CocoIndex applications:

| Aspect | Implementation |
|--------|----------------|
| Web Framework | FastAPI with uvicorn ASGI server |
| Containerization | Docker Compose with multi-service architecture |
| Database | PostgreSQL 17 with pgvector extension |
| Configuration | Environment-based with `.env` files |
| Query Pattern | `@query_handler` decorators with embedding reuse |
| Port Mapping | Separate ports for development (8000) and production (8080) |

The deployment follows best practices for containerized applications while leveraging CocoIndex's incremental processing and query handler capabilities. For related topics, see [CocoInsight: Visualization and Debugging](#11.3) for monitoring deployed flows, and [Incremental Processing and State Management](#9) for understanding how the system maintains index state.

Sources: [examples/fastapi_server_docker/README.md:1-60](), [examples/fastapi_server_docker/requirements.txt:1-8]()

---

# Page: Custom Operations Examples

# Custom Operations Examples

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [docs/docs/core/data_types.mdx](docs/docs/core/data_types.mdx)
- [examples/custom_output_files/main.py](examples/custom_output_files/main.py)
- [examples/face_recognition/main.py](examples/face_recognition/main.py)
- [examples/fastapi_server_docker/main.py](examples/fastapi_server_docker/main.py)
- [examples/image_search/colpali_main.py](examples/image_search/colpali_main.py)
- [examples/image_search/main.py](examples/image_search/main.py)
- [examples/manuals_llm_extraction/main.py](examples/manuals_llm_extraction/main.py)
- [examples/paper_metadata/main.py](examples/paper_metadata/main.py)
- [examples/pdf_elements_embedding/main.py](examples/pdf_elements_embedding/main.py)
- [examples/pdf_embedding/main.py](examples/pdf_embedding/main.py)
- [examples/postgres_source/main.py](examples/postgres_source/main.py)
- [examples/postgres_source/pyproject.toml](examples/postgres_source/pyproject.toml)
- [python/cocoindex/op.py](python/cocoindex/op.py)
- [python/cocoindex/tests/test_typing.py](python/cocoindex/tests/test_typing.py)
- [python/cocoindex/typing.py](python/cocoindex/typing.py)

</details>



This page demonstrates how to implement custom operations in CocoIndex: custom functions, source connectors, and target connectors. These examples show the patterns and best practices for extending CocoIndex with domain-specific logic.

For information about built-in operations, see [Built-in Processing Functions](#6) and [Built-in Data Sources](#5). For the theoretical foundations of the operation system, see [Operation System and Type Marshalling](#4).

---

## Overview of Custom Operations

CocoIndex provides three extension points for custom operations:

| Operation Type | Decorator | Use Case | Complexity |
|---------------|-----------|----------|------------|
| **Function** | `@function` or `@executor_class` | Transform data during flow execution | Low to High |
| **Source Connector** | `@source_connector` | Read data from custom sources | Medium |
| **Target Connector** | `@target_connector` | Export data to custom destinations | Medium to High |

All custom operations must provide type annotations for proper schema inference and validation. The type system bridges Python's dynamic typing with the Rust engine's static typing (see [Type System and Schema Analysis](#4.2) for details).

Sources: [python/cocoindex/op.py:40-89]()

---

## Custom Functions

### Simple Functions with @function Decorator

The `@function` decorator is the simplest way to create custom transformation functions. It wraps a plain Python function and registers it with the engine.

```mermaid
graph LR
    A["@function decorator"] --> B["Function Registration"]
    B --> C["Type Analysis"]
    C --> D["Encoder/Decoder Creation"]
    D --> E["Engine Registration"]
    
    F["Python Function"] --> A
    G["Type Annotations"] --> C
    H["OpArgs Options"] --> A
```

**Pattern: Simple Transformation Function**

```python
@cocoindex.op.function()
def calculate_total_value(price: float, amount: int) -> float:
    return price * amount
```

The decorator analyzes the function signature and automatically:
1. Converts the function name from snake_case to CamelCase for the operation kind
2. Extracts parameter types and return type
3. Creates encoders/decoders for marshalling data across the FFI boundary
4. Registers the operation with the Rust engine

Sources: [python/cocoindex/op.py:489-512](), [examples/postgres_source/main.py:14-19]()

**Example: String Processing**

```python
@cocoindex.op.function()
def make_full_description(
    category: str,
    name: str,
    description: str,
) -> str:
    return f"Category: {category}\nName: {name}\n\n{description}"
```

Sources: [examples/postgres_source/main.py:23-28]()

**Example: Complex Return Types**

```python
@dataclasses.dataclass
class ImageRect:
    min_x: int
    min_y: int
    max_x: int
    max_y: int

@dataclasses.dataclass
class FaceBase:
    rect: ImageRect
    image: bytes

@cocoindex.op.function(
    cache=True,
    behavior_version=1,
    gpu=True,
    arg_relationship=(cocoindex.op.ArgRelationship.RECTS_BASE_IMAGE, "content"),
)
def extract_faces(content: bytes) -> list[FaceBase]:
    """Extract faces from an image."""
    # Implementation details...
```

This example demonstrates:
- Returning complex structured types (dataclasses)
- Using `OpArgs` for caching, GPU execution, and behavior versioning
- Specifying argument relationships for incremental processing optimization

Sources: [examples/face_recognition/main.py:15-75](), [python/cocoindex/op.py:134-156]()

---

### Complex Functions with @executor_class Decorator

For functions requiring initialization, state management, or GPU models, use the `@executor_class` pattern with a spec class and executor class.

```mermaid
graph TB
    subgraph "Definition Time"
        A["FunctionSpec subclass"] --> B["@executor_class decorator"]
        C["Executor class with spec field"] --> B
        D["OpArgs configuration"] --> B
    end
    
    subgraph "Registration"
        B --> E["analyze_schema()"]
        E --> F["Type validation"]
        F --> G["prepare()"]
    end
    
    subgraph "Execution Time"
        G --> H["__call__() per invocation"]
        H --> I["Result encoding"]
    end
```

**Pattern: Spec Class + Executor Class**

```python
class PdfToMarkdown(cocoindex.op.FunctionSpec):
    """Convert a PDF to markdown."""

@cocoindex.op.executor_class(gpu=True, cache=True, behavior_version=1)
class PdfToMarkdownExecutor:
    spec: PdfToMarkdown
    _converter: PdfConverter

    def prepare(self) -> None:
        """Called once before any execution."""
        config_parser = ConfigParser({})
        self._converter = PdfConverter(
            create_model_dict(), config=config_parser.generate_config_dict()
        )

    def __call__(self, content: bytes) -> str:
        """Called for each input."""
        with tempfile.NamedTemporaryFile(delete=True, suffix=".pdf") as temp_file:
            temp_file.write(content)
            temp_file.flush()
            text_any, _, _ = text_from_rendered(self._converter(temp_file.name))
            return text_any
```

**Key components:**

1. **Spec class**: Inherits from `FunctionSpec`, defines configuration parameters
2. **Executor class**: 
   - Must have a `spec` field with type annotation pointing to the spec class
   - `prepare()` method (optional): Initialize expensive resources once
   - `__call__()` method: Execute the transformation for each input
3. **OpArgs**: Passed to `@executor_class` decorator

Sources: [examples/pdf_embedding/main.py:15-37](), [examples/manuals_llm_extraction/main.py:12-34](), [python/cocoindex/op.py:448-475]()

**GPU Execution with Subprocess Isolation**

When `gpu=True` is specified in `OpArgs`, the executor runs in a separate subprocess with GPU resources. This provides:
- Isolation from the main process
- Automatic recovery from GPU memory issues
- Process pooling for efficiency

Sources: [python/cocoindex/op.py:204-209](), [python/cocoindex/subprocess_exec.py]()

**Example: Image Embedding with GPU**

```python
@cocoindex.op.function(cache=True, behavior_version=1, gpu=True)
def embed_image(
    img_bytes: bytes,
) -> cocoindex.Vector[cocoindex.Float32, Literal[768]]:
    """Convert image to embedding using CLIP model."""
    model, processor = get_clip_model()
    image = Image.open(io.BytesIO(img_bytes)).convert("RGB")
    inputs = processor(images=image, return_tensors="pt")
    with torch.no_grad():
        features = model.get_image_features(**inputs)
    return features[0].tolist()
```

Note the return type annotation `cocoindex.Vector[cocoindex.Float32, Literal[768]]` specifying both element type and dimension for proper vector index creation in targets.

Sources: [examples/image_search/main.py:42-54]()

---

## Custom Source Connectors

Custom source connectors enable reading data from arbitrary data sources. They must implement an iterator-based API for listing and fetching data.

```mermaid
graph TB
    subgraph "Source Connector Components"
        A["SourceSpec subclass"] --> B["Connector Class"]
        B --> C["create() class method"]
        C --> D["Executor Instance"]
        D --> E["list() method"]
        D --> F["get_value() method"]
        D --> G["provides_ordinal() method"]
    end
    
    subgraph "Engine Integration"
        E --> H["AsyncIterator of PartialSourceRow"]
        F --> I["PartialSourceRowData"]
        H --> J["Key Encoder"]
        I --> K["Value Encoder"]
        J --> L["Rust Engine"]
        K --> L
    end
```

**Architecture:**

1. **SourceSpec**: Defines configuration (e.g., connection details, filters)
2. **Connector Class**: Decorated with `@source_connector`, provides a factory
3. **Executor Instance**: Created per flow execution, implements data access methods

Sources: [python/cocoindex/op.py:516-767]()

**Required Methods:**

| Method | Signature | Purpose | Returns |
|--------|-----------|---------|---------|
| `create()` | `(spec: SourceSpec) -> Executor` | Factory method, can be async | Executor instance |
| `list()` | `(options: SourceReadOptions) -> Iterator/AsyncIterator` | List all rows | `PartialSourceRow[K, V]` |
| `get_value()` | `(key: K, options: SourceReadOptions) -> PartialSourceRowData[V]` | Fetch single row | Row data or `NON_EXISTENCE` |
| `provides_ordinal()` | `() -> bool` | Whether ordinals are available | Boolean |

**Example: Custom File Source Connector**

Here's a conceptual example showing the structure (not from the provided files, but following the pattern):

```python
class CustomFileSource(cocoindex.op.SourceSpec):
    """Read files from a custom location."""
    base_path: str
    pattern: str

@dataclasses.dataclass
class FileData:
    """Value type for file source."""
    content: str
    size: int

@cocoindex.op.source_connector(
    spec_cls=CustomFileSource,
    key_type=str,  # filename
    value_type=FileData,
)
class CustomFileSourceConnector:
    spec: CustomFileSource
    
    @classmethod
    def create(cls, spec: CustomFileSource):
        """Factory method."""
        instance = cls()
        instance.spec = spec
        return instance
    
    def provides_ordinal(self) -> bool:
        """Ordinals not supported for this source."""
        return False
    
    def list(
        self, options: cocoindex.op.SourceReadOptions
    ) -> Iterator[cocoindex.op.PartialSourceRow[str, FileData]]:
        """Iterate over all files."""
        import glob
        for filepath in glob.glob(f"{self.spec.base_path}/{self.spec.pattern}"):
            filename = os.path.basename(filepath)
            if options.include_value:
                with open(filepath, 'r') as f:
                    content = f.read()
                value = FileData(content=content, size=os.path.getsize(filepath))
            else:
                value = None
            
            yield cocoindex.op.PartialSourceRow(
                key=filename,
                data=cocoindex.op.PartialSourceRowData(value=value)
            )
    
    def get_value(
        self, key: str, options: cocoindex.op.SourceReadOptions
    ) -> cocoindex.op.PartialSourceRowData[FileData]:
        """Fetch a specific file."""
        filepath = os.path.join(self.spec.base_path, key)
        if not os.path.exists(filepath):
            return cocoindex.op.PartialSourceRowData(
                value=cocoindex.op.NON_EXISTENCE
            )
        
        if options.include_value:
            with open(filepath, 'r') as f:
                content = f.read()
            value = FileData(content=content, size=os.path.getsize(filepath))
        else:
            value = None
        
        return cocoindex.op.PartialSourceRowData(value=value)
```

**Key Concepts:**

- **PartialSourceRow**: Contains `key` and `data` (value, ordinal, content_version_fp)
- **SourceReadOptions**: Hints about what data to include (ordinal, content_version_fp, value)
- **NON_EXISTENCE**: Special value indicating a row no longer exists
- **Type Parameters**: `key_type` and `value_type` determine the schema of the source table

Sources: [python/cocoindex/op.py:520-546](), [python/cocoindex/op.py:555-574](), [python/cocoindex/op.py:669-744]()

---

## Custom Target Connectors

Custom target connectors enable exporting data to arbitrary destinations. The example below demonstrates exporting markdown files as HTML to the local filesystem.

```mermaid
graph TB
    subgraph "Target Connector Lifecycle"
        A["TargetSpec subclass"] --> B["@target_connector decorator"]
        C["Connector Class"] --> B
        B --> D["get_persistent_key()"]
        D --> E["Setup Phase"]
        E --> F["apply_setup_change()"]
        F --> G["Execution Phase"]
        G --> H["prepare()"]
        H --> I["mutate()"]
    end
    
    subgraph "Setup Operations"
        F --> J["Create resources"]
        F --> K["Drop resources"]
    end
    
    subgraph "Mutation Operations"
        I --> L["Batch insertions"]
        I --> M["Batch updates"]
        I --> N["Batch deletions"]
    end
```

**Complete Example: Local File Target**

This example exports collected data as HTML files to a local directory:

```python
class LocalFileTarget(cocoindex.op.TargetSpec):
    """Represents the custom target spec."""
    directory: str

@dataclasses.dataclass
class LocalFileTargetValues:
    """Represents value fields of exported data. Used in `mutate` method."""
    html: str

@cocoindex.op.target_connector(spec_cls=LocalFileTarget)
class LocalFileTargetConnector:
    @staticmethod
    def get_persistent_key(spec: LocalFileTarget, target_name: str) -> str:
        """Use the directory path as the persistent key for this target."""
        return spec.directory

    @staticmethod
    def describe(key: str) -> str:
        """(Optional) Return a human-readable description of the target."""
        return f"Local directory {key}"

    @staticmethod
    def apply_setup_change(
        key: str, 
        previous: LocalFileTarget | None, 
        current: LocalFileTarget | None
    ) -> None:
        """
        Apply setup changes to the target.
        Best practice: keep all actions idempotent.
        """
        # Create the directory if it didn't exist.
        if previous is None and current is not None:
            os.makedirs(current.directory, exist_ok=True)

        # Delete the directory with its contents if it no longer exists.
        if previous is not None and current is None:
            if os.path.isdir(previous.directory):
                for filename in os.listdir(previous.directory):
                    if filename.endswith(".html"):
                        os.remove(os.path.join(previous.directory, filename))
                os.rmdir(previous.directory)

    @staticmethod
    def prepare(spec: LocalFileTarget) -> LocalFileTarget:
        """
        (Optional) Prepare for execution. Run common operations before mutations.
        The returned value will be passed as the first element of tuples in `mutate` method.
        
        If not provided, will directly pass the spec to `mutate` method.
        """
        return spec

    @staticmethod
    def mutate(
        *all_mutations: tuple[LocalFileTarget, dict[str, LocalFileTargetValues | None]],
    ) -> None:
        """
        Mutate the target.
        
        The first element of the tuple is the target spec.
        The second element is a dictionary of mutations:
        - The key is the filename, and the value is the mutation.
        - If the value is `None`, the file will be removed.
          Otherwise, the file will be written with the content.
        
        Best practice: keep all actions idempotent.
        """
        for spec, mutations in all_mutations:
            for filename, mutation in mutations.items():
                full_path = os.path.join(spec.directory, filename) + ".html"
                if mutation is None:
                    try:
                        os.remove(full_path)
                    except FileNotFoundError:
                        pass
                else:
                    with open(full_path, "w") as f:
                        f.write(mutation.html)
```

Sources: [examples/custom_output_files/main.py:11-95]()

**Using the Custom Target in a Flow:**

```python
@cocoindex.op.function()
def markdown_to_html(text: str) -> str:
    return _markdown_it.render(text)

@cocoindex.flow_def(name="CustomOutputFiles")
def custom_output_files(
    flow_builder: cocoindex.FlowBuilder, 
    data_scope: cocoindex.DataScope
) -> None:
    data_scope["documents"] = flow_builder.add_source(
        cocoindex.sources.LocalFile(path="data", included_patterns=["*.md"]),
        refresh_interval=timedelta(seconds=5),
    )

    output_html = data_scope.add_collector()
    with data_scope["documents"].row() as doc:
        doc["html"] = doc["content"].transform(markdown_to_html)
        output_html.collect(filename=doc["filename"], html=doc["html"])

    output_html.export(
        "OutputHtml",
        LocalFileTarget(directory="output_html"),
        primary_key_fields=["filename"],
    )
```

Sources: [examples/custom_output_files/main.py:97-123]()

---

## Target Connector Method Details

### get_persistent_key()

Returns a unique identifier for the target instance. This key:
- Must be stable across flow definition changes
- Is used to track target state in the internal database
- Determines whether a target is "the same" across setup operations

**Example:**
```python
@staticmethod
def get_persistent_key(spec: LocalFileTarget, target_name: str) -> str:
    return spec.directory  # Directory path uniquely identifies this target
```

Sources: [python/cocoindex/op.py:948-955](), [examples/custom_output_files/main.py:27-30]()

### apply_setup_change()

Called during `flow.setup()` and `flow.drop()` to create or destroy target resources. Receives:
- `key`: The persistent key
- `previous`: Previous spec (None if creating new target)
- `current`: Current spec (None if dropping target)

**Best Practices:**
- Make all operations idempotent
- Handle three scenarios: creation, update, deletion
- Clean up resources when `current is None`

Sources: [python/cocoindex/op.py:1001-1038](), [examples/custom_output_files/main.py:37-57]()

### prepare()

Optional method called once before processing any mutations. Use it for:
- Opening database connections
- Initializing clients
- Expensive setup operations

The return value is passed as the first element of tuples in `mutate()`.

Sources: [python/cocoindex/op.py:1040-1062](), [examples/custom_output_files/main.py:59-67]()

### mutate()

Core method that applies data mutations to the target. Receives batches of mutations as varargs.

**Signature:**
```python
def mutate(
    *all_mutations: tuple[PreparedSpec, dict[str, ValueStruct | None]]
) -> None
```

**Each tuple contains:**
1. **PreparedSpec**: Return value from `prepare()` (or spec if prepare() not defined)
2. **Mutations dict**: Maps primary key to value struct or None
   - `None` value → delete the row
   - Non-None value → insert or update the row

**Type Annotations:**

The `mutate()` method's type annotation determines how keys and values are decoded:

```python
@staticmethod
def mutate(
    *all_mutations: tuple[LocalFileTarget, dict[str, LocalFileTargetValues | None]],
) -> None:
```

This annotation tells the engine:
- Keys are strings (from `dict[str, ...]`)
- Values are `LocalFileTargetValues` structs
- `None` values represent deletions

Sources: [python/cocoindex/op.py:1064-1115](), [examples/custom_output_files/main.py:69-94](), [python/cocoindex/op.py:863-904]()

---

## Type System Integration

Custom operations must properly annotate types for the engine to infer schemas and create encoders/decoders.

```mermaid
graph LR
    subgraph "Python Side"
        A["Type Annotation"] --> B["analyze_type_info()"]
        B --> C["DataTypeInfo"]
    end
    
    subgraph "FFI Boundary"
        C --> D["make_encoder()"]
        C --> E["make_decoder()"]
        D --> F["Python → Rust"]
        E --> G["Rust → Python"]
    end
    
    subgraph "Engine Side"
        F --> H["EnrichedValueType"]
        H --> I["Schema Validation"]
        G --> J["Value Conversion"]
    end
```

**Return Type Requirements:**

For return values (where Python provides the ground truth), use specific type annotations:
- Primitive types: `str`, `int`, `float`, `bytes`, `bool`
- Annotated types: `cocoindex.Int64`, `cocoindex.Float32`, `cocoindex.Vector[T, Literal[Dim]]`
- Structured types: `@dataclass`, `NamedTuple`, or Pydantic models
- Collections: `list[T]` for LTable, `dict[K, V]` for KTable

**Argument Type Flexibility:**

For arguments (where the engine knows the type), you can use:
- Specific types (same as return types)
- `dict[str, Any]` for any Struct type
- `Any` to skip validation

**Example with Various Type Annotations:**

```python
from typing import Literal
import numpy as np
from numpy.typing import NDArray

@dataclasses.dataclass
class ProcessedData:
    text: str
    confidence: float
    embedding: NDArray[np.float32]

@cocoindex.op.function(behavior_version=1)
def process_with_types(
    # Argument types can be flexible
    raw_data: dict[str, Any],
) -> ProcessedData:
    # Return type must be specific
    return ProcessedData(
        text=raw_data["content"],
        confidence=0.95,
        embedding=np.zeros(768, dtype=np.float32)
    )
```

Sources: [python/cocoindex/op.py:213-351](), [docs/docs/core/data_types.mdx:24-36]()

---

## OpArgs Configuration

The `OpArgs` dataclass configures execution behavior for custom functions:

| Field | Type | Purpose | Default |
|-------|------|---------|---------|
| `gpu` | `bool` | Execute in GPU subprocess | `False` |
| `cache` | `bool` | Enable result caching | `False` |
| `batching` | `bool` | Enable batch processing | `False` |
| `max_batch_size` | `int \| None` | Maximum batch size | `None` |
| `behavior_version` | `int \| None` | Version for cache invalidation | `None` |
| `timeout` | `timedelta \| None` | Execution timeout | `None` |
| `arg_relationship` | `tuple \| None` | Declare input-output relationships | `None` |

**Example with Multiple Options:**

```python
@cocoindex.op.function(
    gpu=True,                  # Run in GPU subprocess
    cache=True,                # Cache results
    behavior_version=1,        # Invalidate cache on version change
    arg_relationship=(         # Declare relationship for optimization
        cocoindex.op.ArgRelationship.EMBEDDING_ORIGIN_TEXT,
        "text"
    )
)
def embed_with_context(text: str) -> cocoindex.Vector[cocoindex.Float32, Literal[768]]:
    # Implementation...
    pass
```

**ArgRelationship Types:**

- `EMBEDDING_ORIGIN_TEXT`: Output is embedding of input text
- `CHUNKS_BASE_TEXT`: Output chunks are derived from input text
- `RECTS_BASE_IMAGE`: Output rectangles are from input image

These relationships enable incremental processing optimizations by tracking data lineage.

Sources: [python/cocoindex/op.py:126-156](), [examples/face_recognition/main.py:34-39]()

---

## Summary: When to Use Each Pattern

| Pattern | Use When | Complexity | Example |
|---------|----------|------------|---------|
| `@function` | Simple stateless transformation | Low | String formatting, calculations |
| `@executor_class` | Need initialization or GPU models | Medium | PDF conversion, ML models |
| `@source_connector` | Reading from custom data sources | Medium | File systems, APIs, databases |
| `@target_connector` | Exporting to custom destinations | High | Custom file formats, APIs, systems |

**Performance Considerations:**

- Use `cache=True` for expensive operations (LLM calls, embeddings)
- Use `gpu=True` for ML models to enable subprocess isolation
- Use `batching=True` for operations that benefit from batch processing
- Always specify `behavior_version` when using caching

**Type System Best Practices:**

- Always annotate return types specifically
- Use `Vector[T, Literal[Dim]]` for vectors going to targets
- Use frozen dataclasses or NamedTuples for composite keys
- Prefer structured types over `dict[str, Any]` for documentation

Sources: [python/cocoindex/op.py:1-513](), [examples/custom_output_files/main.py:1-124](), [examples/face_recognition/main.py:1-125]()

---

# Page: Development Environment Setup

# Development Environment Setup

<details>
<summary>Relevant source files</summary>

The following files were used as context for generating this wiki page:

- [.env.lib_debug](.env.lib_debug)
- [.github/workflows/CI.yml](.github/workflows/CI.yml)
- [.github/workflows/_docs_release.yml](.github/workflows/_docs_release.yml)
- [.github/workflows/_test.yml](.github/workflows/_test.yml)
- [.github/workflows/docs_release.yml](.github/workflows/docs_release.yml)
- [.github/workflows/docs_test.yml](.github/workflows/docs_test.yml)
- [.github/workflows/format.yml](.github/workflows/format.yml)
- [.github/workflows/release.yml](.github/workflows/release.yml)
- [.gitignore](.gitignore)
- [.pre-commit-config.yaml](.pre-commit-config.yaml)
- [.python-version](.python-version)
- [CLAUDE.md](CLAUDE.md)
- [dev/run_cargo_test.sh](dev/run_cargo_test.sh)
- [docs/docs/contributing/setup_dev_environment.md](docs/docs/contributing/setup_dev_environment.md)
- [pyproject.toml](pyproject.toml)
- [python/cocoindex/subprocess_exec.py](python/cocoindex/subprocess_exec.py)
- [uv.lock](uv.lock)

</details>



This document provides a comprehensive guide for setting up a local development environment for CocoIndex. It covers tool installation, build system configuration, development workflow, testing procedures, and CI/CD integration. This is intended for contributors who need to modify and test CocoIndex functionality locally.

For information about deploying CocoIndex in production, see [Production Deployment: FastAPI and Docker](#13.6). For details on the build system and cross-platform wheel generation, see [Build System: Maturin and Cross-Platform Wheels](#12.2).

## Prerequisites

CocoIndex is a hybrid Python-Rust project requiring both language toolchains and specialized build tools.

### Required Tools

| Tool | Version | Purpose |
|------|---------|---------|
| **Python** | 3.11+ | Python interface layer and user-facing API |
| **Rust** | stable | Core engine implementation |
| **uv** | latest | Python package and project management |
| **maturin** | 1.10.0+ | Building PyO3 bindings and Python wheels |

**Sources:** [pyproject.toml:11](), [pyproject.toml:2](), [.github/workflows/_test.yml:55-56]()

### Installation

#### Rust Toolchain

```bash
# Install Rust via rustup
curl --proto '=https' --tlsv1.2 -sSf https://sh.rustup.rs | sh

# Ensure toolchain is up to date
rustup update
```

#### uv Package Manager

```bash
# macOS / Linux
curl -LsSf https://astral.sh/uv/install.sh | sh

# Windows
powershell -ExecutionPolicy ByPass -c "irm https://astral.sh/uv/install.ps1 | iex"
```

**Sources:** [docs/docs/contributing/setup_dev_environment.md:8-29]()

## Build System Architecture

CocoIndex uses `maturin` to build Rust code with PyO3 bindings into platform-specific Python wheels. The build system configuration is defined in `pyproject.toml`.

### Maturin Configuration

```mermaid
graph TB
    subgraph "Build Configuration"
        PYPROJECT["pyproject.toml"]
        CARGO["rust/cocoindex/Cargo.toml"]
    end
    
    subgraph "Build Process"
        MATURIN["maturin<br/>(PyO3 bindings)"]
        RUSTC["rustc compiler"]
        LINKER["Platform linker"]
    end
    
    subgraph "Build Outputs"
        DEV_BUILD["Development Build<br/>profile=dev"]
        RELEASE_BUILD["Release Wheel<br/>profile=release"]
        ENGINE_SO["cocoindex._engine.abi3.so"]
    end
    
    PYPROJECT -->|"bindings=pyo3<br/>module-name=cocoindex._engine"| MATURIN
    CARGO -->|"manifest-path"| MATURIN
    
    MATURIN --> RUSTC
    RUSTC --> LINKER
    
    LINKER -->|"maturin develop"| DEV_BUILD
    LINKER -->|"maturin build --release"| RELEASE_BUILD
    
    DEV_BUILD --> ENGINE_SO
    RELEASE_BUILD --> ENGINE_SO
```

**Key Configuration Parameters:**

- **`bindings = "pyo3"`**: Use PyO3 for Python-Rust interop ([pyproject.toml:58]())
- **`module-name = "cocoindex._engine"`**: Python import path for Rust extension ([pyproject.toml:60]())
- **`manifest-path = "rust/cocoindex/Cargo.toml"`**: Location of primary Rust crate ([pyproject.toml:64]())
- **`profile = "release"`**: Optimized builds for wheels ([pyproject.toml:66]())
- **`editable-profile = "dev"`**: Fast builds for local development ([pyproject.toml:67]())

**Sources:** [pyproject.toml:57-67](), [CLAUDE.md:9-13]()

### Build Profiles

The build system uses different optimization profiles depending on the build target:

| Profile | Use Case | Optimization Level | Debug Symbols |
|---------|----------|-------------------|---------------|
| `dev` | Local development with `maturin develop` | Minimal (fast compilation) | Full |
| `release` | Production wheels with `maturin build --release` | Maximum (slower compilation) | Minimal |

**Sources:** [pyproject.toml:66-67]()

## Initial Repository Setup

### Clone and Sync Dependencies

```bash
# Clone the repository
git clone https://github.com/cocoindex-io/cocoindex.git
cd cocoindex

# Sync Python dependencies (without optional extras)
uv sync --no-dev --group ci

# Or sync all dependencies including optional extras
uv sync --all-extras
```

The `uv sync` command installs all dependencies defined in `pyproject.toml` and generates/updates `uv.lock`. The `--group ci` flag includes dependencies needed for testing and type checking ([pyproject.toml:87-91]()).

### Install Pre-commit Hooks

Pre-commit hooks run automated checks before each commit to maintain code quality:

```bash
uv run pre-commit install
```

This installs hooks defined in `.pre-commit-config.yaml`, which will automatically run on `git commit`.

**Sources:** [docs/docs/contributing/setup_dev_environment.md:32-38](), [pyproject.toml:81-98]()

## Development Workflow

### Initial Build

After cloning or making changes to Rust code, build the Python extension:

```bash
uv run maturin develop
```

This command:
1. Compiles the Rust code in `rust/cocoindex/`
2. Generates the `_engine` Python extension module
3. Installs it as an editable package in the current environment

The build must be repeated after any Rust code changes. Python-only changes do not require rebuilding.

**Sources:** [docs/docs/contributing/setup_dev_environment.md:46-52](), [.github/workflows/_test.yml:67-68]()

### Development Cycle

```mermaid
stateDiagram-v2
    [*] --> EditCode: Start development
    
    state "Code Changes" as EditCode {
        [*] --> RustChanges: Modify Rust code
        [*] --> PythonChanges: Modify Python code
    }
    
    EditCode --> RebuildCheck: Ready to test
    
    state RebuildCheck <<choice>>
    RebuildCheck --> MaturinDevelop: Rust changes
    RebuildCheck --> TypeCheck: Python only changes
    
    MaturinDevelop --> CargoTest: Run Rust tests
    CargoTest --> TypeCheck
    
    TypeCheck --> Pytest: Type check passes
    Pytest --> PreCommit: All tests pass
    
    PreCommit --> CommitReady: Hooks pass
    CommitReady --> [*]: git commit
    
    PreCommit --> EditCode: Hooks fail
    CargoTest --> EditCode: Tests fail
    Pytest --> EditCode: Tests fail
    TypeCheck --> EditCode: Type errors
```

**Command Summary:**

| Change Type | Required Commands |
|-------------|-------------------|
| Rust code only | `uv run maturin develop && cargo test` |
| Python code only | `uv run dmypy run && uv run pytest python/` |
| Both Rust and Python | All commands from both categories |

**Sources:** [CLAUDE.md:22-29](), [.github/workflows/_test.yml:61-74]()

## Pre-commit Hook System

The pre-commit configuration at [.pre-commit-config.yaml]() defines a validation pipeline that runs automatically before each commit.

### Hook Execution Pipeline

```mermaid
graph TB
    subgraph "File Validation Hooks"
        CHECK_CASE["check-case-conflict"]
        CHECK_MERGE["check-merge-conflict"]
        CHECK_SYMLINKS["check-symlinks"]
        DETECT_KEY["detect-private-key"]
        EOF["end-of-file-fixer"]
        TRAILING["trailing-whitespace"]
    end
    
    subgraph "Format Hooks"
        CARGO_FMT["cargo fmt<br/>(Rust)"]
        RUFF_FMT["ruff-format<br/>(Python)"]
    end
    
    subgraph "Dependency Hooks"
        UV_LOCK["uv-lock<br/>(sync uv.lock)"]
    end
    
    subgraph "Build and Test Hooks"
        MATURIN["maturin develop<br/>(rebuild extension)"]
        CARGO_TEST["cargo test<br/>(Rust tests)"]
        MYPY["mypy check<br/>(type checking)"]
        PYTEST["pytest<br/>(Python tests)"]
    end
    
    subgraph "Documentation Hooks"
        GEN_CLI_DOCS["generate-cli-docs"]
    end
    
    CHECK_CASE --> CHECK_MERGE
    CHECK_MERGE --> CHECK_SYMLINKS
    CHECK_SYMLINKS --> DETECT_KEY
    DETECT_KEY --> EOF
    EOF --> TRAILING
    
    TRAILING --> CARGO_FMT
    CARGO_FMT --> RUFF_FMT
    
    RUFF_FMT --> UV_LOCK
    UV_LOCK --> MATURIN
    
    MATURIN --> CARGO_TEST
    CARGO_TEST --> MYPY
    MYPY --> PYTEST
    
    PYTEST --> GEN_CLI_DOCS
```

### Hook Details

**File Validation** ([.pre-commit-config.yaml:6-25]()):
- `check-case-conflict`: Prevents case-insensitive filename conflicts
- `check-merge-conflict`: Detects unresolved merge conflict markers
- `check-symlinks`: Validates symlinks point to valid targets
- `detect-private-key`: Prevents committing private keys
- `end-of-file-fixer`: Ensures files end with single newline
- `trailing-whitespace`: Removes trailing whitespace (except in Python, handled by Ruff)

**Formatting** ([.pre-commit-config.yaml:27-41]()):
- `cargo fmt`: Formats Rust code using `rustfmt`
- `ruff-format`: Formats Python code using Ruff formatter

**Dependency Management** ([.pre-commit-config.yaml:43-47]()):
- `uv-lock`: Ensures `uv.lock` is synchronized with `pyproject.toml`

**Build and Test** ([.pre-commit-config.yaml:49-78]()):
- `maturin develop`: Rebuilds Rust extension if Rust or Python files changed
- `cargo test`: Runs Rust unit tests via `./dev/run_cargo_test.sh`
- `mypy check`: Type checks Python code using mypy daemon (`dmypy`)
- `pytest`: Runs Python tests (only if Python files changed)

**Documentation** ([.pre-commit-config.yaml:80-85]()):
- `generate-cli-docs`: Regenerates CLI documentation from `cli.py`

**Sources:** [.pre-commit-config.yaml:1-85]()

### Skipping Hooks

To skip pre-commit hooks temporarily (not recommended):

```bash
git commit --no-verify
```

## Testing

### Test Organization

CocoIndex has two primary test suites:

| Test Suite | Location | Language | Execution Command |
|------------|----------|----------|-------------------|
| Rust unit/integration tests | `rust/**/tests/` | Rust | `cargo test --verbose` |
| Python unit tests | `python/cocoindex/tests/` | Python | `pytest --capture=no python/cocoindex/tests` |

**Sources:** [.github/workflows/_test.yml:61-74]()

### Running Rust Tests

```bash
# Run all Rust tests
uv run cargo test --verbose

# Run tests without default features
uv run cargo test --no-default-features --verbose

# Run tests for specific package
cargo test -p cocoindex_engine

# Run specific test
cargo test test_name
```

**Troubleshooting: ModuleNotFoundError for `encodings`**

On some systems (particularly with `uv`-managed Python installations), `cargo test` may fail with:

```
ModuleNotFoundError: No module named 'encodings'
```

This occurs because the embedded Python interpreter (via PyO3) cannot locate the Python standard library. The error often shows `sys.prefix=/install` in the traceback.

**Solution:** Use the provided wrapper script:

```bash
./dev/run_cargo_test.sh
```

This script ([dev/run_cargo_test.sh:1-73]()) automatically:
1. Detects the active Python interpreter
2. Computes `PYTHONHOME` using `sys.base_prefix`
3. Constructs `PYTHONPATH` including stdlib, site-packages, and `python/` directory
4. Exports these variables before running `cargo test`

**Sources:** [docs/docs/contributing/setup_dev_environment.md:77-93](), [dev/run_cargo_test.sh:1-73]()

### Running Python Tests

```bash
# Run all Python tests
uv run pytest python/cocoindex/tests

# Run with output capture disabled (see print statements)
uv run pytest --capture=no python/cocoindex/tests

# Run specific test file
uv run pytest python/cocoindex/tests/test_flow.py

# Run specific test function
uv run pytest python/cocoindex/tests/test_flow.py::test_simple_flow
```

### Type Checking

CocoIndex uses mypy with strict type checking enabled ([pyproject.toml:103-142]()):

```bash
# Run mypy via daemon (faster for repeated runs)
uv run dmypy run

# Run standard mypy
uv run mypy
```

The mypy configuration ([pyproject.toml:103-142]()):
- **`strict = true`**: Enables all optional checks
- **`files = ["python", "examples"]`**: Checks both library and examples
- **`mypy_path = "python"`**: Allows `import cocoindex` from examples
- **`explicit_package_bases = true`**: Prevents "Duplicate module" errors
- **`namespace_packages = true`**: Supports example subdirectories

**Sources:** [.github/workflows/_test.yml:70-71](), [pyproject.toml:103-142]()

## Running Examples During Development

### Development Environment Configuration

Load development environment variables:

```bash
source .env.lib_debug
```

This sets ([.env.lib_debug:1-22]()):
- **`RUST_LOG=warn,cocoindex_engine=trace,tower_http=trace`**: Detailed Rust logging
- **`RUST_BACKTRACE=1`**: Full stack traces on panics
- **`COCOINDEX_SERVER_CORS_ORIGINS`**: Allowed CORS origins for local development
- **`COCOINDEX_DEV_ROOT`**: Repository root path
- **`coco-dev-run` function**: Wrapper for running examples with local cocoindex

### Using `coco-dev-run`

The `coco-dev-run` function runs examples using your locally built (editable) CocoIndex package instead of the published PyPI version:

```bash
# Navigate to example directory
cd examples/text_embedding

# Run example with local cocoindex
coco-dev-run cocoindex update main
```

**Implementation:**

```bash
# From .env.lib_debug
coco-dev-run() {
    local pyver
    if [ -f "$COCOINDEX_DEV_ROOT/.python-version" ]; then
        pyver="$(cat "$COCOINDEX_DEV_ROOT/.python-version")"
    else
        pyver="3.11"
    fi

    uv run --python "$pyver" --with-editable "$COCOINDEX_DEV_ROOT" "$@"
}
```

This function:
1. Reads the Python version from `.python-version` (defaults to 3.11)
2. Runs the command with `uv run --with-editable` pointing to the repository root
3. Ensures examples use your local code changes

**Sources:** [.env.lib_debug:1-22](), [docs/docs/contributing/setup_dev_environment.md:54-73]()

## CI/CD Pipeline

Understanding the CI/CD pipeline helps contributors anticipate which checks will run on their pull requests.

### Continuous Integration

The CI workflow runs on every pull request and push to main ([.github/workflows/CI.yml:1-31]()):

```mermaid
graph TB
    subgraph "Trigger Events"
        PR["Pull Request to main"]
        PUSH["Push to main"]
        MANUAL["workflow_dispatch"]
    end
    
    subgraph "CI Job: test"
        TEST_MATRIX["Test Matrix"]
    end
    
    subgraph "Test Platforms"
        UBUNTU["ubuntu-latest<br/>(x86_64)"]
        UBUNTU_ARM["ubuntu-24.04-arm<br/>(aarch64)"]
        MACOS["macos-latest<br/>(aarch64)"]
        MACOS_INTEL["macos-15-intel<br/>(x86_64)"]
        WINDOWS["windows-latest<br/>(x64)"]
    end
    
    subgraph "Test Steps"
        CHECKOUT["Checkout code"]
        SETUP_RUST["Setup Rust toolchain"]
        SETUP_PYTHON["Setup Python 3.11"]
        SETUP_UV["Install uv"]
        SYNC_DEPS["uv sync --no-dev --group ci"]
        RUST_TEST_NO_FEAT["cargo test --no-default-features"]
        RUST_TEST["cargo test"]
        PY_BUILD["maturin develop --strip"]
        MYPY["mypy"]
        PYTEST["pytest python/cocoindex/tests"]
    end
    
    PR --> TEST_MATRIX
    PUSH --> TEST_MATRIX
    MANUAL --> TEST_MATRIX
    
    TEST_MATRIX --> UBUNTU
    TEST_MATRIX --> UBUNTU_ARM
    TEST_MATRIX --> MACOS
    TEST_MATRIX --> MACOS_INTEL
    TEST_MATRIX --> WINDOWS
    
    UBUNTU --> CHECKOUT
    CHECKOUT --> SETUP_RUST
    SETUP_RUST --> SETUP_PYTHON
    SETUP_PYTHON --> SETUP_UV
    SETUP_UV --> SYNC_DEPS
    SYNC_DEPS --> RUST_TEST_NO_FEAT
    RUST_TEST_NO_FEAT --> RUST_TEST
    RUST_TEST --> PY_BUILD
    PY_BUILD --> MYPY
    MYPY --> PYTEST
```

**Platform Matrix** ([.github/workflows/_test.yml:14-22]()):
- **ubuntu-latest**: x86_64 Linux (primary platform)
- **ubuntu-24.04-arm**: aarch64 Linux (ARM servers)
- **macos-latest**: Apple Silicon (M1/M2/M3)
- **macos-15-intel**: Intel Mac (legacy support)
- **windows-latest**: Windows x64

**Environment Variables** ([.github/workflows/_test.yml:24-28]()):
- **`RUSTFLAGS="-C debuginfo=0 -C split-debuginfo=off"`**: Reduce build artifacts
- **`CARGO_INCREMENTAL=0`**: Disable incremental compilation
- **`SCCACHE_GHA_ENABLED=true`**: Enable GitHub Actions cache
- **`RUSTC_WRAPPER=sccache`**: Use sccache for compilation caching

**Sources:** [.github/workflows/CI.yml:1-31](), [.github/workflows/_test.yml:1-99]()

### Format Checking

A separate workflow validates code formatting ([.github/workflows/format.yml:1-56]()):

```mermaid
graph LR
    subgraph "Rust Format Check"
        RUST_CHECKOUT["Checkout code"]
        RUST_TOOLCHAIN["Setup Rust toolchain<br/>with rustfmt"]
        RUST_FMT_CHECK["cargo fmt --check"]
    end
    
    subgraph "Python Format Check"
        PY_CHECKOUT["Checkout code"]
        PY_SETUP["Setup Python 3.11"]
        UV_INSTALL["Install uv"]
        RUFF_CHECK["uv run ruff format --check ."]
    end
    
    RUST_CHECKOUT --> RUST_TOOLCHAIN
    RUST_TOOLCHAIN --> RUST_FMT_CHECK
    
    PY_CHECKOUT --> PY_SETUP
    PY_SETUP --> UV_INSTALL
    UV_INSTALL --> RUFF_CHECK
```

These checks run independently and must both pass. Formatting failures block merging.

**Sources:** [.github/workflows/format.yml:1-56]()

### Release Pipeline

The release workflow ([.github/workflows/release.yml:1-164]()) builds cross-platform wheels and publishes to PyPI:

1. **Generate Third-Party Notices**: Uses `cargo-about` to generate license notices
2. **Build**: Compiles wheels for 5 platform combinations (Linux x86_64/aarch64, macOS x86_64/aarch64, Windows x64)
3. **Test ABI3**: Validates wheels work across Python 3.11, 3.12, 3.13
4. **Build Source Distribution**: Creates sdist for pip
5. **Release**: Publishes to PyPI (only on tag push)
6. **Release Docs**: Deploys documentation to GitHub Pages

**Platform-Specific Containers** ([.github/workflows/release.yml:48-53]()):
- Linux builds use `ghcr.io/rust-cross/manylinux_2_28-cross` for manylinux compatibility
- macOS and Windows use native runners

**Sources:** [.github/workflows/release.yml:1-164]()

## Build Artifacts and Generated Files

### Development Build Output

After running `uv run maturin develop`, the following artifacts are created:

| Path | Description |
|------|-------------|
| `python/cocoindex/_engine.abi3.so` | Compiled Rust extension (Linux/macOS) |
| `python/cocoindex/_engine.abi3.pyd` | Compiled Rust extension (Windows) |
| `python/cocoindex.egg-info/` | Editable install metadata |
| `target/debug/` | Rust debug build artifacts |

### Release Build Output

Running `uv run maturin build --release` produces:

| Path | Description |
|------|-------------|
| `dist/*.whl` | Platform-specific Python wheel |
| `target/release/` | Rust release build artifacts |

**Sources:** [pyproject.toml:57-67]()

## Development Environment Variables

### Required Variables

CocoIndex requires PostgreSQL for state management:

```bash
export COCOINDEX_DATABASE_URL="postgresql://user:pass@localhost:5432/dbname"
```

### Optional Debug Variables

For enhanced debugging and local development:

```bash
# Rust logging
export RUST_LOG=warn,cocoindex_engine=trace,tower_http=trace
export RUST_BACKTRACE=1

# CORS for local CocoInsight UI
export COCOINDEX_SERVER_CORS_ORIGINS=http://localhost:3000

# Application namespace (for multi-tenant development)
export COCOINDEX_APP_NAMESPACE=dev
```

**Sources:** [.env.lib_debug:1-4]()

## Dependency Groups

The `pyproject.toml` defines several dependency groups for different purposes:

| Group | Purpose | Key Dependencies |
|-------|---------|------------------|
| `build-test` | Basic testing | maturin, pytest, pytest-asyncio, mypy |
| `format` | Code formatting | ruff |
| `type-stubs` | Type checking | types-psutil |
| `ci-enabled-optional-deps` | Optional deps for CI | pydantic |
| `ci` | All CI dependencies | Includes build-test, ci-enabled-optional-deps, type-stubs |
| `dev-local` | Local-only tools | pre-commit |
| `dev` | Full development | Includes ci, format, dev-local |

**Install specific groups:**

```bash
# Install CI dependencies
uv sync --group ci

# Install all development dependencies
uv sync --group dev

# Install with optional features
uv sync --all-extras --group dev
```

**Sources:** [pyproject.toml:81-98]()

## Optional Features

CocoIndex has optional dependencies for specific functionality:

| Feature | Dependencies | Use Case |
|---------|-------------|----------|
| `embeddings` | sentence-transformers>=3.3.1 | Text embedding functions |
| `colpali` | colpali-engine | Multimodal document embeddings |
| `lancedb` | lancedb>=0.25.0, pyarrow>=19.0.0 | LanceDB vector storage |
| `all` | All of the above | Full feature set |

**Install with specific features:**

```bash
# Install embedding support only
pip install cocoindex[embeddings]

# Install all features
pip install cocoindex[all]

# Development with all features
uv sync --all-extras --group dev
```

**Sources:** [pyproject.toml:69-79]()

## Summary

The CocoIndex development environment centers on three core tools:
- **uv** for Python dependency management
- **maturin** for building PyO3 bindings
- **pre-commit** for automated validation

The typical development workflow is:
1. Edit code (Rust and/or Python)
2. Run `uv run maturin develop` if Rust changed
3. Run tests (`cargo test`, `pytest`)
4. Commit (pre-commit hooks run automatically)

The CI/CD pipeline validates code across 5 platforms (Linux x86_64/aarch64, macOS x86_64/aarch64, Windows x64) and 3 Python versions (3.11, 3.12, 3.13), ensuring broad compatibility.

**Sources:** [CLAUDE.md:1-76](), [docs/docs/contributing/setup_dev_environment.md:1-94]()
