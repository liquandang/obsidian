**包**
```python
sound/                          最高层级的包
      __init__.py               初始化 sound 包
      formats/                  用于文件格式转换的子包
              __init__.py
              wavread.py
              wavwrite.py
              aiffread.py
              aiffwrite.py
              auread.py
              auwrite.py
              ...
      effects/                  用于音效的子包
              __init__.py
              echo.py
              surround.py
              reverse.py
              ...
      filters/                  用于过滤器的子包
              __init__.py
              equalizer.py
              vocoder.py
              karaoke.py
              ...
```

导入包时，Python 搜索 `sys.path` 里的目录，查找包的子目录。<span style="color:red">需要有 `__init__.py` 文件才能让 Python 将包含该文件的目录当作包来处理，在最简单的情况下，`__init__.py` 可以只是一个空文件，但它也可以执行包的初始化代码或设置 `__all__` 变量。</span>

还可以从包中导入单个模块，例如：

```python
import sound.effects.echo
```

这将加载子模块 `sound.effects.echo`。 它必须通过其全名来引用。
```python
sound.effects.echo.echofilter(input, output, delay=0.7, atten=4)
```

另一种导入子模块的方法是 ：
```python
from sound.effects import echo
```

这也会加载子模块 `echo`，并使其不必加包前缀，因此可按如下方式使用:
```python
echo.echofilter(input, output, delay=0.7, atten=4)
```

Import 语句的另一种变体是直接导入所需的函数或变量：
```python
from sound.effects.echo import echofilter
```

同样，这将加载子模块 `echo`，但这使其函数 `echofilter()` 直接可用:
```python
echofilter(input, output, delay=0.7, atten=4)
```

注意，使用 `from package import item` 时，item 可以是包的子模块（或子包），也可以是包中定义的函数、类或变量等其他名称。`import` 语句首先测试包中是否定义了 item；如果未在包中定义，则假定 item 是模块，并尝试加载。如果找不到 item，则触发 [`ImportError`](https://docs.python.org/zh-cn/3.13/library/exceptions.html#ImportError "ImportError") 异常。

相反，使用 `import item.subitem.subsubitem` 句法时，除最后一项外，每个 item 都必须是包；最后一项可以是模块或包，但不能是上一项中定义的类、函数或变量。

## 从包中导入*

如果包的 `__init__.py` 代码定义了列表 `__all__`，运行 `from package import *` 时，它就是被导入的模块名列表。发布包的新版本时，包的作者应更新此列表。如果包的作者认为没有必要在包中执行导入 * 操作，也可以不提供此列表。例如，`sound/effects/__init__.py` 文件可以包含以下代码：

```python
__all__ = ["echo", "surround", "reverse"]
```

这意味着 `from sound.effects import *` 将导入 `sound.effects` 包的三个命名子模块。

```python
__all__ = [
    "echo",      # 指向 'echo.py' 文件
    "surround",  # 指向 'surround.py' 文件
    "reverse",   # !!! 现在指向 'reverse' 函数 !!!
]

def reverse(msg: str):  # <-- 此名称将覆盖 'reverse.py' 子模块
    return msg[::-1]    #     针对 'from sound.effects import *' 的情况
```

请注意子模块可能会受到本地定义名称的影响。 例如，如果你在 `sound/effects/__init__.py` 文件中添加了一个 `reverse` 函数，`from sound.effects import *` 将只导入 `echo` 和 `surround` 这两个子模块，但 **不会** 导入 `reverse` 子模块，因为它被本地定义的 `reverse` 函数所遮挡。

## 相对导入

导入使用前导点号来表示相对导入所涉及的当前包和上级包。 例如对于 `surround` 模块，可以使用:
```python
from . import echo
from .. import formats
from ..filters import equalizer
```

注意，相对导入基于当前模块名。因为主模块名永远是 `"__main__"` ，所以如果计划将一个模块用作 Python 应用程序的主模块，那么该模块内的导入语句必须始终使用绝对导入。