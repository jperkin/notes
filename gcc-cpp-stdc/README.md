## GCC defining __STDC_VERSION__ in C++ code

GCC on illumos behaves differently compared to other platforms, specifically in
regards to defining `__STDC_VERSION__` in the pre-processor whilst running in
C++ mode.

This is done here:

<https://github.com/gcc-mirror/gcc/blob/releases/gcc-13.2.0/gcc/config/sol2.h#L97-L112>

### Problem

This can cause problems in third-party code which assumes that if
`__STDC_VERSION__` is defined then the code in question must be C, not C++.

Here's an example of third party software doing this from
[cython](https://github.com/cython/cython/blob/3.0.10/Cython/Utility/MemoryView_C.c#L34-L44):

```c
// For standard C/C++ atomics, get the headers first so we have ATOMIC_INT_LOCK_FREE
// defined when we decide to use them.
#if CYTHON_ATOMICS && (defined(__STDC_VERSION__) && \
                        (__STDC_VERSION__ >= 201112L) && \
                        !defined(__STDC_NO_ATOMICS__))
    #include <stdatomic.h>
#elif CYTHON_ATOMICS && (defined(__cplusplus) && ( \
                    (__cplusplus >= 201103L) || \
                    (defined(_MSC_VER) && _MSC_VER >= 1700)))
    #include <atomic>
#endif
```

While this code doesn't affect cython directly, it breaks the build of a
dependency that uses it, such as
[scipy](https://us-central.manta.mnx.io/pkgsrc/public/reports/upstream-trunk/20240515.2249/py311-scipy-1.13.0/build.log),
which tries to use the C stdatomic in C++ code:

```
[1350/1475] Compiling C++ object scipy/optimize/_highs/_highs_wrapper.so.p/meson-generated__highs_wrapper.cpp.o
FAILED: scipy/optimize/_highs/_highs_wrapper.so.p/meson-generated__highs_wrapper.cpp.o 
g++ -Iscipy/optimize/_highs/_highs_wrapper.so.p -Iscipy/optimize/_highs -I../scipy/optimize/_highs -I../scipy/optimize/_highs/cython/src -I../scipy/optimize/_highs/src -I../scipy/_lib/highs/src -I../scipy/_lib/highs/src/io -I../scipy/_lib/highs/src/lp_data -I../scipy/_lib/highs/src/util -I../../../../../../../../opt/local/lib/python3.11/site-packages/numpy/core/include -I/opt/local/include/python3.11 -I/opt/local/include -I/usr/include -fvisibility=hidden -fvisibility-inlines-hidden -fdiagnostics-color=always -D_FILE_OFFSET_BITS=64 -Wall -Winvalid-pch -std=c++17 -O3 -pipe -O2 -msave-args -fno-aggressive-loop-optimizations -D__STDC_FORMAT_MACROS -fPIC -pthread -DNPY_NO_DEPRECATED_API=NPY_1_9_API_VERSION -Wno-class-memaccess -Wno-format-truncation -Wno-non-virtual-dtor -Wno-sign-compare -Wno-switch -Wno-unused-but-set-variable -Wno-unused-variable '-DCMAKE_BUILD_TYPE="RELEASE"' -DFAST_BUILD=ON '-DHIGHS_GITHASH="n/a"' '-DHIGHS_COMPILATION_DATE="2021-07-09"' -DHIGHS_VERSION_MAJOR=1 -DHIGHS_VERSION_MINOR=2 -DHIGHS_VERSION_PATCH=0 -DHIGHS_DIR=/home/pbulk/build/math/py-scipy/work/scipy-1.13.0/scipy/optimize/_highs/../../_lib/highs -UOPENMP -UEXT_PRESOLVE -USCIP_DEV -UHiGHSDEV -UOSI_FOUND -DNDEBUG -DCYTHON_CCOMPLEX=0 -MD -MQ scipy/optimize/_highs/_highs_wrapper.so.p/meson-generated__highs_wrapper.cpp.o -MF scipy/optimize/_highs/_highs_wrapper.so.p/meson-generated__highs_wrapper.cpp.o.d -o scipy/optimize/_highs/_highs_wrapper.so.p/meson-generated__highs_wrapper.cpp.o -c scipy/optimize/_highs/_highs_wrapper.so.p/_highs_wrapper.cpp
scipy/optimize/_highs/_highs_wrapper.so.p/_highs_wrapper.cpp:1619:35: error: 'atomic_int' does not name a type
 1619 |     #define __pyx_atomic_int_type atomic_int
      |                                   ^~~~~~~~~~
```

### Workaround

A workaround is to change all third-party code from:

```c
#if defined(__STDC_VERSION__) ...
```

to:

```c
#if !defined(__cplusplus) && defined(__STDC_VERSION__) ...
```

and that is indeed enough to at least fix the scipy build shown above, but this
is tedious to perform across multiple third-party packages, and would require
continued maintenance into the future.

### Proposed Fixes

#### Remove __STDC_VERSION__

This completely removes the `__STDC_VERSION__` defines from GCC when in C++
mode, making the compiler behave the same on illumos as on other platforms:

```diff
--- gcc/config/sol2.h.orig	2023-08-04 21:07:54.000000000 +0000
+++ gcc/config/sol2.h
@@ -95,22 +95,6 @@ along with GCC; see the file COPYING3.
        library.  */					\
     if (c_dialect_cxx ())				\
       {							\
-	switch (cxx_dialect)				\
-	  {						\
-	  case cxx98:					\
-	  case cxx11:					\
-	  case cxx14:					\
-	    /* C++11 and C++14 are based on C99.	\
-	       libstdc++ makes use of C99 features	\
-	       even for C++98.  */			\
-	    builtin_define ("__STDC_VERSION__=199901L");\
-	    break;					\
-							\
-	  default:					\
-	    /* C++17 is based on C11.  */		\
-	    builtin_define ("__STDC_VERSION__=201112L");\
-	    break;					\
-	  }						\
 	builtin_define ("_XOPEN_SOURCE=600");		\
 	builtin_define ("_LARGEFILE_SOURCE=1");		\
 	builtin_define ("_LARGEFILE64_SOURCE=1");	\
```

Unfortunately this almost immediately causes fallout, when attempting to build
cmake:

```
[ 58%] Building CXX object Source/CMakeFiles/CMakeLib.dir/cmFindProgramCommand.cxx.o
/home/pbulk/build/devel/cmake/work/cmake-3.29.3/Source/cmFileCommand.cxx: In member function 'bool {anonymous}::cURLProgressHelper::UpdatePercentage({anonymous}::cm_curl_off_t, {anonymous}::cm_curl_off_t, std::string&)':
/home/pbulk/build/devel/cmake/work/cmake-3.29.3/Source/cmFileCommand.cxx:1746:38: error: 'lround' is not a member of 'std'; did you mean 'lround'?
 1746 |       this->CurrentPercentage = std::lround(
      |                                      ^~~~~~
In file included from /usr/include/math.h:36,
                 from /opt/local/gcc13/include/c++/13.2.0/bits/std_abs.h:40,
                 from /opt/local/gcc13/include/c++/13.2.0/cstdlib:81,
                 from /opt/local/gcc13/include/c++/13.2.0/ext/string_conversions.h:43,
                 from /opt/local/gcc13/include/c++/13.2.0/bits/basic_string.h:4097,
                 from /opt/local/gcc13/include/c++/13.2.0/string:54,
                 from /home/pbulk/build/devel/cmake/work/cmake-3.29.3/Source/cmFileCommand.h:7,
                 from /home/pbulk/build/devel/cmake/work/cmake-3.29.3/Source/cmFileCommand.cxx:3:
/usr/include/iso/math_c99.h:238:17: note: 'lround' declared here
  238 | extern long int lround(double);
      |                 ^~~~~~
```

The problem is that `sys/feature_tests.h` checks for `__STDC_VERSION__` and
sets the internal `_STDC_C99` or `_STDC_C11` defines accordingly.  These are
then used throughout the system headers to enable features, and in this case
means that `lround()` in `iso/math_c99.h` is not exposed.

Without significant work to the illumos headers, this change is a dead end.

#### Define _STDC_C99 / _STDC_C11 directly

Rather than defining `__STDC_VERSION__`, define `_STDC_C99` and `_STDC_C11`
directly.  While this is frowned upon in the system headers as they are marked
as internal, it feels reasonable to do this in the compiler in a similar manner
to how Sun Studio defined `__C99FEATURES__`, and it shouldn't conflict with
third-party software.

```diff
--- gcc/config/sol2.h.orig	2023-08-04 21:07:54.000000000 +0000
+++ gcc/config/sol2.h
@@ -103,12 +103,13 @@ along with GCC; see the file COPYING3.
 	    /* C++11 and C++14 are based on C99.	\
 	       libstdc++ makes use of C99 features	\
 	       even for C++98.  */			\
-	    builtin_define ("__STDC_VERSION__=199901L");\
+	    builtin_define ("_STDC_C99");		\
 	    break;					\
 							\
 	  default:					\
 	    /* C++17 is based on C11.  */		\
-	    builtin_define ("__STDC_VERSION__=201112L");\
+	    builtin_define ("_STDC_C99");		\
+	    builtin_define ("_STDC_C11");		\
 	    break;					\
 	  }						\
 	builtin_define ("_XOPEN_SOURCE=600");		\
```

Note that for the C++17 or newer case that both `_STDC_C99` and `_STDC_C11`
must be defined, as there are certain sections in the system headers that only
check for the former.

This fixes the scipy build, but in a bulk build was shown to cause two
regressions in packages that specifically use C++03 in their `va_copy()`
handling.  This was tracked down to being due to:

Unpatched:

```shell
$ echo '#include <stdarg.h>' | g++ -std=c++03 -x c++ -dM -E - | grep va_copy
#define __va_copy(d,s) __builtin_va_copy(d,s)
#define va_copy(d,s) __builtin_va_copy(d,s)
```

Patched:

```shell
$ echo '#include <stdarg.h>' | g++ -std=c++03 -x c++ -dM -E - | grep va_copy
#define __va_copy(d,s) __builtin_va_copy(d,s)
```

These definitions aren't coming from the system `stdarg.h`, but from
`/opt/local/gcc13/lib/gcc/x86_64-sun-solaris2.11/13.2.0/include/stdarg.h`.
That file includes this section:

```c
#if !defined(__STRICT_ANSI__) || __STDC_VERSION__ + 0 >= 199900L \
    || __cplusplus + 0 >= 201103L
#define va_copy(d,s)    __builtin_va_copy(d,s)
#endif
```

In order to match the historical behaviour of C++ on illumos the following
patch is also applied:

```diff
--- gcc/ginclude/stdarg.h.orig  2024-05-24 11:43:06.932507694 +0000
+++ gcc/ginclude/stdarg.h
@@ -52,7 +52,7 @@ typedef __builtin_va_list __gnuc_va_list
 #define va_end(v)      __builtin_va_end(v)
 #define va_arg(v,l)    __builtin_va_arg(v,l)
 #if !defined(__STRICT_ANSI__) || __STDC_VERSION__ + 0 >= 199900L \
-    || __cplusplus + 0 >= 201103L
+    || __cplusplus + 0 >= 201103L || (defined(__cplusplus) && defined(__sun))
 #define va_copy(d,s)   __builtin_va_copy(d,s)
 #endif
 #define __va_copy(d,s) __builtin_va_copy(d,s)
```

These changes are now being used in SmartOS pkgsrc trunk
builds, specifically using
[these](https://github.com/TritonDataCenter/pkgsrc-extra/commit/e295cdbf14a6991ff91ad6e5624ddec0f96244ca)
[commits](https://github.com/TritonDataCenter/pkgsrc-extra/commit/0880cfb5c72d83554df732682f97e56fab720662).
