# Building llama-cpp-python with CUDA Support on Windows

**A Complete Guide for Vision-Enabled GGUF Models (Qwen3VL, etc.)**

# Funny Side-note
- All of this happened because I didn't read the included documentation in [ComfyUI-QwenVL](https://github.com/1038lab/ComfyUI-QwenVL)
- There are pre-built wheels with vision support at: https://github.com/JamePeng/llama-cpp-python/releases/
- **BONUS MISTAKE:** I also built the wheel for Python 3.11 when my venv uses Python 3.12 🤦‍♂️
- Lesson learned: Always check your Python version AND read the docs BEFORE spending 30+ minutes compiling!
- **However**, if you need the latest features, specific CUDA versions, or want maximum performance, this build guide is still the way to go!

## Problem Statement

The official llama-cpp-python PyPI releases don't include pre-built Windows wheels with CUDA and vision support. This guide walks you through building from source to enable GPU acceleration for vision-capable GGUF models.

## Prerequisites

### Required Software

1. **Python 3.9-3.12** (tested with 3.11)
   - Download from: https://www.python.org/downloads/

2. **NVIDIA CUDA Toolkit** (version 12.x or 13.x recommended)
   - Download from: https://developer.nvidia.com/cuda-downloads
   - Verify installation: `nvcc --version`

3. **Visual Studio 2022 Build Tools** (CRITICAL)
   - Download: https://aka.ms/vs/17/release/vs_BuildTools.exe
   - **Required components:**
     - ✅ Desktop development with C++
     - ✅ MSVC v143 - VS 2022 C++ x64/x86 build tools
     - ✅ Windows 10 or 11 SDK
   
   **⚠️ IMPORTANT:** CUDA 13.1 does NOT support Visual Studio 2026. You MUST use VS 2022 or earlier.

4. **CMake 3.x or 4.x**
   - Download: https://cmake.org/download/
   - During installation: ✅ Check "Add CMake to system PATH"
   - Verify: `cmake --version`

### Environment Check

Before building, verify your environment:

```cmd
REM Check CUDA
nvcc --version

REM Check CMake
cmake --version

REM Check Visual Studio (should show compiler version)
cl
```

If `cl` is not found, you need to use the **Visual Studio Developer Command Prompt** (see Build Process below).

## Common Issues & Solutions

### Issue 1: "No CUDA toolset found"

**Symptoms:**
```
Found CUDAToolkit: ... (found version "13.1.115")
CUDA Toolkit found
CMake Error: No CUDA toolset found.
```

**Cause:** Using Visual Studio 2026 (or other incompatible version)

**Solution:** Install Visual Studio 2022 Build Tools and use its Developer Command Prompt

### Issue 2: CMake from Python venv conflicts

**Symptom:** Build fails with CMake errors even though system CMake works

**Solution:** Temporarily disable venv CMake:
```cmd
ren "C:\path\to\.venv\Scripts\cmake.exe" "cmake.exe.bak"
```

## Build Process

### Step 1: Clone or Download llama-cpp-python

```cmd
cd C:\temp
git clone --recursive https://github.com/abetlen/llama-cpp-python.git
cd llama-cpp-python
```

**Note:** The `--recursive` flag is important to include llama.cpp submodules.

### Step 2: Open Visual Studio 2022 Developer Command Prompt

**CRITICAL:** Do NOT use a regular command prompt!

1. Press Windows key
2. Search for: **"x64 Native Tools Command Prompt for VS 2022"**
3. Run as Administrator (recommended)

### Step 3: Activate Your Python Virtual Environment (Optional)

If building for a specific project (like ComfyUI):

```cmd
C:\path\to\your\.venv\Scripts\activate.bat
```

### Step 4: Handle venv CMake Conflict (If Applicable)

If you have cmake installed in your venv, temporarily rename it:

```cmd
where cmake
```

If venv cmake appears first:
```cmd
ren "C:\path\to\.venv\Scripts\cmake.exe" "cmake.exe.bak"
```

### Step 5: Build with CUDA Support

```cmd
cd C:\temp\llama-cpp-python

set CMAKE_ARGS=-DGGML_CUDA=on -DCMAKE_CUDA_COMPILER="C:/Program Files/NVIDIA GPU Computing Toolkit/CUDA/v13.1/bin/nvcc.exe" -DCMAKE_CUDA_ARCHITECTURES=native -DCMAKE_GENERATOR="Visual Studio 17 2022" -DCMAKE_GENERATOR_TOOLSET=cuda="C:/Program Files/NVIDIA GPU Computing Toolkit/CUDA/v13.1"

pip install . --no-cache-dir --verbose
```

**Adjust paths if your CUDA is installed elsewhere.** Common locations:
- `C:/Program Files/NVIDIA GPU Computing Toolkit/CUDA/v12.1`
- `C:/Program Files/NVIDIA GPU Computing Toolkit/CUDA/v13.1`

### Step 6: Wait for Compilation

**Expected behavior:**
- Compilation takes 15-30+ minutes depending on your CPU
- You'll see hundreds of compiler warnings (this is normal)
- CPU usage may be moderate (30-60%) - this is expected
- Progress appears to "freeze" during template instantiation phases - be patient!

**Signs of progress:**
- New compilation messages appearing
- Disk activity
- Process still running in Task Manager

### Step 7: Restore venv CMake (If You Renamed It)

```cmd
ren "C:\path\to\.venv\Scripts\cmake.exe.bak" "cmake.exe"
```

### Step 8: Verify Installation

```cmd
python -c "import llama_cpp; print(llama_cpp.__version__)"
```

## Build Configuration Options

### GPU Architecture

```cmd
REM Auto-detect (recommended)
-DCMAKE_CUDA_ARCHITECTURES=native

REM Specific architectures (e.g., RTX 3090 = sm_86)
-DCMAKE_CUDA_ARCHITECTURES=86

REM Multiple architectures
-DCMAKE_CUDA_ARCHITECTURES="75;86;89"
```

### Additional Options

```cmd
REM Enable all optimizations
set CMAKE_ARGS=-DGGML_CUDA=on -DGGML_CUDA_F16=ON -DCMAKE_CUDA_ARCHITECTURES=native

REM For AMD GPUs (ROCm instead of CUDA)
set CMAKE_ARGS=-DGGML_HIPBLAS=on
```

## Alternative: Using Pre-built Wheels

If you don't need the absolute latest version or vision support isn't critical:

```cmd
REM CUDA 12.1
pip install llama-cpp-python --extra-index-url https://abetlen.github.io/llama-cpp-python/whl/cu121

REM CUDA 11.8
pip install llama-cpp-python --extra-index-url https://abetlen.github.io/llama-cpp-python/whl/cu118
```

**Note:** These may not include vision support or the latest features.

## Troubleshooting

### Build Errors with Template Warnings

**Issue:** Hundreds of warnings about "variable declared but never referenced"

**Status:** These are harmless compiler warnings, not errors. Build continues normally.

### "Stuck" During Compilation

**Issue:** No output for 1-2 minutes

**Status:** This is normal during CUDA template instantiation. If no new output for 10+ minutes AND CPU drops to 0%, then investigate.

### Out of Memory During Build

**Solution:** Close other applications, or add:
```cmd
set CMAKE_ARGS=%CMAKE_ARGS% -DCMAKE_BUILD_TYPE=Release
```

### CUDA Version Mismatch

**Issue:** PyTorch or other libraries require different CUDA versions

**Solution:** Build multiple wheels for different CUDA versions, or use conda environments to isolate CUDA dependencies.

## Saving the Wheel for Later

After successful build, save the wheel file:

```cmd
REM Wheel is in dist/ folder
copy C:\temp\llama-cpp-python\dist\llama_cpp_python-*.whl C:\somewhere\safe\

REM Install later
pip install C:\somewhere\safe\llama_cpp_python-X.X.X-*.whl
```

**Wheel filename includes:**
- Python version (cp311 = Python 3.11)
- Platform (win_amd64)
- Build date/hash

## Compatibility Matrix

| CUDA Version | Visual Studio | Python | Status |
|--------------|---------------|--------|--------|
| 13.1 | VS 2022 (v143) | 3.9-3.12 | ✅ Tested |
| 13.1 | VS 2026 | Any | ❌ Not Compatible |
| 12.x | VS 2022 (v143) | 3.9-3.12 | ✅ Recommended |
| 12.x | VS 2019 (v142) | 3.9-3.12 | ✅ Should work |
| 11.8 | VS 2019/2022 | 3.9-3.12 | ✅ Should work |

## Performance Notes

**Expected performance with CUDA:**
- 5-10x faster inference than CPU
- Vision models (Qwen3VL) benefit significantly from GPU
- First run may be slower (kernel compilation)

## Credits & Community

- llama-cpp-python: https://github.com/abetlen/llama-cpp-python
- llama.cpp: https://github.com/ggerganov/llama.cpp

## License

This guide is provided as-is for educational purposes. Follow the respective licenses of the software you're building.

---

**Last Updated:** January 2026  
**Tested Configuration:**
- Windows 11
- CUDA 13.1
- Visual Studio 2022 Build Tools
- Python 3.11
- CMake 4.2.1
