================================
使用CFFI进行嵌入
================================

.. contents::

您可以使用CFFI生成C代码，该代码将您选择的API导出到任何想要与此C代码链接的C应用程序。 您自己定义的此API最终将作为 ``.so/.dll/.dylib``
库的API， 或者您可以在较大的应用程序中静态链接它。

可能的用例:

* 将用Python编写的库直接显露给C/C++程序。

* 使用Python为已经编写的现有C/C++程序制作"插件"以加载它们。

* 使用Python实现更大的C/C++应用程序的一部分(使用静态链接)。

* 在Python中编写一个小的C/C++包装器， 隐藏了应用程序实际上是用Python编写的事实 (创建自定义命令行界面; 用于分发目的; 或者只是简单地对应用程序进行逆向工程).

总体思路如下:

* 您编写并执行Python脚本，该脚本生成带有您选择的API的 ``.c`` 文件 (并可选择将其编译为 ``.so/.dll/.dylib``)。 该脚本还提供了一些Python代码在 ``.so`` 中"封装"。

* 在运行时，C应用程序加载此 ``.so/.dll/.dylib`` (或与 ``.c`` 源代码静态链接)，而不必知道它是从Python和CFFI生成的。

* 第一次调用C函数时，Python被初始化并执行封装的Python代码。

* 封装的Python代码定义了更多实现API的C函数的Python函数，然后用于所有后续的C函数调用。

这种方法的目标之一是完全独立于CPython C API: 没有 ``Py_Initialize()`` 和 ``PyRun_SimpleString()``
甚至没有 ``PyObject``。 它在CPython和PyPy上的工作方式相同。

这完全是 *版本1.5中的新功能。*  (PyPy包含自5.0版以来的CFFI 1.5。)


用法
-----

.. __: overview.html#embedding

有关快速介绍，请参阅 `概述页面中的段落`__ 。在本节中，我们将更详细地解释每一步。 我们将在这里使用这个稍微扩展的例子:

.. code-block:: c

    /* file plugin.h */
    typedef struct { int x, y; } point_t;
    extern int do_stuff(point_t *);

.. code-block:: c

    /* file plugin.h, Windows-friendly version */
    typedef struct { int x, y; } point_t;

    /* When including this file from ffibuilder.set_source(), the
       following macro is defined to '__declspec(dllexport)'.  When
       including this file directly from your C program, we define
       it to 'extern __declspec(dllimport)' instead.

       With non-MSVC compilers we simply define it to 'extern'.
       (The 'extern' is needed for sharing global variables;
       functions would be fine without it.  The macros always
       include 'extern': you must not repeat it when using the
       macros later.)
    */
    #ifndef CFFI_DLLEXPORT
    #  if defined(_MSC_VER)
    #    define CFFI_DLLEXPORT  extern __declspec(dllimport)
    #  else
    #    define CFFI_DLLEXPORT  extern
    #  endif
    #endif

    CFFI_DLLEXPORT int do_stuff(point_t *);

.. code-block:: python

    # file plugin_build.py
    import cffi
    ffibuilder = cffi.FFI()

    with open('plugin.h') as f:
        # read plugin.h and pass it to embedding_api(), manually
        # removing the '#' directives and the CFFI_DLLEXPORT
        data = ''.join([line for line in f if not line.startswith('#')])
        data = data.replace('CFFI_DLLEXPORT', '')
        ffibuilder.embedding_api(data)

    ffibuilder.set_source("my_plugin", r'''
        #include "plugin.h"
    ''')

    ffibuilder.embedding_init_code("""
        from my_plugin import ffi

        @ffi.def_extern()
        def do_stuff(p):
            print("adding %d and %d" % (p.x, p.y))
            return p.x + p.y
    """)

    ffibuilder.compile(target="plugin-1.5.*", verbose=True)
    # or: ffibuilder.emit_c_code("my_plugin.c")

运行上面的代码会生成一个 *DLL*，即，一个可动态加载的库。 它是Windows上的扩展名为 ``.dll``，Mac OS/X上的 ``.dylib`` 或其他平台上的 ``.so`` 文件。 像往常一样，它是通过生成一些中间 ``.c`` 代码然后调用常规平台特定的C编译器来生成的。 有关使用生成的库的C语言级别问题的一些指示，请参见 下文__。

.. __: `关于使用.so的问题`_

以下是有关上述方法的一些细节:

* **ffibuilder.embedding_api(source):** 解析给定的C源，它声明了您希望由DLL导出的函数。 它还可以声明类型，常量和全局变量，它们是DLL的C语言级别API的一部分。

  在 ``source`` 文件中找到的函数将在 ``.c`` 文件中自动定义: 它们将包含在第一次调用Python解释器时初始化Python解释器的代码，然后是调用附加的Python函数的代码 (使用
  ``@ffi.def_extern()``， 请参阅下一点)。

  另一方面，全局变量不会自动生成。 您必须在 
  ``ffibuilder.set_source()`` 中显式地编写它们的定义，作为常规C代码 (参见接下来的一点)。

* **ffibuilder.embedding_init_code(python_code):** 这给出了初始化时间Python源代码。  此代码在DLL中被复制("封装")。 在运行时，代码在首次初始化DLL时执行，就在Python本身初始化之后。 这个新初始化的Python解释器有一个额外的"内置"模块，可以神奇地加载而无需访问任何文件，使用类似 "``from my_plugin import ffi,
  lib``" 的一行。 名称 ``my_plugin`` 来自
  ``ffibuilder.set_source()`` 的第一个参数。 从Python的角度来看，这个模块代表了“调用者的C语言世界”。

  初始化时间Python代码可以像往常一样导入其他模块或包。 您可能会遇到典型的Python问题，例如需要先手动设置 ``sys.path``。

  对于 ``ffibuilder.embedding_api()`` 中声明的每个函数， 初始化时间Python代码或其导入的模块之一应使用装饰器 ``@ffi.def_extern()`` 将相应的Python函数附加到它。

  如果初始化时间Python代码因异常而失败，那么您将获得打印到stderr的traceback以及更多信息，以帮助您识别错误的 ``sys.path`` 等问题。 如果某个函数在C代码尝试调用它时仍未附加，则还会向stderr打印一条错误消息，该函数返回零/null。

  请注意，CFFI模块从不调用 ``exit()``，但CPython本身包含调用 ``exit()`` 的代码， 例如，如果导入 ``site`` 失败。 这可能会在将来解决。

* **ffibuilder.set_source(c_module_name, c_code):** 从Python的角度设置模块的名称。 它还提供了更多的C代码，这些代码将包含在生成的C代码中。 在简单的例子中，它可以是一个空字符串。 您可以在其中 ``#include`` 其他一些文件，定义全局变量等。 宏
  ``CFFI_DLLEXPORT`` 可用于此C语言代码: 它扩展到特定于平台的方式表示"应该从DLL导出以下声明"。 例如， 您可以将 "``extern int
  my_glob;``" 放在 ``ffibuilder.embedding_api()`` 和 "``CFFI_DLLEXPORT int
  my_glob = 42;``" 放在 ``ffibuilder.set_source()`` 中。

  目前，``ffibuilder.embedding_api()`` 中声明的任何类型也必须存在于 ``c_code`` 中。 如果此代码在上面的示例中包含类似 ``#include "plugin.h"`` 的行，则这是自动的。

* **ffibuilder.compile([target=...] [, verbose=True]):** 制作C代码并编译它。 默认情况下，它会生成一个名为
  ``c_module_name.dll``，``c_module_name.dylib`` 或
  ``c_module_name.so`` 的文件，但可以使用可选的 ``target`` 关键字参数更改默认值。 你可以使用带有文字 ``*`` ``target="foo.*"`` 在Windows上请求一个名为
  ``foo.dll`` 的文件，在OS/X上请求 ``foo.dylib`` ，在其他地方使用 ``foo.so``。 指定备用目标的一个原因是包括Python模块名称中通常不允许的字符，例如
  "``plugin-1.5.*``"。

  对于更复杂的情况，您可以调用
  ``ffibuilder.emit_c_code("foo.c")`` 并使用其他方法编译生成的 ``foo.c``
  文件。 CFFI的编译逻辑基于标准库 ``distutils`` 包，它是为了制作CPython扩展模块而开发和测试的; 它可能并不总是适合制作通用DLL。 此外，如果您不想制作独立的 ``.so/.dll/.dylib`` 文件，只需获取C代码即可: 这个C文件可以作为更大的应用程序的一部分进行编译和静态链接。


阅读更多
------------

如果您正在阅读有关嵌入的此页面，并且您已经不熟悉CFFI，请参阅下面的内容:

* 对于 ``@ffi.def_extern()`` 函数，整数C类型只是作为Python整数传递; 简单的指向结构和基本数组的指针都很简单。  但是，迟早您需要在 此处__ 详细了解此主题。

* ``@ffi.def_extern()``: 请参阅 `此处的文档,`__ 特别是如果Python函数引发异常会发生什么。

* 要创建附加到C数据的Python对象，一种常见的解决方案是使用 ``ffi.new_handle()``。 请参阅 此处__ 的文档。

* 在嵌入模式中，主要方向是调用Python函数的C代码。 这与CFFI的常规扩展模式相反， 其中主要方向是调用C的Python代码。 这就是为什么页面 `使用ffi/lib对象`_ 首先讨论后者，以及为什么"C代码调用Python"的方向通常在该页面中被称为"回调"。 如果您还需要让Python代码调用C代码，请阅读下面有关
  `嵌入和扩展`_ 的更多信息。

* ``ffibuilder.embedding_api(source)``: 遵循与
  ``ffibuilder.cdef()`` 相同的语法， `文档在此。`__  您也可以使用 "``...``"
  语法，但在实践中它可能没有 ``cdef()`` 那么有用。 另一方面，预计通常需要提供给 ``ffibuilder.embedding_api()`` 的C语言source与您希望提供给DLL用户的某些 ``.h`` 文件的内容完全相同。 这就是上面的例子这样做的原因::

      with open('foo.h') as f:
          ffibuilder.embedding_api(f.read())

  请注意，这种方法的缺点是 ``ffibuilder.embedding_api()``
  不支持 ``#ifdef`` 指令。你可能不得不使用更复杂的表达式::

      with open('foo.h') as f:
          lines = [line for line in f if not line.startswith('#')]
          ffibuilder.embedding_api(''.join(lines))

  如上例所示，您也可以使用 ``ffibuilder.set_source()`` 中的相同 ``foo.h``::

      ffibuilder.set_source('module_name', r'''
          #include "foo.h"
      ''')


.. __: using.html#working
.. __: using.html#def-extern
.. __: ref.html#ffi-new-handle
.. __: cdef.html#cdef

.. _`使用ffi/lib对象`: using.html


疑难解答
---------------

* 错误消息

    cffi extension module 'c_module_name' has unknown version 0x2701

  表示正在运行的Python解释器位于早于1.5的CFFI版本。 必须在正在运行的Python中安装CFFI 1.5或更高版本。

* 在PyPy上，错误消息

    debug: pypy_setup_home: directories 'lib-python' and 'lib_pypy' not
    found in pypy's shared library location or in any parent directory

  表示找到了 ``libpypy-c.so`` 文件，但未在此位置找到标准库。 至少在某些Linux发行版中会出现这种情况，因为它们将 ``libpypy-c.so`` 放在 ``/usr/lib/``,
  中，而不是我们推荐的方式，这是: 将该文件保存在
  ``/opt/pypy/bin/`` 中，并在 ``/usr/lib/`` 中添加符号链接。
  最快的解决方法是手动进行更改。


关于使用.so的问题
--------------------------

本段描述的问题不一定是CFFI特有的。 它假定您已经获得了如上所述的 ``.so/.dylib/.dll`` 文件，但是您在使用它时遇到了麻烦。 (总之: 这是一团糟。 这是我自己的经验，通过使用Google和查看来自各种平台的报告。 请报告本段中的任何不准确之处或更好的方法。)

* CFFI生成的文件应遵循此命名模式: Linux上的 ``libmy_plugin.so``， Mac上的 ``libmy_plugin.dylib`` 或Windows上的 ``my_plugin.dll`` (Windows上没有 ``lib`` 前缀)。

* 首先请注意，此文件不包含Python解释器， 也不包含Python的标准库。 你仍然需要它在某个地方。 有一些方法可以将它压缩为较少数量的文件， 但这超出了CFFI的范围 (请报告您是否成功使用了其中一些方法， 以便我可以在此处添加一些链接).

* 在我们称之为"主程序"的地方， ``.so`` 可以动态使用 (例如通过在主程序中调用 ``dlopen()`` 或 ``LoadLibrary()``)， 也可以在编译时使用 (例如通过用 ``gcc -lmy_plugin`` 编译它 )。 如果您正在为程序构建插件，则始终使用前一种情况，并且程序本身不需要重新编译。 后一种情况是为了使CFFI库更紧密地集成在主程序中。

* 在编译时使用的情况下: 你可以在 ``-Lsome/path/`` 之前添加gcc选项 ``-lmy_plugin`` 来描述
  ``libmy_plugin.so`` 的位置。 在某些平台上，特别是Linux，如果能找到 ``libmy_plugin.so`` 而不是 ``libpython27.so`` 或 ``libpypy-c.so``， ``gcc`` 会报错。 要修复它，您需要调用
  ``LD_LIBRARY_PATH=/some/path/to/libpypy gcc``。

* 实际执行主程序时，需要找到
  ``libmy_plugin.so`` 以及 ``libpython27.so`` 或 ``libpypy-c.so``。
  对于PyPy，解压缩PyPy发行版，并在 ``bin`` 子目录中获得 ``libpypy-c.so`` 的完整目录结构，或者在顶级目录中的Windows  ``pypy-c.dll`` 上获取完整目录结构; 你不能移动这个文件，只是指向它。 指向它的一种方法是使用一些环境变量运行主程序:
  Linux上的 ``LD_LIBRARY_PATH=/some/path/to/libpypy``，OS/X上的
  ``DYLD_LIBRARY_PATH=/some/path/to/libpypy``。

* 如果使用内部硬编码的路径编译 ``libmy_plugin.so``，则可以避免 ``LD_LIBRARY_PATH`` 问题。  在Linux中，这是由 ``gcc -Wl,-rpath=/some/path`` 完成的。 你可以把这个选项放在 ``ffibuilder.set_source("my_plugin", ...,
  extra_link_args=['-Wl,-rpath=/some/path/to/libpypy'])`` 中。 该路径可以以 ``$ORIGIN`` 开头，表示"``libmy_plugin.so`` 所在的目录"。 然后，您可以指定相对于该位置的路径，例如 ``extra_link_args=['-Wl,-rpath=$ORIGIN/../venv/bin']``。
  U使用se ``ldd libmy_plugin.so`` 查看 ``$ORIGIN`` 扩展后当前编译的路径。)

  在此之后，您不再需要 ``LD_LIBRARY_PATH`` 来在运行时找到
  ``libpython27.so`` 或 ``libpypy-c.so``。从理论上讲，它还应该包括对主要程序的 ``gcc`` 调用。 如果rpath以 ``$ORIGIN`` 开头，我在Linux上没有 ``LD_LIBRARY_PATH`` 就无法很好的使用 ``gcc`` 

* 可以使用相同的rpath技巧让主程序在没有 ``LD_LIBRARY_PATH`` 的情况下首先找到
  ``libmy_plugin.so``.
  (如果主程序使用 ``dlopen()`` 将其作为动态插件加载，则不适用。) 您可以使用 ``gcc
  -Wl,-rpath=/path/to/libmyplugin`` 创建主程序，可能使用 ``$ORIGIN``。  ``$ORIGIN`` 中的 ``$`` 会导致各种shell问题: 如果使用通用shell，则需要说明 ``gcc
  -Wl,-rpath=\$ORIGIN``。 从Makefile中，你需要说明一些类似 ``gcc -Wl,-rpath=\$$ORIGIN`` 的语句。

* 在某些Linux发行版上，特别是Debian，CPython C扩展模块的 ``.so`` 文件可能会被编译而不会说明它们依赖于 ``libpythonX.Y.so``。如果嵌入器使用 ``dlopen(...,
  RTLD_LOCAL)`` 这使得这样的Python系统不适合嵌入。 您得到一个 ``undefined symbol`` 错误。 参见
  `问题 #264`__。  解决方法是首先调用
  ``dlopen("libpythonX.Y.so", RTLD_LAZY|RTLD_GLOBAL)``，这将强制首先加载 ``libpythonX.Y.so``。

.. __: https://bitbucket.org/cffi/cffi/issues/264/


使用多个CFFI制作的DLL
-----------------------------

多个CFFI制作的DLL可以由相同的过程使用。

请注意，进程中所有CFFI制作的DLL共享一个Python解释器。 这种效果与通过组装大量不相关的包来构建大型Python应用程序所获得的效果相同。 其中一些可能是从标准库中修补某些函数的库，例如，其他部分可能出乎意料。


多线程
--------------

基于Python的标准全局解释器锁(Global Interpreter Lock)，多线程应该透明地工作。

如果两个线程在Python尚未初始化时都尝试调用C函数，则会发生死锁。 一个线程继续初始化并阻塞另一个线程。 只有在执行初始化时间Python代码时才允许另一个线程继续。

如果两个线程调用两个不同的CFFI制造的DLL，Python初始化本身仍将被序列化，但两段初始化时间的Python代码不会。 其思想是，事先没有理由让一个DLL等待另一个DLL的初始化完成。

初始化之后，Python的标准全局解释器锁启动。 最终结果是当一个CPU在执行Python代码时，没有其他CPU可以从同一进程的另一个线程执行更多Python代码。 每隔一段时间，锁会切换到一个不同的线程， 这样就不会出现任何单个线程无限期阻塞。


测试
-------

出于测试目的，可以在正在运行的Python解释器中导入CFFI制造的DLL，而不是像C共享库一样加载。

您可能在文件名方面存在一些问题: 例如，在Windows上，Python期望的文件被称为 ``c_module_name.pyd``，但CFFI制造的DLL被称为 ``target.dll``。 基本名称
``target`` 是 ``ffibuilder.compile()`` 中指定的目标，在Windows上，扩展名为 ``.dll`` 而不是 ``.pyd``。 您必须重命名或复制文件，或者在POSIX上使用符号链接。

然后该模块就像常规的CFFI扩展模块一样工作。 它使用 "``from c_module_name import ffi, lib``" 导入，并在 ``lib`` 对象上公开所有C函数。 您可以通过调用这些C函数来测试它。 DLL内部封装的初始化时间Python代码在第一次完成此类调用时执行。


嵌入和扩展
-----------------------

嵌入模式与CFFI的非嵌入模式不兼容。

您可以在同一构建脚本中同时使用 ``ffibuilder.embedding_api()`` 和
``ffibuilder.cdef()``。 你把前面想要由DLL导出的声明放在前面; 你只需要在C和Python之间共享C函数和类型，而不是从DLL中导出。

作为一个例子，考虑你希望直接用C语言编写DLL导出的C函数的情况，也许在调用Python函数之前处理一些情况。 为此，您不能将函数的签名放在 ``ffibuilder.embedding_api()``。 (请注意，如果您使用 ``ffibuilder.embedding_api(f.read())`` 则需要更多修改。)
您只能在
``ffibuilder.set_source()`` 中编写自定义函数定义，并使用宏CFFI_DLLEXPORT作为前缀:

.. code-block:: c

    CFFI_DLLEXPORT int myfunc(int a, int b)
    {
        /* implementation here */
    }

如果需要，这个函数可以使用"回调"的一般机制调用Python函数, 这是因为它是从C到Python的调用，尽管在这种情况下它不会调用任何东西:

.. code-block:: python

    ffibuilder.cdef("""
        extern "Python" int mycb(int);
    """)

    ffibuilder.set_source("my_plugin", r"""

        static int mycb(int);   /* the callback: forward declaration, to make
                                   it accessible from the C code that follows */

        CFFI_DLLEXPORT int myfunc(int a, int b)
        {
            int product = a * b;   /* some custom C code */
            return mycb(product);
        }
    """)

然后Python初始化代码需要包含以下行:

.. code-block:: python

    @ffi.def_extern()
    def mycb(x):
        print "hi, I'm called with x =", x
        return x * 10

这个 ``@ffi.def_extern`` 将一个Python函数附加到C回调 ``mycb()``，在这种情况下，它不会从DLL导出。
然而，当调用 ``mycb()`` 时，会发生Python的自动初始化， 如果它恰好是从C调用的第一个函数。 更确切地说，调用 ``myfunc()`` 时不会发生这种情况: 这只是一个C函数，没有额外的代码如魔法般地镶嵌在它周围。 它只发生在 ``myfunc()`` 调用
``mycb()`` 时。

如上面的解释提示，这就是 ``ffibuilder.embedding_api()``
实际实现直接调用Python代码的函数调用的方式;
在这里，我们只是明确地分解它，以便在中间添加一些自定义C代码。

如果您需要强制从C代码中调用Python，在调用第一个 ``@ffi.def_extern()`` 之前进行初始化，你可以通过调用没有参数的C函数 ``cffi_start_python()`` 来实现。 它返回一个整数0或-1，以判断初始化是否成功。 目前， 无法阻止初始化失败， 也无法将traceback和更多信息转储到stderr。
