=======================================================
概览
=======================================================

.. contents::
   

第一部分介绍了一个使用CFFI从Python编译共享对象(DLL)中调用C函数的简单工作示例。 CFFI非常灵活，涵盖了第二部分中介绍的其他几个用例。 第三部分展示了如何将Python函数导出到嵌入在C或C ++应用程序中的Python解释器。 最后两节深入探讨了CFFI库。

确保你有 `安装cffi`__.

.. __: installation.html

.. _out-of-line-api-level:
.. _real-example:


主要使用方式
------------------

使用CFFI的主要方式是作为一些已经编译的共享对象的接口，这是由其他方法提供的。  想象一下，您有一个系统安装的共享对象名为 ``piapprox.dll`` (Windows) 或
``libpiapprox.so`` (Linux和其它) 或 ``libpiapprox.dylib`` (OS X)，导出函数 ``float pi_approx(int n);`` 在给定迭代次数的情况下计算pi的一些近似值。 你想从Python调用这个函数。 请注意，此方法与静态库 ``piapprox.lib`` (Windows) 或 ``libpiapprox.a`` 同样适用。

创建文件 ``piapprox_build.py``:

.. code-block:: python

      from cffi import FFI
      ffibuilder = FFI()

      # cdef() expects a single string declaring the C types, functions and
      # globals needed to use the shared object. It must be in valid C syntax.
      ffibuilder.cdef("""
          float pi_approx(int n);
      """)

      # set_source() gives the name of the python extension module to
      # produce, and some C source code as a string.  This C code needs
      # to make the declarated functions, types and globals available,
      # so it is often just the "#include".
      ffibuilder.set_source("_pi_cffi",
      """
      	   #include "pi.h"   // the C header of the library
      """,
      	   libraries=['piapprox'])   # library name, for the linker

      if __name__ == "__main__":
          ffibuilder.compile(verbose=True)

执行此脚本。 如果一切正常，它应该产生
``_pi_cffi.c``，然后在其上调用编译器。 生成的
``_pi_cffi.c`` 包含 ``set_source()`` 中给出的字符串的副本，
在这个例子中 ``#include "pi.h"``。 之后，它包含所有函数的胶水代码，在上面的 ``cdef()`` 中声明的类型和全局变量。

在运行时，您可以像这样使用扩展模块:

.. code-block:: python

    from _pi_cffi import ffi, lib
    print(lib.pi_approx(5000))

就这样！在本页的其余部分，我们将介绍一些更高级的示例和其他CFFI模式。  特别是，`如果您没有已安装的C库来调用`_ ，这是一个完整的示例。

有关 ``FFI`` 类的 ``cdef()`` 和 ``set_source()`` 方法的更多信息，请参阅 `准备和分发模块`__。

.. __: cdef.html

当您的示例有效时，手动运行构建脚本的常见替代方法是将其作为 ``setup.py``  的一部分运行。 以下是使用Setuptools分发的示例:

.. code-block:: python

    from setuptools import setup

    setup(
        ...
        setup_requires=["cffi>=1.0.0"],
        cffi_modules=["piapprox_build:ffibuilder"], # "filename:global"
        install_requires=["cffi>=1.0.0"],
    )


其他CFFI模式
----------------

CFFI可以用于四种模式之一: "ABI" 与 "API" 级别，
每个都有 "in-line" 或 "out-of-line" 准备(或编译)。

**ABI 模式** 以二进制级别访问库，而较快的 **API 模式** 使用C编译器访问它们。  我们解释了这个区别，更多细节如下__。

.. __: `abi-versus-api`_

在 **in-line 模式** 下，每次导入Python代码时都会设置所有内容。  在 **out-of-line 模式** 下，你有一个单独的准备步骤(可能还有C编译)，它产生一个模块，你的主程序可以导入该模块。


简单示例 (ABI 级别，in-line)
+++++++++++++++++++++++++++++++++++

对于那些使用过 ctypes_ 的人来说可能看起来很熟悉。

.. code-block:: python

    >>> from cffi import FFI
    >>> ffi = FFI()
    >>> ffi.cdef("""
    ...     int printf(const char *format, ...);   // copy-pasted from the man page
    ... """)                                  
    >>> C = ffi.dlopen(None)                     # loads the entire C namespace
    >>> arg = ffi.new("char[]", b"world")        # equivalent to C code: char arg[] = "world";
    >>> C.printf(b"hi there, %s.\n", arg)        # call printf
    hi there, world.
    17                                           # this is the return value
    >>>

请注意 ``char *`` 参数需要一个 ``bytes`` 对象。  如果你有一个
``str`` (或Python 2 上 ``unicode`` ) 你需要使用 ``somestring.encode(myencoding)`` 显示编码。

*Windows上的Python 3:* ``ffi.dlopen(None)`` 不起作用。 这个问题很乱，而且无法解决。 如果您尝试从系统上存在的特定DLL调用函数，则不会发生此问题: 然后你使用 ``ffi.dlopen("path.dll")`` 。

*此示例不调用任何C编译器。它在所谓的ABI模式下工作，这意味着如果你调用某个函数或访问cdef()中稍微错误声明结构的某些字段，它将崩溃。*

如果使用C编译器安装模块是一个选项，强烈建议使用API​​模式。 (它也更快)


Struct/Array 示例 (minimal，in-line)
+++++++++++++++++++++++++++++++++++++++

.. code-block:: python

    from cffi import FFI
    ffi = FFI()
    ffi.cdef("""
        typedef struct {
            unsigned char r, g, b;
        } pixel_t;
    """)
    image = ffi.new("pixel_t[]", 800*600)

    f = open('data', 'rb')     # binary mode -- important
    f.readinto(ffi.buffer(image))
    f.close()

    image[100].r = 255
    image[100].g = 192
    image[100].b = 128

    f = open('data', 'wb')
    f.write(ffi.buffer(image))
    f.close()

这可以用作 struct_ 和
array_ 模块的更灵活的替换，并替换 ctypes_ 。  你可以调用 ``ffi.new("pixel_t[600][800]")``
并得到一个二维数组。

.. _struct: http://docs.python.org/library/struct.html
.. _array: http://docs.python.org/library/array.html
.. _ctypes: http://docs.python.org/library/ctypes.html

*此示例不调用任何C编译器。*

这个例子也承认与 out-of-line 等价。 它类似于上面的第一个 `主要使用方式`_ 示例，
但将 ``None`` 作为第二个参数传递给
``ffibuilder.set_source()``。 接着在主程序中写入
``from _simple_example import ffi`` 然后从行 ``image =
ffi.new("pixel_t[]", 800*600)`` 开始，与上面的
in-line 示例相同的内容。


API模式，调用C标准库
++++++++++++++++++++++++++++++++++++++++

.. code-block:: python

    # file "example_build.py"

    # Note: we instantiate the same 'cffi.FFI' class as in the previous
    # example, but call the result 'ffibuilder' now instead of 'ffi';
    # this is to avoid confusion with the other 'ffi' object you get below

    from cffi import FFI
    ffibuilder = FFI()

    ffibuilder.set_source("_example",
       r""" // passed to the real C compiler,
            // contains implementation of things declared in cdef()
            #include <sys/types.h>
            #include <pwd.h>

            // We can also define custom wrappers or other functions
            // here (this is an example only):
            static struct passwd *get_pw_for_root(void) {
                return getpwuid(0);
            }
        """,
        libraries=[])   # or a list of libraries to link with
        # (more arguments like setup.py's Extension class:
        # include_dirs=[..], extra_objects=[..], and so on)

    ffibuilder.cdef("""
        // declarations that are shared between Python and C
        struct passwd {
            char *pw_name;
            ...;     // literally dot-dot-dot
        };
        struct passwd *getpwuid(int uid);     // defined in <pwd.h>
        struct passwd *get_pw_for_root(void); // defined in set_source()
    """)

    if __name__ == "__main__":
        ffibuilder.compile(verbose=True)

您需要运行一次 ``example_build.py`` 脚本以在文件 ``_example.c`` 中生成"源代码"，并将其编译为常规C扩展模块。  (CFFI根据 ``set_source()`` 的第二个参数是否为 ``None`` 来选择要生成Python模块或C模块)

*这个步骤需要一个C编译器。它产生一个名为例如_example.so或_example.pyd的文件。 如果需要，它可以像任何其他扩展模块一样以预编译形式分发。*

然后，在您的主程序中，您使用:

.. code-block:: python

    from _example import ffi, lib

    p = lib.getpwuid(0)
    assert ffi.string(p.pw_name) == b'root'
    p = lib.get_pw_for_root()
    assert ffi.string(p.pw_name) == b'root'

请注意 ``struct
passwd`` 与C设计确切无关 (它是"API 级别"，而不是"ABI 级别")。 它需要一个C编译器才能运行 ``example_build.py``，但它比尝试完全正确地获取 ``struct
passwd`` 字段的细节要便携得多。 同样，在 ``cdef()`` 中我们将 ``getpwuid()`` 声明为采用 ``int`` 参数; 在某些平台上，这可能稍微不正确，但并不重要。

另请注意，在运行时，API模式比ABI模式更快。

要使用Setuptools进行分发，将其集成到 ``setup.py`` :

.. code-block:: python

    from setuptools import setup

    setup(
        ...
        setup_requires=["cffi>=1.0.0"],
        cffi_modules=["example_build.py:ffibuilder"],
        install_requires=["cffi>=1.0.0"],
    )


.. _`如果您没有已安装的C库来调用`:

API 模式，调用C语言源码而不是编译库
+++++++++++++++++++++++++++++++++++++++++++++++++++++++++

如果要调用某些未预编译的库，但是你有C语言源代码，那么最简单的解决方案是创建一个从这个库中的C语言源代码编译的扩展模块，和额外的CFFI包装器(用于封装C语言源代码的库并构建扩展模块)。 例如，从 ``pi.c`` 和 ``pi.h`` 文件开始:

   .. code-block:: C

      /* filename: pi.c*/
      # include <stdlib.h>
      # include <math.h>
       
      /* Returns a very crude approximation of Pi
         given a int: a number of iteration */
      float pi_approx(int n){
      
        double i,x,y,sum=0;
      
        for(i=0;i<n;i++){
      
          x=rand();
          y=rand();
      
          if (sqrt(x*x+y*y) < sqrt((double)RAND_MAX*RAND_MAX))
            sum++; }
      
        return 4*(float)sum/(float)n; }

   .. code-block:: C

      /* filename: pi.h*/
      float pi_approx(int n);
      
创建一个脚本名为 ``pi_extension_build.py``，构建C语言扩展:

   .. code-block:: python

      from cffi import FFI
      ffibuilder = FFI()
      
      ffibuilder.cdef("float pi_approx(int n);")
   
      ffibuilder.set_source("_pi",  # name of the output C extension
      """
          #include "pi.h"',
      """,
          sources=['pi.c'],   # includes pi.c as additional sources
          libraries=['m'])    # on Unix, link with the math library
   
      if __name__ == "__main__":
          ffibuilder.compile(verbose=True)

构建扩展:
   
   .. code-block:: shell

      python pi_extension_build.py

注意到，在工作目录下，生成的输出文件:
``_pi.c``，``_pi.o`` 和编译的C语言扩展 (例如，在Linux上叫 ``_pi.so`` )。  它可以被PYthon调用:

   .. code-block:: python
   
       from _pi.lib import pi_approx
   
       approx = pi_approx(10)
       assert str(pi_approximation).startswith("3.")
   
       approx = pi_approx(10000)
       assert str(approx).startswith("3.1")  


.. _performance:

单纯的性能 (API level，out-of-line)
+++++++++++++++++++++++++++++++++++++++++++++++

`以上部分`__ 的变型，其目标不是调用现有的C库，而是编译并调用直接在构建脚本中编写的一些C语言函数:

.. __: real-example_

.. code-block:: python

    # file "example_build.py"

    from cffi import FFI
    ffibuilder = FFI()

    ffibuilder.cdef("int foo(int *, int *, int);")

    ffibuilder.set_source("_example",
    r"""
        static int foo(int *buffer_in, int *buffer_out, int x)
        {
            /* some algorithm that is seriously faster in C than in Python */
        }
    """)

    if __name__ == "__main__":
        ffibuilder.compile(verbose=True)

.. code-block:: python

    # file "example.py"

    from _example import ffi, lib

    buffer_in = ffi.new("int[]", 1000)
    # initialize buffer_in here...

    # easier to do all buffer allocations in Python and pass them to C,
    # even for output-only arguments
    buffer_out = ffi.new("int[]", 1000)

    result = lib.foo(buffer_in, buffer_out, 1000)

*您需要一个C编译器来运行example_build.py一次。 它产生一个文件名为 _example.so 或 _example.pyd。 如果可以，它可以像任何其他扩展模块一样以预编译形式分发。*


.. _out-of-line-abi-level:

Out-of-line，ABI 模式
++++++++++++++++++++++

out-of-line ABI 模式是常规(API) out-of-line
模式和in-line ABI 模式的混合。 它允许您使用 ABI 模式，具有其优点 (不需要C编译器) 和问题 (更容易崩溃).

这种混合模式可以大大减少导入时间，因为解析较大C头文件很慢。 它还允许您在构建期间进行更详细的检查，而不必担心性能
(例如 根据系统上检测到的库版本，使用小块声明多次调用 ``cdef()`` )。 

.. code-block:: python

    # file "simple_example_build.py"

    from cffi import FFI

    ffibuilder = FFI()
    # Note that the actual source is None
    ffibuilder.set_source("_simple_example", None)
    ffibuilder.cdef("""
        int printf(const char *format, ...);
    """)

    if __name__ == "__main__":
        ffibuilder.compile(verbose=True)

运行会产生 ``_simple_example.py``。 您的主程序仅导入此生成的模块，而不再是 ``simple_example_build.py``:

.. code-block:: python

    from _simple_example import ffi

    lib = ffi.dlopen(None)      # Unix: open the standard C library
    #import ctypes.util         # or, try this on Windows:
    #lib = ffi.dlopen(ctypes.util.find_library("c"))

    lib.printf(b"hi there, number %d\n", ffi.cast("int", 2))

注意这个 ``ffi.dlopen()``，不像in-line 模式,
不会调用任何额外的魔法来定位库: 它必须是路径名 (带或不带目录)，根据C的
``dlopen()`` 或 ``LoadLibrary()`` 函数的要求。 这意味着
``ffi.dlopen("libfoo.so")`` 没问题，但 ``ffi.dlopen("foo")`` 却不行。
在后一种情况下，你可以用
``ffi.dlopen(ctypes.util.find_library("foo"))`` 替换它。 此外，None仅在Unix上被识别以打开标准C库。

出于分发目的，请记住生成了一个新的
``_simple_example.py`` 文件。 您可以在项目的源文件中静态包含它，或者，使用Setuptools，您可以在 ``setup.py`` 中这样编写:

.. code-block:: python

    from setuptools import setup

    setup(
        ...
        setup_requires=["cffi>=1.0.0"],
        cffi_modules=["simple_example_build.py:ffibuilder"],
        install_requires=["cffi>=1.0.0"],
    )

总之，当您希望声明许多C语言结构但不需要与共享对象快速交互时，此模式很有用。例如，它对于解析二进制文件很有用。


In-line，API 模式
++++++++++++++++++

"API level + in-line" 模式存在错误，但很久就会弃用。  它曾经用 ``lib = ffi.verify("C header")`` 。
具有 ``set_source("modname", "C header")`` 的out-of-line 变型是首选的，并且当项目规模增大时避免了许多问题。


.. _embedding:

嵌入
---------

*版本1.5中的新功能。*

CFFI可用于 嵌入__: 创建一个标准的动态链接库 (Windows下 ``.dll`` ，其他地方 ``.so``)
可以在C应用程序中使用。

.. code-block:: python

    import cffi
    ffibuilder = cffi.FFI()

    ffibuilder.embedding_api("""
        int do_stuff(int, int);
    """)

    ffibuilder.set_source("my_plugin", "")

    ffibuilder.embedding_init_code("""
        from my_plugin import ffi

        @ffi.def_extern()
        def do_stuff(x, y):
            print("adding %d and %d" % (x, y))
            return x + y
    """)

    ffibuilder.compile(target="plugin-1.5.*", verbose=True)

这个简单的示例将 ``plugin-1.5.dll`` 或 ``plugin-1.5.so`` 创建为具有单个导出函数 ``do_stuff()`` 的DLL。 您使用要在内部使用的解释器执行上面的脚本一次; 它可以是CPython 2.x或3.x或PyPy。 然后可以从应用程序"照常"使用此DLL; 应用程序不需要知道它正在与使用Python和CFFI创建的库进行通信。 在运行时，当应用程序调用 ``int do_stuff(int,
int)`` 时，Python解释器会自动初始化并且被 ``def
do_stuff(x, y):`` 调用。 `请参阅有关嵌入的文档中的详细信息。`__

.. __: embedding.html
.. __: embedding.html


究竟发生了什么？
-----------------------

CFFI接口在与C语言相同的级别上运行————您使用与C语言中相同的语法声明类型和函数定义它们。　这意味着大多数文档或示例都可以直接从手册页中复制。

声明可以包含 **类型，函数，常量**
和 **和全局变量。** 传递给 ``cdef()`` 的内容不得包含其他内容; 特别是，``#ifdef`` 或 ``#include``
指令是不支持的。 上面例子中的cdef就是这样————他们声明“在C级别中有一个具有此给定签名的函数”，或者“存在具有此形状的结构类型”。

在 ABI 示例中， ``dlopen()`` 手动调用加载库。
在二进制级别，程序被分成多个命名空间 - 一个全局命名空间(在某些平台上)，每个库加一个命名空间。  因此
``dlopen()`` 返回一个 ``<FFILibrary>`` 对象，并且该对象具有来自该库的所有函数，常量和变量符号作为属性，并且已在
``cdef()`` 中声明。 如果要加载多个相互依赖的库，则只能调用一次 ``cdef()`` ，但可以多次调用 ``dlopen()`` 。

相反，API模式更像C语言程序: C链接器(静态或动态)负责查找使用的任何符号。
你将库名 ``libraries`` 作为
``set_source()`` 的关键字参数，但永远不需要说明哪个符号来自哪个库。
``set_source()`` 的其他常见参数包括 ``library_dirs`` 和
``include_dirs``; 所有这些参数都传递给标准的 
distutils/setuptools.

``ffi.new()`` 行分配C对象。 除非使用可选的第二个参数，否则它们最初用零填充。 如果指定，这个参数给出了一个"初始值"，就像你可以用C代码初始化全局变量一样。

实际的 ``lib.*()`` 函数调用应该是显而易见的: 就像C一样。


.. _abi-versus-api:

ABI 与 API
--------------

在二进制级别("ABI")访问C库充满了问题，特别是在非Windows平台上。

ABI级别最直接的缺点是调用函数需要通过非常通用的*libffi*库，这很慢(并且在非标准平台上并不总是完美测试)。 API模式改为编译直接调用目标函数的CPython C包装器。 它可以更快(并且比libffi工作得更好)。

更喜欢API模式的根本原因是 *C库通常用于与C编译器一起使用。* 你不应该做猜测结构中字段的位置。
上面的 "真实示例" 显示了CFFI如何使用C编译器: 此示例使用 ``set_source(..., "C source...")`` 而不是
``dlopen()``。 使用这种方法时，
我们的优点是我们可以在 ``cdef()`` 中的不同位置使用字面 "``...``"，缺少的信息将在C编译器的帮助下完成。 CFFI会将其转换为单个C源文件，其中包含未经修改的"C源代码"部分，后跟一些"魔术"C语言代码和从 ``cdef()`` 派生的声明。 编译此C语言文件时，生成的C扩展模块将包含我们需要的所有信息- 或者C编译器将发出警告或错误，例如.如果我们错误地声明了某些函数的签名。

请注意  ``set_source()`` 中的"C source" 部分可以包含任意C代码。 您可以使用它来声明一些用C编写的辅助函数。 要将这些帮助程序导出到Python，请将它们的签名放在 ``cdef()`` 中。
(您可以在"C source"部分中使用 ``static`` C关键字，如``static int myhelper(int x) { return x * 42; }`` 因为这些辅助只是在同一个C文件中生成的"魔术"C代码中引用。)

这可以用于例如将"crazy"宏包装到更标准的C函数中。 额外的C语言层在其他方面也很有用，喜欢调用期望一些复杂的参数结构的函数，你喜欢用C而不是Python构建。 (另一方面，如果您只需要调用"类似函数"的宏，那么您可以直接在 ``cdef()`` 中声明它们，就好像它们是函数一样。)

生成的C语言代码应该在运行它的平台(或Python版本)上独立相同，因此在简单的情况下，您可以直接分发预生成的C语言代码并将其视为常规C扩展模块(这取决于CPython上的 ``_cffi_backend`` 模块。)   `上面示例`__ 中的特殊Setuptools行是针对更复杂的情况，我们需要重新生成C源代码————例如: 因为重新生成此文件的Python脚本本身会查看系统以了解它应包含的内容。

.. __: real-example_
