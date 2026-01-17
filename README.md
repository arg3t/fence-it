# Memory Fencing Optimization for LLVM

[![LLVM](https://img.shields.io/badge/LLVM-Compiler_Pass-blue.svg)](https://llvm.org/)
[![C++17](https://img.shields.io/badge/C++-17-blue.svg)](https://isocpp.org/)
[![CMake](https://img.shields.io/badge/CMake-3.12+-green.svg)](https://cmake.org/)
[![License](https://img.shields.io/badge/License-MIT-Green.svg)](LICENSE)

This project provides LLVM compiler passes that implement memory fencing optimization for concurrent programs under relaxed memory models. It includes automated fence insertion and a max-flow min-cut optimization for Total Store Ordering (TSO) and Partial Store Ordering (PSO).

Sequentially-consistent (SC) fences enforce a happens-before relationship between operations preceding and following the fence, ensuring all operations before the fence complete prior to executing those after it at both hardware and compiler levels. Fences are inserted at the LLVM IR stage, allowing the backend to lower them to target architecture instructions or ignore them when unnecessary (for example, SC fences are effectively ignored on x86).

## Project Overview

This project implements:

- Fence insertion passes to ensure correctness under TSO and PSO
- Fence optimization using max-flow min-cut to remove redundant fences
- A litmus test suite covering classical concurrent programming patterns

### Memory Models Supported

| Memory Model | Description | Use Case |
|--------------|-------------|----------|
| **TSO** (Total Store Ordering) | Allows load-load, load-store, and store-store reordering, but prohibits store-load reordering | x86/x64 architectures |
| **PSO** (Partial Store Ordering) | More relaxed than TSO, allows store-store reordering | SPARC architectures |

### Fence Insertion Rules

The passes insert fences based on consecutive atomic memory operation pairs. Fences are only introduced between two relaxed atomic memory operations, as C11 memory ordering for other operations already provides the required guarantees.

| Op1 | Order1 | Op2 | Order2 | TSO | PSO |
|-----|--------|-----|--------|----- |-----|
| R | RLX | R | RLX | R-Fence-R | R-Fence-R |
| R | RLX | W | RLX | R-Fence-W | R-Fence-W |
| W | RLX | R | RLX | R-W | R-W |
| W | RLX | W | RLX | W-Fence-W | W-W |
| R | ACQ | * | * | R-Op2 | R-Op2 |
| * | * | W | REL | Op1-W | Op1-W |
| W | REL | R | ACQ | W-R | W-R |
| * | SEQ_CST | * | SEQ_CST | Op1-Op2 | Op1-Op2 |
| * | ACQ_REL | * | ACQ_REL | Op1-Op2 | Op1-Op2 |

Note: The main difference between TSO and PSO is that PSO allows Write-Write reordering while TSO does not.

## Architecture

The project consists of three main LLVM passes:

```
fencing/
├── FenceTSO.cpp          # TSO fence insertion pass
├── FencePSO.cpp          # PSO fence insertion pass
├── FenceOptimization.cpp # Max-flow based fence optimization
└── FencingPasses.h       # Pass interface definitions
```

### Algorithm Overview

Fence Insertion Algorithm: The TSO and PSO passes traverse each function starting from the entry block, following all possible control flows while tracking the last atomic memory operation. When a consecutive operation pair that is too relaxed appears, the algorithm inserts two fences per operation pair: one after the initial operation and one before the second operation.

Fence Optimization Algorithm:
1. Graph construction by removing fence instructions and inserting nodes before and after their locations.
2. Network flow construction to connect source and sink and form a flow graph.
3. Min-cut calculation using Ford-Fulkerson max-flow.
4. Optimal placement by inserting fences only at min-cut locations.

This approach minimizes fence overhead while maintaining correctness.

### Litmus Tests

The project includes implementations of classical concurrent programming patterns used to validate fence synthesis:

- MP (Message Passing): producer-consumer synchronization
- SB (Store Buffering): symmetric store-load pattern testing write-read reordering
- LB (Load Buffering): circular dependency pattern testing read-write reordering
- IRIW (Independent Reads of Independent Writes): multi-reader consistency pattern
- Branch control flow tests for fence insertion across conditional branches

Each test covers different consecutive memory operation types (read-write, write-write, write-read) with various memory ordering combinations to validate both TSO and PSO fence synthesis methods.

## Getting Started

### Prerequisites

- LLVM 14+ with development headers
- CMake 3.12+
- C++17 compatible compiler
- llvm-lit for running tests

### Building the Project

```bash
# Configure with CMake
cmake -DLLVM_LIT=$(which lit) -DLLVM_DIR=/usr/lib64/cmake/llvm -S . -B build

# Build the passes
cmake --build build

# Navigate to build directory
cd build && make
```

Tip: Adjust `LLVM_DIR` to point to your LLVM installation's cmake directory.

### Running the Passes

TSO fence insertion:
```bash
opt -load-pass-plugin ../build/fencing/FencingPass.so -S -passes=fence-tso input.ll
```

PSO fence insertion:
```bash
opt -load-pass-plugin ../build/fencing/FencingPass.so -S -passes=fence-pso input.ll
```

Fence optimization:
```bash
opt -load-pass-plugin ../build/fencing/FencingPass.so -S -passes=fence-tso,fence-opt input.ll
```

### Testing

```bash
# Run all tests
make test

# Run LLVM lit tests specifically
make run-lit-tests
```

## Educational Value

This project demonstrates:

- Relaxed memory models and their impact on reordering
- Correctness preservation in concurrent programs
- Graph algorithms (max-flow min-cut) applied to compiler optimization
- LLVM pass development

## Research Background

Authors: Mihnea Bernevig, Yigit Colakoglu

This implementation builds upon established research in memory fencing optimization:

- Sequential consistency and happens-before relationships
- Fence insertion algorithms for relaxed memory models
- Min-cut optimization based on "Partially redundant fence elimination for x86, arm, and power processors" (Morisset & Zappa Nardelli, 2017)
- Hardware-level memory reordering behaviors

The project demonstrates practical application of graph theory (max-flow min-cut) to compiler optimization.

## Project Structure

```
cs4560_fencing/
├── fencing/           # Core LLVM passes
├── litmus_tests/      # C++ concurrent test programs
├── tests/             # LLVM IR test cases
├── CMakeLists.txt     # Build configuration
└── README.md          # This file
```

## References

- Morisset, R., & Zappa Nardelli, F. (2017). Partially redundant fence elimination for x86, arm, and power processors. In Proceedings of the 26th International Conference on Compiler Construction (CC 2017) (pp. 1–10). Austin, TX, USA: Association for Computing Machinery. https://doi.org/10.1145/3033019.3033021
