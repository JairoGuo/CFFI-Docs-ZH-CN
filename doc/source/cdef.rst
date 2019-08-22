======================================
编写和分发模块
======================================

.. contents::

在项目中使用CFFI有三种或四种不同的方法。
按顺序排列:

* **"in-line", "ABI 模式"**:

  .. code-block:: python

    import cffi

    ffi = cffi.FFI()
    ffi.cdef("C-like declarations")
    lib = ffi.dlopen("libpath")

    # use ffi and lib here

.. _out-of-line-abi:

* **"out-of-line",** 但仍然是 **"ABI 模式"，** 对于组织代码和减少导入时间很有用:

  .. code-block:: python

    # in a separate file "package/foo_build.py"
    import cffi

    ffibuilder = cffi.FFI()
    ffibuilder.set_source("package._foo", None)
    ffibuilder.cdef("C-like declarations")

    if __name__ == "__main__":
        ffibuilder.compile()

  运行 ``python foo_build.py`` 会生成一个文件 ``_foo.py``，然后可以在主程序中导入该文件:

  .. code-block:: python

    from package._foo import ffi
    lib = ffi.dlopen("libpath")

    # use ffi and lib here

.. _out-of-line-api:

* **"out-of-line", "API 模式"** 为您提供了在C级别而不是二进制级别访问C库的最大灵活性和速度:

  .. code-block:: python

    # in a separate file "package/foo_build.py"
    import cffi

    ffibuilder = cffi.FFI()
    ffibuilder.set_source("package._foo", r"""real C code""")   # <=
    ffibuilder.cdef("C-like declarations with '...'")

    if __name__ == "__main__":
        ffibuilder.compile(verbose=True)

  运行 ``python foo_build.py`` 生成一个文件 ``_foo.c`` 并调用C编译器将其转换为文件 ``_foo.so`` (或
  ``_foo.pyd`` 或 ``_foo.dylib``)。 它是一个C扩展模块，可以在主程序中导入:

  .. code-block:: python

    from package._foo import ffi, lib
    # no ffi.dlopen()

    # use ffi and lib here

.. _distutils-setuptools:

* 最后，在编写 ``setup.py`` 时，您可以(但不必) 使用CFFI的 **Distutils** 或
  **Setuptools 集成** 。 对于
  Distutils (仅在 out-of-line API 模式):

  .. code-block:: python

    # setup.py (requires CFFI to be installed first)
    from distutils.core import setup

    import foo_build   # possibly with sys.path tricks to find it

    setup(
        ...,
        ext_modules=[foo_build.ffibuilder.distutils_extension()],
    )

  对于Setuptools (out-of-line, 但适用于ABI或API模式;
  推荐):

  .. code-block:: python

    # setup.py (with automatic dependency tracking)
    from setuptools import setup

    setup(
        ...,
        setup_requires=["cffi>=1.0.0"],
        cffi_modules=["package/foo_build.py:ffibuilder"],
        install_requires=["cffi>=1.0.0"],
    )

  再次注意，``foo_build.py`` 示例包含以下行，这意味着仅在导入 ``package.foo_build`` 时实际上不编译 ``ffibuilder``， 它将由Setuptools逻辑独立编译，使用Setuptools提供的编译参数:

  .. code-block:: python

    if __name__ == "__main__":    # not when running with setuptools
        ffibuilder.compile(verbose=True)

* 请注意，尝试查找项目使用的所有模块的一些捆绑工具(如PyInstaller)将在 out-of-line模式下避免 ``_cffi_backend``，因为您的程序不包含显式 ``import
  cffi`` 或 ``import _cffi_backend``。 您需要显式添加
  ``_cffi_backend`` (作为PyInstaller中的"hidden import"，但通常也可以通过在主程序中添加 ``import
  _cffi_backend`` 来更好地完成它)。

请注意，CFFI实际上包含两个不同的 ``FFI`` 类。 页面 `使用ffi/lib对象`_ 描述了常用功能。
这是你从上面的 ``from package._foo import ffi`` 中得到的。
另一方面，扩展的 ``FFI`` 类是从
``import cffi; ffi_or_ffibuilder = cffi.FFI()`` 得到的; 它具有相同的功能 (用于 in-line 使用)，但也有下面描述的额外方法 (编写 FFI)。 注意: 当代码是关于生成 ``_foo.so`` 时，我们在out-of-line上下文中使用名称 ``ffibuilder``
而不是 ``ffi``; 这是尝试将它与后来
``from _foo import ffi`` 所获得的不同 ``ffi`` 对象区分开来。

.. _`使用ffi/lib对象`: using.html

这种功能分离的原因是使用CFFI外联的常规程序根本不需要导入 ``cffi`` 纯Python包。 (在内部，它仍然需要 ``_cffi_backend``，一个CFFI附带的C扩展模块; 这就是为什么CFFI也在 ``install_requires=..`` 上面列出的原因。将来，这可能会拆分为仅安装
``_cffi_backend`` 的不同PyPI包。)

请注意，确实存在一些小的差异: 值得注意的是，从 ``from _foo import
ffi`` 返回一个用C语言编写的类型的对象，它不允许你向它添加随机属性 (它也没有Python版本的所有下划线前缀内部属性)。
类似地，除了对全局变量的写入之外，C版本返回的 ``lib`` 对象是只读的 此外，``lib.__dict__`` 在版本1.2之前不起作用，或者如果 ``lib`` 恰好声明一个名为 ``__dict__`` 的名称 (使用 ``dir(lib)`` 代替)。 对于连续版本中添加的 ``lib.__class__``， ``lib.__all__`` 和 ``lib.__name__`` 也是如此。


.. _cdef:

ffi/ffibuilder.cdef(): 声明类型和函数
----------------------------------------------------

**ffi/ffibuilder.cdef(source)**: 解析给定的C源。它在C源中注册所有函数，类型，常量和全局变量。 这些类型可以在 ``ffi.new()`` 和其他函数中立即使用。 在您可以访问函数和全局变量之前，您需要为 ``ffi`` 提供另一条信息: 它们实际上来自那里 (您可以使用 ``ffi.dlopen()`` 或
``ffi.set_source()`` 执行此操作)。

.. _`all types listed above`:

内部解析C语言源代码 (使用 ``pycparser``)。此代码不能包含 ``#include``。  它通常应该是从手册页中提取的自包含声明。它可以假设存在的唯一事物是标准类型:

* char, short, int, long, long long (both signed 和 unsigned)

* float, double, long double

* intN_t, uintN_t (对于 N=8,16,32,64), intptr_t, uintptr_t, ptrdiff_t,
  size_t, ssize_t

* wchar_t (如果后端支持).  *版本1.11中的新功能:*
  char16_t 和 char32_t.

* _Bool 和 bool (相等)。 如果C编译器没有直接支持，则使用 ``unsigned char`` 的大小声明它。

* FILE.  `看这里。`__

* 如果在Windows上运行，则定义所有 `常见的Windows类型`_ (``DWORD``, ``LPARAM``, 等)。  例外:
  ``TBYTE TCHAR LPCTSTR PCTSTR LPTSTR PTSTR PTBYTE PTCHAR`` 不会自动定义; 参见 `ffi.set_unicode()`_。

* stdint.h中的其他标准整数类型，如 ``intmax_t``，只要它们映射到1,2,4或8字节的整数即可。不支持更大的整数。

.. __: ref.html#file
.. _`常见的Windows类型`: http://msdn.microsoft.com/en-us/library/windows/desktop/aa383751%28v=vs.85%29.aspx

声明还可以在各个地方包含 "``...``"; 这些是将由编译器完成的占位符。 有关它的更多信息，请参阅 `让C编译器填补空白`_。


请注意，上面列出的所有标准类型名称仅作为 *默认值* 处理 (除了那些是C语言中的关键词)。 如果您的 ``cdef`` 包含重新定义上述类型之一的显式typedef，则忽略上述默认值。 (这有点难以干净地实现，因此在某些极端情况下它可能会失败，尤其是错误 ``Multiple type specifiers
with a type tag``。 如果确实如此，请将其报告为错误。)

可以多次调用 ``ffi.cdef()``。请注意，很多时候调用 ``ffi.cdef()`` 的速度很慢，主要考虑in-line模式.

``ffi.cdef()`` 调用可选地接受一个额外参数: ``packed`` 或 ``pack``。 如果传递 ``packed=True``，则在此cdef中声明的所有结构都是"packed"的。 (如果您需要packed和非packed结构，请按顺序使用多个cdef。)  这与GCC中的 ``__attribute__((packed))`` 的含义相似。 它指定所有结构字段的对齐方式应为一个字节。 (请注意，到目前为止，packed属性对位字段没有影响，这意味着它们可能与GCC上的packed方式不同。
此外，这对使用 ``"...;"`` 声明的结构没有影响，稍后会详细介绍 `让C编译器填补空白`_。)
*版本1.12中的新功能:*  在ABI模式中， 你也可以传递 ``pack=n``，整数 ``n`` 必须是2的幂。 则任何字段的对齐限制为 ``n``，否则将大于 ``n``。 传递 ``pack=1`` 相当于传递
``packed=True``。 这是为了模拟MSVC编译器中的 ``#pragma pack(n)``。 在Windows上，默认值为 ``pack=8`` (从cffi 1.12开始); 在其他平台上，默认值为 ``pack=None``。

请注意，您可以在 ``cdef()`` 中使用类型修饰符 ``const`` 和 ``restrict``
(但不是 ``__restrict`` 或 ``__restrict__``) ，但这对运行时获得的cdata对象没有影响 (他们永远不会是 ``const``)。 效果仅限于知道全局变量是否为常量。  此外，*版本1.3中的新功能:* 当使用 ``set_source()`` 或 ``verify()``， 这两个限定符将从cdef复制到生成的C语言代码中; 这修复了C编译器的警告。

如果从具有额外宏的源代码复制粘贴代码，请注意一个技巧 (例如，Windows文档使用SAL注释，如 ``_In_`` 或 ``_Out_``)。 必须在给cdef()的字符串中删除这些提示，但可以像这样以编程方式完成::

    ffi.cdef(re.sub(r"\b(_In_|_Inout_|_Out_|_Outptr_)(opt_)?\b", " ",
      """
        DWORD WINAPI GetModuleFileName(
          _In_opt_ HMODULE hModule,
          _Out_    LPTSTR  lpFilename,
          _In_     DWORD   nSize
        );
      """))

另请注意，pycparser是底层C解析器，它以下列格式识别类似预处理器的指令: ``# NUMBER
"FILE"``。 例如， 如果你把 ``# 42 "foo.h"`` 放在传递给 ``cdef()`` 的字符串的中间，之后会出现两行错误，然后会报告一条以 ``foo.h:43:`` 开头的错误消息 (给出数字42的行是指令后面的行).  *版本1.10.1中的新功能:*  CFFI自动将行
``# 1 "<cdef source string>"`` 放在您给
``cdef()`` 的字符串之前。


.. _`ffi.set_unicode()`:

**ffi.set_unicode(enabled_flag)**: Windows: 如果 ``enabled_flag`` 为True, 在C中启用 ``UNICODE`` 和 ``_UNICODE`` 定义， 并声明类型 ``TBYTE TCHAR LPCTSTR PCTSTR LPTSTR PTSTR PTBYTE
PTCHAR`` 是 (指针) ``wchar_t``。 如果 ``enabled_flag`` 为False，则声明这些类型为 (指针 ) 普通8位字符。
(如果不调用
``set_unicode()`` 则根本不用预先声明这些类型。)

这个方法背后的原因是很多标准函数都有两个版本，比如 ``MessageBoxA()`` 和 ``MessageBoxW()``。 官方接口是 ``MessageBox()`` 其参数类似于
``LPTCSTR``。 根据是否定义 ``UNICODE``，标准头将通用函数名重命名为两个专用版本之一，并声明正确的 (unicode 或 not) 类型。

U通常，正确的做法是使用True调用此方法。 请注意 (特别是在Python 2上) ，之后，您需要将unicode字符串作为参数而不是字节字符串传递。


.. _loading-libraries:

ffi.dlopen(): 以ABI模式加载库
-------------------------------------------

``ffi.dlopen(libpath, [flags])``: 此函数打开一个共享库并返回类似模块的库对象。 当您对系统的ABI级别访问权限有限制时，可以使用此选项。 (依赖于ABI详细信息，获取崩溃而不是C编译器错误/警告，以及调用C函数的更高开销)。 如有疑问，请在概述中再次阅读
`ABI与API`_ 。

.. _`ABI与API`: overview.html#abi-versus-api

您可以使用库对象来调用先前由 ``ffi.cdef()`` 声明的函数，读取常量以及读取或写入全局变量。 请注意，只要使用 ``dlopen()`` 加载每个函数并使用正确的函数访问函数，就可以使用单个 ``cdef()`` 来声明多个库中的函数。

``libpath`` 是共享库的文件名，它可以包含完整路径 (在这种情况下，它在标准位置搜索，如 ``man dlopen`` 中所述)，是否包含扩展名。
或者，如果 ``libpath`` 为None，则返回标准C库
(在Linux上可以用来访问glibc的功能)。 请注意，在使用Python 3的Windows中 ``libpath`` `不能为None`__。

.. __: http://bugs.python.org/issue23606

让我再说一遍: 这提供了对库的ABI级访问，因此您需要手动声明所有类型，而不是在创建库时。没有检查。不匹配可能导致随机崩溃。另一方面，API级访问更安全。速度方面，API级别的访问速度要快得多 (对性能有相反的误解是很常见的)。

请注意，只有函数和全局变量存在于库对象中;
这些类型存在于 ``ffi`` 例中，与库对象无关。
这是由于C模型: 您在C中声明的类型不依赖于特定库，只要您 ``#include`` 其标题即可; 但是你不能在程序中调用函数而不在程序中链接它，因为 ``dlopen()`` 在C中动态地执行。

对于可选的 ``flags`` 参数， 参见 ``man dlopen`` (在Windows上被忽略)。 它默认为 ``ffi.RTLD_NOW``。

此函数返回一个"library"对象，当它超出范围时会被关闭。确保在需要时保留库对象。 (或者， out-of-line FFI有一个方法
``ffi.dlclose(lib)``。)

.. _dlopen-note:

注意: 如果无法直接找到库，则来自in-line ABI模式的旧版本的 ``ffi.dlopen()`` 会尝试使用 ``ctypes.util.find_library()``。 更新的 out-of-line ``ffi.dlopen()`` 不再自动执行此操作; 它只是将它接收的参数传递给底层的 ``dlopen()`` 或 ``LoadLibrary()`` 函数。 如果需要，您可以使用 ``ctypes.util.find_library()`` 或任何其他方式查找库的文件名。 这也意味着
``ffi.dlopen(None)`` 不再适用于Windows; 尝试改为
``ffi.dlopen(ctypes.util.find_library('c'))``。


ffibuilder.set_source(): 编写out-of-line模块
------------------------------------------------------

**ffibuilder.set_source(module_name, c_header_source, [\*\*keywords...])**:
编写ffi以生成一个名为
``module_name`` 的外部模块。

``ffibuilder.set_source()`` 本身不会写任何文件，而只是记录其参数以供日后使用。 因此可以在 ``ffibuilder.cdef()`` 之前或之后调用它。

在 **ABI 模式，** 你调用 ``ffibuilder.set_source(module_name, None)``。 参数是要生成的Python模块的名称 (或包内带点号名称)。 在此模式下，不会调用C编译器。

在 **API 模式，** ``c_header_source`` 参数是一个字符串，将粘贴到生成的.c文件中。 通常，它被指定为
``r""" ...multiple lines of C code... """`` (例如，``r`` 前缀允许这些行包含文字 ``\n``)。 这段C语言代码通常包含一些 ``#include``，但也可能包含更多内容，例如自定义"包装器"C语言函数的定义。 目标是可以像这样生成.c文件::

    // C file "module_name.c"
    #include <Python.h>

    ...c_header_source...

    ...magic code...

其中"魔术代码"是从 ``cdef()`` 自动生成的。
例如， 如果 ``cdef()`` 包含 ``int foo(int x);``， 然后魔术代码将包含用整数参数调用函数 ``foo()`` 的逻辑，它本身包含在一些CPython或PyPy特定的代码中。

``set_source()`` 的关键字参数控制C编译器的调用方式。 它们直接传递给 distutils_ 或 setuptools_， 至少包括 ``sources``, ``include_dirs``, ``define_macros``,
``undef_macros``, ``libraries``, ``library_dirs``, ``extra_objects``,
``extra_compile_args`` 和 ``extra_link_args``。 您通常至少需要 ``libraries=['foo']`` 才能在Windows上与 ``libfoo.so`` 或
``libfoo.so.X.Y``, 或 ``foo.dll`` 链接。 ``sources`` 是一组编译和链接在一起的额外.c文件 (始终生成上面显示的文件
``module_name.c`` 并自动添加为 ``sources`` 的第一个参数)。 `有关其他参数的更多信息`__ 请参阅distutils文档。

.. __: http://docs.python.org/distutils/setupscript.html#library-options
.. _distutils: http://docs.python.org/distutils/setupscript.html#describing-extension-modules
.. _setuptools: https://pythonhosted.org/setuptools/setuptools.html

内部处理的额外关键字参数是 ``source_extension``， 默认为 ``".c"``。 生成的文件实际上调用 ``module_name + source_extension``。 例如
C++ (但请注意，仍存在一些已知的C与C ++兼容性问题):

.. code-block:: python

    ffibuilder.set_source("mymodule", r'''
    extern "C" {
        int somefunc(int somearg) { return real_cpp_func(somearg); }
    }
    ''', source_extension='.cpp')

.. _pkgconfig:

**ffibuilder.set_source_pkgconfig(module_name, pkgconfig_libs,
c_header_source, [\*\*keywords...])**:

*版本1.12中的新功能。*  这相当于 ``set_source()``， 但它首先使用列表 ``pkgconfig_libs`` 中给出的包名称调用系统实用程序 ``pkg-config``。 它收集以这种方式获得的信息，并将其添加到明确提供的
``**keywords`` (如果有) 中。这也许不应该在Windows上使用。

如果未安装 ``pkg-config`` 程序或不知道所请求的库，则调用将失败并显示 ``cffi.PkgConfigError``。  如果有必要，您可以捕获此错误并尝试直接调用 ``set_source()``。 (理想情况下，如果 ``ffibuilder``
实例没有方法 ``set_source_pkgconfig()``，您也应该这样做，以支持旧版本的cffi。)


让C编译器填补空白
------------------------------------

如果您使用的是C编译器 ("API 模式")， 那么:

*  获取或返回整数或浮点数参数的函数可能被误报: 如果是一个函数由 ``cdef()`` 声明为接受一个
   ``int``，但实际上需要一个 ``long``，然后C编译器处理差异。

*  检查其他参数: 如果将 ``int *`` 参数传递给期望 ``long *`` 的函数，则会收到编译警告或错误。

*  类似地，在 ``cdef()`` 中声明的大多数其他事情都被检查，达到目前为止我们实现的最佳效果; mistakes给出编译警告或错误。

此外，您可以在 ``cdef()`` 中的各个位置使用 "``...``" (按照字面意思, 点点点) ，以便让C编译器填写详细信息。 这些地方是:

*  结构声明: 以 "``...;``" 结尾的任何 ``struct { }`` 作为最后一个"字段"是部分的: 可能缺少字段和/或已将其声明为无序。
   此声明将由编译器更正。 (但请注意，您只能访问您声明的字段，而不能访问其他字段。)  任何不使用 "``...``" 的 ``struct``
   声明都被认为是精确的，但这是检查的: 如果不正确，你会收到错误。

*  整数类型: 语法 "``typedef
   int... foo_t;``" 将类型 ``foo_t`` 声明为整数类型，其未指定精确大小和符号。 编译器会搞清楚。 (请注意，这需要 ``set_source()``; 它不适用于 ``verify()``。)  ``int...`` 可以用
   ``long...`` 或 ``unsigned long long...`` 或任何其他原始整数类型替换，但不起作用。 该类型将始终映射到Python中的
   ``(u)int(8,16,32,64)_t`` 之中，但在生成的C代码中，仅使用 ``foo_t``。

* *版本1.3中的新功能:* 浮点类型: "``typedef
  float... foo_t;``" (或者等价 "``typedef double... foo_t;``")
  将 ``foo_t`` 声明为一个float或一个double; 编译器会弄清楚它是什么。请注意，如果实际的C类型更大
  (``long double`` 在某些平台上)，则编译将失败。
  问题是Python"float"类型不能用于存储额外的精度。 (使用不是点点点的语法 ``typedef long
  double foo_t;`` 像往常一样，它返回的值不是Python浮点数，而是cdata "long double" 对象。)

*  未知类型: 语法 "``typedef ... foo_t;``" 将类型
   ``foo_t`` 声明为不确定。 主要用于API采用并返回
   ``foo_t *`` 而无需查看 ``foo_t``。也适用于 "``typedef ... *foo_p;``"， 它声明指针类型
   ``foo_p`` 而不给不确定类型本身命名。 请注意， 这种不确定的结构没有已知的大小， 这会阻止某些操作执行 (大多数情况下像在C语言中)。 *您不能使用此语法来声明特定类型， 如整数类型！它仅声明不确定的类似结构的类型。* 在某些情况下，你需要说
   ``foo_t`` 不是不确定的，而只是一个你不知道任何字段的结构; 然后你会使用 "``typedef struct { ...; } foo_t;``"。

*  数组长度: 当用作结构字段或全局变量时， 数组可以具有未指定的长度， 如 "``int n[...];``" 中所示。 
   长度由C编译器完成。
   这与 "``int n[];``" 略有不同， 因为后者意味着即使对C编译器也不知道长度， 因此不会尝试完成它。 这支持多维数组: "``int n[...][...];``"。

   *版本1.2中的新功能:* "``int m[][...];``"， 即 ``...`` 可以在最里面的维度中使用， 而不是也用在最外面的维度中。 在给出的示例中，假定 ``m`` 数组的长度不为C编译器所知， 但是每个项的长度 (如子数组 ``m[0]``) 总是为C编译器所知。
   换句话说， 只有最外层的维度可以在C和CFFI中指定为
   ``[]``， 但任何维度都可以在CFFI中以
   ``[...]`` 的形式给出。

*  枚举: 如果您不知道声明的常量的确切顺序 (或值)，请使用以下语法: "``enum foo { A, B, C, ... };``"
   (后面有个 "``...``")。 C编译器将用于计算常量的确切值。 另一种语法是
   "``enum foo { A=..., B, C };``" 或甚至
   "``enum foo { A=..., B=..., C=... };``"。 与结构一样，没有 "``...``" 的 ``enum`` 被认为是精确的，并且这被检查。

*  整数常量和宏: 你可以在 ``cdef`` 中写一行
   "``#define FOO ...``"，使用任何宏名FOO都使用 ``...`` 作为一个值。 如果将宏定义为整数值， 则该值将通过库对象的属性提供。 通过编写声明
   ``static const int FOO;`` 可以实现相同的效果。 后者更通用， 因为它支持除整数类型之外的其他类型 (注意: 然后用C语言语法将 ``const`` 与变量名一起写入， 如
   ``static char *const FOO;``)。

目前，不支持自动查找在哪个地方需要的各种整数或浮点类型， 除了以下情况：如果这样的类型是显式命名的。 对于整数类型，请使用 ``typedef int... the_type_name;`` 或其他类型， 如
``typedef unsigned long... the_type_name;``。 两者都是等价的， 并且由真实的C语言类型取代， 它必须是整数类型。
同样， 对于浮点类型， 请使 ``typedef float...
the_type_name;`` 或等效的 ``typedef double...  the_type_name;``。
请注意，这种方式无法检测到 ``long double``。

在函数参数或返回类型的情况下， 当它是一个简单的整数/浮点类型时， 你可以简单地误判它。 如果你将函数 ``void f(long)`` 误认为是 ``void f(int)``， 它仍然有效 (但你必须使用适合int的参数调用它)。 它的工作原理是因为C编译器会为我们进行转换。 此参数和返回类型的C语言级转换仅适用于常规函数， 而不适用于函数指针类型; 目前，它也不适用于可变函数。

对于更复杂的类型， 您别无选择， 只能精确。 例如， 你不能将 ``int *`` 参数误认为是 ``long *``，或者是全局数组 ``int a[5];`` 误认为 ``long a[5];``。 CFFI认为 `上面列出的所有类型`_ 视为原始类型  (所以 ``long long a[5];`` 和 ``int64_t a[5]`` 是不同的声明)。 其中的原因在 `关于问题的讨论`__ 中有详细说明。

.. __: https://bitbucket.org/cffi/cffi/issues/265/cffi-doesnt-allow-creating-pointers-to#comment-28406958


ffibuilder.compile() 等: 编译out-of-line模块
--------------------------------------------------------

您可以使用以下某个函数来实际生成使用 ``ffibuilder.set_source()`` 和
``ffibuilder.cdef()`` 编写的.py或.c文件。

请注意，这些函数不会覆盖具有完全相同内容的.py/.c文件，以保留最后修改时间。在某些情况下，无论如何都需要更新最后修改时间，请在调用函数之前删除该文件。

*版本1.8中的新功能:*  ``emit_c_code()`` 或
``compile()`` 生成的C语言代码包含 ``#define Py_LIMITED_API``。 这意味着在CPython >= 3.2时，编译此源会生成二进制.so/.dll， 它应该适用于任何CPython >= 3.2的版本 (而不是仅适用于相同版本的CPython x.y)。 但是， 标准的 ``distutils``
包仍会产生一个名为
``NAME.cpython-35m-x86_64-linux-gnu.so`` 的文件。 您可以手动将其重命名为
``NAME.abi3.so``， 或使用setuptools版本26或更高版本。 另请注意， 使用Python的调试版本进行编译实际上并不会定义
``Py_LIMITED_API``， 因为这样做会使 ``Python.h`` 不适当。

*版本1.12中的新功能:* ``Py_LIMITED_API`` 现在也在Windows上定义。
如果你使用 ``virtualenv``， 你需要它的最新版本: 早于16.0.0的版本不用将 ``python3.dll`` 复制到虚拟环境中。 如果升级 ``virtualenv`` 是一个真正的问题， 您可以手动编辑C代码以删除第一行 ``# define
Py_LIMITED_API``。

**ffibuilder.compile(tmpdir='.', verbose=False, debug=None):**
显式生成.py或.c文件，并(假如是.c)编译它。 输出文件是（或者是）放在 ``tmpdir`` 给出的目录中。 在这里给出的示例中， 我们在构建脚本中使用
``if __name__ == "__main__": ffibuilder.compile()``,  如果直接执行它们， 这会使它们重建当前目录中的.py/.c文件。 (注意: 如果在调用 ``set_source()`` 时指定了包，则使用 ``tmpdir`` 相应子目录。)

*版本1.4中的新功能:* ``verbose`` 参数。 如果为True，则打印通常的distutils输出，包括调用编译器的命令行。 (在将来的版本中， 此参数可能默认更改为True。)

*版本1.8.1中的新功能:* ``debug`` 参数。如果设置为bool，它将控制是否以调试模式编译C代码。 默认的None表示使用本机Python的 ``sys.flags.debug``。
从版本1.8.1开始，如果您运行的是调试模式Python，则默认情况下会以调试模式编译C代码 (请注意，无论如何必须在Windows上执行此操作)。

**ffibuilder.emit_python_code(filename):** 生成给定的.py文件 (与用于ABI模式的 ``ffibuilder.compile()`` 相同， 具有要写入的显式命名文件)。 如果您愿意，可以将此.py文件包含在您自己的发行版中: 对于任何Python版本(2或3)都是相同的。 

**ffibuilder.emit_c_code(filename):** 生成给定的.c文件(用于API模式)而不编译它。 如果您有其他方法来编译它， 则可以使用， 例如 如果您想与一些更大的构建系统集成，这些系统将为您编译这个文件。 您还可以分发.c文件: 除非您使用的构建脚本依赖于操作系统或平台，否则.c文件本身是通用的 (如果在不同的操作系统上生成， 使用不同版本的CPython或使用PyPy，它将完全相同; 它是通过生成适当的 ``#ifdef`` 来完成的)。

**ffibuilder.distutils_extension(tmpdir='build', verbose=True):** 用于基于distutils的 ``setup.py`` 文件。 如果需要，在给定的 ``tmpdir`` 中调用会创建.c文件，并返回
``distutils.core.Extension`` 实例。

对于Setuptools，在 ``setup.py`` 中使用
``cffi_modules=["path/to/foo_build.py:ffibuilder"]`` 行代替。 这行要求Setuptools导入并使用CFFI提供的帮助程序，CFFI执行文件 ``path/to/foo_build.py`` (与
``execfile()`` 一样)， 并查找名为 ``ffibuilder`` 的全局变量。 你也可以这样 ``cffi_modules=["path/to/foo_build.py:maker"]``， 其中
``maker`` 命名为全局函数; 调用它时没有参数，应该返回一个 ``FFI`` 对象。


ffi/ffibuilder.include(): 合并多个CFFI接口
------------------------------------------------------------

**ffi/ffibuilder.include(other_ffi)**: 包括在另一个FFI实例中定义的类型定义(typedef)，结构体(struct)，联合(union)，枚举(enum)和常量(const)。 这适用于大型项目， 其中一个基于CFFI的接口依赖于在不同的基于CFFI的接口中声明的某些类型。

*请注意，每个库只应使用一个ffi对象; ffi.include()的预期用法是要与几个相互依赖的库进行交互。*  对于一个库，请创建一个 ``ffi``
对象。 (如果一个文件太大，你可以从几个Python文件中通过相同的 ``ffi`` 编写多个 ``cdef()`` 调用。)

对于out-of-line模块，``ffibuilder.include(other_ffibuilder)``
行应该出现在构建脚本中，而 ``other_ffibuilder`` 参数应该是来自另一个构建脚本的另一个FFI实例。 当两个构建脚本转换为生成的文件时，比如 ``_ffi.so`` 和
``_other_ffi.so``，然后导入 ``_ffi.so`` 在内部导致
``_other_ffi.so`` 被导入。 此时， ``_other_ffi.so`` 中的实际声明与 ``_ffi.so`` 中的实际声明相结合。

``ffi.include()`` 的用法是C语言中 ``#include`` 的cdef级别等价物， 其中程序的一部分可能包含在另一部分中为其自身使用而定义的类型和函数。 您可以在 ``ffi`` 对象 (以及 *包含* 支持的关联 ``lib`` 对象) 上看到包含支持声明的类型和常量。 在API模式下，您还可以直接查看函数和全局变量。
在ABI模式下，必须通过 ``other_ffi`` 上的 ``dlopen()`` 方法返回的原始 ``other_lib`` 对象访问这些对象。


ffi.cdef()限制
----------------------

``cdef()`` 和一些C99应该支持所有的ANSI C *声明*。 (这不包括任何 ``#include`` 或 ``#ifdef``。)
已知缺少的功能是C99， 或GCC或MSVC扩展:

* 任何 ``__attribute__`` 或 ``#pragma pack(n)``

* 其他类型: 特殊大小的浮点和定点类型，向量类型等。

* 自版本1.11起cffi支持C99类型 ``float _Complex`` 和 ``double _Complex``，但不支持libffi: 您不能使用复杂参数或返回值调用C函数，除非它们是直接API模式函数。 完全不支持 ``long double _Complex`` 类型 (声明并使用它，好像它是一个包含两个
  ``long double`` 的数组，并在C语言中使用set_source()编写包装函数).

* ``__restrict__`` 或 ``__restrict`` 分别是GCC和MSVC的扩展。 他们不被认可。 但是 ``restrict`` 是一个C关键字并被接受 (和忽略)。

注意像 ``int field[];`` 结构中的结构被解释为可变长度结构。 另一方面，像 ``int field[...];`` 这样的声明是数组，其长度将由编译器完成。  你可以使用 ``int field[];``
对于实际上不是可变长度的数组字段; 它也可以工作，但在这种情况下，由于CFFI认为它无法向C编译器询问数组的长度，因此可以减少安全检查：例如，您可能会通过在构造函数中传递过多的数组项来覆盖以下字段。

*版本1.2中的新功能:*
可以访问线程局部变量 (``__thread``)， 以及定义为动态宏的变量 (``#define myvar  (*fetchme())``)。 在1.2版之前， 您需要编写getter/setter函数。

请注意，如果在不使用 ``const`` 的情况下在 ``cdef()`` 中声明变量，CFFI会假定它是一个读写变量并生成两段代码，一段用于读取，一段用于写入。如果变量实际上不能用C代码写入，由于某种原因，它将无法编译。 在这种情况下，您可以将其声明为常量: 例如， 你会使用 ``foo_t *const
myglob;`` 而不是 ``foo_t *myglob;``。  另请注意 ``const foo_t *myglob;``  是一个 *变量;* 它包含一个指向常量 ``foo_t`` 的变量指针。


调试dlopenC库
-------------------------------

在 ``dlopen()`` 设置中，一些C库实际上很难正确使用。 这是因为大多数C语言库都是针对它们与另一个程序 *链接* 的情况下使用静态链接或动态链接进行测试， 但是从C语言编写的程序，在启动时，使用链接器的功能而不是 ``dlopen()``。

这有时可能会产生问题。 你会在另一个设置中遇到与CFFI相同的问题， 比如使用 ``ctypes`` 甚至是调用 ``dlopen()`` 的普通C代码。 本节包含一些通常有用的环境变量(在Linux上)，可以在调试这些问题时提供帮助。

**export LD_TRACE_LOADED_OBJECTS=all**

    提供了很多信息，有时大多取决于设置。 输出有关动态链接器的详细调试信息。 如果设置为 ``all``， 则打印它具有的所有调试信息，如果设置为 ``help`` 打印有关可在此环境变量中指定哪些类别的帮助消息

**export LD_VERBOSE=1**

    (glibc自2.1起) 如果设置为非空字符串，则在查询有关程序的信息时输出有关程序的符号版本控制信息 (即，已设置 ``LD_TRACE_LOADED_OBJECTS``， 或者已为动态链接程序提供了 ``--list`` 或 ``--verify`` 选项)。

**export LD_WARN=1**

    (仅限ELF)(glibc自2.1.3起) 如果设置为非空字符串，则警告未解析的符号。


ffi.verify(): in-line API模式
------------------------------

**ffi.verify()** 支持向后兼容，但已弃用。 ``ffi.verify(c_header_source, tmpdir=..,  ext_package=.., modulename=.., flags=.., **kwargs)`` 从 ``ffi.cdef()`` 生成并编译C语言文件， 如 ``ffi.set_source()`` 在API模式下，然后立即加载并返回动态库对象。 使用一些重要的逻辑来决定是否必须重新编译动态库; 请参阅以下有关控制它的方法。

``c_header_source`` 和额外关键字参数的含义与 ``ffi.set_source()`` 中的含义相同。

``ffi.verify()`` 的一个剩余用例将以下修改以明确查找任何类型的大小， 以字节为单位， 并立即在Python中使用它 (例如因为需要编写构建脚本的其余部分):

.. code-block:: python

    ffi = cffi.FFI()
    ffi.cdef("const int mysize;")
    lib = ffi.verify("const int mysize = sizeof(THE_TYPE);")
    print lib.mysize

``ffi.verify()`` 的额外参数:
    
*  ``tmpdir`` 控制C语言文件的创建和编译位置。 除非设置了 ``CFFI_TMPDIR`` 环境变量， 默认值为
   ``directory_containing_the_py_file/__pycache__`` 使用.py文件的目录名，该文件包含对
   ``ffi.verify()`` 的实际调用。 (这有一点修改，但通常与您的库的.pyc文件的位置一致。
   名称 ``__pycache__`` 本身来自Python 3。)

*  ``ext_package`` 控制应该从哪个包中查找编译的扩展模块。 这仅在分发基于ffi.verify()的模块后才有用。

*  ``tag`` 参数在扩展模块的名称中间插入一个额外的字符串: ``_cffi_<tag>_<hash>``。
   有用的是提供更多的上下文，例如调试时。

*  ``modulename`` 参数可用于强制特定模块名称，覆盖名称 ``_cffi_<tag>_<hash>``。  小心使用，例如如果要将变量信息传递给 ``verify()`` 但仍希望模块名称始终相同 (例如 本地文件的绝对路径)。在这种情况下，不计算散列，如果模块名称已经存在，则无需进一步检查即可重复使用。 每当您更改源时，请务必使用其他方法清除 ``tmpdir``。

* ``source_extension`` 与 ``ffibuilder.set_source()`` 中的含义相同。

*  可选的 ``flags`` 参数 (在Windows上被忽略) 默认为
   ``ffi.RTLD_NOW``; 参见 ``man dlopen``。 (使用 ``ffibuilder.set_source()``， 您将使用 ``sys.setdlopenflags()``。)

*  如果需要列出传递给C编译器的本地文件，则可选的 ``relative_to`` 参数很有用::

     ext = ffi.verify(..., sources=['foo.c'], relative_to=__file__)

   与上行大致相同的::

     ext = ffi.verify(..., sources=['/path/to/this/file/foo.c'])

   除了生成的库的默认名称是根据参数 ``sources`` 的CRC校验和和构建的， 以及您给 ``ffi.verify()`` 的大多数其他参数， 但不是 ``relative_to``。
   因此，如果您使用第二行，它将在您的项目安装后停止查​​找已编译的库，因为 ``'/path/to/this/file'`` 突然改变了。  第一行没有这个问题。

请注意，在开发期间，每次更改传递给 ``cdef()`` 或 ``verify()`` 的C源时，后者将根据从这些字符串计算的两个CRC32哈希值创建新的模块文件名。 这会在 ``__pycache__`` 目录中创建越来越多的文件。建议您不时的清理它。
一个很好的方法是在测试套件中添加对
``cffi.verifier.cleanup_tmpdir()`` 的调用。 或者，您可以手动删除整个 ``__pycache__`` 目录。

另一个缓存目录可以作为 ``verify()`` 的 ``tmpdir`` 参数，通过环境变量 ``CFFI_TMPDIR``，或者在调用 ``verify`` 之前调用 ``cffi.verifier.set_tmpdir(path)``。
。


从CFFI 0.9升级到CFFI 1.0
-----------------------------------

CFFI 1.0是向后兼容的，但考虑转向1.0中的新out-of-line方法仍然是一个好主意。 这是步骤。

**ABI 模式**，如果您的CFFI项目使用 ``ffi.dlopen()``:

.. code-block:: python

    import cffi

    ffi = cffi.FFI()
    ffi.cdef("stuff")
    lib = ffi.dlopen("libpath")

*如果* "stuff" 部分足够大以至于导入时间是一个问题，那么按照 `out-of-line但仍然是ABI模式`__
的描述重写它。 可选， 另请参见 `setuptools集成`__ 段落。

.. __: out-of-line-abi_
.. __: distutils-setuptools_


**API 模式**， 如果您的CFFI项目使用 ``ffi.verify()``:

.. code-block:: python

    import cffi

    ffi = cffi.FFI()
    ffi.cdef("stuff")
    lib = ffi.verify("real C code")

然后你应该按照上面的 `out-of-line,
API 模式`__ 中的描述重写它。 它避免了一些导致
``ffi.verify()`` 随着时间推移增加一些额外参数的问题。然后查看 `distutils 或 setuptools`__ 段落。 另外，请记住从 ``setup.py`` 中删除 ``ext_package=".."``，这有时需要使用 ``verify()`` 但只是与 ``set_source()`` 产生混淆。

.. __: out-of-line-api_
.. __: distutils-setuptools_

以下示例应该适用于旧版本(1.0之前版本)和新版本的CFFI版本， 支持这两个版本在旧版本的PyPy上运行非常重要 (CFFI 1.0在PyPy < 2.6中不起作用):

.. code-block:: python

    # in a separate file "package/foo_build.py"
    import cffi

    ffi = cffi.FFI()
    C_HEADER_SRC = r'''
        #include "somelib.h"
    '''
    C_KEYWORDS = dict(libraries=['somelib'])

    if hasattr(ffi, 'set_source'):
        ffi.set_source("package._foo", C_HEADER_SRC, **C_KEYWORDS)

    ffi.cdef('''
        int foo(int);
    ''')

    if __name__ == "__main__":
        ffi.compile()

并在主程序中:

.. code-block:: python

    try:
        from package._foo import ffi, lib
    except ImportError:
        from package.foo_build import ffi, C_HEADER_SRC, C_KEYWORDS
        lib = ffi.verify(C_HEADER_SRC, **C_KEYWORDS)

(不论好坏， 这个最新技巧可以更普遍地用于允许导入"执行"，即使没有生成 ``_foo`` 模块。)

编写一个兼容CFFI 0.9和1.0的 ``setup.py`` 脚本需要显式检查我们可以拥有的CFFI版本， 它被硬编码为PyPy中的内置模块:

.. code-block:: python

    if '_cffi_backend' in sys.builtin_module_names:   # PyPy
        import _cffi_backend
        requires_cffi = "cffi==" + _cffi_backend.__version__
    else:
        requires_cffi = "cffi>=1.0.0"

然后我们使用 ``requires_cffi`` 变量根据需要为
``setup()`` 提供不同的参数，例如:

.. code-block:: python

    if requires_cffi.startswith("cffi==0."):
        # backward compatibility: we have "cffi==0.*"
        from package.foo_build import ffi
        extra_args = dict(
            ext_modules=[ffi.verifier.get_extension()],
            ext_package="...",    # if needed
        )
    else:
        extra_args = dict(
            setup_requires=[requires_cffi],
            cffi_modules=['package/foo_build.py:ffi'],
        )
    setup(
        name=...,
        ...,
        install_requires=[requires_cffi],
        **extra_args
    )
