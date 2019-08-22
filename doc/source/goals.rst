目标
-----

该接口基于 `LuaJIT's FFI`_，并遵循如下一些原则:

* 目标是在不学习第三种编程语言的情况下从Python调用C语言代码:
  现有的替代方案要求用户学习特定领域语言 (Cython_，SWIG_) 或 API (ctypes_)。 CFFI设计目的要求用户只知道C和Python，最大限度地减少需要学习API的额外部分。

* 将所有与Python相关的逻辑代码保存在Python中，这样您就不需要编写很多C语言代码 
  (不同于 `CPython原生C扩展`_).

* 首选方法是在API (Application
  Programming Interface)级别运行 : C语言编译器根据您编写的声明直接链接C语言结构。
  或者，也可以选择ABI级别（Application Binary Interface），这种方法通过 ctypes_ 运行。
  但是，在非Windows平台上，C语言库通常具有指定的C API但不具有ABI (例如 他们可能将 "struct" 记录为至少包含这些字段，但可能更多)。

* 尽量完成。 目前不支持一些C99结构，但所有C89结构都应该支持，包括宏 
  (包括 "abuses"，你可以 `手动包装`_ 在看起来很神奇的C语言函数)。

* 尝试同时支持PyPy和CPython，为IronPython和Jython等其他Python实现提供合理的路径。

* 注意，与 `Weave`_ **不同** 的是，这个项目并不是要在Python中嵌入可执行C代码。
  而是从Python调用现有的语言库。

* 没有支持 C++。  
  有时，围绕C++代码编写一个C包装器然后用CFFI调用这个C API是合理的。
  否者，看看其他项目。  我会推荐 cppyy_，它有一些相似之处 (并且还可以在CPython和PyPy上高效工作)。

.. _`LuaJIT's FFI`: http://luajit.org/ext_ffi.html
.. _`Cython`: http://www.cython.org
.. _`SWIG`: http://www.swig.org/
.. _`CPython原生C扩展`: http://docs.python.org/extending/extending.html
.. _`native C extensions`: http://docs.python.org/extending/extending.html
.. _`ctypes`: http://docs.python.org/library/ctypes.html
.. _`Weave`: http://wiki.scipy.org/Weave
.. _`cppyy`: http://cppyy.readthedocs.io/en/latest/
.. _`手动包装`: overview.html#abi-versus-api

开始阅读 `概述`__.

.. __: overview.html


意见和错误
-----------------

联系我们的最佳方式是在 ``irc.freenode.net`` 的IRC ``#pypy`` 频道。  随意讨论那里或 `邮件列表`_ 中的问题。 请向 `问题跟踪器`_ 报告任何错误。

作为基本规则，当需要解决设计问题时，我们选择"最像C"的解决方案。 我们希望这个模块拥有访问C语言代码所需的一切，仅此而已。

--- 作者，Armin Rigo 和 Maciej Fijalkowski，Jairo(翻译者)

--- 本文档未获得文档翻译版权，仅供学习参考，如有侵权，则立即删除

.. _`问题跟踪器`: https://bitbucket.org/cffi/cffi/issues
.. _`邮件列表`: https://groups.google.com/forum/#!forum/python-cffi
