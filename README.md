# 🚀 Jonathan

> A tiny, fast, header-only hooking library for x86/x64 Windows. No bullshit, just hooks.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](https://opensource.org/licenses/MIT)

Jonathan is designed for performance-critical applications where low latency and a small footprint are essential. It has no external dependencies, is fully thread-safe, and provides a clean, modern C++ API.

---

## ✨ Features

-   **📦 Header-Only**: Just drop `Jonathan.hpp` into your project.
-   **💻 x86 & x64 Support**: Works seamlessly on both architectures.
-   **⚡️ High Performance**: Prefers 5-byte `rel32` jumps when possible, automatically falling back to 14-byte `abs64` jumps for distant targets.
-   **🧠 Smart Allocation**: Jump pads are allocated in executable memory pages near the original function to improve CPU cache performance.
-   **🔒 Thread-Safe**: All public API functions are protected by a spinlock.
-   **🔧 Configurable**: Disable thread freezing for UWP/AppContainer compatibility.
-   **✨ RAII Handle**: Includes an optional `Jonathan::hook_handle` for easy, exception-safe hook management.
-   **💨 Batched Operations**: Queue multiple hooks to be enabled or disabled at once with a single `apply_queued()` call.

---

## 💾 Installation

This is a header-only library.

1.  Copy `Include/Jonathan.hpp` into your project's include directory.
2.  `#include "Jonathan.hpp"` wherever you need it.

That's it.

---

## 🏁 Quick Start

Here's a simple example of hooking `MessageBoxA`.

```cpp
#include <Windows.h>
#include "Jonathan.hpp"
#include <iostream>

// Define the function prototype for our hook
using MessageBoxA_t = int(WINAPI*)(HWND, LPCSTR, LPCSTR, UINT);

// This will hold the original function
MessageBoxA_t oMessageBoxA = nullptr;

// Our detour function
int WINAPI hMessageBoxA(HWND hWnd, LPCSTR lpText, LPCSTR lpCaption, UINT uType) {
    std::cout << "MessageBoxA hooked!" << std::endl;
    std::cout << "Caption: " << lpCaption << std::endl;
    std::cout << "Text: " << lpText << std::endl;

    // Call the original function
    return oMessageBoxA(hWnd, "Hooked!", "Hooked!", uType);
}

int main() {
    // 1. Initialize Jonathan
    if (Jonathan::init() != Jonathan::status::ok) {
        return 1;
    }

    HMODULE user32 = GetModuleHandleA("user32.dll");
    if (!user32) return 1;

    void* target = (void*)GetProcAddress(user32, "MessageBoxA");
    if (!target) return 1;

    // 2. Create the hook
    if (Jonathan::create_hook(target, &hMessageBoxA, (void**)&oMessageBoxA) != Jonathan::status::ok) {
        return 1;
    }

    // 3. Enable the hook
    if (Jonathan::enable_hook(target) != Jonathan::status::ok) {
        return 1;
    }

    // Call the function to test the hook
    MessageBoxA(NULL, "Hello", "Test", 0);

    // 4. Disable and remove the hook
    Jonathan::disable_hook(target);
    Jonathan::remove_hook(target);

    // 5. Shutdown Jonathan
    Jonathan::shutdown();

    return 0;
}
```

### RAII Usage (Recommended)

The `hook_handle` class manages the lifetime of a hook automatically.

```cpp
#include "Jonathan.hpp"

int main() {
    if (Jonathan::init() != Jonathan::status::ok) return 1;

    void* target = (void*)GetProcAddress(GetModuleHandleA("user32.dll"), "MessageBoxA");

    {
        // Create the handle. Hook is created and enabled automatically.
        Jonathan::hook_handle<MessageBoxA_t> handle(target, &hMessageBoxA);

        // Get the original function from the handle
        oMessageBoxA = handle.original();
        
        MessageBoxA(NULL, "Hello from RAII", "Test RAII", 0);

        // Hook is automatically disabled and removed when 'handle' goes out of scope
    }

    Jonathan::shutdown();
    return 0;
}
```

---

## ⚙️ Configuration

You can configure Jonathan's behavior by defining these macros before including the header:

-   `JONATHAN_CFG_FREEZE_THREADS`: Set to `0` to disable thread suspension during patching. This is required for environments like UWP or Windows AppContainer where you don't have permission to suspend threads. Defaults to `1` (enabled).

**Example:**
```cpp
#define JONATHAN_CFG_FREEZE_THREADS 0
#include "Jonathan.hpp"
```

---

## 📜 License

This project is licensed under the MIT License. See the `LICENSE` file for details.
