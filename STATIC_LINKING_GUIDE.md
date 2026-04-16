# PE-Sieve Static Linking Guide

## Overview
This guide explains how to statically link PE-sieve into your application after building it as a static library.

> **PE_Sieve.lib is self-contained.** All dependencies (libpeconv, sig_finder) are compiled directly into the library. You do **not** need separate `.lib` files for these dependencies.

## Required Files

### 1. Library File
- **PE_Sieve.lib** - The compiled static library (includes libpeconv + sig_finder)
  - Debug build: `x64\Debug\PE_Sieve.lib` or `Win32\Debug\PE_Sieve.lib`
  - Release build: `x64\Release\PE_Sieve.lib` or `Win32\Release\PE_Sieve.lib`

### 2. Public API Headers (Required)
Copy these headers to your project's include directory:
```
pe-sieve/include/
├── pe_sieve_api.h           # Main API functions (PESieve_scan, etc.)
├── pe_sieve_types.h         # Configuration structures and enums
└── pe_sieve_return_codes.h  # Return code constants
```

### 3. Dependency Headers (Required for Compilation)
These directories must be in your include path because PE-sieve's internal headers reference them:
```
pe-sieve/libpeconv/include/     # PE manipulation library headers
pe-sieve/paramkit/paramkit/include/  # Parameter handling headers
pe-sieve/sig_finder/include/    # Signature matching headers
```
> **Note:** You only need the *headers* from these directories. Their source code is already compiled into `PE_Sieve.lib`.

## Project Configuration

### Visual Studio Project Settings

#### Include Directories
Add to **C/C++ → General → Additional Include Directories**:
```
$(SolutionDir)pe-sieve\include
$(SolutionDir)pe-sieve\libpeconv\include
$(SolutionDir)pe-sieve\paramkit\paramkit\include
$(SolutionDir)pe-sieve\sig_finder\include
```

#### Preprocessor Definitions
Add to **C/C++ → Preprocessor → Preprocessor Definitions**:
```
PESIEVE_STATIC_LIB
```
> **Important:** This disables `__declspec(dllimport)` and uses direct linking.

#### Library Directories
Add to **Linker → General → Additional Library Directories**:
```
$(SolutionDir)pe-sieve\x64\$(Configuration)
```

#### Additional Dependencies
Add to **Linker → Input → Additional Dependencies**:
```
PE_Sieve.lib
psapi.lib
version.lib
```

### CMake Configuration

```cmake
# Add include directories
include_directories(
    ${CMAKE_SOURCE_DIR}/pe-sieve/include
    ${CMAKE_SOURCE_DIR}/pe-sieve/libpeconv/include
    ${CMAKE_SOURCE_DIR}/pe-sieve/paramkit/paramkit/include
    ${CMAKE_SOURCE_DIR}/pe-sieve/sig_finder/include
)

# Define static library macro
add_definitions(-DPESIEVE_STATIC_LIB)

# Link the library
target_link_libraries(YourProject
    ${CMAKE_SOURCE_DIR}/pe-sieve/x64/Release/PE_Sieve.lib
    psapi
    version
)
```

### Command Line (cl.exe)

```cmd
cl.exe YourApp.cpp ^
    /I"C:\path\to\pe-sieve\include" ^
    /I"C:\path\to\pe-sieve\libpeconv\include" ^
    /I"C:\path\to\pe-sieve\paramkit\paramkit\include" ^
    /I"C:\path\to\pe-sieve\sig_finder\include" ^
    /D"PESIEVE_STATIC_LIB" ^
    /link ^
    "C:\path\to\pe-sieve\x64\Release\PE_Sieve.lib" ^
    psapi.lib version.lib
```

## Sample Code

### Basic Usage Example

```cpp
#define PESIEVE_STATIC_LIB  // Must be defined BEFORE including headers
#include <pe_sieve_api.h>
#include <pe_sieve_types.h>
#include <iostream>

int main()
{
    // Initialize parameters with default values
    PEsieve_params params = { 0 };
    params.pid = 1234;  // Target process ID
    
    // Configure scan settings
    params.quiet = true;
    params.no_hooks = false;
    params.shellcode = SHELLC_PATTERNS;
    params.data = PE_DATA_SCAN_ALWAYS;
    params.minidump = false;
    params.json_output = false;
    params.log = false;
    
    // Perform the scan
    PEsieve_report report = PESieve_scan(&params);
    
    // Check results
    if (report.scanned == ERROR_SCAN_FAILURE) {
        std::cerr << "Scan failed!" << std::endl;
        return 1;
    }
    
    std::cout << "Scan Results:" << std::endl;
    std::cout << "  Scanned:    " << report.scanned << std::endl;
    std::cout << "  Suspicious: " << report.suspicious << std::endl;
    std::cout << "  Replaced:   " << report.replaced << std::endl;
    std::cout << "  Hooked:     " << report.hooked << std::endl;
    std::cout << "  Implanted:  " << report.implanted << std::endl;
    
    return 0;
}
```

### Advanced Usage with JSON Output

```cpp
#define PESIEVE_STATIC_LIB
#include <pe_sieve_api.h>
#include <pe_sieve_types.h>
#include <windows.h>
#include <iostream>
#include <vector>

int main()
{
    PEsieve_params params = { 0 };
    params.pid = GetCurrentProcessId();  // Scan current process
    params.quiet = true;
    params.shellcode = SHELLC_PATTERNS_OR_STATS;
    params.iat = PE_IATS_UNFILTERED;
    params.data = PE_DATA_SCAN_ALWAYS;
    
    // Get JSON report
    size_t buf_needed = 0;
    std::vector<char> json_buffer;
    
    // First call to get required buffer size
    PEsieve_report report = PESieve_scan_ex(
        &params,
        pesieve::REPORT_JSON,
        nullptr,
        0,
        &buf_needed
    );
    
    if (buf_needed > 0) {
        // Allocate buffer and get the report
        json_buffer.resize(buf_needed);
        report = PESieve_scan_ex(
            &params,
            pesieve::REPORT_JSON,
            json_buffer.data(),
            json_buffer.size(),
            &buf_needed
        );
        
        std::cout << "JSON Report:\n" << json_buffer.data() << std::endl;
    }
    
    return 0;
}
```

### Scanning Multiple Processes

```cpp
#define PESIEVE_STATIC_LIB
#include <pe_sieve_api.h>
#include <pe_sieve_types.h>
#include <windows.h>
#include <tlhelp32.h>
#include <iostream>
#include <vector>

std::vector<DWORD> GetAllProcessIds()
{
    std::vector<DWORD> pids;
    HANDLE snapshot = CreateToolhelp32Snapshot(TH32CS_SNAPPROCESS, 0);
    
    if (snapshot != INVALID_HANDLE_VALUE) {
        PROCESSENTRY32 pe32 = { sizeof(PROCESSENTRY32) };
        
        if (Process32First(snapshot, &pe32)) {
            do {
                pids.push_back(pe32.th32ProcessID);
            } while (Process32Next(snapshot, &pe32));
        }
        CloseHandle(snapshot);
    }
    return pids;
}

int main()
{
    PEsieve_params params = { 0 };
    params.quiet = true;
    params.shellcode = SHELLC_PATTERNS;
    params.no_hooks = false;
    params.data = PE_DATA_SCAN_NO_DEP;
    
    std::vector<DWORD> pids = GetAllProcessIds();
    int suspicious_count = 0;
    
    for (DWORD pid : pids) {
        params.pid = pid;
        PEsieve_report report = PESieve_scan(&params);
        
        if (report.scanned != ERROR_SCAN_FAILURE && report.suspicious > 0) {
            std::cout << "PID " << pid << " has " 
                      << report.suspicious << " suspicious module(s)" 
                      << std::endl;
            suspicious_count++;
        }
    }
    
    std::cout << "\nTotal suspicious processes: " << suspicious_count << std::endl;
    return 0;
}
```

## Important Notes

### 1. Runtime Requirements
- Target process must have appropriate access rights
- Running as Administrator may be required for scanning protected processes
- Antivirus may flag PE-sieve functionality (false positive)

### 2. Thread Safety
- PE-sieve functions are generally thread-safe for scanning different processes
- Avoid concurrent scans of the same process from multiple threads

### 3. Performance Considerations
- Scanning is CPU-intensive, especially with shellcode detection enabled
- Consider using a separate thread for scanning to avoid blocking UI
- Cache results when possible

### 4. Platform Support
- Windows only (uses Windows-specific APIs)
- Both 32-bit and 64-bit builds available
- Match your application architecture with PE_Sieve.lib architecture

## Troubleshooting

### Linker Error: "unresolved external symbol"
**Cause:** Missing `PESIEVE_STATIC_LIB` definition or incorrect library path

**Solution:**
```cpp
#define PESIEVE_STATIC_LIB  // Add this BEFORE including headers
#include <pe_sieve_api.h>
```

> **Note:** Since libpeconv and sig_finder are compiled into `PE_Sieve.lib`, you should **not** see unresolved externals for `peconv::` or `sig_finder::` symbols. If you do, ensure you're linking the correct build of `PE_Sieve.lib`.

### Compiler Error: "cannot open include file"
**Cause:** Include directories not configured correctly

**Solution:** Verify all four include paths are added:
- `pe-sieve/include`
- `pe-sieve/libpeconv/include`
- `pe-sieve/paramkit/paramkit/include`
- `pe-sieve/sig_finder/include`

### Runtime Error: Access Denied
**Cause:** Insufficient privileges to access target process

**Solution:** Run as Administrator or adjust target process selection

### Build Configuration Mismatch
**Cause:** Mixing Debug/Release or x86/x64 builds

**Solution:** Ensure PE_Sieve.lib matches your application's configuration

## API Reference

### Core Functions

#### `PESieve_scan`
```cpp
PEsieve_report PESieve_scan(const PEsieve_params *args);
```
Performs a scan with the specified parameters.

#### `PESieve_scan_ex`
```cpp
PEsieve_report PESieve_scan_ex(
    const PEsieve_params *args,
    const PEsieve_rtype rtype,
    char* json_buf,
    size_t json_buf_size,
    size_t *buf_needed_size
);
```
Extended scan function with JSON report support.

#### `PESieve_help`
```cpp
void PESieve_help(void);
```
Displays help information (MessageBox on Windows).

#### `PESieve_version`
```cpp
extern const DWORD PESieve_version;
```
PE-sieve version as a DWORD.

### Key Structures

#### `t_params` / `PEsieve_params`
Main configuration structure - see `pe_sieve_types.h` for all fields.

#### `t_report` / `PEsieve_report`
Scan result structure with counters for detected issues.

## Additional Resources

- **PE-sieve GitHub**: https://github.com/hasherezade/pe-sieve
- **libpeconv Documentation**: https://github.com/hasherezade/libpeconv
- **API Documentation**: See comments in `pe_sieve_api.h` and `pe_sieve_types.h`

## License

PE-sieve is licensed under BSD 2-Clause License. Ensure compliance when distributing your application.
