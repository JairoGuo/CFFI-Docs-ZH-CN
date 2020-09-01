================================
CFFI 参考
================================

.. contents::


FFI 接口
-------------

*这个页面记录了运行时接口的两种类型 "FFI" 和
"CompiledFFI"。 这两种类型彼此非常相似。 如果导入out-of-line模块，则会获得CompiledFFI对象。 您从显式编写cffi.FFI()获得FFI对象。 与CompiledFFI不同，FFI类型还记录了其他方法在* `下一页`__.

.. __: cdef.html


ffi.NULL
++++++++

**ffi.NULL**: 类型为 ``<cdata 'void *'>`` 的常量NULL。


ffi.error
+++++++++

**ffi.error**: 他在各种情况下引发Python异常。 (不要将它与 ``ffi.errno`` 混淆。)


ffi.new()
+++++++++

**ffi.new(cdecl, init=None)**:
根据指定的C语言类型分配实例并返回指向它的指针。 指定的C语言类型必须是指针或数组: ``new('X *')`` 分配一个X并返回一个指向它的指针，而 ``new('X[10]')`` 分配一个10个X的数组并返回一个引用它的数组 (它主要像指针一样工作，就像在C中一样).
您还可以使用 ``new('X[]', n)`` 来分配非常数长度为n的数组。 请参阅其他有效初始化值的 `详细文档`__。

.. __: using.html#working

当返回的 ``<cdata>`` 对象超出范围时，将释放内存。 换句话说，返回的 ``<cdata>`` 对象拥有它指向的类型 ``cdecl`` 值的所有权。 这意味着只要此对象保持活动状态，就可以使用原始数据，但不能长时间使用。 将指针复制到其他地方的内存时要小心，例如 进入另一个结构。
另外，这意味着像 ``x = ffi.cast("B *", ffi.new("A *"))``
或 ``x = ffi.new("struct s[1]")[0]`` 这样的一行是错误的: 新分配的对象超出范围即刻，因此立即被释放，``x`` 是垃圾变量。 唯一可以这样做的是，对于指向结构体的指针和指向联合类型的指针，有一种特殊情况: 在 ``p = ffi.new("struct-or-union *", ..)`` 之后, 然后 ``p`` or ``p[0]`` 使内存保持活动状态。

在应用可选的初始化值之前，返回的内存最初被清除(用零填充)。 有关性能，请参阅 `ffi.new_allocator()`_ 以获取分配非零初始化内存的方法。

*版本1.12中的新功能:* 另见 ``ffi.release()``。


ffi.cast()
++++++++++

**ffi.cast("C type", value)**: 类似于C语言转换(cast): 返回使用给定值初始化的命名C语言类型的实例。 该值在任何类型的整数或指针之间转换。


.. _ffi-errno:
.. _ffi-getwinerror:

ffi.errno, ffi.getwinerror()
++++++++++++++++++++++++++++

**ffi.errno**: 从此线程中最近的C语言调用收到的 ``errno`` 的值，并传递给后面的C语言调用。 (这是一个线程本地读写属性。)

**ffi.getwinerror(code=-1)**: 在Windows上，除了 ``errno`` 之外，我们还在函数调用中保存和恢复 ``GetLastError()`` 值。 此函数将此错误代码作为元组 ``(code，message)`` 返回，在引发WindowsError时添加类似Python的可读消息。 如果给出参数 ``code``， 则将该代码格式化为消息而不是使用 ``GetLastError()``。
(则将该代码格式化为消息而不是使用 ``GetLastError()``
函数)


.. _ffi-string:
.. _ffi-unpack:

ffi.string(), ffi.unpack()
++++++++++++++++++++++++++

**ffi.string(cdata, [maxlen])**: 从'cdata'返回一个Python字符串(或unicode字符串)。

- 如果'cdata'是一个指针或字符数组或字节，则返回以null结尾的字符串。 返回的字符串将一直延伸到第一个空字符。 'maxlen'参数限制我们查找空字符的距离。 如果'cdata'是一个数组，那么'maxlen'默认为它的长度。 请参阅下面的 ``ffi.unpack()`` 以获取继续经过第一个空字符的方法。  *Python 3:* 这会返回一个 ``bytes``，而不是 ``str``.

- 如果'cdata'是wchar_t的指针或数组，则返回遵循相同规则的unicode字符串。 *版本1.11中的新功能:* 可以是char16_t或char32_t。

- 如果'cdata'是单个字符或字节或wchar_t或charN_t，则将其作为字节字符串或unicode字符串返回。  (请注意，在某些情况下，单个wchar_t或char32_t可能需要长度为2的Python unicode字符串。)

- 如果'cdata'是枚举，则将枚举数的值作为字符串返回。如果该值超出范围，则只返回字符串整数。

**ffi.unpack(cdata, length)**: 解包给定长度的C语言数据数组，返回Python字符串/ unicode/list。 'cdata'应该是一个指针;如果是数组，则首先将其转换为指针类型。 *版本1.6中的新功能。*

- 如果'cdata'是指向'char'的指针，则返回一个字节字符串。它不会在第一个空值处停止。 (一个等效的方法是
  ``ffi.buffer(cdata, length)[:]``。)

- 如果'cdata'是指向'wchar_t'的指针，则返回一个unicode字符串。
  ('length'以wchar_t的数量来衡量;它不是以字节为单位的大小。)  *版本1.11中的新功能:* 也可以是char16_t或char32_t。

- 如果'cdata'是指向其他任何内容的指针，则返回给定'length'的列表。 (一个较慢的方法是 ``[cdata[i] for i in
  range(length)]``。)


.. _ffi-buffer:
.. _ffi-from-buffer:

ffi.buffer(), ffi.from_buffer()
+++++++++++++++++++++++++++++++

**ffi.buffer(cdata, [size])**: 返回一个缓冲区对象，该对象引用给定'cdata'指向的原始C数据，其长度为'size'字节。  Python调用"一个缓冲区"，或者更准确地说是"支持缓冲区接口的对象"，是一个表示一些原始内存的对象，可以传递给各种内置或扩展函数;这些内置函数可以读取或写入原始内存直接，无需额外的拷贝。

'cdata' 参数必须是指针或数组。 如果未指定，缓冲区的大小是 ``cdata`` 指向的大小，或者数组的整个大小。

以下是buffer()有用的几个示例:

-  使用 ``file.write()`` 和 ``file.readinto()`` 与这样的缓冲区 (对于以二进制模式打开的文件)

-  覆盖结构的内容: 如果 ``p`` 是指向一个cdata，使用 ``ffi.buffer(p)[:] = newcontent``，其中 ``newcontent`` 是一个字节对象 (``str`` in Python 2)。

请记住，就像在C语言中一样，您可以使用 ``array + index`` 来获取指向数组的索引项的指针。 (在C中你可能更自然地编写
``&array[index]``，但这是等价的。)

返回的对象的类型不是内置 ``buffer`` ，也不是 ``memoryview``
类型，因为这些类型的API在Python版本中变化太大。
相反，除了支持缓冲区接口之外，它还具有以下Python API (Python 2的 ``buffer`` 子集。):

- ``buf[:]`` or ``bytes(buf)``: 将数据复制出缓冲区，返回常规字节字符串 (或 ``buf[start:end]`` 作为一个部分)

- ``buf[:] = newstr``: 将数据复制到缓冲区中 (或 ``buf[start:end]
  = newstr``)

- ``len(buf)``， ``buf[index]``， ``buf[index] = newchar``: 作为一系列字符访问。

``ffi.buffer(cdata)`` 返回的缓冲区对象使
``cdata`` 对象保持活动状态: 如果它最初是一个拥有的cdata，那么只要缓冲区处于活动状态，它的拥有内存就不会被释放。

Python 2/3兼容性说明: 你应该避免使用 ``str(buf)``，
因为它在Python 2和Python 3之间产生不一致的结果。
(这类似于 ``str()`` 在常规字节字符串上给出不一致的结果)。 请改用 ``buf[:]``。

*版本1.10中的新功能:* ``ffi.buffer`` 此时是返回的缓冲区对象的类型; ``ffi.buffer()`` 实际上调用了构造函数。

**ffi.from_buffer([cdecl,] python_buffer, require_writable=False)**:
返回一个cdata数组 (默认情况下为 ``<cdata 'char[]'>``)，指向给定Python对象的数据，该对象必须支持缓冲区接口。 请注意， ``ffi.from_buffer()`` 将通用Python缓冲区对象转换为cdata对象，而 ``ffi.buffer()`` 执行相反的转换。 两个调用实际上都不会复制任何数据。

``ffi.from_buffer()`` 用于包含大量原始数据的对象，如 字节数组(bytearrays)
或 ``array.array`` 或 numpy数组。 它支持旧的 *缓冲区* API (在Python 2.x中) 和新的 *memoryview* API。 请注意，如果传递只读缓冲区对象，则仍会获得常规 ``<cdata 'char[]'>``; 如果原始缓冲区不希望您这样做，那么您有责任不在那里写。
*特别是，永远不要修改字节串！*

只要 ``ffi.from_buffer()`` 返回的cdata对象处于活动状态，原始对象就会保持活动状态 (并且在内存视图的情况下被锁定)。

一个常见的用例是调用一个带有一些  的c函数，该 ``char *`` 指向一个python对象的内部缓冲区; 对于这种情况，您可以直接将 ``ffi.from_buffer(python_buffer)`` 作为参数传递给调用。

*版本1.10中的新功能:* ``python_buffer`` 可以是支持buffer/memoryview接口的任何东西 (unicode字符串除外)。 以前，1.7版本版本支持bytearray对象 (小心，如果你调整bytearray的大小 ``<cdata>`` 对象将指向释放的内存); 版本1.8及以上版本支持字节字符串。

*版本1.12中的新功能*: 添加了可选的第 *一个* 参数 ``cdecl`` 和关键字参数 ``require_writable``:

* ``cdecl`` 默认为 ``"char[]"``，但是可以为结果指定不同的数组或（从1.13版本开始）指针类型。 像 ``"int[]"`` 这样的值将返回一个整数数组而不是字符， 其长度将设置为适合缓冲区的整数数。 (如果划分不准确，则向下舍入)。 像 ``"int[42]"`` 或 ``"int[2][3]"`` 这样的值将返回一个正好为42(相应的2乘3)整数的数组， 如果缓冲区太小则会引发ValueError。 指定 ``"int[]"`` 和使用旧代码 ``p1 =
  ffi.from_buffer(x); p2 = ffi.cast("int *", p1)`` 间的区别在于， 只要 ``p2`` 在使用，旧代码就需要保持 ``p1`` 活动， 因为只有 ``p1`` 保持底层python对象活动和锁定。 (另外，
  ``ffi.from_buffer("int[]", x)`` 提供了更好的数组绑定检查。)

  *版本1.13中的新功能:* ``cdecl`` 可以是指针类型。  果它指向一个结构或联合，则可以像往常一样编写 ``p.field`` 而不是 ``p[0].field``。  您也可以访问 ``p[n]``; n请注意，在这种情况下，CFFI不会执行任何边界检查。 还要注意， ``p[0]`` 不能用于保持缓冲区活动 (不像 ``ffi.new()``)。


* 如果 ``require_writable`` 设置为True，则如果从 ``python_buffer`` 得的缓冲区是只读的 (例如，如果 ``python_buffer`` 是字节字符串)。 则函数将失败。确切的异常是由对象本身引发的，对于字节这样的东西，它随Python版本而变化，所以不要依赖它。 (在版本1.12之前，使用修改可以实现相同的效果:
  调用 ``ffi.memmove(python_buffer, b"", 0)``。 如果对象是可写的，这没有效果，但如果它是只读的，则会失败。)  请记住，CFFI没有实现C关键字 ``const``: 即使您将 ``require_writable`` 显式设置为False，您仍然会得到常规的读写cdata指针。

*版本1.12中的新功能:* 另见 ``ffi.release()``。


ffi.memmove()
+++++++++++++

**ffi.memmove(dest, src, n)**: 将 ``n`` 个字节从内存区域
``src`` 复制到内存区域 ``dest``。 见下面的例子。 受C函数 ``memcpy()`` 和 ``memmove()`` 的启发————就像后者一样，这些区域可以重叠。  ``dest`` 和 ``src`` 中的每一个都可以是cdata指针，也可以是支持buffer/memoryview接口的python对象。
在 ``dest`` 的情况下，buffer/memoryview 必须是可写的。
*版本1.3中的新功能。*  例:

* ``ffi.memmove(myptr, b"hello", 5)`` 将
  ``b"hello"`` 的5个字节复制到 ``myptr`` 指向的区域。

* ``ba = bytearray(100); ffi.memmove(ba, myptr, 100)`` 将100个字节从 ``myptr`` 复制到bytearray ``ba`` 中。

* ``ffi.memmove(myptr + 1, myptr, 100)`` 将100个字节从 ``myptr`` 的内存移到 ``myptr + 1`` 的内存。

在1.10之前的版本中，``ffi.from_buffer()`` 对缓冲区的类型有限制，这使得 ``ffi.memmove()`` 更加通用。

.. _ffi-typeof:
.. _ffi-sizeof:
.. _ffi-alignof:

ffi.typeof(), ffi.sizeof(), ffi.alignof()
+++++++++++++++++++++++++++++++++++++++++

**ffi.typeof("C type" or cdata object)**: 返回与解析后的字符串对应的
``<ctype>`` 类型的对象，或者返回cdata实例的C语言类型。通常，您不需要调用此函数或在代码中显式操作 ``<ctype>`` 对象: 任何接受C类型的地方都可以接收字符串或预先解析的  ``ctype``
对象 (由于字符串的缓存，因此没有真正的性能差异)。 它在编写类型检查时仍然有用，
例如:

.. code-block:: python
  
    def myfunction(ptr):
        assert ffi.typeof(ptr) is ffi.typeof("foo_t*")
        ...

还要注意，从字符串 ``"foo_t*"`` 到
``<ctype>`` 对象的映射存储在一些内部字典中。 这样可以确保只有一个 ``<ctype 'foo_t *'>`` 对象，因此您可以使用  ``is`` 运算符来比较它。 缺点是字典项目现在是唯一的。 将来，我们可能会添加易懂的改造旧未使用的旧条目。 同时，请注意，如果使用许多不同长度的字符串(如 ``"int[%d]" % length`` )来命名类型，则会创建许多不唯一的缓存项。

**ffi.sizeof("C type" or cdata object)**: 以字节为单位返回参数的大小。 参数可以是C类型，也可以是cdata对象，就像C语言中等效的 ``sizeof`` 算符一样。

对于 ``array = ffi.new("T[]", n)``，然后 ``ffi.sizeof(array)`` 返回
``n * ffi.sizeof("T")``.  *版本1.9中的新功能:* 类似的规则适用于末尾具有可变大小数组的结构。更准确地说，如果
``p`` 由 ``ffi.new("struct foo *", ...)`` 返回，则
``ffi.sizeof(p[0])`` 此时返回总分配大小。 在以前的版本中，它只用于返回 ``ffi.sizeof(ffi.typeof(p[0]))``，这是忽略可变大小部分的结构的大小。 (请注意，由于对齐， ``ffi.sizeof(p[0])`` 可能返回小于 ``ffi.sizeof(ffi.typeof(p[0]))`` 的值。)

**ffi.alignof("C type")**: 返回参数的自然对齐大小(以字节为单位)。 对应于GCC中的 ``__alignof__`` 运算符。


.. _ffi-offsetof:
.. _ffi-addressof:

ffi.offsetof(), ffi.addressof()
+++++++++++++++++++++++++++++++

**ffi.offsetof("C struct or array type", \*fields_or_indexes)**: 返回给定字段结构中的偏移量。对应于C语言中的 ``offsetof()``。

在嵌套结构的情况下，您可以给出几个字段名称。 在指针或数组类型的情况下，您还可以提供与数组项对应的数值。 例如， ``ffi.offsetof("int[5]", 2)``
等于两个整数的大小，也是如此。 ``ffi.offsetof("int *", 2)``。


**ffi.addressof(cdata, \*fields_or_indexes)**: 相当于C语言中的 '&'运算符:

1. ``ffi.addressof(<cdata 'struct-or-union'>)`` 返回一个cdata，它是指向此结构或联合的指针。 返回的指针只有是原始的 ``cdata`` 对象才有效;如果它是直接从 ``ffi.new()`` 获得的，请确保它保持活动状态。

2. ``ffi.addressof(<cdata>, field-or-index...)`` 返回给定结构或数组中的字段或数组项的地址。 对于嵌套结构或数组，您可以提供多个字段或索引以递归查看。 注意，``ffi.addressof(array, index)``
也可以表示为 ``array + index``: 在CFFI和C中都是如此，其中 ``&array[index]`` 只是 ``array + index``。

3. ``ffi.addressof(<library>, "name")`` 从给定的库对象返回指定函数或全局变量的地址。
对于函数，它返回一个包含指向函数的指针的常规cdata对象。

请注意，案例1. 不能用于获取原始或指针的地址，而只能用于获取结构或联合。
实现起来很困难，因为只有结构和联合在内部存储为数据的间接指针。 如果你需要一个可以获取地址的C语言int，首先使用 ``ffi.new("int[1]")``; 同样，对于指针，使用 ``ffi.new("foo_t *[1]")``。


.. _ffi-cdata:
.. _ffi-ctype:

ffi.CData, ffi.CType
++++++++++++++++++++

**ffi.CData, ffi.CType**: 在本文档的其余部分中称为 ``<cdata>`` 和 ``<ctype>`` 的对象的Python类型。请注意，某些cdata对象实际上可能是 ``ffi.CData`` 的子类， 并且与ctype类似， 因此您应该检查 ``if isinstance(x, ffi.CData)``。  此外， ``<ctype>`` 对象具有许多内建属性: ``kind`` 和 ``cname`` 总是存在，根据它们的类型，它们也可能有
``item``, ``length``, ``fields``, ``args``, ``result``, ``ellipsis``,
``abi``, ``elements`` 和 ``relements``。

*版本1.10中的新功能:* ``ffi.buffer`` 现在也是 `一种类型`__。

.. __: #ffi-buffer


.. _ffi-gc:

ffi.gc()
++++++++

**ffi.gc(cdata, destructor, size=0)**:
返回指向相同数据的新cdata对象。 稍后，当这个新的cdata对象被垃圾收集时，将调用
``destructor(old_cdata_object)``。  用法示例:
``ptr = ffi.gc(lib.custom_malloc(42), lib.custom_free)``.
请注意， ``ffi.new()`` 返回类似的对象，返回的指针对象具有所有权，这意味着只要这个确切的返回对象被垃圾收集，就会调用析构函数。

*版本1.12中的新功能:* 另见 ``ffi.release()``。

**ffi.gc(ptr, None, size=0)**:
删除对常规调用 ``ffi.gc`` 返回的对象的所有权，并且在垃圾收集时不会调用析构函数。 该对象在本地修改，并且该函数返回 ``None``。  *版本1.7中的新功能: ffi.gc(ptr, None)*

请注意，对于有限的资源应该避免使用 ``ffi.gc()``，或者 (cffi低于1.11) 用于大内存分配。  在PyPy上尤其如此: 它的GC不知道返回的 ``ptr`` 有多少内存或多少资源。 只有在分配了足够的内存时，它才会运行GC (因此可能比你预期的更晚地运行析构函数)。 而且，析构函数在PyPy当时的任何线程中被调用，这对于某些C语言库来说可能是一个问题。 在这些情况下，请考虑使用自定义 ``__enter__()`` 和 ``__exit__()`` 方法编写包装类，在已知时间点分配和释放C语言数据，并在 ``with``
语句中使用它。 在cffi 1.12中，另见 ``ffi.release()``。

*版本1.11中的新功能:* ``size`` 参数。 如果给定，这应该是 ``ptr`` 保持活动的大小(以字节为单位)的估计值。 该信息被传递给垃圾收集器，解决了上述问题的一部分。 ``size`` 参数在PyPy上最为重要; 在CPython上，到目前为止它被忽略了，但是将来它也可以用来更友好地触发循环引用GC (参见 CPython
`问题 31105`__)。

可以使用负 ``size`` 调用 ``ffi.gc(ptr, None, size=0)``，以撤销估量。 但这不是强制性的:
如果大小估计不匹配，则不会有任何不同步。 它只会使得下一次GC开始或多或少提前开始。

请注意，如果您有多个 ``ffi.gc()`` 对象，则将以随机顺序调用相应的析构函数。 如果您需要特定顺序，参见 `问题 340`__ 的讨论。

.. __: http://bugs.python.org/issue31105
.. __: https://foss.heptapod.net/pypy/cffi/-/issues/340

.. _ffi-new-handle:
.. _ffi-from-handle:

ffi.new_handle(), ffi.from_handle()
+++++++++++++++++++++++++++++++++++

**ffi.new_handle(python_object)**: 返回 ``void *`` 类型的非NULL cdata，其中包含对 ``python_object`` 的不透明引用。  您可以将其传递给C函数或将其存储到C语言结构中。 稍后，您可以使用 **ffi.from_handle(p)** 从具有相同 ``void *`` 指针的值中检索原始 ``python_object``。
*调用 ffi.from_handle(p) 无效，如果 new_handle() 返回的cdata对象未保持活动状态，则可能会崩溃!*

请参阅下面的 `典型用法示例`_。

(如果你想知道，这个 ``void *`` 是不是 ``PyObject *``
指针。 无论如何，这对PyPy没有意义。)

``ffi.new_handle()/from_handle()`` 函数在 *概念* 上的工作方式如下:

* ``new_handle()`` 返回包含Python对象引用的cdata对象; 我们将它们统称为"句柄"cdata对象。 这些句柄cdata对象中的 ``void *`` 值是随机的但是唯一的。

* ``from_handle(p)`` 搜索所有实时"句柄"cdata对象，以获得与其 ``void *`` 值具有相同值 ``p`` 的对象。 然后它返回该句柄cdata对象引用的Python对象。 如果没有找到，则会出现"未定义的行为" (即崩溃)。

"句柄"cdata对象使Python对象保持活动状态，类似于 ``ffi.new()`` 返回一个使一块内存保持活动状态的cdata对象。 如果句柄cdata对象本身不再存在，则关联 ``void * -> python_object`` 将失效，而
``from_handle()`` 将崩溃。

*版本1.4中的新功能:* 对 ``new_handle(x)`` 的两次调用保证返回具有不同 ``void *`` 值的cdata对象，即使使用相同的 ``x`` 也是如此。 这是一个有用的功能，可以避免以下技巧中出现意外重复的问题: 如果你需要保持“句柄”，直到明确要求释放它，但没有一个自然的Python端附加它，那么最简单的是将它 ``add()`` 到一个全局集合。 稍后可以通过
``global_set.discard(p)`` 稍后可以通过 ``p`` 为任何cdata对象，其 ``void *``
值比较相等。

.. _`典型用法示例`:

用法示例: 假设你有一个C语言库，你必须调用一个
``lib.process_document()`` 函数来调用一些回调。 ``process_document()`` 函数接收指向回调和 ``void *`` 参数的指针。 然后使用等于提供值的 ``void
*data`` 参数调用回调。 在这种典型情况下，您可以像这样实现它 (out-of-line API 模式)::

    class MyDocument:
        ...

        def process(self):
            h = ffi.new_handle(self)
            lib.process_document(lib.my_callback,   # the callback
                                 h,                 # 'void *data'
                                 args...)
            # 'h' stays alive until here, which means that the
            # ffi.from_handle() done in my_callback() during
            # the call to process_document() is safe

        def callback(self, arg1, arg2):
            ...

    # the actual callback is this one-liner global function:
    @ffi.def_extern()
    def my_callback(arg1, arg2, data):
        return ffi.from_handle(data).callback(arg1, arg2)


.. _ffi-dlopen:
.. _ffi-dlclose:

ffi.dlopen(), ffi.dlclose()
+++++++++++++++++++++++++++

**ffi.dlopen(libpath, [flags])**: 打开并将"句柄"作为 ``<lib>`` 对象返回到动态库。 参见 `准备和分发模块`_。

**ffi.dlclose(lib)**:显式关闭 ``ffi.dlopen()`` 返回的 ``<lib>`` 对象。

**ffi.RLTD_...**: 常量: ``ffi.dlopen()`` 的标志。


ffi.new_allocator()
+++++++++++++++++++

**ffi.new_allocator(alloc=None, free=None, should_clear_after_alloc=True)**:
返回一个新的分配器。 是一个可调用的，其行为类似于
``ffi.new()`` ，但使用提供的低级 ``alloc`` 和 ``free``
函数。 *版本1.2中的新功能。*

``alloc()`` 是以size作为唯一参数调用的。如果返回空值，则引发MemoryError。 稍后，如果 ``free`` 不是None，则将使用 ``alloc()`` 的结果作为参数调用它。  两者都可以是Python函数，也可以直接是C语言函数。 如果只有 ``free`` 是None，则不调用释放函数。 如果 ``alloc`` 和 ``free`` 都为None，则使用默认的alloc/free组合。 (换句话说，调用 ``ffi.new(*args)`` 等同于 ``ffi.new_allocator()(*args)``。)

如果 ``should_clear_after_alloc`` 设置为False，则假定 ``alloc()`` 返回的内存已被清除 (或者你对内存垃圾没问题); 否则CFFI会清除它。 例: 为了提高性能，如果使用 ``ffi.new()`` 来分配大内存块，使初始内容保持未初始化状态，则可以执行以下操作::

    # at module level
    new_nonzero = ffi.new_allocator(should_clear_after_alloc=False)

    # then replace `p = ffi.new("char[]", bigsize)` with:
        p = new_nonzero("char[]", bigsize)

**注意:** 以下是一般性警告，特别适用于
(但不仅限于) PyPy 5.6或更早版本 (PyPy > 5.6 尝试说明 ``ffi.new()`` 或自定义分配器返回的内存; CPython 使用引用计数)。 如果您进行了大量的分配，那么就无法保证何时释放内存。  如果要确保内存被及时释放 (例如，在分配更多内存之前)，则应同时避免 ``new()`` 和 ``new_allocator()()`` 。

另一种方法是声明并调用C语言 ``malloc()`` 和 ``free()``
函数，或者像 ``mmap()`` 和 ``munmap()`` 这样的变体。 然后，您可以精确地控制分配和释放内存的时间。 例如,
将这两行添加到现有的 ``ffibuilder.cdef()``::

    void *malloc(size_t size);
    void free(void *ptr);

然后手动调用这两个函数::

    p = lib.malloc(n * ffi.sizeof("int"))
    try:
        my_array = ffi.cast("int *", p)
        ...
    finally:
        lib.free(p)

在cffi版本1.12中，您确实可以使用 ``ffi.new_allocator()`` 但是使用
``with`` 语句 (请参阅 ``ffi.release()``) 来强制在已知点调用释放函数。 以上相当于此代码::

    my_new = ffi.new_allocator(lib.malloc, lib.free)  # at global level
    ...
    with my_new("int[]", n) as my_array:
        ...

**Warning:** 由于存在错误， ``p = ffi.new_allocator(..)("struct-or-union *")``
可能不遵循 ``p`` 或 ``p[0]`` 使内存保持活动的规则， 该规则适用于普通的 ``ffi.new("struct-or-union *")`` 分配器。
在某些情况下，如果仅引用 ``p[0]``，则会释放内存。  原因是该规则不适用于 ``ffi.gc()``， 有时会在
``ffi.new_allocator()()`` 的实现中使用; 这可能会在将来的版本中修复。


.. _ffi-release:

ffi.release() and the context manager
+++++++++++++++++++++++++++++++++++++

**ffi.release(cdata)**: 从 ``ffi.new()``，``ffi.gc()``，``ffi.from_buffer()`` 或
``ffi.new_allocator()()`` 释放cdata对象持有的资源。 之后不得使用cdata对象。
cdata对象的普通Python析构函数释放相同的资源，但这允许在已知的时间释放，而不是在将来的某个未指定的点释放。
*版本1.12中的新功能。*

``ffi.release(cdata)`` 相当于 ``cdata.__exit__()``，这意味着您可以使用 ``with`` 语句来确保在块末尾释放cdata。 (在版本1.12及以上)::

    with ffi.from_buffer(...) as p:
        do something with p

效果更为精确，如下所示:

* 对于从 ``ffi.gc(destructor)`` 返回的对象，``ffi.release()`` 将导致立即调用 ``destructor``。

* 在自定义分配器返回的对象上，立即调用自定义自由函数。

* 在CPython上, ``ffi.from_buffer(buf)`` 锁定缓冲区，因此可以使用 ``ffi.release()`` 在已知时间解锁它。 在PyPy上，没有锁定 (到目前为止); ``ffi.release()`` 的效果仅限于删除链接，即使cdata对象保持活动状态，也允许对原始缓冲区对象进行垃圾回收。

* 在CPython上，这个方法对 ``ffi.new()`` 返回的对象没有影响(到目前为止)，因为内存是与cdata对象内联分配的，不能独立释放。 可能会在将来的cffi版本中修复它。

* 在PyPy上, ``ffi.release()`` 立即释放 ``ffi.new()`` 内存。它很有用，因为否则内存将保持活动状态，直到下一次GC发生。
  如果使用 ``ffi.new()`` 分配大量内存并且不使用 ``ffi.release()`` 分配大量内存并且不使用，PyPy (>= 5.7) 会更频繁地运行其GC以进行补偿，因此分配的总内存应保持在边界内无论如何; 但是显式调用 ``ffi.release()`` 应该通过降低GC运行的频率来提高性能。

在 ``ffi.release(x)`` 之后，不要再使用 ``x`` 指向的任何内容。作为此规则的一个例外，您可以为完全相同的cdata对象x多次调用 ``ffi.release(x)``; 第一个之后的调用被忽略。


ffi.init_once()
+++++++++++++++

**ffi.init_once(function, tag)**: 运行 ``function()`` 一次。 ``tag`` 应该是标识函数的原始对象，如字符串: ``function()`` 仅在我们第一次看到 ``tag`` 时调用。 ``function()`` 的返回值将被当前和所有将来的 ``init_once()`` 用相同的标记记住并返回。 如果从多个线程并行调用 ``init_once()`` 则所有调用都会阻塞，直到执行 ``function()`` 为止。 如果
``function()`` 引发异常，则会传播它，并且不会缓存任何内容 (即 如果我们捕获异常并再次尝试 ``init_once()``，将再次调用 ``function()``。).  *版本1.4中的新功能。*

例::

    from _xyz_cffi import ffi, lib

    def initlib():
        lib.init_my_library()

    def make_new_foo():
        ffi.init_once(initlib, "init")
        return lib.make_foo()

如果已经调用了 ``function()``，则 ``init_once()`` 被优化为非常快速地运行。 (在PyPy上，成本为零————JIT通常会删除它生成的机器代码中的所有内容。)

*注意:* ``init_once()`` 的一个 动机__ 是嵌入式案例中
"subinterpreters" 的CPython概念。  如果使用的是
out-of-line API 模式，即使存在多个子解释器，也只调用一次 ``function()``，并且所有子解释器之间共享其返回值。 目标是模仿传统的cpython C扩展模块的init代码总共只执行一次，即使有子解释器。在上面的示例中，C函数 ``init_my_library()`` 总共调用一次，而不是每个子解释器调用一次。 因此，避免
``function()`` 中的python级副作用。 (因为它们只会应用于第一个子解释器中运行); 相反，返回一个值，如下例所示::

   def init_get_max():
       return lib.initialize_once_and_get_some_maximum_number()

   def process(i):
       if i > ffi.init_once(init_get_max, "max"):
           raise IndexError("index too large!")
       ...

.. __: https://foss.heptapod.net/pypy/cffi/-/issues/233

.. _ffi-getctype:
.. _ffi-list-types:

ffi.getctype(), ffi.list_types()
++++++++++++++++++++++++++++++++

**ffi.getctype("C type" or <ctype>, extra="")**: 返回给定C类型的字符串表示形式。 如果非空，则追加"额外"字符串 (或插入更复杂的情况下的正确位置);它可以是要声明的变量的名称，也可以是类型的额外部分，如
like ``"*"`` 或 ``"[5]"``。 例如
``ffi.getctype(ffi.typeof(x), "*")`` 返回C类型"指向与x相同类型的指针"的字符串表示形式; 并且
``ffi.getctype("char[80]", "a") == "char a[80]"``。

**ffi.list_types()**: 返回此FFI实例已知的用户类型名称。这将返回一个包含三个名称列表的元组:
``(typedef_names, names_of_structs, names_of_unions)``。 *版本1.6中的新功能。*


.. _`准备和分发模块`: cdef.html#loading-libraries


转换
-----------

本节介绍了在 *写入* C数据结构(或将参数传递给函数调用)以及从C语言数据结构中 *读取* (或获取函数调用的结果)时允许的所有转换。最后一列给出了允许的特定于类型的操作。

+---------------+-----------------------------------------------+-------------------------------------+--------------------------------+
|    C语言类型  |           写入                                |        读取                         |    其他操作                    |
+===============+===============================================+=====================================+================================+
| 整形和枚举    | 一个整数或int()返回的任何东西                 | Python int或long，具体取决于类型    | int(), bool()                  |
| `[5]`         | (但不是浮点数!)。 必须在范围内。              | (版本1.10:或者bool)                 | `[6]`,                         |
|               |                                               |                                     | ``<``                          |
+---------------+-----------------------------------------------+-------------------------------------+--------------------------------+
|   ``char``    | 一个长度为1或类似<cdata char>的字符串         | 长度为1的字符串                     | int(), bool(),                 |
|               |                                               |                                     | ``<``                          |
+---------------+-----------------------------------------------+-------------------------------------+--------------------------------+
| ``wchar_t``,  | 一个长度为1的unicode                          | 一个长度为1的unicode                |                                |
| ``char16_t``, | (如果是代理(码元)，则可能是2个)               | (如果是代理(码元)，则可能是2)       | int(),                         |
| ``char32_t``  | 或其他类似的<cdata>                           |                                     | bool(), ``<``                  |
| `[8]`         |                                               |                                     |                                |
+---------------+-----------------------------------------------+-------------------------------------+--------------------------------+
|  ``float``,   | 浮点数或float()返回的任何东西                 | 一个Python浮点数                    | float(), int(),                |
|  ``double``   |                                               |                                     | bool(), ``<``                  |
+---------------+-----------------------------------------------+-------------------------------------+--------------------------------+
|``long double``| 类似带有 ``long double`` 的<cdata>，          | 一个<cdata>，以避免失去精度 `[3]`   | float(), int(),                |
|               | 或者float()返回的任何东西                     |                                     | bool()                         |
|               |                                               |                                     |                                |
|               |                                               |                                     |                                |
+---------------+-----------------------------------------------+-------------------------------------+--------------------------------+
| ``float``     | 一个复数或任何complex()返回的任何东西         | 一个Python复数                      | complex(),                     |
| ``_Complex``, |                                               |                                     | bool()                         |
| ``double``    |                                               |                                     | `[7]`                          |
| ``_Complex``  |                                               |                                     |                                |
+---------------+-----------------------------------------------+-------------------------------------+--------------------------------+
|  指针         | 类似兼容类型的<cdata>                         |  一个<cdata>                        |``[]`` `[4]`,                   |
|               | (即相同类型或 ``void*``，或作为数组 `[1]`     |                                     |``+``, ``-``,                   |
|               |                                               |                                     |bool()                          |
|               |                                               |                                     |                                |
|               |                                               |                                     |                                |
+---------------+-----------------------------------------------+                                     |                                |
|  ``void *``   | 类似带有任何指针或数组类型的<cdata>           |                                     |                                |
|               |                                               |                                     |                                |
|               |                                               |                                     |                                |
+---------------+-----------------------------------------------+                                     +--------------------------------+
| 指向结构体    | 与指针相同                                    |                                     | ``[]``, ``+``,                 |
| 或 联合的指针 |                                               |                                     | ``-``, bool(),                 |
|               |                                               |                                     | 和read/write struct 字段       |
|               |                                               |                                     |                                |
+---------------+-----------------------------------------------+                                     +--------------------------------+
| 函数指针      | 与指针相同                                    |                                     | bool(),                        |
|               |                                               |                                     | call `[2]`                     |
+---------------+-----------------------------------------------+-------------------------------------+--------------------------------+
|  数组         | 列表或元组的元素                              | 一个<cdata>                         |len(), iter(),                  |
|               |                                               |                                     |``[]`` `[4]`,                   |
|               |                                               |                                     |``+``, ``-``                    |
+---------------+-----------------------------------------------+                                     +--------------------------------+
| ``char[]``,   | 与数组或Python字节字符串相同                  |                                     | len(), iter(),                 |
| ``un/signed`` |                                               |                                     | ``[]``, ``+``,                 |
| ``char[]``,   |                                               |                                     | ``-``                          |
| ``_Bool[]``   |                                               |                                     |                                |
+---------------+-----------------------------------------------+                                     +--------------------------------+
|``wchar_t[]``, | 与数组或Python unicode字符串相同              |                                     | len(), iter(),                 |
|``char16_t[]``,|                                               |                                     | ``[]``,                        |
|``char32_t[]`` |                                               |                                     | ``+``, ``-``                   |
|               |                                               |                                     |                                |
+---------------+-----------------------------------------------+-------------------------------------+--------------------------------+
| 结构体        | 字段值的列表或元组或字典，或相同类型的<cdata> | 一个<cdata>                         | read/write 字段                |
|               |                                               |                                     |                                |
|               |                                               |                                     |                                |
|               |                                               |                                     |                                |
+---------------+-----------------------------------------------+                                     +--------------------------------+
| 联合          | 与struct相同，但最多只有一个字段              |                                     | read/write 字段                |
|               |                                               |                                     |                                |
+---------------+-----------------------------------------------+-------------------------------------+--------------------------------+

`[1]` ``item *`` 是函数参数中的 ``item[]`` :

   在函数声明中，根据C标准， ``item *``
   参数与 ``item[]`` 参数相同 (并且 ``ffi.cdef()`` 不记录差异)。 所以当你调用这样一个函数时，你可以传递一个C类型接受的参数，例如将一个Python字符串传递给一个 ``char *`` 参数
   (因为它适用于 ``char[]`` 参数) 或 ``int *`` 参数的整数列表 (它适用于 ``int[]`` 参数)。 请注意，即使您要传递单个 ``item``，也需要在长度为1的列表中指定它; 例如，``struct point_s
   *`` 参数可能会传递为 ``[[x, y]]`` 或 ``[{'x': 5, 'y':
   10}]``。

   作为优化，CFFI假定具有 ``char *`` 参数的函数，您传递Python字符串将不会实际修改传入的字符数组，因此直接传递Python字符串对象内的指针。
   (在PyPy上，这种优化仅在PyPy 5.4和CFFI 1.8之后才可用。)

`[2]` C函数调用在GIL释放时完成。

   请注意，我们假设被调用的函数不使用Python.h中的Python API。例如，我们之后不会检查它们是否设置了Python异常。您可以解决它，但不建议将CFFI与 ``Python.h`` 混合使用。 (如果您这样做，在PyPy和Windows等某些平台上，您可能需要显式链接到 ``libpypy-c.dll`` 才能访问CPython C API兼容层;实际上，PyPy上的CFFI生成的模块本身并没有链接到
   ``libpypy-c.dll`` 。但实际上，首先不要这样做。)

`[3]` ``long double`` 的支持:

   我们在cdata对象中保留 ``long double`` 值以避免丢失精度。 普通Python浮点数只包含 ``double`` 的精度。 如果你真的想将这样的对象转换为常规的Python float (即 C
   ``double``)， 请调用 ``float()``。 如果你需要对这些数字进行算术而没有任何精度损失，你需要定义和使用一系列C函数，如 ``long double add(long double
   a, long double b);``。

`[4]` 切片 ``x[start:stop]``:

   只要你明确指定 ``start`` 和 ``stop``，就允许切片 (并且不给任何 ``step``)。  它给出了一个"view"cdata对象，它是从 ``start`` 到 ``stop`` 的所有元素(数据项)。
   它是数组类型的cdata (所以例如将它作为参数传递给C函数只会将其转换为指向 ``start`` 元素的指针).
   与索引一样，负边界意味着真正的负索引，如在C中。 至于切片赋值，它接受任何可迭代的，包括元素列表或另一个类似数组的cdata对象，但长度必须匹配。
   (请注意，此行为与初始化不同: 例如 你可以这样 ``chararray[10:15] = "hello"``，但是指定的字符串必须是正确的长度; 没有添加隐式空字符。)

`[5]` 枚举像int一样处理:

   与C一样，枚举类型主要是int类型 (unsigned 或 signed， int 或
   long; 请注意，GCC的首选是unsigned)。 例如，读取结构的枚举字段会返回一个整数。 要象征性地比较它们的值，请使用 ``if x.field ==
   lib.FOO`` 之类的代码。 如果你真的想要将它们的值作为字符串，请使用
   ``ffi.string(ffi.cast("the_enum_type", x.field))``。

`[6]` 原始cdata上的bool() :

   *版本1.7中的新功能。*  在以前的版本中，它只适用于指针; 对于原语，它总是返回True。

   *N版本1.10中的新功能:*  C语言类型 ``_Bool`` 或 ``bool`` 现在转换为Python布尔值。 如果 C ``_Bool`` 碰巧包含不同于0和1的值，则会出现异常 (这种情况在C中触发未定义的行为;如果你真的必须与依赖于它的库接口，不要在CFFI端使用 ``_Bool``)。
   此外，从字节字符串转换为 ``_Bool[]`` 时，只接受字节 ``\x00`` 和 ``\x01``。

`[7]` libffi不支持复数:

   *版本1.11中的新功能:* CFFI现在直接支持复数。
   但请注意，libffi没有。 这意味着CFFI无法调用直接作为参数类型或返回complex类型的C函数，除非它们直接使用API​​模式。

`[8]` ``wchar_t``, ``char16_t`` 和 ``char32_t``

   请参阅下面的 `Unicode字符类型`_。


.. _file:

支持文件
++++++++++++++++

您可以使用 ``FILE *`` 参数声明C函数，并使用Python文件对象调用它们。 如果需要，你也可以这样做 ``c_f
= ffi.cast("FILE *", fileobj)`` 然后传递 ``c_f``。

但请注意，CFFI通过尽力而为的方法来做到这一点。 如果您需要更好地控制缓冲，刷新和及时关闭
``FILE *``，那么您不应该对 ``FILE *`` 使用此特殊支持。
相反，您可以使用fdopen()处理您明确使用的常规 ``FILE *`` cdata对象，如下所示:

.. code-block:: python

    ffi.cdef('''
        FILE *fdopen(int, const char *);   // from the C <stdio.h>
        int fclose(FILE *);
    ''')

    myfile.flush()                    # make sure the file is flushed
    newfd = os.dup(myfile.fileno())   # make a copy of the file descriptor
    fp = lib.fdopen(newfd, "w")       # make a cdata 'FILE *' around newfd
    lib.write_stuff_to_file(fp)       # invoke the external function
    lib.fclose(fp)                    # when you're done, close fp (and newfd)

无论如何，对 ``FILE *`` 的特殊支持在CPython 3.x和PyPy上以类似的方式实现，因为这些Python实现的文件本身不是基于 ``FILE *``。 这样做显式地提供了更多的控制。


.. _unichar:

Unicode字符类型
+++++++++++++++++++++++

``wchar_t`` 类型与底层平台具有相同的签名。 例如， 在Linux上，它是一个带符号的32位整数。
但是， ``char16_t`` 和 ``char32_t`` 类型 (*版本1.11中的新功能*)
始终是无符号的。

请注意，CFFI假定这些类型在本机字节序中包含UTF-16或UTF-32字符。更确切地说:

* 假设 ``char32_t`` 包含UTF-32或UCS4，它只是unicode码位;

* 假设 ``char16_t`` 包含UTF-16，即UCS2加代理(码元);

* 假设 ``wchar_t`` 包含UTF-32或UTF-16，基于其实际平台定义的大小为4或2个字节。

这一假设是真是假，C语言没有说明。
理论上，您正在链接的C语言库可以使用其中一种具有不同含义的类型。 然后，您需要自己处理它， 例如，在 ``cdef()`` 使用 ``uint32_t`` 而不是 ``char32_t`` ，并手动构建预期的 ``uint32_t`` 数组。

Python本身可以使用 ``sys.maxunicode == 65535`` 或
``sys.maxunicode == 1114111`` (Python >= 3.3 始终是 1114111)。 这改变了代理的处理方式 (这是一对16位"字符"，实际上代表一个值大于65535的码位)。 如果您的Python是 ``sys.maxunicode == 1114111``，那么它可以存储任意unicode码位; 从Python unicodes转换为UTF-16时会自动插入代理，并在转换回时自动删除。 另一方面，如果你的Python是 ``sys.maxunicode == 65535``，那么它就是另一种方式: 从Python unicodes转换为UTF-32时会删除代理，并在转换回时添加。 换句话说，仅在存在大小不匹配时才进行代理转换。

请注意，未指定Python的内部表示。 例如， 在CPython >= 3.3， 它将使用1或2或4字节数组，具体取决于字符串实际包含的内容。 使用CFFI，当您将Python字节字符串传递给期望 ``char*`` 的C函数时，我们直接传递指向现有数据的指针，而无需临时缓冲区; 但是，由于内部表示的变化，使用unicode字符串参数和 ``wchar_t*`` / ``char16_t*`` /
``char32_t*`` 类型无法完成相同的操作。因此，为了保持一致性，CFFI总是为unicode字符串分配一个临时缓冲区。

**警告:** 现在，如果你将 ``char16_t`` 和 ``char32_t`` 与
``set_source()`` 一起使用，你必须确保自己的类型是由你提供给 ``set_source()`` 的C语言源代码中声明的。 如果 ``#include`` 显式使用它们的库，则会声明它们，例如，使用C++ 11时。 否则，您需要在Linux上使用 ``#include
<uchar.h>``，或者通常使用类似于 ``typedef
uint16_t char16_t;`` 的方法。 这不是由CFFI自动完成的，因为
``uchar.h`` 不是跨平台的标准，如果类型恰好已经定义，写上面的 ``typedef`` 会崩溃。
