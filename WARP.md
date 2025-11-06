# WARP.md

This file provides guidance to WARP (warp.dev) when working with code in this repository.

## Build Commands

**Primary build system: CMake (Makefile is deprecated)**

```bash
# Basic CPU-only build
cmake -B build
cmake --build build --config Release -j $(nproc)

# CUDA backend
cmake -B build -DGGML_CUDA=ON
cmake --build build --config Release -j $(nproc)

# Metal backend (macOS)
cmake -B build -DGGML_METAL=ON
cmake --build build --config Release -j $(nproc)

# Debug build (single-config generators)
cmake -B build -DCMAKE_BUILD_TYPE=Debug
cmake --build build

# Debug build (multi-config generators like Xcode)
cmake -B build -G "Xcode"
cmake --build build --config Debug
```

Built binaries are placed in `build/bin/`.

## Testing

```bash
# Run all tests
ctest --test-dir build --output-on-failure -j $(nproc)

# Run server-specific unit tests (requires Python venv)
cd tools/server/tests
source ../../../.venv/bin/activate
./tests.sh

# Manual inference test
./build/bin/llama-cli --version
./build/bin/llama-cli -m model.gguf -p "Hello" -n 10
```

Expected: 2-3 network-related tests may fail in isolated environments. Core functionality tests should pass.

## Linting and Formatting

```bash
# C++ formatting (ALWAYS run before committing)
git clang-format

# Python environment (use for all Python tools)
source .venv/bin/activate

# Pre-commit hooks
pre-commit run --all-files
```

Configuration files:
- `.clang-format`: C++ style (4 spaces, 120 column limit, same-line braces)
- `.flake8`: Python linting (max-line-length=125)
- `pyrightconfig.json`: Python type checking

## Local CI Validation

```bash
# Run full CI locally before submitting PRs
mkdir tmp
bash ./ci/run.sh ./tmp/results ./tmp/mnt
```

Runtime: 30-60 minutes. Add `ggml-ci` to commit messages to trigger heavy CI workloads on custom infrastructure.

## Architecture

### Core Components

- **`src/llama.cpp`** (~8000 lines): Main library implementation
- **`include/llama.h`** (~2000 lines): Public C API - changes require careful consideration
- **`ggml/`**: Tensor computation library (vendored dependency)
- **`common/`**: Shared utilities across examples/tools
- **`examples/`**: 30+ example applications
- **`tools/`**: Production-ready utilities (server, benchmarks, etc.)

### Key Executables (in `build/bin/`)

- **`llama-cli`**: Main inference tool with conversation mode
- **`llama-server`**: OpenAI-compatible HTTP server
- **`llama-quantize`**: Model quantization utility
- **`llama-perplexity`**: Model evaluation/quality metrics
- **`llama-bench`**: Performance benchmarking
- **`llama-run`**: Comprehensive inference example (used with RamaLama)
- **`llama-simple`**: Minimal API example for developers

### Model Conversion

Python scripts in root directory:
- `convert_hf_to_gguf.py`: HuggingFace → GGUF format
- `convert_lora_to_gguf.py`: LoRA adapters → GGUF
- `convert_llama_ggml_to_gguf.py`: Legacy GGML → GGUF

All scripts require Python venv (`.venv`).

### Backend Support

llama.cpp supports multiple hardware backends:
- **CPU**: AVX/AVX2/AVX512/NEON optimized
- **CUDA**: NVIDIA GPUs
- **Metal**: Apple Silicon
- **Vulkan**: Cross-platform GPU
- **SYCL**: Intel/NVIDIA GPUs
- **HIP**: AMD GPUs
- **MUSA**: Moore Threads GPUs
- **CANN**: Ascend NPU

Backend-specific code is isolated and does not require adherence to core coding guidelines.

## Coding Standards

### C/C++
- Plain C/C++ with minimal dependencies
- Avoid modern STL constructs, templates, and third-party libraries
- Use `snake_case` for functions/variables/types
- Use sized integer types (`int32_t`) in public API
- Enum values: uppercase with enum name prefix (`LLAMA_VOCAB_TYPE_NONE`)
- Naming pattern: `<class>_<method>` where method is `<action>_<noun>`
- Declare structs as `struct foo {}`, not `typedef struct foo {} foo`
- Vertical alignment for readability
- 4 spaces indentation, 120 column limit, same-line braces
- Pointer/reference style: `void * ptr`, `int & ref`

### Python
- Activate `.venv` before running any Python tools
- Follow `.flake8` configuration (max-line-length=125)
- Use pyright for type checking

### Naming Conventions

Optimize for longest common prefix:
```cpp
// Good
int number_small;
int number_big;

// Bad
int small_number;
int big_number;
```

Function naming:
```cpp
llama_model_init();           // class: "llama_model", method: "init"
llama_sampler_chain_remove(); // class: "llama_sampler_chain", method: "remove"
llama_sampler_get_seed();     // class: "llama_sampler", method: "get_seed"
```

## Development Workflow

1. Format code: `git clang-format`
2. Build: `cmake --build build --config Release`
3. Test: `ctest --test-dir build --output-on-failure`
4. If modifying server: Run server tests in `tools/server/tests`
5. Manual validation: Test relevant tools in `build/bin/`
6. Performance check: Use `llama-bench` and `llama-perplexity`

### Performance Validation

```bash
# Benchmark inference
./build/bin/llama-bench -m model.gguf

# Evaluate perplexity
./build/bin/llama-perplexity -m model.gguf -f dataset.txt

# Test backend operations
./build/bin/test-backend-ops
```

## Important Guidelines

- **Cross-platform compatibility**: Test on Linux, macOS, Windows when possible
- **Performance-critical**: This is an inference library optimized for speed
- **API stability**: Changes to `include/llama.h` require maintainer approval
- **Minimal dependencies**: Avoid adding external dependencies
- **PR requirements**:
  - Create separate PRs for each feature/fix
  - Execute local CI before publishing
  - Verify perplexity and performance are not negatively affected
  - If modifying ggml operators, add test cases to `test-backend-ops`
  - Squash-merge format: `<module> : <commit title> (#<issue_number>)`

## Testing Philosophy

- Core functionality must work without network access
- Backend tests require access to at least two different backends for consistency validation
- Server tests require Python venv dependencies (pytest, aiohttp)
- 38 tests total covering tokenizers, grammar, sampling, backends, integration

## Model Format

llama.cpp uses GGUF format for models. Download pre-quantized models from HuggingFace:

```bash
# CLI automatically downloads from HF
llama-cli -hf ggml-org/gemma-3-1b-it-GGUF

# Server with auto-download
llama-server -hf ggml-org/gemma-3-1b-it-GGUF --port 8080
```

Set `MODEL_ENDPOINT` environment variable for alternative sources (e.g., ModelScope).

## GGML Tensor Library

llama.cpp uses ggml for tensor operations. Key characteristics:
- Tensors stored in row-major order
- Dimension 0 = columns, 1 = rows, 2 = matrices
- Matrix multiplication convention: `C = ggml_mul_mat(ctx, A, B)` means $C^T = A B^T$

For ggml understanding, see examples in `ggml/examples/`: `simple`, `gpt-2`, `mnist`.

