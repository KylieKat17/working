# CPSC 3220 - Project 1  
### Heap-Leak Counter & Syscall Tracer  
**Author:** Kylie Gilbert · kgilbe3@g.clemson.edu · Fall 2025  

---

## 1). Overview
This submission contains three runnable deliverables:

| File / Target | Purpose |
|---------------|---------|
| **`mem_shim.so`** | LD\_PRELOAD shim that wraps the four glibc heap APIs, records every live allocation in an internal hash table, and prints a leak summary (`LEAK <n>` lines + a `TOTAL` line) when the program exits |
| **`leakcount`** | Launcher that sets **`LD_PRELOAD`** to `mem_shim.so`, then `execvp()`s the user’s program, forwarding its exit status |
| **`sctracer`** | `ptrace`-based system-call counter; writes a `<nr>\t<count>` histogram (ascending syscall number) to a file supplied by the user |

All code compiles warning-free with **`gcc -std=c11 -Wall -Wextra -pedantic`** and runs on the course x86-64 VM

---

## 2). Building

```bash
# inside project-1 directory
make            # builds leakcount, mem_shim.so, sctracer
make clean      # removes generated binaries
````

*Requires GCC >= 9 and glibc (I compiled using clangd 16.0.2)*

---

## 3). Running

### 3.1 Heap-leak counter

```bash
./leakcount  ./my_program  [arg0 arg1 …]

# stderr example:
#   LEAK  42
#   LEAK  128
#   TOTAL 2 170
```

### 3.2 Syscall tracer

```bash
./sctracer  results.sys  ./my_program  [args …]
head results.sys
# 0     1
# 1     5
# 60    1
```

`leakcount` returns the same exit code as the target; `sctracer` mirrors the child’s exit/signal status per project spec

---

## 4). Design Notes

### 4.1 `mem_shim.so`

* **Hash table** – 4096 chained buckets (`HASH_BUCKETS`), index = `(ptr >> 3) & (4096-1)` (dropping three alignment bits gives good key dispersion)
* **Entry structure**:

  ```c
  typedef struct MemEntry {
      void   *allocPtr;   // address returned to caller
      size_t  bytes;      // size requested
      struct MemEntry *next;
  } MemEntry;
  ```
* **Thread-safety** – single `pthread_mutex_t` (`tableLock`) guards each insert/remove; allocator symbols are resolved once with `pthread_once`
* **Interception** – wrappers call `realMalloc/realFree/…` (resolved via `dlsym(RTLD_NEXT, …)`), then update the table
* **Destructor** – `reportLeaks()` iterates every bucket, prints one `LEAK <bytes>` per surviving entry, and ends with `TOTAL <count> <sumBytes>`
* **Known TODOs** (left as comments, do not affect correctness):

  * Replace the `while` loop in `removeAllocation()` with a `for` if desired
  * Minor cosmetic “3u / 1u” literal tweaks noted in comments

### 4.2 `leakcount`

* Locates the shim at runtime via
  `dirname(argv[0]) + "/mem_shim.so"` → portable untar/run
* Overwrites any existing `LD_PRELOAD` so the course grader always loads *our* shim first

### 4.3 ` sctracer`

* Child: `PTRACE_TRACEME → SIGSTOP → execvp`
* Parent: sets `PTRACE_O_TRACESYSGOOD`; counts only **entry stops** (flag `isEntering`)
* Histogram: fixed array `[0…1023]`; prints only non-zero rows

---

## 5). Known Limitations

1. Programs that terminate via `_exit()` (or static binaries) skip C destructors, so `mem_shim.so` won’t print a summary
2. `sctracer` assumes the x86-64 user-regs layout (`orig_rax` syscall number)
3. Hash-table itself is intentionally not freed at exit (the process is dying anyway)
4. Was unable to complete desired refactoring of `mem_shim.c` so there are some missing comments and leftover to-dos, but hey, it runs!

---

## 6). Testing Tips

```bash
# tiny leak generator with no static destructors
cat > alloc_bomb.c <<'EOF'
#include <stdlib.h>
int main(void){ for(int i=0;i<100;i++) malloc(64); }
EOF
gcc -o alloc_bomb alloc_bomb.c

./leakcount ./alloc_bomb        # prints 100 leaks
./sctracer  bomb.sys ./alloc_bomb
head bomb.sys                   # syscall histogram
```

Avoid `/bin/ls` or C++ programs with global destructors—those free memory after the shim’s destructor runs and may skew counts (yes, I tried it, yes, it broke stuff)

---

*Submission includes*: source (.c), `makefile`, and this `README.md`
