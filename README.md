# Arbalest - Dynamic Data Inconsistency Detector for OpenMP programs

This directory and its sub-directories contain the source code for a customized LLVM 15.
We modified ThreadSanitizer (compiler-rt/lib/tsan) to implement a prototype of our data inconsistency detector.
In addition, we also implemented all OpenMP Tool interface (OMPT) callbacks for OpenMP device offloading according to the 5.2 version specification.

Note: this prototype only supports the x86-64 architecture

## How to install Arbalest

We have provided a bash script to help you install Arbalest.  

```c
install_arbalest.sh [BUILD_DIR] [INSTALL_DIR]

// [BUILD_DIR]: the directory where cmake & ninja compile the source code
// [INSTALL_DIR]: the directory where the customized LLVM will be installed
```

## How to use Arbalest
We will use the following example (example.cpp) to show how to use Arbalest  
```c
     1	#include <cstdio>
     2	#define N 1000
     3
     4	int main() {
     5	    int a[N];
     6	    #pragma omp target teams distribute map(from: a[0:N]) // map-type should be "tofrom"
     7	    for (int i = 0; i < N; i++) {
     8	        a[i] += i;                                 // read uninitialized value from a[i]
     9	    }
    10
    11	    printf("a[%d] = %d\n", 3, a[3]);
    12	    return 0;
    13	}

```

### Compile the OpenMP program with OpenMP and ThreadSanitizer enabled
```c
   clang++ -fopenmp -fopenmp-targets=x86_64-pc-linux-gnu -fsanitize=thread -g -o example.exe example.cpp
```

### Execute the OpenMP program
```c
   export TSAN_OPTIONS='ignore_noninstrumented_modules=1' // this option is needed to avoid false positives
   ./example.exe
```

### Arbalest's Output

Note: The line numbers in the Arbalest report may experience slight discrepancies, either shifting up or down. The underlying issue lies in ThreadSanitizer's inability to accurately retrieve every line number when OpenMP is enabled. We are actively working on resolving this issue to ensure accurate line numbers in the report.

```c
arbalest successfully starts 
LLVMSymbolizer: error reading file: No such file or directory
==================
WARNING: ThreadSanitizer: data mapping issue (uninitialized access) (pid=15148) on the target 
  Read of size 4 at 0x7ffc94e4a3f0 by main thread:
    #0 .omp_outlined._debug__ /home/lyu/Test/example.cpp:7:5 (tmpfile_KSsVh6+0x9b9)
    #1 .omp_outlined. /home/lyu/Test/example.cpp:6:5 (tmpfile_KSsVh6+0xc04)
    #2 __kmp_invoke_microtask <null> (libomp.so+0xb9202)

  Location is stack of main thread.

  Location is global '??' at 0x7ffc94e2c000 ([stack]+0x1e3f0)

SUMMARY: ThreadSanitizer: data mapping issue (uninitialized access) /home/lyu/Test/example.cpp:7:5 in .omp_outlined._debug__
==================
==================
WARNING: ThreadSanitizer: data mapping issue (uninitialized access) (pid=15148) on the target 
  Read of size 4 at 0x7ffc94e4a3f8 by main thread:
    #0 .omp_outlined._debug__ /home/lyu/Test/example.cpp:8:14 (tmpfile_KSsVh6+0xb04)
    #1 .omp_outlined. /home/lyu/Test/example.cpp:6:5 (tmpfile_KSsVh6+0xc04)
    #2 __kmp_invoke_microtask <null> (libomp.so+0xb9202)

  Location is stack of main thread.

  Location is global '??' at 0x7ffc94e2c000 ([stack]+0x1e3f8)

SUMMARY: ThreadSanitizer: data mapping issue (uninitialized access) /home/lyu/Test/example.cpp:8:14 in .omp_outlined._debug__
==================
a[3] = 1114118
ThreadSanitizer: reported 2 warnings
```


