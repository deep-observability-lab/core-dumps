# 1.  Introduction

**Breakpad** is an open-source crash reporting system developed by Google. It provides a lightweight way for C++ applications to automatically generate _minidump_ files whenever they crash. Unlike full core dumps, minidumps are compact and easy to collect in production.

With Breakpad you can:
- Detect and capture application crashes in real-world deployments.
- Store crash dumps locally or upload them to a server.
- Symbolize stack traces later, turning raw memory addresses into function names and line numbers.
- Debug issues post-mortem without interrupting users.

Breakpad is cross-platform, but this guide focuses only on **Linux + C++**, giving you a practical workflow from setup to crash analysis. It’s a perfect fit for developers who want reliable crash diagnostics without heavy dependencies.

---
# 2. Prerequisites

Before building Breakpad on Linux, make sure your system has the standard development tools.

### Required tools
```bash
sudo apt update
sudo apt install -y build-essential autoconf automake libtool pkg-config git
```
- **build-essential** → compilers, linker, make.
- **autoconf/automake/libtool** → needed for Breakpad’s build system.
- **pkg-config** → helps locate libraries.
- **git** → to clone the source.
### Optional 
- **depot_tools** → Google’s helper scripts if you want to use `fetch breakpad` instead of plain `git clone`.

---

# 3. Getting the Source

You can fetch Breakpad with these commands : 
```bash
git clone https://chromium.googlesource.com/breakpad/breakpad
cd breakpad
git clone https://chromium.googlesource.com/linux-syscall-support src/third_party/lss
```

- The extra `linux-syscall-support (lss)` library is required for Linux builds.
- After this, your tree should look like:

---
# 4. Building Breakpad

From the `breakpad` directory:
```bash
cd breakpad
./configure
make
```

- `./configure` will detect your environment.
- `make` builds libraries and tools (like `minidump_stackwalk`).

To install system-wide:
```bash
sudo make install
```

This typically installs:
- **Libraries** → `/usr/local/lib/`
- **Headers** → `/usr/local/include/breakpad/`
- **Tools** (like `minidump_stackwalk`) → `/usr/local/bin/`
You can also skip installation and use the built artifacts directly from the build tree.

---
# 5. Integrating Breakpad into a C++ App

After installing, you can use Breakpad’s client library to capture crashes.  
Minimal example:
``` cpp
#include <client/linux/handler/exception_handler.h>
#include <iostream>

using namespace google_breakpad;

// Callback when a dump is written
bool DumpCallback(const MinidumpDescriptor& descriptor,
                  void* context,
                  bool succeeded) {
    std::cout << "Minidump created at: " << descriptor.path() << std::endl;
    return succeeded;
}

int main() {
    // Set where dumps will be written
    MinidumpDescriptor descriptor("./dumps");
    ExceptionHandler eh(descriptor, nullptr, DumpCallback, nullptr, true, -1);

    // Trigger a crash
    int* ptr = nullptr;
    *ptr = 42; // segfault

    return 0;
}

```
### Build it
```bash
g++ main.cpp -lbreakpad_client -o crash_app
```
Now run `./crash_app` and you’ll see a `.dmp` file inside `./dumps/`.

---
# 6. Symbols & Symbolization

For human-readable stack traces, Breakpad needs **symbol files**.
Build your program with debug symbols:
1.  Build your program with debug symbols:
	```bash
	g++ -g main.cpp -lbreakpad_client -o crash_app
	```
2. Use `dump_syms` to generate symbols:
	```bash
	dump_syms crash_app > crash_app.sym
	```
3. Place it in the correct directory structure:
```css
symbols/
  crash_app/
    <build_id>/
      crash_app.sym
```
- `<build_id>` comes from the binary itself (Breakpad tools check this automatically).

---
# 7. Collecting Minidumps

Your app will produce `.dmp` files whenever it crashes.  
Typical workflow:
- **Local debugging:** keep dumps in `./dumps/` and process them yourself.  
- **Production:** upload dumps to a server, then process them later.
Breakpad doesn’t enforce an uploader — you write your own (e.g., HTTP POST with the `.dmp` file).

---
# 8. Processing Minidumps (Server/Tooling)

Use `minidump_stackwalk` to turn dumps into stack traces:
```bash
minidump_stackwalk ./dumps/<dumpfile>.dmp ./symbols > crash_report.txt
```
This will produce a human-readable backtrace, mapping addresses to function names and source lines.

