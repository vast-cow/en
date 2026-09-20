---
title: "How to Use `getopt` with `getopt.h` / `getopt.c` in MSVC and Licensing Considerations"
description: "Integrate MinGW-w64 getopt source files into an MSVC project, parse short and long options, and preserve the applicable third-party license notices."
pubDatetime: 2026-06-23T10:16:23.013Z
updatedDate: 2026-06-23T10:20:36.477Z
---

## Prerequisites

The Windows MSVC environment does not provide the POSIX-style `getopt.h` / `getopt()` commonly used on Unix-like systems. As a result, when building C/C++ code originally written for Linux with MSVC, you will typically encounter errors such as:

```text
fatal error C1083: cannot open include file: 'getopt.h'
```

Even if you add the header file, linking will still fail unless implementations of `getopt()`, `getopt_long()`, `optind`, `optarg`, and related symbols are available.

MSYS2 UCRT64 includes a MinGW-w64-derived `getopt.h`. The MSYS2 package `mingw-w64-ucrt-x86_64-headers` is distributed as MinGW-w64 headers for Windows and belongs to the UCRT64 repository. The package license is listed as `ZPL-2.1 AND LGPL-2.1-or-later`. ([MSYS2 Packages][1])

However, the `getopt.h` file itself contains a disclaimer stating that no copyright has been assigned and that it has been placed in the Public Domain. ([GitHub][2])

This article explains how to incorporate this MinGW-w64-derived `getopt.h` and `getopt.c` into an MSVC project to use `getopt()` and `getopt_long()`.

---

## Conclusion

The safest way to use `getopt` with MSVC is the following structure:

```text
your_project/
  include/
    getopt.h
  src/
    getopt.c
    main.c
```

Then compile `getopt.c` as part of your project:

```bat
cl /Iinclude src\main.c src\getopt.c
```

What you should **not** do is add MSYS2's entire `/ucrt64/include` or `/ucrt64/lib` directories to an MSVC project. This mixes MinGW-w64 and MSVC CRTs, headers, and libraries, which can easily lead to build and link problems.

---

## 1. Prepare the Files

Copy the following two files derived from MinGW-w64 into your project:

```text
getopt.h
getopt.c
```

If MSYS2 UCRT64 is installed, `getopt.h` is typically located at:

```text
C:\msys64\ucrt64\include\getopt.h
```

However, instead of adding the entire `C:\msys64\ucrt64\include` directory to the include path of an MSVC project, it is safer to copy only the required `getopt.h` into your project.

`getopt.c` is the implementation file from the MinGW-w64 CRT. The MSYS2 package `mingw-w64-ucrt-x86_64-crt` is distributed as MinGW-w64 CRT for Windows, and its package license is listed as `ZPL-2.1`. ([MSYS2 Packages][3])

However, the `getopt.c` file itself contains a permissive license originating from Todd C. Miller and BSD-style license text originating from NetBSD. These licenses permit redistribution in source and binary form, provided that copyright notices, conditions, and disclaimers are retained or reproduced. ([GitHub][4])

---

## 2. Minimal Example

### `main.c`

```c
#include <stdio.h>
#include <getopt.h>

int main(int argc, char *argv[])
{
    int opt;

    while ((opt = getopt(argc, argv, "ab:c")) != -1) {
        switch (opt) {
        case 'a':
            printf("option -a\n");
            break;

        case 'b':
            printf("option -b with argument: %s\n", optarg);
            break;

        case 'c':
            printf("option -c\n");
            break;

        default:
            printf("usage: %s [-a] [-b value] [-c]\n", argv[0]);
            return 1;
        }
    }

    for (int i = optind; i < argc; ++i) {
        printf("non-option argument: %s\n", argv[i]);
    }

    return 0;
}
```

### Build

From Developer Command Prompt for Visual Studio:

```bat
cl /Iinclude src\main.c src\getopt.c
```

Example execution:

```bat
main.exe -a -b hello file1 file2
```

```text
option -a
option -b with argument: hello
non-option argument: file1
non-option argument: file2
```

---

## 3. Example Using `getopt_long()`

MinGW-w64's `getopt.h` also includes declarations for `struct option`, `no_argument`, `required_argument`, `optional_argument`, `getopt_long()`, and `getopt_long_only()`. ([GitHub][2])

### `main.c`

```c
#include <stdio.h>
#include "getopt.h"

int main(int argc, char *argv[])
{
    static struct option long_options[] = {
        { "help",    no_argument,       0, 'h' },
        { "output",  required_argument, 0, 'o' },
        { "verbose", no_argument,       0, 'v' },
        { 0,         0,                 0,  0  }
    };

    int opt;
    int option_index = 0;

    while ((opt = getopt_long(argc, argv, "ho:v", long_options, &option_index)) != -1) {
        switch (opt) {
        case 'h':
            printf("help\n");
            break;

        case 'o':
            printf("output: %s\n", optarg);
            break;

        case 'v':
            printf("verbose\n");
            break;

        default:
            printf("usage: %s [--help] [--output file] [--verbose]\n", argv[0]);
            return 1;
        }
    }

    return 0;
}
```

### Example Execution

```bat
main.exe --verbose --output result.txt
```

```text
verbose
output: result.txt
```

---

## 4. Using It in a Visual Studio Project

For a `.vcxproj` project, configure it as follows.

### File Layout

```text
project/
  include/
    getopt.h
  src/
    getopt.c
    main.c
```

### Settings

| Item                                             | Setting                                  |
| ------------------------------------------------ | ---------------------------------------- |
| C/C++ → General → Additional Include Directories | `$(ProjectDir)include`                   |
| Source Files                                     | Add `getopt.c`                           |
| Linker                                           | No special additional libraries required |

If `getopt.c` is not added to the project, compilation may succeed but linking will fail. `getopt.h` only provides declarations; the actual definitions of `getopt()`, `optind`, and related symbols are contained in `getopt.c`. MinGW-w64's `getopt.c` provides definitions for `optind`, `opterr`, `optopt`, `optarg`, as well as implementations of `getopt()`, `getopt_long()`, and `getopt_long_only()`. ([GitHub][4])

---

## 5. Do Not Mix MSYS2 Include/Lib Directories Directly into MSVC

Avoid configurations like:

```text
C:\msys64\ucrt64\include
C:\msys64\ucrt64\lib
```

Adding these directories wholesale to MSVC include paths or library paths can mix MinGW-w64 headers and import libraries with MSVC's Windows SDK, UCRT, and CRT.

Problematic areas include:

| Area            | Issue                                                       |
| --------------- | ----------------------------------------------------------- |
| CRT headers     | MSVC CRT and MinGW-w64 CRT have different assumptions       |
| Windows headers | Windows SDK versions and MinGW-w64 versions may conflict    |
| `.a` libraries  | GCC-oriented import libraries differ from MSVC `.lib` files |
| ABI             | C++ and runtime boundary incompatibilities can occur        |

Therefore, if you only want to use `getopt` with MSVC, it is better to copy `getopt.h` and `getopt.c` locally and compile them with MSVC rather than linking against the MSYS2 environment itself.

---

## 6. Licensing Considerations

### `getopt.h`

MinGW-w64's `getopt.h` contains a disclaimer stating that no copyright has been assigned and that it has been placed in the Public Domain. ([GitHub][2])

As a result, merely including `getopt.h` does not normally impose GPL/LGPL-style source disclosure obligations on your program.

### `getopt.c`

The file requiring attention is `getopt.c`.

It contains at least the following license texts:

| Origin                     | Nature                                |
| -------------------------- | ------------------------------------- |
| Todd C. Miller portions    | Permissive license similar to ISC/BSD |
| NetBSD Foundation portions | BSD 2-clause style license            |

Both are permissive licenses and generally allow inclusion in commercial or closed-source products. However, the license terms require that copyright notices, conditions, and disclaimers be retained in source redistributions and reproduced in documentation or accompanying materials for binary redistributions. ([GitHub][4])

### Practical Compliance

When distributing binaries, provide something like:

```text
THIRD-PARTY-NOTICES.txt
LICENSES/
  getopt.txt
```

For safety, place the license comment block from the beginning of `getopt.c` into `getopt.txt` verbatim.

---

## 7. What Happens to the License of Your Binary?

Even if `getopt.h` and `getopt.c` are statically compiled into your application using MSVC, your entire application does **not** become GPL or LGPL.

The practical situation is:

| Item                         | Treatment                                              |
| ---------------------------- | ------------------------------------------------------ |
| Your own source code         | Any license you choose                                 |
| `getopt.h`                   | Contains Public Domain disclaimer                      |
| `getopt.c`                   | Subject to permissive license conditions               |
| Binary distribution          | Include copyright notices, conditions, and disclaimers |
| Source disclosure obligation | Normally none                                          |
| Commercial use               | Normally allowed                                       |

The important point is that the requirement is generally a **notice obligation**, not a **source disclosure obligation**.

---

## 8. Recommended Distribution Layout

Whether commercial or non-commercial, the following structure is recommended:

```text
dist/
  your_app.exe
  README.txt
  THIRD-PARTY-NOTICES.txt
```

Example `THIRD-PARTY-NOTICES.txt`:

```text
This product includes getopt implementation derived from MinGW-w64 / OpenBSD / NetBSD sources.

The getopt.h file from MinGW-w64 states that it has no copyright assigned
and is placed in the Public Domain.

The getopt.c file contains permissive license notices from Todd C. Miller
and The NetBSD Foundation, Inc. The original copyright notices, permission
notice, conditions, and disclaimers are reproduced below.

[Paste the license comment block from the beginning of getopt.c here verbatim]
```

Even if your distribution is entirely in Japanese, it is safest to include the original English license text. If you add a translation, do not remove the original text.

---

## 9. Implementation Notes

### Resetting `optind`

If `getopt()` needs to be run multiple times within the same process, `optind` must be reset:

```c
optind = 1;
```

The MinGW-w64 implementation also supports BSD-style `optreset`, but for portability, using `optind = 1` is generally preferable. `getopt.h` itself contains comments recommending that developers avoid relying on the non-standard BSD `optreset` for portability reasons. ([GitHub][2])

### Using from C++

`getopt.h` supports `extern "C"` and can therefore be used from C++ source files. ([GitHub][2])

```cpp
#include "getopt.h"
```

However, `getopt.c` should naturally be compiled as a C source file. In Visual Studio, leave its extension as `.c` when adding it to the project.

### Separate Issue from `/utf-8` and Unicode

`getopt()` operates on narrow strings from `main(int argc, char *argv[])`.

If you need precise handling of Unicode command lines on Windows, consider using `wmain(int argc, wchar_t *argv[])` or `CommandLineToArgvW()` instead.

`getopt` should be viewed as a POSIX-style parser for narrow-character `argv`.

---

## 10. Summary

When using `getopt` with MSVC, the safest approach is:

1. Copy `getopt.h` and `getopt.c` into your project.
2. Compile `getopt.c` as part of the project with MSVC.
3. Do not add MSYS2's `/ucrt64/include` or `/ucrt64/lib` directories wholesale to MSVC.
4. Treat `getopt.h` as having a Public Domain disclaimer.
5. Treat `getopt.c` as permissively licensed and include copyright notices, conditions, and disclaimers in distributed materials.
6. Your own binary normally incurs no source disclosure obligation, but you should provide appropriate third-party notices.

In practical terms, the key points for MSVC are: **build `getopt.c` together with your project, not just the header**, and **preserve the license notices from `getopt.c` in distributed materials**.

[1]: https://packages.msys2.org/package/mingw-w64-ucrt-x86_64-headers "Package: mingw-w64-ucrt-x86_64-headers - MSYS2 Packages"
[2]: https://raw.githubusercontent.com/mingw-w64/mingw-w64/master/mingw-w64-headers/crt/getopt.h "raw.githubusercontent.com"
[3]: https://packages.msys2.org/package/mingw-w64-ucrt-x86_64-crt "Package: mingw-w64-ucrt-x86_64-crt - MSYS2 Packages"
[4]: https://raw.githubusercontent.com/mingw-w64/mingw-w64/master/mingw-w64-crt/misc/getopt.c "raw.githubusercontent.com"
