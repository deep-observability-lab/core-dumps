
## ## Chapter 0 — Introduction

Crashpad is Google’s crash reporting system, designed to capture and preserve crash information when an application encounters a fatal error. Unlike simple core dumps, Crashpad provides:

- **A persistent handler process** (`crashpad_handler`) that survives crashes.
- **Reliable capture** of state even in severe crashes (segfaults, illegal instructions, stack overflows).
- **Structured storage** in a local crash database (`pending/`, `completed/`, `uploads/`).
- **Annotations**: Metadata like product, version, or runtime context.
- **Cross-platform support**: Used in Chromium, Electron, and other large projects.

Crashpad separates the **application process** (your program) from the **handler process**. The application tells the handler: “If I crash, record everything you can.” When a crash occurs, Crashpad writes a **minidump** (`.dmp` file) that can later be symbolized and inspected.

---
## 1-Installing and Building Crashpad

Crashpad uses Chromium’s **depot_tools**. Steps :
```bash
# Install prerequisites
sudo apt-get update
sudo apt-get install git python3 g++ pkg-config libglib2.0-dev

# Clone depot_tools
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
export PATH=path/to/depot_tools:$PATH

# Get Crashpad source
mkdir crashpad && cd crashpad
fetch crashpad

# Build Crashpad
cd crashpad
gn gen out/Default
ninja -C out/Default

```
Key output:
- `out/Default/crashpad_handler` (the handler executable)
- Libraries in `out/Default/obj/` (`libclient.a`, `libutil.a`, etc.)

---
## 2-Crashpad concepts 

- **CrashpadClient**: Starts the handler.
- **crashpad_handler**: External process that saves crashes.
- **CrashReportDatabase**: Stores crash dumps (`.dmp` files) and metadata.
- **Annotations**: Key/value pairs describing your app (product name, version, etc.).
- **Symbolization and stack walking**: Converting raw crash addresses into function names using `.sym` files and stack walking using minidump_stackwalk tool.
---
## 3-Minimal C++ Example

In our first example (`multiple_files`):
- We created a small C++ program that initializes Crashpad.
- It starts the handler process, points it to a local database (`./db`), and attaches annotations.
- The program then deliberately dereferences a null pointer .
- Crashpad catches the segmentation fault and writes a minidump to `db/pending/`.
**Crashpad’s role here**:  
It runs in the background, and when the app crashes, the handler writes all relevant process state (registers, stack, memory around fault) into a `.dmp` file. The app itself terminates, but the dump survives for later debugging.

---
##  4- Pure C Example (with C++ Wrapper)

Crashpad is a C++ library. To integrate it with a C project:

- We wrote the main setup code in **C++** to initialize Crashpad.
- The actual crash logic lived in a **C file** (`crash.c`) with a `run_c_crash()` function.
- The wrapper (`wrapper.cpp`) called `run_c_crash()` after starting the handler.
- In C, we allocated and freed a node structure, then accessed it after free (a **use-after-free crash**).

**Crashpad’s role here**:  
Even though the bug occurs entirely inside the C function, the handler still records it. This demonstrates that Crashpad doesn’t care if the code is C or C++ — once initialized, it captures crashes from the whole process.
The dump contains a backtrace pointing into the C code, which can be symbolized for clear function/line mapping.

---
##  5- Shared Library Example

We extended the concept to a **shared library**:

- A shared library (`libcrasher.so`) was built in C with a function (`crash_me()`) that performs a null pointer dereference.
- The main program (in C++) dynamically loaded the library with `dlopen` and called `crash_me()`.
- The crash originated inside the shared library.

**Crashpad’s role here**:  
It doesn’t matter whether the crash occurs in the main binary or in a dynamically loaded library. The handler records the exact faulting instruction and stack trace. When symbolized, the dump shows the crash inside the shared library function (`crash_me`).

This example proves Crashpad can handle modular programs that mix main executables and dynamically loaded plugins.

---
## 6- Working with Crash Dumps

After running any example, Crashpad saves dumps in:
```bash
./db/pending/
```
Each file is a `.dmp`. To inspect:
1. **Create symbol files** (using `dump_syms` from Breakpad):	
```bash
dump_syms example_cpp > example_cpp.sym
```
2.  **Organize symbols** into the right folder structure (`./symbols/<binary_id>/`).
3. **Stack walk the dump**:
```bash
minidump_stackwalk ./db/completed/<dumpfile>.dmp ./symbols/
```
This prints a human-readable backtrace with source file names and line numbers.

---
## 7- Configuring Crash Database

Crashpad provides a local database to manage reports. With `CrashReportDatabase::Initialize`:
- **Completed reports** go to `db/completed/`.
- **Pending reports** go to `db/pending/` (waiting for upload).
- **Uploads** would be retried if enabled.

You can disable uploads during local testing:
```cpp
auto db = crashpad::CrashReportDatabase::Initialize("./db");
if (db && db->GetSettings()) {
    db->GetSettings()->SetUploadsEnabled(false);
}
```
This ensures reports remain local.

