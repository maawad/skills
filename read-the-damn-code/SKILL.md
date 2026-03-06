---
name: read-the-damn-code
description: When writing or debugging code in specific domains (Triton, HIP/ROCm, etc.), clone the canonical upstream repo and read the actual source instead of guessing. Use when the user is writing Triton kernels, debugging HIP or ROCm runtime behavior, or when the agent is inferring behavior of these systems without evidence.
---

# Read the damn code

When working in the domains below, **clone the relevant repo and read the source code**. Do not rely on memory or inference for API behavior, semantics, or bugs—use the repo as the source of truth.

## Principle

- Clone the canonical repo when the task touches that domain.
- Search and read the actual code (headers, implementation, tests) before answering or writing code.
- Prefer grep/search in the cloned tree over guessing from documentation or recall.

## Domain → Repo mapping

### Triton (GPU kernel language, compiler)

- **Repo**: https://github.com/triton-lang/triton  
- **Clone**: `git clone --depth 1 --filter=blob:none https://github.com/triton-lang/triton.git`
- **Where to read** (verified):  
  - Language/ops/semantics: `python/triton/language/`  
  - MLIR/codegen: `lib/` — `Dialect/` (Triton, TritonGPU, TritonNvidiaGPU, etc.), `Conversion/`, `Analysis/`, `Target/`, `Tools/`  
  - Python backends/driver: `python/triton/backends/`  
  - Third-party: `third_party/`  
  - Tests: `test/` (mixed Python + C++/lit), `unittest/`
- **When**: Writing Triton kernels, debugging Triton codegen or runtime behavior, questions about Triton ops or scheduling.

### HIP / ROCm runtime and libraries

AMD uses two monorepos (both large). Use **sparse checkout** to fetch only the components you need.

- **rocm-systems** (runtime, HIP, driver stack): https://github.com/ROCm/rocm-systems  
  - Layout: all components under **`projects/<name>/`**. Key ones: `projects/hip/` (API, `include/`), `projects/clr/` (contains `hipamd/`, `rocclr/`), `projects/hip-tests/`, `projects/aqlprofile/`, `projects/rdc/`, `projects/rccl/`, `projects/rocr-runtime/`, plus rocprofiler*, rocm-core, rocminfo, etc.
- **rocm-libraries** (math/libs): https://github.com/ROCm/rocm-libraries  
  - Layout: all libraries under **`projects/<name>/`**. Examples: `projects/rocblas/`, `projects/miopen/`, `projects/hipblas/`, `projects/hipsparse/`, `projects/rocthrust/`, `projects/rocwmma/`, `projects/hipdnn/`, `projects/rocfft/`, `projects/composablekernel/`, plus hipcub, rocprim, rocrand, rocsolver, hipsolver, hipblaslt, hipsparselt, hiptensor, hiprand, hipfft. Also top-level `dnn-providers/`.

**Sparse checkout (recommended)** — clone only the components you need:

```bash
# rocm-systems: e.g. HIP + CLR (runtime) only
git clone --filter=blob:none --sparse https://github.com/ROCm/rocm-systems.git
cd rocm-systems
git sparse-checkout set projects/hip projects/clr projects/hip-tests

# rocm-libraries: e.g. one or more libraries
git clone --filter=blob:none --sparse https://github.com/ROCm/rocm-libraries.git
cd rocm-libraries
git sparse-checkout set projects/rocblas projects/miopen projects/rocthrust
```

Add or remove `projects/<name>` paths as needed. To add more later: `git sparse-checkout add projects/rdc`.

**Full clone** (if you need the whole tree):

```bash
git clone --depth 1 --filter=blob:none https://github.com/ROCm/rocm-systems.git
git clone --depth 1 --filter=blob:none https://github.com/ROCm/rocm-libraries.git
```

**Where to read**:  
- Systems: `rocm-systems/projects/<component>/` (e.g. `projects/hip/include/`, `projects/clr/hipamd/`).  
- Libraries: `rocm-libraries/projects/<library>/` (e.g. `projects/rocblas/`, `projects/miopen/`).

- **When**: Debugging HIP runtime bugs, understanding HIP API behavior, driver/device behavior, ROCm stack issues, or any ROCm library behavior.

## Workflow

1. Identify the domain (Triton vs HIP/ROCm vs other).
2. Clone the repo(s) above into a scratch dir (e.g. workspace or temp) if not already present.
3. Use grep/search and read the relevant files before answering or writing code.
4. Cite file paths and line ranges when giving answers or fixes.

## Adding more domains

Add new sections in the same format: **Repo**, **Clone**, **Where to read**, **When**. Keep the rule: if the task touches that stack, clone and read—don’t guess.
