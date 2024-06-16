---
title: "Whitebox fuzzing Python C Extensions"
date: 2024-06-08T21:10:59+02:00
lastmod: 2024-06-08T21:10:59+02:00
author: "stulle123"
summary: "Whitebox fuzzing Python C Extensions with Atheris, libFuzzer and AFL++."
tags: 
- Fuzzing
- Python
showToc: true
TocOpen: true
draft: false
disableShare: true
ShowReadingTime: true
ShowBreadCrumbs: true
ShowPostNavLinks: true
ShowWordCount: true
UseHugoToc: true
---

In this short series of blog posts I will show you how to fuzz Python native extensions with [Atheris](https://github.com/google/atheris), [libFuzzer](https://www.llvm.org/docs/LibFuzzer.html) and [AFL++](https://github.com/AFLplusplus/AFLplusplus).

In this post I will focus on whitebox fuzzing which means the source code of the native extensions is available to us. In the next post, I will blog about blackbox or binary-only fuzzing which is the opposite case.

## Why should I fuzz native extensions?

Behind the scenes many Python modules use C/C++ for running performance-critical code. For example, running `cloc` on numpy `1.26.4` results in 170,000 lines of C code. As we all know, C/C++ is prone to memory corruption and bugs in native extensions might even lead to escaping the Python sandbox (see [example exploit](https://medium.com/hackernoon/python-sandbox-escape-via-a-memory-corruption-bug-19dde4d5fea5) in numpy `1.11.0`).

```bash
~/numpy$ cloc *
    2041 text files.
    2039 unique files.                                          
     162 files ignored.

github.com/AlDanial/cloc v 1.90  T=4.56 s (416.3 files/s, 183300.1 lines/s)
-------------------------------------------------------------------------------
Language                     files          blank        comment           code
-------------------------------------------------------------------------------
Python                         736          47161          81270         170695
C                              173          40866          71153         167309
reStructuredText               479          23464          16957          65645
CSV                             30              0              0          36702
C/C++ Header                   230           4563           7477          32005
SVG                             26             48             11          19314
C++                             32           2405           1360          16052
Cython                          19           2875           7664           5723
SWIG                             9            330            538           2573
Meson                           17            194            344           2458
diff                             9            350           1556           1817
Fortran 90                      59            120             91            953
Fortran 77                      36             28            109            537
YAML                             5             63             62            354
Markdown                        10            110              0            342
Bourne Shell                    10             45             96            311
make                             4             48             44            213
TOML                             2             33             12            173
sed                              1              0             12            139
JSON                             1             18              0             73
DOS Batch                        1             11              8             55
CSS                              1             17              6             51
CMake                            1              9              9             47
INI                              3              1              0             38
NAnt script                      1              7              0             31
HTML                             1              2              0             21
TeX                              1              0              0             20
-------------------------------------------------------------------------------
SUM:                          1897         122768         188779         523651
-------------------------------------------------------------------------------
```

Despite many other techniques for detecting bugs, fuzzing can be great and easy way to find memory corruption bugs in Python native extensions. As an example, [Trail of Bits](https://www.trailofbits.com/) found a [bug](https://github.com/agronholm/cbor2/issues/198#issuecomment-1869630196) in the `cbor2` module which was detected by the [Atheris](https://github.com/google/atheris) fuzzer which we are going to introduce below.

## How can I find fuzzing targets?

If you have the source code available, this is more or less straightforward. For example, go to Github and identify native functions that parse obscure/complex data structures (e.g., file parsers). Ideally, such functions should handle inputs that can be controlled by an attacker via untrusted channels (e.g., sockets, files, etc.). Another interesting fuzzing target are functions that work with Python objects, see the warning from the [Python documentation](https://docs.python.org/3/c-api/memory.html):

> To avoid memory corruption, extension writers should never try to operate on Python objects with the functions exported by the C library: malloc(), calloc(), realloc() and free().

In cases where you don't have the native extension's source code, you can simply import it into Python and run `dir(<native_extension>)` to list its available methods. You could also just use a [decompiler](https://dogbolt.org/) and look for [PyArg_ParseTuple](https://docs.python.org/3/c-api/arg.html) symbols to identify exported functions.

## Fuzz hello world extension with Atheris

I prepared a hello world native extension as a vulnerable example to get you started. In this section we'll fuzz it with [Atheris](https://github.com/google/atheris) (version `2.3.0`) which is a coverage-guided fuzzer for Python code as well as native C extensions.

First, go through the required setup steps:

0) Make sure you have a working Docker installation.
1) Clone this repository: `$ git clone -b fuzz-extensions-in-2024 https://github.com/stulle123/fuzzing_native_python_extensions` and `cd` into it
2) Grab two :coffee: :coffee: and build the Docker container: `$ docker build -t fuzz -f Dockerfile` (this will take a while)
3) Run it: `$ docker run -it --rm -v $(pwd):/app/output/ --name fuzz fuzz bash` (all your changes will be lost once you exit the container, only the fuzzer's crash files will be saved to your host's current working directory)
4) Finally, we can manually trigger the bug:

```bash
[fuzz e20a39df2e4a] /app $ LD_PRELOAD=$(python3 -c "import atheris; print(atheris.path())")/asan_with_fuzzer.so \
    ASAN_OPTIONS="allocator_may_return_null=1,detect_leaks=0,external_symbolizer_path=$CLANG_DIR/bin/llvm-symbolizer" \
    python3 -c "import memory; memory.corruption(b'FUZZ')"
```

5) Next, trigger the bug with Atheris. Crash files will be stored on your host's current working directory.

```bash
[fuzz e20a39df2e4a] /app $ LD_PRELOAD=$(python3 -c "import atheris; print(atheris.path())")/asan_with_fuzzer.so \
    ASAN_OPTIONS="allocator_may_return_null=1,detect_leaks=0,external_symbolizer_path=$CLANG_DIR/bin/llvm-symbolizer" \
    python3 ./fuzzing_native_python_extensions/atheris/atheris_fuzz.py -artifact_prefix=/app/output
```

## Fuzz Numpy with Atheris

Next, we will look at a real-world integer overflow bug in Numpy `v1.11.0` which I took from Gabe Pike's great [blog post](https://medium.com/hackernoon/python-sandbox-escape-via-a-memory-corruption-bug-19dde4d5fea5). Can you spot it :wink:? ([Spoiler](https://github.com/stulle123/fuzzing_native_python_extensions/blob/main/course/trigger_bug_in_numpy_1.11.0.py))

```c
NPY_NO_EXPORT PyObject *
PyArray_Resize(PyArrayObject *self, PyArray_Dims *newshape, int refcheck,
        NPY_ORDER order)
{
    // npy_intp is `long long`
    npy_intp* new_dimensions = newshape->ptr;
    npy_intp newsize = 1;
    int new_nd = newshape->len;
    int k;
    // NPY_MAX_INTP is MAX_LONGLONG (0x7fffffffffffffff)
    npy_intp largest = NPY_MAX_INTP / PyArray_DESCR(self)->elsize;
    for(k = 0; k < new_nd; k++) {
        newsize *= new_dimensions[k];
        if (newsize <= 0 || newsize > largest) {
            return PyErr_NoMemory();
        }
    }
    if (newsize == 0) {
        sd = PyArray_DESCR(self)->elsize;
    }
    else {
        sd = newsize*PyArray_DESCR(self)->elsize;
    }
    /* Reallocate space if needed */
    new_data = realloc(PyArray_DATA(self), sd);
    if (new_data == NULL) {
        PyErr_SetString(PyExc_MemoryError,
                “cannot allocate memory for array”);
        return NULL;
    }
    ((PyArrayObject_fields *)self)->data = new_data;
```

As with the other example, let's first trigger the bug manually and then again with Atheris:

0) Trigger the bug with my example [trigger_bug_in_numpy_1.11.0.py](https://github.com/stulle123/fuzzing_native_python_extensions/blob/main/course/trigger_bug_in_numpy_1.11.0.py) script:

```bash
[fuzz e20a39df2e4a] /app $ LD_PRELOAD=$($CC -print-file-name=libclang_rt.ubsan_standalone-x86_64.so) \
    ASAN_OPTIONS="allocator_may_return_null=1,detect_leaks=0,external_symbolizer_path=$CLANG_DIR/bin/llvm-symbolizer" \
    python3 ./fuzzing_native_python_extensions/course/trigger_bug_in_numpy_1.11.0.py 
```

1) Now, find the bug with Atheris:

```bash
[fuzz e20a39df2e4a] /app $ LD_PRELOAD=$(python3 -c "import atheris; print(atheris.path())")/asan_with_fuzzer.so \
    ASAN_OPTIONS="allocator_may_return_null=1,detect_leaks=0,external_symbolizer_path=$CLANG_DIR/bin/llvm-symbolizer" \
    python3 ./fuzzing_native_python_extensions/course/numpy_fuzz.py -artifact_prefix=/app/output
```

## Fuzz hello world extension with libFuzzer

Atheris runs [libFuzzer](https://www.llvm.org/docs/LibFuzzer.html) under the hood, so no surprise that you can also fuzz Python native extensions with it:

0) Build the fuzzing harness:

```bash
[fuzz e20a39df2e4a] /app $ clang++ $(python3-config  --embed --cflags) $(python3-config --embed --ldflags) \
    -fsanitize=address,fuzzer -g -o lib_fuzz ./fuzzing_native_python_extensions/libfuzzer/lib_fuzz.c
```

1) Run it: 

```bash
[fuzz e20a39df2e4a] /app $ ASAN_OPTIONS="allocator_may_return_null=1,detect_leaks=0,external_symbolizer_path=$CLANG_DIR/bin/llvm-symbolizer" \
    ./lib_fuzz -artifact_prefix=/app/output
```

## Fuzz hello world extension with AFL++

Fuzzing Python native extensions with [AFL++](https://github.com/AFLplusplus/AFLplusplus) (version `4.22a`) is also straightforward:

0) Remove the `memory` Python module and re-build it again:

```bash
[fuzz e20a39df2e4a] /app $ pip3 uninstall -y memory && rm -rf ./fuzzing_native_python_extensions/build/ \
    CC=afl-clang-fast CXX=afl-clang-fast++ \
    LD=afl-clang-fast LDSHARED="clang -shared" python3 -m pip install ./fuzzing_native_python_extensions
```

1) Build the fuzzing harness:

```bash
[fuzz e20a39df2e4a] /app $ afl-clang-fast $(python3-config --embed --cflags) $(python3-config --embed --ldflags) \
    -o afl_fuzz ./fuzzing_native_python_extensions/afl++/whitebox_fuzz.c
```

2) Run it:

```bash
[fuzz e20a39df2e4a] /app $ afl-fuzz -i ./fuzzing_native_python_extensions/afl++/in -o /app/output -- ./afl_fuzz
```