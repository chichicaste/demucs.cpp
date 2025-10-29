# Building demucs.cpp on Windows with MSVC

This guide explains how to compile demucs.cpp on Windows using Visual Studio 2022 and MSVC.

## Prerequisites

### Required Software

1. **Visual Studio 2022** with C++ components:
   - MSVC C++ build tools
   - CMake for Windows
   - Windows 10/11 SDK

2. **vcpkg** (C++ package manager):
   ```powershell
   git clone https://github.com/Microsoft/vcpkg.git
   cd vcpkg
   .\bootstrap-vcpkg.bat
   .\vcpkg integrate install
   ```

### Dependencies via vcpkg

Install OpenBLAS (optimized linear algebra library):

```powershell
.\vcpkg install openblas:x64-windows
```

**Note:** vcpkg will automatically install OpenBLAS to `C:\vcpkg\installed\x64-windows\`. CMake will detect this installation automatically.

## Clone the Repository

Make sure to clone with submodules to get all vendored libraries (Eigen, libnyquist, googletest):

```powershell
git clone --recurse-submodules https://github.com/sevagh/demucs.cpp
cd demucs.cpp
```

## Changes Made for MSVC Compatibility

### 1. CMakeLists.txt - Compiler-Specific Flags

Modified [`CMakeLists.txt`](CMakeLists.txt:16) to use compiler-specific flags:

**For MSVC:**
- `/W4` - Warning level 4
- `/EHsc` - Standard C++ exception handling
- `/O2` - Speed optimization (Release)
- `/Od /Zi` - No optimization + debug info (Debug)

**For GCC/Clang:**
- `-Wall -Wextra` - Comprehensive warnings
- `-Ofast -march=native` - Aggressive optimizations
- Math-specific fast flags

Flags are set **after** the `project()` command so CMake can detect the compiler correctly.

### 2. std::filesystem::path to std::string Conversion

MSVC requires explicit conversion from `std::filesystem::path` to `std::string`. Added `.string()` calls in all CLI files:

- [`cli-apps/demucs.cpp`](cli-apps/demucs.cpp:230)
- [`cli-apps/demucs_ft.cpp`](cli-apps/demucs_ft.cpp:307)
- [`cli-apps/demucs_mt.cpp`](cli-apps/demucs_mt.cpp:227)
- [`cli-apps/demucs_ft_mt.cpp`](cli-apps/demucs_ft_mt.cpp:284)
- [`cli-apps/demucs_v3.cpp`](cli-apps/demucs_v3.cpp:230)
- [`cli-apps/demucs_v3_mt.cpp`](cli-apps/demucs_v3_mt.cpp:225)

**Example change:**
```cpp
// Before (causes error in MSVC)
write_audio_file(target_waveform, p_target);

// After (MSVC compatible)
write_audio_file(target_waveform, p_target.string());
```

### 3. Eigen::half Operator Overload Ambiguity

MSVC is stricter with operator overload resolution. Added explicit cast to `float` in test files:

- [`test/test_layers.cpp`](test/test_layers.cpp:498)
- [`test/test_layers_v3.cpp`](test/test_layers_v3.cpp:498)

**Example change:**
```cpp
// Before (ambiguous in MSVC)
Eigen::half val = x(i, j, k, l);
x_stddev += (val - x_mean) * (val - x_mean);

// After (explicit)
Eigen::half val = x(i, j, k, l);
float val_f = static_cast<float>(val);
x_stddev += (val_f - x_mean) * (val_f - x_mean);
```

## Build Process

### 1. Configure Project with CMake

From the project root directory:

```powershell
mkdir build
cd build
cmake .. -G "Visual Studio 17 2022" -A x64
```

**Available CMake options:**
- `USE_OPENBLAS=ON` (default) - Use OpenBLAS
- `USE_AMD_AOCL=OFF` - Use AMD AOCL (Linux only)
- `USE_INTEL_MKL=OFF` - Use Intel MKL

### 2. Build the Project

Build in Release mode with 8 parallel threads:

```powershell
cmake --build . --config Release --parallel 8
```

Or build in Debug mode:

```powershell
cmake --build . --config Debug --parallel 8
```

### 3. Output Executables Location

Compiled executables are located in:
```
build\Release\
├── demucs.cpp.main.exe          # Single model
├── demucs_ft.cpp.main.exe       # Fine-tuned (4 models)
├── demucs_mt.cpp.main.exe       # Single model multi-threaded
├── demucs_ft_mt.cpp.main.exe    # Fine-tuned multi-threaded
├── demucs_v3.cpp.main.exe       # Demucs v3
├── demucs_v3_mt.cpp.main.exe    # Demucs v3 multi-threaded
└── demucs.cpp.test.exe          # Unit tests
```

## Download Model Weights

Pre-converted ggml weights are available on [Hugging Face](https://huggingface.co/datasets/Retrobear/demucs.cpp/tree/main):

```powershell
# Create directory for models
mkdir ggml-demucs
cd ggml-demucs

# Download models (using browser or curl)
# - ggml-model-htdemucs-4s-f16.bin (81 MB) - 4-source model
# - ggml-model-htdemucs-6s-f16.bin (53 MB) - 6-source model
# - ggml-model-htdemucs_ft_*.bin (81 MB each) - Fine-tuned models
# - ggml-model-hdemucs_mmi-v3-f16.bin (160 MB) - v3 model
```

## Convert Weights from PyTorch (Optional)

If you want to convert your own weights from PyTorch:

### 1. Set Up Python Environment

```powershell
# Using conda/mamba
conda create --name demucscpp python=3.11
conda activate demucscpp
pip install -r .\scripts\requirements.txt
```

### 2. Convert Models

**Standard 4-source model:**
```powershell
python .\scripts\convert-pth-to-ggml.py .\ggml-demucs
```

**6-source model:**
```powershell
python .\scripts\convert-pth-to-ggml.py .\ggml-demucs --six-source
```

**Fine-tuned models (all):**
```powershell
python .\scripts\convert-pth-to-ggml.py .\ggml-demucs --ft-drums --ft-vocals --ft-bass --ft-other
```

**v3 model:**
```powershell
python .\scripts\convert-pth-to-ggml.py .\ggml-demucs --v3
```

## Running demucs.cpp

### Single Model (4-source)

```powershell
.\build\Release\demucs.cpp.main.exe .\ggml-demucs\ggml-model-htdemucs-4s-f16.bin C:\path\to\your\song.wav .\output\
```

### Fine-tuned Model (BagOfModels)

```powershell
.\build\Release\demucs_ft.cpp.main.exe .\ggml-demucs\ C:\path\to\your\song.wav .\output\
```

### Multi-threaded Version (recommended for better performance)

```powershell
# Using 4 threads
.\build\Release\demucs_mt.cpp.main.exe .\ggml-demucs\ggml-model-htdemucs-4s-f16.bin C:\path\to\your\song.wav .\output\ 4
```

### v3 Model

```powershell
.\build\Release\demucs_v3.cpp.main.exe .\ggml-demucs\ggml-model-hdemucs_mmi-v3-f16.bin C:\path\to\your\song.wav .\output\
```

## Output

Separated audio files will be saved in the specified output directory:

```
output\
├── target_0_drums.wav
├── target_1_bass.wav
├── target_2_other.wav
└── target_3_vocals.wav
```

For the 6-source model, additional files are generated:
- `target_4_guitar.wav`
- `target_5_piano.wav`

## Running Tests

```powershell
.\build\Release\demucs.cpp.test.exe
```

## Troubleshooting

### Error: "Cannot find OpenBLAS"

Make sure vcpkg is properly integrated:
```powershell
.\vcpkg integrate install
```

### Error: "CMake cannot find compiler"

Open "Developer Command Prompt for VS 2022" or "Developer PowerShell for VS 2022" from the Start menu.

### Type Conversion Warnings

Warnings C4244, C4305, C4267 are normal and don't affect functionality. They are type conversions that occur in mathematical operations with Eigen.

### Slow Performance

- Use multi-threaded versions (`*_mt.exe`) for better performance
- Make sure to compile in Release mode, not Debug
- Consider using more threads based on your CPU (e.g., 8 threads for 8-core CPU)

## Differences from Linux Version

1. **Compiler Flags:** MSVC uses `/` instead of `-` for flags
2. **Optimizations:** MSVC uses `/O2` instead of `-Ofast -march=native`
3. **File Paths:** Windows uses `\` but C++ accepts `/` in paths
4. **Extensions:** Executables have `.exe` extension

## Performance

For details on performance, benchmarks, and comparisons with PyTorch, see [PERFORMANCE.md](./.github/PERFORMANCE.md).

## License

See [LICENSE](LICENSE) for license details.