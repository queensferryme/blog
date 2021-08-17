---
title: "Flake8 插件编程指南"
date: 2021-08-17T10:07:09+08:00
tags:
  - python
---

上周我花了两三天时间给 [flake8](https://flake8.pycqa.org/) 写了一个插件：[flake8-too-many](https://github.com/queensferryme/flake8-too-many)。这个插件可以算是 [RSSerpent](https://github.com/RSSerpent/RSSerpent) 项目的一个派生产品：我希望通过把这个插件应用到 RSSerpent 项目上，来保证其代码质量。本文将以 flake8-too-many 为例，介绍什么是 [setuptools](https://github.com/pypa/setuptools) 的 [entry point](https://setuptools.readthedocs.io/en/latest/userguide/entry_point.html) 机制、以及flake8 插件的基本结构。

<!--more-->

## 加载插件

>   阅读本小节需要你对 Python Packaging 有基本的了解。

在开始编写插件之前，我们需要先弄清楚 flake8 是如何加载插件的 —— 答案就是：通过 setuptools 的 entry point 机制。如果你此前有使用 Python 编写命令行应用的经验，你可能已经接触过 entry point 了。

当你使用 Python 编写了一个命令行应用并发布到 PyPI 上以后，你可能会希望你的用户可以直接调用 `my_cli_app [<OPTIONS>]`，而不是 `python -m my_cli_app [<OPTIONS>]` 或者 `python my_cli_app.py [<OPTIONS>]`。前者看起来更简洁方便，而且用户也无需关心这个命令行应用到底是用 Python / Rust 还是其他什么语言编写的。setuptools 的 entry point 机制就可以帮助我们实现前者的效果。

以 [Flask](https://github.com/pallets/flask) 为例，许多用户会注意到，尽管 Flask 是一个 Web 开发框架，但是它也提供了一个 `flask` 命令辅助开发。Flask 的开发者们在配置文件（[setup.cfg#L44-L46](https://github.com/pallets/flask/blob/53eef278efd59222be406ff5be737f48c92f8313/setup.cfg#L44-L46)）中将 `flask` 命令行应用注册到名为 `console_scripts` 的 entry point 下，并声明 `flask` 命令的入口在 `flask` 包 `cli` 模块中的 [`main`](https://github.com/pallets/flask/blob/53eef278efd59222be406ff5be737f48c92f8313/src/flask/cli.py#L985) 变量。

```ini
[options.entry_points]
console_scripts =
    flask = flask.cli:main
```

`console_scripts` 是一个特殊的 entry point。Python 包管理器在安装有 `console_scripts` 这个 entry point 的包时会自动为其创建相应的可执行文件，这样我们就有了一个命令行应用。

[![aGEnDNo2rXUl9dp](https://i.loli.net/2021/08/17/aGEnDNo2rXUl9dp.png)](https://i.loli.net/2021/08/17/aGEnDNo2rXUl9dp.png)

但实际上，entry point 也可以是其他自定义的字符串。例如 `flake8.extension`（[setup.py#L13-L18](https://github.com/PyCQA/flake8/blob/542145fb7cda98375467554a4070e232ba37f865/example-plugin/setup.py#L13-L18)）：

```python
entry_points={
    "flake8.extension": [
        "X1 = flake8_example_plugin:ExampleOne",
        "X2 = flake8_example_plugin:ExampleTwo",
    ],
} 
```

如果某个 Python 包（例如 `flake8-too-many`）注册了 `flake8.extension` 这个 entry point，并且和 `flake8` 位于同一个 Python 环境下，那么 `flake8` 主程序（[flake8/plugins/manager.py#L451-L454](https://github.com/PyCQA/flake8/blob/542145fb7cda98375467554a4070e232ba37f865/src/flake8/plugins/manager.py#L451-L454)）就可以通过 [`importlib.metadata`](https://docs.python.org/3.10/library/importlib.metadata.html) 来动态地发现并加载这些包，从而实现插件加载。

```python
>>> from importlib.metadata import entry_points
>>> eps = entry_points(group="flake8.extension")
>>> for ep in eps:
...     plugin = ep.load()
...
```

>   注意，如果你在使用 Python 3.10+ 版本，可以直接使用标准库中的 `importlib.metadata`，否则推荐使用 PyPI 上的 backport 版本 [`importlib-metadata`](https://pypi.org/project/importlib-metadata/)。

当然，如果你使用 [Poetry](https://python-poetry.org/) （而非 setuptools）来打包你的 Python 应用，你同样可以使用 entry point（[文档](https://python-poetry.org/docs/pyproject/#plugins)）。以 flake8-too-many 为例，只需在 [`pyproject.toml`](https://github.com/queensferryme/flake8-too-many/blob/9a01c9efa7ca6239716f6582ed05e48cbca8e279/pyproject.toml#L37-L38) 中写上：

```toml
[tool.poetry.plugins."flake8.extension"]
TMN = "flake8_too_many:Checker"
```



值得一提的是，RSSerpent 的插件也是基于 entry point 机制的。RSSerpent 使用了 `rsserpent.plugins` 这个 entry point。

## 定义插件

flake8 有两种插件（[文档](https://flake8.pycqa.org/en/latest/plugin-development/index.html)），本文要讨论的则是 check 插件：它的职责是检查源代码并报错。

以 flake8-too-many 为例：插件的入口是 `flake8_too_many` 包中的 `Checker` 类。

```toml
[tool.poetry.plugins."flake8.extension"]
TMN = "flake8_too_many:Checker"
```

插件所有报错的错误代码均应以字符串 "TMN" 开头，随后以三个数字结尾。

```python
# see https://github.com/queensferryme/flake8-too-many/blob/master/flake8_too_many/messages.py
TMN001 = "TMN001 function has too many arguments ({} > {})."
TMN002 = "TMN002 function returns too many values ({} > {})."
TMN003 = "TMN003 function has too many return statements ({} > {})."
TMN004 = "TMN004 unpacking has too many targets ({} > {})."
```

插件的入口 [`Checker`](https://github.com/queensferryme/flake8-too-many/blob/9a01c9efa7ca6239716f6582ed05e48cbca8e279/flake8_too_many/checker.py#L11) 类必须包含 `name` 和 `version` 字段，用以指定插件的名字和版本。类名本身可以随意。

```python
class Checker:
    name = "flake8-too-many"
    version = "0.1.1"
```

插件需要定义 `__init__` 方法，并声明需要哪些信息（[文档](https://flake8.pycqa.org/en/latest/plugin-development/plugin-parameters.html#indicating-desired-data)）。Flake8 则会检查插件 `__init__` 方法的签名，并将相应的信息传给插件。例如，flake8-too-many 需要源代码的 AST 树：

```python
def __init__(self, tree: ast.Module) -> None:
    self.tree = tree
```

此外，插件还需要一个 `run` 方法，这个方法内应包含插件检查源代码并抛出错误的逻辑。

```python
def run(self) -> Iterable[Tuple[int, int, str, Type["Checker"]]]:
    # do something ...
    for error in errors:
        yield (error.row, error.col, error.msg, Checker)
```

>   flake8-too-many 使用 Python 的 [ast](https://docs.python.org/3/library/ast.html) 模块对 `self.tree` 做了一些检查，详见 [`visitor.py`](https://github.com/queensferryme/flake8-too-many/blob/master/flake8_too_many/visitor.py)

检查到的错误应以生成器的形式用 `yield` 返回。返回值是一个四元组：第一个元素是错误所在行号，第二个则是列号，第三个则是含有错误代码和错误[信息](https://github.com/queensferryme/flake8-too-many/blob/9a01c9efa7ca6239716f6582ed05e48cbca8e279/flake8_too_many/messages.py)的字符串（在本例中错误代码应以 "TMN" 开头），第四个则是插件类本身（以便 flake8 知道是哪个插件抛出了该错误）。

此外插件还可以有 `add_options` 和 `parse_options` 这两个 `@classmethod` 类方法。如果你的插件有额外的可配置参数，你应该在这两个方法声明。

```python
from argparse import Namespace
from flake8.options.manager import OptionManager

# ...
# ignore the indentation error

@classmethod
def add_options(cls, optioin_manager: OptionManager) -> None:
    option_manager.add_option(
        "--some-configurable-option",
        default=10,
        parse_from_config=True,
        type=int,
        help="Some help message",
    )

@classmethod
def parse_options(cls, options: Namespace) -> None:
    cls.some_configurable_option = options.some_configurable_option
```

这样你就可以使用命令行参数 `flake8 --some-configurable-option=100`，或者 `.flake8` 配置文件来对插件进行配置：

```ini
[flake8]
# ...
some-configurable-option = 100
```

---

~~付费查看剩余内容~~ 本文内容就是这些啦～拜拜👋
