# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

OpenEvolve is an open-source implementation of Google DeepMind's AlphaEvolve system - an evolutionary coding agent that uses LLMs to optimize code through iterative evolution. The framework can evolve code in multiple languages (Python, R, Rust, Metal shaders, etc.) for tasks like scientific computing, optimization, GPU kernel optimization, and algorithm discovery.

## Technology Stack & External Services

### Core Dependencies

**Python Runtime**
- Python >=3.10 required
- Build: setuptools>=42, wheel

**Core Libraries**
- `openai>=1.0.0` - Universal LLM client (works with OpenAI, Gemini, local models)
- `pyyaml>=6.0` - Configuration management
- `numpy>=1.22.0` - Numerical operations and feature calculations
- `tqdm>=4.64.0` - Progress bars for evolution iterations
- `flask` - Visualization web server

**Development Dependencies**
- `pytest>=7.0.0`, `pytest-asyncio>=0.21.0` - Testing framework
- `black>=22.0.0` - Code formatter (100 char line length)
- `isort>=5.10.0` - Import sorting
- `mypy>=0.950` - Type checking
- `requests>=2.28.0` - HTTP client for integration tests

### LLM API Providers (OpenAI-compatible)

OpenEvolve uses a universal OpenAI-compatible client that works with multiple providers:

**Supported Providers**
1. **OpenAI** (api.openai.com/v1)
   - Models: gpt-4, gpt-4-turbo, gpt-3.5-turbo, o1, o3, o3-mini
   - Authentication: OPENAI_API_KEY environment variable
   - Docs: https://platform.openai.com/docs

2. **Google Gemini** (generativelanguage.googleapis.com/v1beta/openai/)
   - Models: gemini-2.0-flash, gemini-2.0-flash-lite, gemini-2.5-pro, gemini-2.5-flash
   - Authentication: OPENAI_API_KEY (with Gemini API key)
   - Docs: https://ai.google.dev/gemini-api/docs

3. **Cerebras** (api.cerebras.ai/v1)
   - Fast inference cloud service
   - Models: Various Llama models
   - Docs: https://cerebras.ai/

4. **Local Models via Ollama** (localhost:11434/v1)
   - Models: llama3.1, codellama, mistral, qwen, etc.
   - Self-hosted, no API key required
   - Docs: https://ollama.ai/

5. **OptiLLM Proxy** (localhost:8000/v1)
   - Advanced routing, rate limiting, test-time compute
   - Supports mixture-of-agents, web search integration
   - Docs: https://github.com/codelion/optillm

**LLM Configuration**
- Set `llm.api_base` in config.yaml to change provider
- Use `llm.models` for ensemble with weights
- Separate `llm.evaluator_models` for code feedback

### Optional Services & Tools

**OptiLLM Integration**
- Install: `pip install optillm`
- Use for: Model routing, fallbacks, test-time compute enhancements
- Example config: `examples/web_scraper_optillm/config.yaml`

**Visualization Dependencies**
- Flask (included in core)
- Additional: See `scripts/requirements.txt`

**Example-Specific Dependencies**
Each example may require additional libraries:
- MLX: Apple Silicon GPU computing (`examples/mlx_metal_kernel_opt/`)
- R: Statistical computing (`examples/r_robust_regression/`)
- Rust: Systems programming (`examples/rust_adaptive_sort/`)
- Scientific: scipy, matplotlib, etc. (various math examples)

### API Authentication

**Environment Variables**
```bash
# Required for LLM access
export OPENAI_API_KEY="your-api-key"

# Optional: Override API base
export OPENAI_API_BASE="https://api.openai.com/v1"
```

**Configuration File**
```yaml
llm:
  api_base: "https://generativelanguage.googleapis.com/v1beta/openai/"
  api_key: null  # Uses OPENAI_API_KEY env var by default
```

### External Documentation Links

- **OpenAI API**: https://platform.openai.com/docs/api-reference
- **Google Gemini API**: https://ai.google.dev/gemini-api/docs
- **Ollama**: https://ollama.ai/library
- **OptiLLM**: https://github.com/codelion/optillm
- **OpenEvolve Examples**: https://github.com/codelion/openevolve/tree/main/examples

## Essential Commands

### Development Setup

```bash
# Install in development mode with all dependencies
pip install -e ".[dev]"

# Or install just the base package
pip install -e .

# Using Makefile
make install          # Install base
make install-dev      # Install with dev dependencies (pytest, optillm)
```

### Running Tests

```bash
# Run unit tests only (fast, no LLM required)
python -m unittest discover tests
make test

# Run integration tests (requires optillm and local LLM)
make test-integration

# Run all tests
make test-all
```

### Code Formatting

```bash
# Format with Black (line length: 100)
python -m black openevolve examples tests scripts
make lint
```

### Running OpenEvolve

```bash
# Basic evolution run
python openevolve-run.py path/to/initial_program.py path/to/evaluator.py \
  --config path/to/config.yaml \
  --iterations 1000

# Using library API (recommended)
openevolve-run path/to/initial_program.py path/to/evaluator.py \
  --config path/to/config.yaml \
  --iterations 1000

# Resume from checkpoint
python openevolve-run.py path/to/initial_program.py path/to/evaluator.py \
  --config path/to/config.yaml \
  --checkpoint path/to/checkpoint_directory \
  --iterations 50

# Run specific example
cd examples/function_minimization
python ../../openevolve-run.py initial_program.py evaluator.py --config config.yaml --iterations 50
```

### Visualization

```bash
# Install visualization dependencies first
pip install -r scripts/requirements.txt

# View evolution tree interactively
python scripts/visualizer.py

# View specific checkpoint
python scripts/visualizer.py --path examples/function_minimization/openevolve_output/checkpoints/checkpoint_100/
```

## High-Level Architecture

OpenEvolve implements a sophisticated multi-process evolutionary system with quality-diversity optimization (MAP-Elites), island-based evolution, and LLM-driven code mutations.

### Core Components

1. **Controller (`openevolve/controller.py`)**: Main orchestrator that manages the evolution loop
   - Coordinates between database, LLM ensemble, evaluator, and prompt sampler
   - Manages checkpointing and state persistence
   - Tracks absolute best program across all islands
   - Uses ProcessPoolExecutor for parallel iteration execution

2. **Database (`openevolve/database.py`)**: Implements MAP-Elites algorithm with island-based evolution
   - **MAP-Elites Grid**: Multi-dimensional feature space where each cell holds the best program for that feature combination
   - **Island Model**: Multiple isolated populations (islands) evolve independently
   - **Migration**: Periodic gene flow between adjacent islands (ring topology) prevents premature convergence
   - **Feature Dimensions**: Can be built-in (complexity, diversity) or custom metrics from evaluator
   - **Sampling Strategies**: Elite selection, exploitation, exploration with configurable ratios
   - Tracks absolute best program separately from MAP-Elites grid

3. **Evaluator (`openevolve/evaluator.py`)**: Flexible evaluation system with cascade pattern
   - **Cascade Evaluation**: Multi-stage filtering to reject bad programs early (saves compute)
     - Stage 1: Quick validation
     - Stage 2: Basic performance testing
     - Stage 3: Comprehensive evaluation
   - **TaskPool**: Parallel evaluation of multiple programs concurrently
   - **Artifact Collection**: Captures stdout, stderr, execution context for LLM feedback
   - **LLM Feedback** (optional): Uses LLM to evaluate code quality
   - Dynamically loads evaluation function from provided file

4. **LLM Integration (`openevolve/llm/`)**: Robust ensemble system
   - **Model Ensemble**: Multiple models with configurable weights (e.g., 80% fast model, 20% smart model)
   - **Separate Ensembles**: Different models for code evolution vs code feedback
   - **Universal API**: Works with OpenAI, Google Gemini, local models (Ollama), proxies (OptiLLM)
   - **Retry Logic**: Automatic retries with exponential backoff
   - **Async Generation**: Non-blocking LLM calls

5. **Iteration (`openevolve/iteration.py`)**: Worker process that executes a single evolution step
   - Samples parent program and inspirations from database
   - Builds prompt with context (top programs, artifacts, previous programs)
   - Generates code mutation via LLM
   - Evaluates new program
   - Stores results and artifacts in database
   - Each iteration runs in fresh process with database snapshot for true parallelism

6. **Prompt System (`openevolve/prompt/`)**: Context-aware prompt generation
   - **Template System**: Customizable templates with variable substitution
   - **Template Stochasticity**: Random variations in phrasing for diversity
   - **Context Building**: Includes top programs, inspirations, artifacts, metrics
   - **Diff vs Full Rewrite**: Supports both incremental changes and complete rewrites
   - **Island Isolation**: Only shows programs from same island to maintain diversity

7. **Process Parallel Controller (`openevolve/process_parallel.py`)**: Manages parallel execution
   - Uses ProcessPoolExecutor for true multi-core parallelism
   - Each worker process has its own copy of database, evaluator, LLM ensemble
   - Serializes results back to main process
   - Handles worker initialization and cleanup

### Key Architectural Patterns

#### Quality-Diversity Optimization (MAP-Elites)

- Programs are mapped to cells in a multi-dimensional feature grid
- Each cell stores the best program for that feature combination
- Maintains diversity while optimizing performance
- Feature dimensions can be built-in (complexity, diversity) or custom from evaluator

#### Island-Based Evolution

- Multiple populations (islands) evolve in isolation
- Prevents premature convergence to local optima
- Periodic migration shares best solutions between adjacent islands (ring topology)
- Migration based on generation count, not iteration count (lazy migration)

#### Double-Selection Pattern

- Programs selected as parents (for mutation) differ from inspirations (shown to LLM)
- Allows different selection strategies for exploitation vs exploration

#### Artifact Side-Channel

- Programs can return debugging data, error messages, execution traces
- Small artifacts (<10KB by default) stored as JSON in database
- Large artifacts saved to disk with references in database
- Artifacts included in next generation's prompt for feedback loop

#### Process Worker Pattern

- Each iteration runs in fresh Python process
- Workers load database snapshot for isolation
- True parallelism (no GIL contention)
- Failure in one worker doesn't crash system

#### Fitness Calculation

- Uses 'combined_score' if provided by evaluator
- Otherwise averages all numeric metrics EXCEPT those in feature_dimensions
- Feature dimensions are for diversity (MAP-Elites), not fitness

### Code Evolution Markers

Mark code sections to evolve using:

```python
# EVOLVE-BLOCK-START
# Code to evolve goes here
# EVOLVE-BLOCK-END
```

For full-file evolution, the entire program is evolved without markers.

### Configuration System

YAML-based hierarchical configuration with extensive options:

- **LLM**: Model ensembles, API endpoints, temperature, retries
- **Evolution**: Diff-based vs full rewrites, code length limits, early stopping
- **Database**: Population size, island count, migration settings, feature dimensions
- **Evaluator**: Timeout, cascade thresholds, parallel evaluations, LLM feedback
- **Prompt**: Templates, system messages, artifact rendering, stochasticity

Key configuration concepts:

- `feature_dimensions`: Dimensions for MAP-Elites (diversity, NOT fitness)
- `feature_bins`: Number of bins per dimension (int or dict)
- `migration_interval`: Generations between island migrations
- `cascade_thresholds`: Fitness thresholds for evaluation stages
- `random_seed`: Set to 42 by default for full reproducibility

See `configs/default_config.yaml` for all options with explanations.

### Library API Usage

OpenEvolve can be used as a library without external files:

```python
from openevolve import run_evolution, evolve_function

# Evolution with inline code
result = run_evolution(
    initial_program='''
    # EVOLVE-BLOCK-START
    def fibonacci(n):
        if n <= 1: return n
        return fibonacci(n-1) + fibonacci(n-2)
    # EVOLVE-BLOCK-END
    ''',
    evaluator=lambda path: {"score": benchmark_fib(path)},
    iterations=100
)

# Evolve Python functions directly
result = evolve_function(
    your_function,
    test_cases=[...],
    iterations=50
)
```

### Important Implementation Details

#### Checkpoint/Resume System

- Automatic checkpointing every N iterations (configurable)
- Saves entire system state: database, best program, configuration, RNG state
- Seamless resume from any checkpoint
- Checkpoints stored in `output_dir/checkpoints/checkpoint_N/`

#### Parallel Evaluation

- Multiple programs evaluated concurrently via TaskPool
- Configurable concurrency: `evaluator.parallel_evaluations`
- Async/await based for efficient I/O

#### Error Resilience

- Individual iteration failures don't crash the system
- Extensive retry logic in LLM calls
- Timeout protection for evaluation
- Graceful degradation when workers fail

#### Prompt Engineering

- Template-based system with variable substitution
- Context includes: current code, top programs, artifacts, metrics
- Evolution history for learning from mistakes
- Island-aware: only shows programs from same island

#### Reproducibility

- Every component is seeded: LLM, database, evaluation
- Default seed=42 for immediate reproducibility
- Hash-based isolation prevents cross-contamination
- Deterministic evolution across machines

### Development Notes

#### Requirements

- Python >=3.10 required
- OpenAI-compatible API for LLM (OpenAI, Gemini, Ollama, etc.)
- Optional: pytest for integration tests, optillm for proxy

#### Testing

- Unit tests: `tests/` - No LLM required
- Integration tests: `tests/integration/` - Requires local LLM via optillm
- Use `make test-integration` to run with automatic optillm setup

#### Code Style

- Black formatter with 100 character line length
- Type hints encouraged but not required
- Docstrings for public APIs

#### Key Files

- `openevolve/controller.py` - Main evolution loop
- `openevolve/database.py` - MAP-Elites + island model
- `openevolve/iteration.py` - Single evolution step
- `openevolve/evaluator.py` - Program evaluation
- `openevolve/llm/ensemble.py` - LLM model management
- `openevolve/prompt/sampler.py` - Prompt generation
- `openevolve/api.py` - High-level library API

#### Common Patterns

- Artifacts: Small stored in DB, large on disk with 10KB threshold
- Metrics: Always return dict from evaluator with numeric values
- Feature dimensions: Return raw values, OpenEvolve handles binning automatically
- Process safety: Workers get database snapshots, not shared memory
