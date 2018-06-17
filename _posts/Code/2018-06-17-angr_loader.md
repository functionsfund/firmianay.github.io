---
layout: post
title: angr 源码分析 —— cle Loader 类
category: Code
tags: angr
keywords: angr
description:
---


- [Loader 类](#loader-类)


## Loader 类
The loader loads all the objects and exports an abstraction of the memory of the process. What you see here is an
address space with loaded and rebased binaries.

```python
class Loader(object):

    def __init__(self, main_binary, auto_load_libs=True,
                 force_load_libs=(), skip_libs=(),
                 main_opts=None, lib_opts=None, custom_ld_path=(), use_system_libs=True,
                 ignore_import_version_numbers=True, case_insensitive=False, rebase_granularity=0x1000000,
                 except_missing_libs=False, aslr=False,
                 page_size=0x1, extern_size=0x8000):
```
```
:param main_binary:         The path to the main binary you're loading, or a file-like object with the binary
                            in it.

The following parameters are optional.

:param auto_load_libs:      Whether to automatically load shared libraries that loaded objects depend on.
:param force_load_libs:     A list of libraries to load regardless of if they're required by a loaded object.
:param skip_libs:           A list of libraries to never load, even if they're required by a loaded object.
:param main_opts:           A dictionary of options to be used loading the main binary.
:param lib_opts:            A dictionary mapping library names to the dictionaries of options to be used when
                            loading them.
:param custom_ld_path:      A list of paths in which we can search for shared libraries.
:param use_system_libs:     Whether or not to search the system load path for requested libraries. Default True.
:param ignore_import_version_numbers:
                            Whether libraries with different version numbers in the filename will be considered
                            equivalent, for example libc.so.6 and libc.so.0
:param case_insensitive:    If this is set to True, filesystem loads will be done case-insensitively regardless of
                            the case-sensitivity of the underlying filesystem.
:param rebase_granularity:  The alignment to use for rebasing shared objects
:param except_missing_libs: Throw an exception when a shared library can't be found.
:param aslr:                Load libraries in symbolic address space. Do not use this option.
:param page_size:           The granularity with which data is mapped into memory. Set to 1 if you are working
                            in a non-paged environment.

:ivar memory:               The loaded, rebased, and relocated memory of the program.
:vartype memory:            cle.memory.Clemory
:ivar main_object:          The object representing the main binary (i.e., the executable).
:ivar shared_objects:       A dictionary mapping loaded library names to the objects representing them.
:ivar all_objects:          A list containing representations of all the different objects loaded.
:ivar requested_names:      A set containing the names of all the different shared libraries that were marked as a
                            dependency by somebody.
:ivar initial_load_objects: A list of all the objects that were loaded as a result of the initial load request.

When reference is made to a dictionary of options, it requires a dictionary with zero or more of the following keys:

- backend :             "elf", "pe", "mach-o", "ida", "blob" : which loader backend to use
- custom_arch :         The archinfo.Arch object to use for the binary
- custom_base_addr :    The address to rebase the object at
- custom_entry_point :  The entry point to use for the object
```
