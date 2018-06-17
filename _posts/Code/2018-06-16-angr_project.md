---
layout: post
title: angr 源码分析 —— Project 类
category: Code
tags: angr
keywords: angr
description:
---


- [Project 类](#project-类)
- [load_shellcode](#load_shellcode)

源码版本是 7.8.6.16。推荐使用 angr-dev，以便获得更快的更新。

## Project 类
Project 类是 angr 的主类。它包含了一组二进制文件及其相互关系，然后会对这些二进制文件进行分析。
```python
class Project(object):

    def __init__(self, thing,
                 default_analysis_mode=None,
                 ignore_functions=None,
                 use_sim_procedures=True,
                 exclude_sim_procedures_func=None,
                 exclude_sim_procedures_list=(),
                 arch=None, simos=None,
                 load_options=None,
                 translation_cache=True,
                 support_selfmodifying_code=False,
                 store_function=None,
                 load_function=None,
                 analyses_preset=None,
                 engines_preset=None,
                 **kwargs):
```
```
:param thing:                       The path to the main executable object to analyze, or a CLE Loader object.

The following parameters are optional.

:param default_analysis_mode:       The mode of analysis to use by default. Defaults to 'symbolic'.
:param ignore_functions:            A list of function names that, when imported from shared libraries, should
                                    never be stepped into in analysis (calls will return an unconstrained value).
:param use_sim_procedures:          Whether to replace resolved dependencies for which simprocedures are
                                    available with said simprocedures.
:param exclude_sim_procedures_func: A function that, when passed a function name, returns whether or not to wrap
                                    it with a simprocedure.
:param exclude_sim_procedures_list: A list of functions to *not* wrap with simprocedures.
:param arch:                        The target architecture (auto-detected otherwise).
:param simos:                       a SimOS class to use for this project.
:param bool translation_cache:      If True, cache translated basic blocks rather than re-translating them.
:param support_selfmodifying_code:  Whether we aggressively support self-modifying code. When enabled, emulation
                                    will try to read code from the current state instead of the original memory,
                                    regardless of the current memory protections.
:type support_selfmodifying_code:   bool
:param store_function:              A function that defines how the Project should be stored. Default to pickling.
:param load_function:               A function that defines how the Project should be loaded. Default to unpickling.
:param analyses_preset:             The plugin preset for the analyses provider (i.e. Analyses instance).
:type analyses_preset:              angr.misc.PluginPreset
:param engines_preset:              The plugin preset for the engines provider (i.e. EngineHub instance).
:type engines_preset:               angr.misc.PluginPreset

Any additional keyword arguments passed will be passed onto ``cle.Loader``.

:ivar analyses:     The available analyses.
:type analyses:     angr.analysis.Analyses
:ivar entry:        The program entrypoint.
:ivar factory:      Provides access to important analysis elements such as path groups and symbolic execution results.
:type factory:      AngrObjectFactory
:ivar filename:     The filename of the executable.
:ivar loader:       The program loader.
:type loader:       cle.Loader
:ivar surveyors:    The available surveyors.
:type surveyors:    angr.surveyors.surveyor.Surveyors
:ivar storage:      Dictionary of things that should be loaded/stored with the Project.
:type storage:      defaultdict(list)
```
Project 构造函数执行流程如下：
1. 加载二进制文件，获得一个 `cle.Loader` 实例。首先对加载的 `thing` 做判断，如果是一个 `cle.loader` 类实例，则将其设置为 `self.loader` 成员变量；否则如果是一个流，或者是一个二进制文件，则创建一个新的 `cle.Loader`。然后该 project 被放入字典 `projects`（从流加载的除外）。
2. 判断二进制文件的 CPU 架构，如果是自己指定，则从 archinfo 里匹配，否则从 `self.loader` 获取。
3. 对相关的默认、公共和私有属性进行设置。
4. 设置 project 的插件中心。
    1. 从参数、loader、arch 或者默认值中获取预设的引擎。该过程创建了一个 `angr.EngineHub` 类实例。
    2. 从参数或者默认值中获取预设的分析。创建了一个 `angr.AnalysesHub` 类实例。
    3. 同样的创建 `angr.Surveyors` 和 `angr.KnowledgeBase` 类实例。
5. 确定 guest OS。创建了一个 `angr.SimOS` 或者其子类实例。
6. 根据库函数适当地注册 simprocedures。调用了内部函数 `_register_object`。
7. 执行 OS 特定的配置。

```python
    def hook(self, addr, hook=None, length=0, kwargs=None, replace=False):

    def hook_symbol(self, symbol_name, obj, kwargs=None, replace=None):

    def _hook_decorator(self, addr, length=0, kwargs=None):
```
hook 用于将某段代码替换为其他的操作。其中参数 `hook` 是一个 `SimProcedure` 的实例，如果没有指定该参数，则假设函数作为装饰器使用。被 hook 的地址及实例 `hook` 被放入字典 `self._sim_procedures`。`hook_symbol()` 首先解析符号名得到对应的地址，然后调用 `hook()`。

```python
    def execute(self, *args, **kwargs):

    def terminate_execution(self):
```
为符号执行提供的 API，十分方便。它被设计为在 hook 后执行，并将执行结果返回给模拟管理器。
- 当不带参数执行时，该函数从程序入口点开始
- 当指定了参数 `state` 为一个 SimState 时，从该 state 开始
- 另外，它还可以接受所有传递给 `project.factory.full_init_state` 的任意关键字参数


## load_shellcode
根据一串原始字节码加载一个新的 Project。
```python
def load_shellcode(shellcode, arch, start_offset=0, load_address=0):
```
```
:param shellcode:       The data to load
:param arch:            The name of the arch to use, or an archinfo class
:param start_offset:    The offset into the data to start analysis (default 0)
:param load_address:    The address to place the data in memory (default 0)
```
