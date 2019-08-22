================================
使用 ffi/lib 对象
================================

.. contents::

把这个页面放在枕边作为参考


.. _working:

使用指针，结构体和数组
--------------------------------------------

C语言代码的整数和浮点值映射到Python的常规 ``int``，``long`` 和 ``float``。 而且，C语言类型 ``char``
对应于Python中的单字符字符串。  (如果要将其映射到小整数，请使用 ``signed char`` 或 ``unsigned char``。)

同样，C语言类型 ``wchar_t`` 对应于单字符
unicode字符串。 请注意，在某些情况下(一个narrow的Python构建，具有底层的4字节wchar_t类型)，单个wchar_t字符可能对应于一对代理(码元，码位)，它们表示为长度为2的unicode字符串。 如果需要将这样的2-chars unicode字符串转换为整数，则 ``ord(x)`` 不起作用; 使用
``int(ffi.cast('wchar_t', x))`` 替换。

*版本1.11中的新功能:* 除了 ``wchar_t`` 之外，C语言类型
``char16_t`` 和 ``char32_t`` 的工作方式相同，但具有已知的固定大小。
在以前的版本中，这可以使用 ``uint16_t`` 和
``int32_t`` 实现，但不能自动转换为Python unicodes。

指针，结构和数组更复杂: 他们没有明显的Python等价物。 因此，它们对应类型
``cdata`` 的对象，例如，它们被打印为
``<cdata 'struct foo_s *' 0xa3290d8>``。

``ffi.new(ctype, [initializer])``: 此函数构建并返回给定 ``ctype`` 的新cdata对象。 ctype通常是一些描述C类型的常量字符串。 它必须是指针或数组类型。 如果是指针，例如 ``"int *"`` 或 ``struct foo *``，然后它为一个 ``int`` 或 ``struct foo`` 分配内存。 如果是数组，例如 ``int[10]``，然后它为10个
``int`` 分配内存。 在这两种情况下，返回的cdata都是 ``ctype`` 类型。

内存最初用零填充。 如下所述，也可以给出初始值。

例::

    >>> ffi.new("int *")
    <cdata 'int *' owning 4 bytes>
    >>> ffi.new("int[10]")
    <cdata 'int[10]' owning 40 bytes>

    >>> ffi.new("char *")          # allocates only one char---not a C string!
    <cdata 'char *' owning 1 bytes>
    >>> ffi.new("char[]", "foobar")  # this allocates a C string, ending in \0
    <cdata 'char[]' owning 7 bytes>

与C不同，返回的指针对象对分配的内存具有 *所有权*: 当这个确切的对象被垃圾收集时，内存被释放。  如果在C语言级别，你在其他地方存储了一个指向内存的指针，接着确保你还可以根据需要保持对象存活。 (如果您立即将返回的指针强制转换为其他类型的指针，这也适用: 只有原始对象拥有所有权，所以你必须保持它的存活。 一旦忽略它，那么转换的指针将指向垃圾! 换句话说，所有权规则附加到包装器cdata对象: 它们不是也不能，附加到底层的原始内存中。) 
例:

.. code-block:: python

    global_weakkeydict = weakref.WeakKeyDictionary()

    def make_foo():
        s1   = ffi.new("struct foo *")
        fld1 = ffi.new("struct bar *")
        fld2 = ffi.new("struct bar *")
        s1.thefield1 = fld1
        s1.thefield2 = fld2
        # here the 'fld1' and 'fld2' object must not go away,
        # otherwise 's1.thefield1/2' will point to garbage!
        global_weakkeydict[s1] = (fld1, fld2)
        # now 's1' keeps alive 'fld1' and 'fld2'.  When 's1' goes
        # away, then the weak dictionary entry will be removed.
        return s1

通常你不需要弱字典: 例如，要使用包含指向 ``char *`` 指针的指针的 ``char * *`` 参数调用函数，这样就足够了:

.. code-block:: python

    p = ffi.new("char[]", "hello, world")    # p is a 'char *'
    q = ffi.new("char **", p)                # q is a 'char **'
    lib.myfunction(q)
    # p is alive at least until here, so that's fine

然而，这总是错的 (使用释放的内存):

.. code-block:: python

    p = ffi.new("char **", ffi.new("char[]", "hello, world"))
    # WRONG!  as soon as p is built, the inner ffi.new() gets freed!

出于同样的原因，这也是错误的:

.. code-block:: python

    p = ffi.new("struct my_stuff")
    p.foo = ffi.new("char[]", "hello, world")
    # WRONG!  as soon as p.foo is set, the ffi.new() gets freed!


cdata对象主要支持与C中相同的操作: 你可以从指针，数组和结构中读取或写入。  取消引用一个指针通常在C语言中语法为 ``*p``，这不是有效的Python，所以你必须使用替代语法 ``p[0]``
(这也是有效的C)。 另外，C语言中的``p.x`` 和 ``p->x``
语法都在Python中成为 ``p.x``。

我们有 ``ffi.NULL`` 同 C语言 ``NULL`` 在相同位置使用。
与后者类似，它实际上被定义为 ``ffi.cast("void *",
0)``。 例如，读取一个NULL指针返回一个 ``<cdata 'type *'
NULL>``，你可以检查, 例如通过与
``ffi.NULL`` 比较。

C中的 ``&`` 运算符没有一般等价物 (因为它不适合模型，而且这里似乎不需要它)。 有 `ffi.addressof()`__，但仅限于某些情况。 例如，你不能在Python中获取数字的 "地址"; 同样,
你不能获取CFFI指针的地址。 如果你有这种C语言代码::

    int x, y;
    fetch_size(&x, &y);

    opaque_t *handle;      // some opaque pointer
    init_stuff(&handle);   // initializes the variable 'handle'
    more_stuff(handle);    // pass the handle around to more functions

那么你需要像这样重写它，用逻辑上指向变量的指针替换C中的变量:

.. code-block:: python

    px = ffi.new("int *")
    py = ffi.new("int *")              arr = ffi.new("int[2]")
    lib.fetch_size(px, py)    -OR-     lib.fetch_size(arr, arr + 1)
    x = px[0]                          x = arr[0]
    y = py[0]                          y = arr[1]

    p_handle = ffi.new("opaque_t **")
    lib.init_stuff(p_handle)   # pass the pointer to the 'handle' pointer
    handle = p_handle[0]       # now we can read 'handle' out of 'p_handle'
    lib.more_stuff(handle)

.. __: ref.html#ffi-addressof


在C中返回指针或数组或结构类型的任何操作都会为您提供一个新的cdata对象。 与"原始"的方式不同，这些新的cdata对象没有所有权: 它们仅仅是对现有内存的引用。

作为上述规则的例外，取消引用一个拥有
*struct* 或 *union* 对象的指针会返回一个"共同拥有"相同内存的cdata struct 或 union 对象。 因此，在这种情况下，有两个对象可以保持相同的内存存活。 这样做是为了你真正想拥有一个struct对象但没有任何合适的位置来保存原始指针对象 (由
``ffi.new()`` 返回)。

例:

.. code-block:: python

    # void somefunction(int *);

    x = ffi.new("int *")      # allocate one int, and return a pointer to it
    x[0] = 42                 # fill it
    lib.somefunction(x)       # call the C function
    print x[0]                # read the possibly-changed value

``ffi.cast("type", value)`` 是C转换(casts)提供的等价物。
他们应该像在C中一样工作。 另外，这是获取整数或浮点类型的cdata对象的唯一方法::

    >>> x = ffi.cast("int", 42)
    >>> x
    <cdata 'int' 42>
    >>> int(x)
    42

将指针强制转换为int，将它转换为 ``intptr_t`` 或 ``uintptr_t``，
由C定义为足够大的整数类型 (例如32位)::

    >>> int(ffi.cast("intptr_t"，pointer_cdata))    # signed
    -1340782304
    >>> int(ffi.cast("uintptr_t", pointer_cdata))   # unsigned
    2954184992L

``ffi.new()``
的可选第二个参数给出的初始值可以是您用作C代码的初始值的任何东西,
使用列表或元组而不是使用C语法 ``{ .., .., .. }``。
例::

    typedef struct { int x, y; } foo_t;

    foo_t v = { 1, 2 };            // C syntax
    v = ffi.new("foo_t *", [1, 2]) # CFFI equivalent

    foo_t v = { .y=1, .x=2 };                // C99 syntax
    v = ffi.new("foo_t *", {'y': 1, 'x': 2}) # CFFI equivalent

与C一样，字符数组也可以从字符串初始化，在这种情况下，隐式附加终止空字符::

    >>> x = ffi.new("char[]", "hello")
    >>> x
    <cdata 'char[]' owning 6 bytes>
    >>> len(x)        # the actual size of the array
    6
    >>> x[5]          # the last item in the array
    '\x00'
    >>> x[0] = 'H'    # change the first item
    >>> ffi.string(x) # interpret 'x' as a regular null-terminated string
    'Hello'

同样，可以从unicode字符串初始化wchar_t或char16_t或char32_t的数组，并在cdata对象上调用 ``ffi.string()`` 返回存储在源数组中的当前unicode字符串 (必要时添加代理(码元，码位))。
有关更多详细信息，请参阅 `Unicode字符类型`__ 部分。

.. __: ref.html#unichar

请注意，与Python列表或元组不同，但与C类似，你不能使用负数在最后的C数组中索引。

更一般地说，C语言数组类型的长度可以在C类型中未指定，只要它们的长度可以从初始化值得到，就像在C中::

    int array[] = { 1, 2, 3, 4 };           // C syntax
    array = ffi.new("int[]", [1, 2, 3, 4])  # CFFI equivalent

作为扩展，初始化值也可以只是一个数字，而且给出了长度 (如果你只想要零初始化)::

    int array[1000];                  // C syntax
    array = ffi.new("int[1000]")      # CFFI 1st equivalent
    array = ffi.new("int[]", 1000)    # CFFI 2nd equivalent

如果长度实际上不是常数，这将非常有用，以避免像 ``ffi.new("int[%d]" % x)`` 这样的事情。 实际上，不建议这样做:
``ffi`` 通常缓存字符串 ``"int[]"`` 不需要一直重新解析它。

C99支持可变大小结构，只要初始化值说明数组长度:

.. code-block:: python

    # typedef struct { int x; int y[]; } foo_t;

    p = ffi.new("foo_t *", [5, [6, 7, 8]]) # length 3
    p = ffi.new("foo_t *", [5, 3])         # length 3 with 0 in the array
    p = ffi.new("foo_t *", {'y': 3})       # length 3 with 0 everywhere

最后，请注意，任何用作初始化值的Python对象也可以在没有 ``ffi.new()`` 的情况下直接用于数组项或结构字段的赋值。 实际上，``p = ffi.new("T*", initializer)`` 是等价于 ``p = ffi.new("T*"); p[0] = initializer``。  例:

.. code-block:: python

    # if 'p' is a <cdata 'int[5][5]'>
    p[2] = [10, 20]             # writes to p[2][0] and p[2][1]

    # if 'p' is a <cdata 'foo_t *'>, and foo_t has fields x, y and z
    p[0] = {'x': 10, 'z': 20}   # writes to p.x and p.z; p.y unmodified

    # if, on the other hand, foo_t has a field 'char a[5]':
    p.a = "abc"                 # writes 'a', 'b', 'c' and '\0'; p.a[4] unmodified

在函数调用中，传递参数时，这些规则也可以使用;
请参见 `函数调用`_.


Python 3支持
----------------

支持Python 3，但要注意的要点是C语言
类型 ``char`` 对应Python类型 ``bytes``，而不是 ``str``。 在将所有Python字符串传递给CFFI或从CFFI接收它们时，您有责任将所有Python字符串编码/解码为字节。

这仅涉及 ``char`` 类型和派生类型; 在Python 2中接受字符串的API的其他部分继续接受Python 3中的字符串。


调用类似main的一个例子
---------------------------------------

想象一下，我们有某些类似这个:

.. code-block:: python

   from cffi import FFI
   ffi = FFI()
   ffi.cdef("""
      int main_like(int argv, char *argv[]);
   """)
   lib = ffi.dlopen("some_library.so")

现在，一切都很简单，除了，我们如何在这里创建 ``char**`` 参数?
第一个想法:

.. code-block:: python

   lib.main_like(2, ["arg0", "arg1"])

不起作用，因为初始化值接收两个Python ``str`` 对象，
期望它是 ``<cdata 'char *'>`` 对象。 您需要显式使用
``ffi.new()`` 来创建这些对象:

.. code-block:: python

   lib.main_like(2, [ffi.new("char[]", "arg0"),
                     ffi.new("char[]", "arg1")])

请注意，两个 ``<cdata 'char[]'>`` 对象在调用期间保持活动状态: 它们仅在列表本身被释放时释放，并且仅在调用返回时释放列表。

如果你想要构建一个你想要重复使用的 "argv" 变量，那么需要更加小心:

.. code-block:: python

   # DOES NOT WORK!
   argv = ffi.new("char *[]", [ffi.new("char[]", "arg0"),
                               ffi.new("char[]", "arg1")])

在上面的示例中，只要构建"argv"，就会释放内部"arg0"字符串。 您必须确保保留对内部 "char[]" 对象的引用，直接或通过保持列表活动，像这样:

.. code-block:: python

   argv_keepalive = [ffi.new("char[]", "arg0"),
                     ffi.new("char[]", "arg1")]
   argv = ffi.new("char *[]", argv_keepalive)


函数调用
--------------

调用C函数时，传递参数主要遵循与分配给结构字段相同的规则，返回值遵循与读取结构字段相同的规则。  例如:

.. code-block:: python

    # int foo(short a, int b);

    n = lib.foo(2, 3)     # returns a normal integer
    lib.foo(40000, 3)     # raises OverflowError

您可以将 ``char *`` 参数传递给普通的Python字符串 (但是不要将普通的Python字符串传递给带有 ``char *``
参数的函数并且改变它!):

.. code-block:: python

    # size_t strlen(const char *);

    assert lib.strlen("hello") == 5

您还可以将unicode字符串作为 ``wchar_t *`` 或 ``char16_t *`` 或
``char32_t *`` 参数传递。  请注意C语言在使用 ``type *`` 或 ``type[]`` 的参数声明之间没有区别。  例如， ``int *`` 完全等价于 ``int[]`` (甚至 ``int[5]``; 5被忽略了)。 对于CFFI， 这意味着您始终可以传递参数以转换为 ``int *`` 或 ``int[]``。  例如:

.. code-block:: python

    # void do_something_with_array(int *array);

    lib.do_something_with_array([1, 2, 3, 4, 5])    # works for int[]

请参见 `参考: 转换`__ 类似于传递 ``struct foo_s *`` 参数的方法————但一般来说，
在这种情况下传递
``ffi.new('struct foo_s *', initializer)`` 更清晰。

__ ref.html#conversions

CFFI支持传递和返回结构和联合到函数和回调。  例:

.. code-block:: python

    # struct foo_s { int a, b; };
    # struct foo_s function_returning_a_struct(void);

    myfoo = lib.function_returning_a_struct()
    # `myfoo`: <cdata 'struct foo_s' owning 8 bytes>

出于性能，通过编写 ``lib.some_function`` 获得的非可变API级别级函数不是 ``<cdata>``
对象，而是不同类型的对象 (在CPython上，``<built-in
function>``)。 这意味着您不能将它们直接传递给其他C
语言函数期望的函数指针参数。 只有 ``ffi.typeof()``
才能使用它们。 要获取包含常规函数指针的cdata，请使用 ``ffi.addressof(lib, "name")``。

支持的参数和返回类型有一些（模糊的）限制。 这些限制来自libffi，仅适用于调用 ``<cdata>`` 函数指针; 换句话说，如果您使用API​​模式，它们不适用于不可变参数 ``cdef()`` 声明的函数。 限制是您不能直接作为参数传递或返回类型:

* 联合(union) (但是联合 *指针* 是不受限制的);

* 一个使用位字段的结构体(struct) (但这样的结构体 *指针* 是不受限制的);

* 在 ``cdef()`` 中用 "``...``" 声明的结构体。.

在API模式下，你可以解决这些限制: 例如，如果你需要从Python调用这样的函数指针，您可以改为编写一个自定义C语言函数，该函数接受函数指针和真实参数，并从C语言执行调用。 然后在 ``cdef()`` 中声明自定义C语言函数并从Python中使用它。


可变函数调用
-----------------------

C语言中的可变参数函数 (以 "``...``" 作为最后一个参数结束) 可以正常声明和调用，但可变部分中传递的所有参数必须是cdata对象。
这是因为无法猜测，如果你写了这个::

    lib.printf("hello, %d\n", 42)   # doesn't work!

你真的认为42作为C语言 ``int`` 传递，并不是
``long`` 或 ``long long``。 ``float`` 与
``double`` 会出现同样的问题。 所以你必须强制你想要的C语言类型是cdata对象,
必要时使用 ``ffi.cast()``:

.. code-block:: python
  
    lib.printf("hello, %d\n", ffi.cast("int", 42))
    lib.printf("hello, %ld\n", ffi.cast("long", 42))
    lib.printf("hello, %f\n", ffi.cast("double", 42))

但理所当然:

.. code-block:: python

    lib.printf("hello, %s\n", ffi.new("char[]", "world"))

请注意，如果您使用的是 ``dlopen()``，``cdef()`` 中的函数声明必须与C中的原始声明完全匹配，像往常一样————尤其如此，如果此函数在C语言中是可变参数的，那么它的 ``cdef()``
声明也必须是可变的。 您不能使用固定参数在
``cdef()`` 中声明它，即使你打算只用这些参数类型调用它。 原因是某些体系结构具有不同的调用约定，具体取决于函数签名是否固定。 (在x86-64上，如果某些参数是 ``double`` 类型的，有时可以在PyPy的JIT生成的代码中看到差异。)

注意函数签名 ``int foo();`` 由CFFI解释为等同于 ``int foo(void);``。 这与C标准不同，
其中 ``int foo();`` 真的像 ``int foo(...);`` 并且可以使用任何参数调用。  (这个特征是C89之前的遗留: 在不依赖于特定于编译器的扩展的情况下，在 ``foo()`` 的主体中根本无法访问参数。 现在几乎所有代码都使用 ``int foo();`` 的真实意思是 ``int foo(void);``。)


内存压力 (PyPy)
----------------------

本段仅适用于PyPy，因为它的垃圾收集器(GC)与CPython不同。 在C语言代码中，通常有一对函数，一个执行内存分配或获取其他资源，而另一个让它们再次释放。  根据您构建Python代码的方式，只有在GC决定可以释放特定(Python)对象时才会调用释放函数。 这种情况尤其明显:

* 如果你使用 ``__del__()`` 方法来调用释放函数。

* 如果你使用 ``ffi.gc()`` 而不使用 ``ffi.release()``。

* 如果在确定的时间调用释放函数，则不会发生这种情况，例如在常规 ``try: finally:`` 语句块。 然而它确实发生 *在生成器内*———— 如果生成器没有明确释放但忘记了``yield``这一点，则封闭在 ``finally`` 块中的代码仅在下一个GC处调用。

在这些情况下，您可能必须使用内置函数
``__pypy__.add_memory_pressure(n)``。 它的参数 ``n`` 是对要添加的内存压力的估量。 例如，如果这对C语言函数我们谈论的是 ``malloc(n)`` 和 ``free()`` 或类似的函数，则在 ``malloc(n)`` 之后调用 ``__pypy__.add_memory_pressure(n)``。 这样做不总是问题的完整答案，
但它使下一个GC发生得更早，这通常就足够了。

如果内存分配是间接的，则同样适用，例如 C语言函数分配一些内部数据结构。 在这种情况下，使用参数 ``n`` 调用
``__pypy__.add_memory_pressure(n)`` 这是一个粗略估量。 知道确切的大小并不重要，并且在调用释放函数后不必再次手动降低内存压力。 如果您正在为分配/释放函数编写包装器，你应该在前者中调用
``__pypy__.add_memory_pressure()``，即使用户可以在 ``finally:`` 块的已知点调用后者。.

如果这个解决方案还不够，或者，如果获取的资源不是内存，而是其他更有限的内容(如文件描述符)，那么没有比重构代码更好的方法来确保在已知点调用释放函数而不是由GC间接调用。

请注意，在PyPy <= 5.6中，上面的讨论也适用于
``ffi.new()``. 在更新版本的PyPy中，both ``ffi.new()`` 和
``ffi.new_allocator()()`` 都会自动解释它们创建的内存压力。 (如果您需要支持较旧和较新的PyPy，
无论如何，尝试调用 ``__pypy__.add_memory_pressure()``; 最好扩大估量而不是考虑内存压力。)


.. _extern-python:
.. _`extern "Python"`:

外部 "Python" (新式回调)
-------------------------

当语言C代码需要一个指向函数的指针，该函数调用您选择的Python函数时，以下是在
out-of-line API模式下执行此操作的方法。 
关于 回调_ 的下一节描述了ABI模式解决方案。


这是 *1.4版本中的新功能*。 如果向后兼容性存在问题，
请使用旧式 回调_。 (原始回调调用较慢，并且与libffi的回调具有相同的问题; 值得注意的是，请看 警告__ 。 本节中描述的新样式根本不使用libffi的回调。)

.. __: Callbacks_

在构建器脚本中，在cdef中声明一个以
``extern "Python"`` 为前缀的函数::

    ffibuilder.cdef("""
        extern "Python" int my_callback(int, int);

        void library_function(int(*callback)(int, int));
    """)
    ffibuilder.set_source("_my_example", r"""
        #include <some_library.h>
    """)

然后，函数 ``my_callback()`` 在应用程序代码中的Python中实现::

    from _my_example import ffi, lib

    @ffi.def_extern()
    def my_callback(x, y):
        return 42

通过获取 ``lib.my_callback`` 获得 ``<cdata>`` 指针函数对象。 这个 ``<cdata>`` 可以传递给C代码， 然后像回调一样工作: 当C代码调用此函数指针时，调用Python函数 ``my_callback``。 (您需要将 ``lib.my_callback`` 传递给C语言代码，而不是 ``my_callback``: 后者只是上面的Python函数，不能传递给C语言。)

CFFI通过将 ``my_callback`` 定义为静态C语言函数来实现此功能，写在 ``set_source()`` 代码之后。 ``<cdata>``
然后指向此功能。 这个函数的作用是调用Python函数对象，该函数在运行时附加
``@ffi.def_extern()``。

``@ffi.def_extern()`` 装饰器应该应用于 **全局函数，** 一个用于同名的每个 ``extern "Python"`` 函数。

为了支持某些极端情况，可以通过再次调用 ``@ffi.def_extern()`` 来重新定义附加的Python函数，但是不建议这样做!  更好地为此名称附加单个全局Python函数，并首先灵活地写出来。 这是因为每个 ``extern "Python"`` 函数只能变成一个C语言函数。 调用 ``@ffi.def_extern()`` 会再次更改此函数的C逻辑以调用新的Python函数; 旧的Python函数不再可调用。 从 ``lib.my_function`` 获得的C语言函数指针始终是此C语言函数的地址，即 它保持不变。

外部 "Python" 和 ``void *`` 参数
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

如前所述，您不能使用 ``extern "Python"`` 来生成可变数量的语言C函数指针。 然而，在纯C语言代码中也无法实现该结果。 因此，C通常使用 ``void *data`` 参数定义回调。  您可以使用 ``ffi.new_handle()`` 和 ``ffi.from_handle()`` 通过 ``void *`` 参数传递Python对象。 例如，如果回调的C语言类型是::

    typedef void (*event_cb_t)(event_t *evt, void *userdata);

并通过调用此函数来注册事件::

    void event_cb_register(event_cb_t cb, void *userdata);

然后你会在构建脚本中编写这个::

    ffibuilder.cdef("""
        typedef ... event_t;
        typedef void (*event_cb_t)(event_t *evt, void *userdata);
        void event_cb_register(event_cb_t cb, void *userdata);

        extern "Python" void my_event_callback(event_t *, void *);
    """)
    ffibuilder.set_source("_demo_cffi", r"""
        #include <the_event_library.h>
    """)

并在您的主应用程序中注册这样的事件::

    from _demo_cffi import ffi, lib

    class Widget(object):
        def __init__(self):
            userdata = ffi.new_handle(self)
            self._userdata = userdata     # must keep this alive!
            lib.event_cb_register(lib.my_event_callback, userdata)

        def process_event(self, evt):
            print "got event!"

    @ffi.def_extern()
    def my_event_callback(evt, userdata):
        widget = ffi.from_handle(userdata)
        widget.process_event(evt)

其他一些库没有明确的 ``void *`` 参数，但是允许您将 ``void *`` 附加到现有结构中。 例如，
库可能会说 ``widget->userdata`` 是为应用程序保留的通用字段。 如果事件的签名现在是这样的话::

    typedef void (*event_cb_t)(widget_t *w, event_t *evt);

然后你可以使用低级 ``widget_t *`` 中的 ``void *`` 字段::

    from _demo_cffi import ffi, lib

    class Widget(object):
        def __init__(self):
            ll_widget = lib.new_widget(500, 500)
            self.ll_widget = ll_widget       # <cdata 'struct widget *'>
            userdata = ffi.new_handle(self)
            self._userdata = userdata        # must still keep this alive!
            ll_widget.userdata = userdata    # this makes a copy of the "void *"
            lib.event_cb_register(ll_widget, lib.my_event_callback)

        def process_event(self, evt):
            print "got event!"

    @ffi.def_extern()
    def my_event_callback(ll_widget, evt):
        widget = ffi.from_handle(ll_widget.userdata)
        widget.process_event(evt)

从C语言直接访问外部 "Python"
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

如果你想直接从 ``set_source()``  编写的C代码访问一些 ``extern "Python"`` 函数，你需要在之前写一个声明。 默认情况下，它必须是静态的，但请参阅 `下一段`__ 。) 在C代码之后由CFFI添加此函数的实际实现————这是必需的，因为声明可能使用由 ``set_source()`` 定义的类型 (例如 上面的 ``event_t`` 来自 ``#include``)，所以在此之前不能生成它。

.. __: `extern-python-c`_

::

    ffibuilder.set_source("_demo_cffi", r"""
        #include <the_event_library.h>

        static void my_event_callback(widget_t *, event_t *);

        /* here you can write C code which uses '&my_event_callback' */
    """)

这也可以用来编写直接调用Python的自定义C代码。 这是一个例子 (在这种情况下效率很低，但如果 ``my_algo()`` 中的逻辑要复杂得多，则可能会有用)::

    ffibuilder.cdef("""
        extern "Python" int f(int);
        int my_algo(int);
    """)
    ffibuilder.set_source("_example_cffi", r"""
        static int f(int);   /* the forward declaration */

        static int my_algo(int n) {
            int i, sum = 0;
            for (i = 0; i < n; i++)
                sum += f(i);     /* call f() here */
            return sum;
        }
    """)

.. _extern-python-c:

外部 "Python+C"
~~~~~~~~~~~~~~~~~

使用 ``extern "Python"`` 声明的函数在C源中生成为 ``static`` 函数。 但是， 在某些情况下，将它们设置为非静态是很方便的，通常当您想要从其他C语言源文件直接调用它们时。 要做到这一点，你可以说 ``extern "Python+C"`` 而不只是 ``extern "Python"``。  *版本1.6中的新功能*。

+------------------------------------+--------------------------------------+
| 如果cdef包含                       | 然后CFFI生成                         |
+------------------------------------+--------------------------------------+
| ``extern "Python" int f(int);``    | ``static int f(int) { /* code */ }`` |
+------------------------------------+--------------------------------------+
| ``extern "Python+C" int f(int);``  | ``int f(int) { /* code */ }``        |
+------------------------------------+--------------------------------------+

名称 ``extern "Python+C"`` 来自于我们想要两种意义上的外部函数: 作为一个 ``extern "Python"``，并且作为非静态的C语言函数。

你不能让CFFI生成额外的宏或其他特定于编译器的东西，比如GCC  ``__attribute__``。 您只能控制该功能是否应该是 ``static`` 的。 但通常，
这些属性必须与函数 *头文件* 一起写入，如果函数 *实现* 不重复它们就没问题::

    ffibuilder.cdef("""
        extern "Python+C" int f(int);      /* not static */
    """)
    ffibuilder.set_source("_example_cffi", r"""
        /* the forward declaration, setting a gcc attribute
           (this line could also be in some .h file, to be included
           both here and in the other C files of the project) */
        int f(int) __attribute__((visibility("hidden")));
    """)


外部 "Python": 参考
~~~~~~~~~~~~~~~~~~~~~~~~~~

``extern "Python"`` 必须出现在cdef()中。 就像C++ ``extern
"C"`` 语法一样，它也可以用于围绕一组函数的大括号::

    extern "Python" {
        int foo(int);
        int bar(int);
    }

The ``extern "Python"`` 函数现在不能是可变参数函数。 这可以在将来实施。 (这个 `演示`__ 展示了如何做到这一点，但它有点冗长。)

.. __: https://bitbucket.org/cffi/cffi/src/default/demo/extern_python_varargs.py

每个对应的Python回调函数都是使用
``@ffi.def_extern()`` 装饰器定义的。 编写此函数时要小心: 如果它引发异常，或尝试返回错误类型的对象，那么异常无法传播。 而是将异常打印到stderr，并使C语言级回调返回默认值。 这可以通过 ``error`` 和
``onerror`` 来控制，如下面所描述的。

.. _def-extern:

``@ffi.def_extern()`` 装饰器接受这些可选参数:

* ``name``: cdef中写入的函数的名称。 默认情况下，它取自您装饰的Python函数的名称。

.. _error_onerror:

* ``error``: 如果Python函数引发异常则返回一个值。 默认为0或null。 异常仍然打印到stderr，所以这应该只用作最后的解决方案。

* ``onerror``: 如果你想确保捕获所有异常，使用
  ``@ffi.def_extern(onerror=my_handler)``。 如果发生异常并指定了
  ``onerror``，然后调用 ``onerror(exception, exc_value,
  traceback)``。 这在某些情况下非常有用，在这种情况下，您不能简单地编写 ``try: except:`` 在主回调函数中，
  因为它可能无法捕获信号处理程序引发的异常: 如果在C中发生信号，则尽快调用Python信号处理程序，这是在进入回调函数之后但在执行 ``try:`` *之前*。 如果信号处理程序抛出，
  我们还没有进入 ``try: except:``。

  如果调用 ``onerror`` 并正常返回，然后假设它自己处理异常并且没有任何内容打印到stderr。 如果 ``onerror`` 抛出，则会打印两个traceback信息。
  最后，``onerror`` 本身可以在C语言中提供回调的结果值，但不一定: 如果它只是返回 None————或者如果 ``onerror`` 本身失败————那么将使用 ``error`` 的值，如果有的话。

  注意下面的技巧: 在 ``onerror`` 中，您可以按如下方式访问原始回调参数。 首先检查 ``traceback`` 是否为
  None (它是 None 例如 如果整个函数成功运行但转换返回值时出错: 这在调用后发生).  如果 ``traceback`` 不是 None，
  ``traceback.tb_frame`` 是最外层函数的框架，
  即直接用
  ``@ffi.def_extern()`` 装饰的函数的框架。 所以你可以通过阅读 ``traceback.tb_frame.f_locals['argname']`` 获得该帧中 ``argname`` 的值。


.. _Callbacks:
.. _回调:

回调 (旧式)
---------------------

以下是如何创建一个包含函数指针的新 ``<cdata>`` 对象，该函数调用您选择的Python函数::

    >>> @ffi.callback("int(int, int)")
    >>> def myfunc(x, y):
    ...    return x + y
    ...
    >>> myfunc
    <cdata 'int(*)(int, int)' calling <function myfunc at 0xf757bbc4>>

请注意 ``"int(*)(int, int)"`` 是C *函数指针* 类型，而
``"int(int, int)"`` 是C *函数* 类型。 两者都可以指定为
ffi.callback() 结果是相同的。

.. warning::

    为ABI模式提供回调或向后兼容。 如果你正在使用 out-of-line API 模式，建议使用 `extern "Python"`_ 机制而不是回调: 它提供更快，更清洁的代码。 它还避免了旧式回调的几个问题:

    - 在不太常见的架构上，更容易在回调上崩溃 (`例如在NetBSD上`__);

    - 在 PAX and SELinux 等强化系统上，额外的内存保护可能会产生干扰 (例如，在 SELinux 上您需要在 ``deny_execmem`` 设置为 ``off`` 的情况下运行)。

    - `On Mac OS X，`__ 您需要为应用程序提供授权
      ``com.apple.security.cs.allow-unsigned-executable-memory``。

    还要注意尝试针对此问题的cffi修复————请参阅 ``ffi_closure_alloc`` 分支————但没有合并，因为它使用 ``fork()`` 创建潜在的 `memory corruption`__。

    换一种说法: 是的，允许在程序中写入+执行内存是危险的; 这就是为什么存在上述各种"强化"选项的原因。 但与此同时，这些选择为另一次攻击打开了大门: 如果程序分支然后尝试调用任何 ``ffi.callback()``，然后，这会立即导致崩溃————或者，攻击者只需要很少的工作就可以执行任意代码。 对我来说，它听起来比原来的问题更危险，这就是为什么cffi没有与他一起使用的原因。

    要在受影响的平台上一劳永逸地解决问题，您需要重构所涉及的代码，以便它不再使用 ``ffi.callback()``.

.. __: https://github.com/pyca/pyopenssl/issues/596
.. __: https://bitbucket.org/cffi/cffi/issues/391/
.. __: https://bugzilla.redhat.com/show_bug.cgi?id=1249685

警告: 与ffi.new()一样，ffi.callback()返回一个拥有其C语言数据所有权的cdata。 (在这种情况下，必要的C语言数据包含用于执行回调的libffi数据结构。)  这意味着只要此cdata对象处于活动状态，就只能调用回调。
如果将函数指针存储到C语言代码中，只要可以调用回调，就确保你也保持这个对象的存活。。
最简单的方法是始终只在模块级使用 ``@ffi.callback()``，并尽可能使用 `ffi.new_handle()`__ 传递 "context" 信息。 例:

.. __: ref.html#new-handle

.. code-block:: python

    # a good way to use this decorator is once at global level
    @ffi.callback("int(int, void *)")
    def my_global_callback(x, handle):
        return ffi.from_handle(handle).some_method(x)


    class Foo(object):

        def __init__(self):
            handle = ffi.new_handle(self)
            self._handle = handle   # must be kept alive
            lib.register_stuff_with_callback_and_voidp_arg(my_global_callback, handle)

        def some_method(self, x):
            print "method called!"

(另请参阅上面关于 `extern "Python"`_ 的部分，使用相同的一般风格。)

请注意，不支持可变参数函数类型的回调。 解决方法是添加自定义C代码。 在下面的示例中，回调获取第一个参数，该参数计算传递多少额外的 ``int``
参数:

.. code-block:: python

    # file "example_build.py"

    import cffi

    ffibuilder = cffi.FFI()
    ffibuilder.cdef("""
        int (*python_callback)(int how_many, int *values);
        void *const c_callback;   /* pass this const ptr to C routines */
    """)
    ffibuilder.set_source("_example", r"""
        #include <stdarg.h>
        #include <alloca.h>
        static int (*python_callback)(int how_many, int *values);
        static int c_callback(int how_many, ...) {
            va_list ap;
            /* collect the "..." arguments into the values[] array */
            int i, *values = alloca(how_many * sizeof(int));
            va_start(ap, how_many);
            for (i=0; i<how_many; i++)
                values[i] = va_arg(ap, int);
            va_end(ap);
            return python_callback(how_many, values);
        }
    """)
    ffibuilder.compile(verbose=True)

.. code-block:: python
    
    # file "example.py"

    from _example import ffi, lib

    @ffi.callback("int(int, int *)")
    def python_callback(how_many, values):
        print ffi.unpack(values, how_many)
        return 0
    lib.python_callback = python_callback

弃用: 你也可以使用 ``ffi.callback()`` 作为装饰器而不是直接作为 ``ffi.callback("int(int, int)", myfunc)``。 这是不鼓励的: 使用这个样式，我们更有可能在它仍在使用时过早忘记回调对象。

``ffi.callback()`` 装饰器也接受可选参数
``error``，并从CFFI版本1.2接受可选参数 ``onerror``。
这两个工作方式与 `上面描述的 "extern Python"`__ 相同。

.. __: error_onerror_



Windows: 调用约定
----------------------------

在Win32上，函数可以有两个主要的调用约定: "cdecl" (默认)，或 "stdcall" (也被称为 "WINAPI")。 还有其他罕见的调用约定，但这些都不受支持。
*版本1.3中的新功能*。

当您从Python发出调用到C时，实现是这样的，它适用于这两个主要调用约定中的任何一个; 你不必指定它。 However，但是，如果您操作"函数指针"类型的变量或声明回调，则调用约定必须正确。 这是通过在类型中写入 ``__cdecl`` 或 ``__stdcall``
来完成的，就像在C中一样::

    @ffi.callback("int __stdcall(int, int)")
    def AddNumbers(x, y):
        return x + y

或::

    ffibuilder.cdef("""
        struct foo_s {
            int (__stdcall *MyFuncPtr)(int, int);
        };
    """)

支持 ``__cdecl`` 但始终是默认值，因此可以省略它。 在 ``cdef()`` 中，您还可以使用 ``WINAPI`` 作为
``__stdcall`` 的等效项。 如上所述，它几乎不需要 (但不会受到影响) 在 ``cdef()`` 中声明一个普通函数时说明 ``WINAPI`` 或 ``__stdcall``。 (如果使用 ``ffi.addressof()`` 显式指向此函数的指针，或者函数是 ``extern "Python"``，仍然可以看到差异。)

这些调用约定说明符被接受但在32位Windows以外的任何平台上都被忽略。

在1.3之前的CFFI版本中，调用约定说明符无法识别。 在API模式下，您可以通过使用间接来解决它，就像关于 回调_
(``"example_build.py"``) 一节中的示例一样。 在ABI模式下无法使用stdcall回调。


FFI 接口
-------------

(FFI接口的参考已移至 `下一页`__.)

.. __: ref.html
