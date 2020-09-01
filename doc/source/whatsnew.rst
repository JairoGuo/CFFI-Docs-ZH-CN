======================
有什么新变化
======================


v1.14.1
=======

* CFFI源代码现在托管在 `hosted on Heptapod`_ 上。

* 改进了对 ``typedef int my_array_t[...];`` 的支持，在API模式下有一个显式的点-点-点 (`issue #453`_)

* Windows（32位和64位）：针对返回结构的函数的ABI模式调用的多项修复。

* 在aarch64上对MacOS 11的实验支持。

* 以及其他一些小的更改和错误修复。

.. _`hosted on Heptapod`: https://foss.heptapod.net/pypy/cffi/
.. _`issue #453`: https://foss.heptapod.net/pypy/cffi/issues/453



v1.14
=====

* 现在可以使用已打开的C语言库的句柄(如 ``void *``) 调用 ``ffi.dlopen()`` 。

* 仅CPython: 修复了
  ``lib.myfunc([large list])`` 之类的调用的堆栈溢出问题。  例如，如果函数声明为带
  ``float *`` 参数，则该数组会临时转换为C语言的float数组。 但是，无论大小如何，此临时存储使用 ``alloca()`` 的代码。 现在，此问题已解决。

  该修复程序涉及所有模式: in-line/out-of-line API/ABI.  还要注意，您的API模式C语言扩展模块需要使用cffi 1.14重新生成才能获得修复; 即对于API模式，此修复程序位于生成的C语言源代码中。
  (从cffi 1.14生成的C源代码也可以在具有旧版本cffi的不同环境中运行时工作。 同样，此更改对PyPy而言也没有区别。)

  作为一种适用于所有版本的cffi的解决方法，您可以编写
  ``lib.myfunc(ffi.new("float[]", [large list]))``, 它是等效的，但明确地将中间数组构建为堆上的常规Python对象。

* 修复了CPython 3.x上 ``ffi.getwinerror()`` 内部的内存泄漏。


v1.13.2
=======

* 重新发布，因为Linux wheels附带了libffi的附加版本，该版本非常旧，有缺陷(`issue #432`_).

.. _`issue #432`: https://bitbucket.org/cffi/cffi/issues/432/


v1.13.1
=======

* 弃用在 ``cdef()`` 中声明仅具有
  ``void *foo;`` 的全局变量的方式。  您应该始终使用存储类，例如 ``extern void
  *foo;`` 或 ``static void *foo;``。  这些对于 ``cdef()`` 都是等效的，但是不推荐使用裸版本的原因是(据我所知)在真正的C头文件中总是错误的。

* 修复 ``RuntimeError: found a situation in which we try
  to build a type recursively`` (`issue #429`_).

* 修复了 `issue #427`_ ，其中嵌入逻辑初始化代码中的多线程错误会导致CPython 3.7上死锁。

.. _`issue #429`: https://bitbucket.org/cffi/cffi/issues/429/
.. _`issue #427`: https://bitbucket.org/cffi/cffi/issues/427/


v1.13
=====

* 除了 ``"type[]"`` 现在还支持 ``ffi.from_buffer("type *", ..)``。  然后，您可以编写 ``p.field`` 来访问项目，而不仅仅是 ``p[0].field``。    注意不要执行边界检查，因为
  ``p[n]`` 可能会超出范围访问数据。

* 修复包含未命名位域的结构，像 ``int : 1;``。

* 调用“函数指针”类型的cdata时，如果指针碰巧为NULL，则给出RuntimeError而不是崩溃

* 支持枚举定义中的常量之间的更多二进制操作(PR #96)

* 如果在预处理器行中使用引号，则使错误发出的警告关闭

* 使用 ``ffi.cdef("""struct X { void(*fnptr)(struct X); };""")`` 检测一个极端情况，该情况会将C语言代码进行无限递归


旧版本
==============

v1.12.3
-------

* 修复以var大小数组结尾的嵌套结构类型（＃405）。

* 添加对在 ``ffi.cdef()`` 中整数常量末尾使用 ``U`` 和 ``L`` 符的支持（感谢Guillaume）。

* 更多3.8修复。


v1.12.2
-------

* 添加了临时解决方法以在CPython 3.8.0a2上进行编译。


v1.12.1
-------

* Windows上的CPython 3：
  我们再次默认不再使用 ``Py_LIMITED_API`` 进行编译，因为这些模块 *仍然* 无法与virtualenv一起使用。问题是它在CPython <= 3.4中不起作用，并且由于技术原因，我们无法根据Python的版本自动启用此标志。 

  就像之前一样，`问题 #350`_ 提到了一个解决方法，如果您仍然需要 
  ``Py_LIMITED_API`` 标志，并且您不关心virtualenv，*或* 者您确定您的模块不会在CPython <= 3.4上使用: 将 
  ``define_macros=[("Py_LIMITED_API", None)]`` 作为关键字传递给
  ``ffibuilder.set_source()`` 调用。


v1.12
-------

* `直接支持pkg-config`__.

* ``ffi.from_buffer()`` 接受一个新的可选的第一个参数，该参数给出结果的数组类型。它还需要一个可选的关键字参数
  ``require_writable`` 来拒绝只读Python缓冲区。

* ``ffi.new()``，``ffi.gc()`` 或 ``ffi.from_buffer()`` 现在可以通过使用 ``with``
  关键字或通过调用新的 ``ffi.release()`` 在已知时间释放cdata对象。

* Windows，CPython 3.x: cffi模块再次与 ``python3.dll``
  链接。 这使得它们与CPython版本无关，就像它们在其他平台上一样。 **它需要virtualenv 16.0.0。**

* 如果 ``p`` 本身是另一个cdata ``int[4]``，则接受像 ``ffi.new("int[4]", p)`` 这样的表达式。

* CPython 2.x: ``ffi.dlopen()`` 在Posix上使用非ascii文件名失败

* CPython: 如果一个线程是从C启动然后运行Python代码 (使用回调或嵌入解决方案)，那么以前版本的cffi将包含可能的崩溃和/或内存泄漏。 希望这已得到修复 (参见 `问题 #362`_).

* 支持 ``ffi.cdef(..., pack=N)``，其中N是2的幂。
  在MSVC上模拟 ``#pragma pack(N)`` 的方法。此外，Windows上的默认值现在为 ``pack=8``，就像在MSVC上一样。 这可能会对角落情况产生影响，尽管我在CFFI的背景下无法想到这一点。
  旧方法 ``ffi.cdef(..., packed=True)`` 保持不变，相当于 ``pack=1`` (比如说，像 ``int`` 这样的字段应该对齐到1字节而不是4字节)。

.. __: cdef.html#pkgconfig
.. _`问题 #362`: https://bitbucket.org/cffi/cffi/issues/362/



**注：本文档不对旧版本文档的更新内容进行翻译，如有需要，阅读下面内容或自行翻译**

v1.11.5
-------

* `Issue #357`_: fix ``ffi.emit_python_code()`` which generated a buggy
  Python file if you are using a ``struct`` with an anonymous ``union``
  field or vice-versa.

* Windows: ``ffi.dlopen()`` should now handle unicode filenames.

* ABI mode: implemented ``ffi.dlclose()`` for the in-line case (it used
  to be present only in the out-of-line case).

* Fixed a corner case for ``setup.py install --record=xx --root=yy``
  with an out-of-line ABI module.  Also fixed `Issue #345`_.

* More hacks on Windows for running CFFI's own ``setup.py``.

* `Issue #358`_: in embedding, to protect against (the rare case of)
  Python initialization from several threads in parallel, we have to use
  a spin-lock.  On CPython 3 it is worse because it might spin-lock for
  a long time (execution of ``Py_InitializeEx()``).  Sadly, recent
  changes to CPython make that solution needed on CPython 2 too.

* CPython 3 on Windows: we no longer compile with ``Py_LIMITED_API``
  by default because such modules cannot be used with virtualenv.
  `问题 #350`_ mentions a workaround if you still want that and are not
  concerned about virtualenv: pass ``define_macros=[("Py_LIMITED_API",
  None)]`` as a keyword to the ``ffibuilder.set_source()`` call.

.. _`Issue #345`: https://bitbucket.org/cffi/cffi/issues/345/
.. _`Issue #350`: https://bitbucket.org/cffi/cffi/issues/350/
.. _`Issue #358`: https://bitbucket.org/cffi/cffi/issues/358/
.. _`Issue #357`: https://bitbucket.org/cffi/cffi/issues/357/



v1.11.4
-------

* Windows: reverted linking with ``python3.dll``, because
  virtualenv does not make this DLL available to virtual environments
  for now.  See `Issue #355`_.  On Windows only, the C extension
  modules created by cffi follow for now the standard naming scheme
  ``foo.cp36-win32.pyd``, to make it clear that they are regular
  CPython modules depending on ``python36.dll``.

.. _`Issue #355`: https://bitbucket.org/cffi/cffi/issues/355/


v1.11.3
-------

* Fix on CPython 3.x: reading the attributes ``__loader__`` or
  ``__spec__`` from the cffi-generated lib modules gave a buggy
  SystemError.  (These attributes are always None, and provided only to
  help compatibility with tools that expect them in all modules.)

* More Windows fixes: workaround for MSVC not supporting large
  literal strings in C code (from
  ``ffi.embedding_init_code(large_string)``); and an issue with
  ``Py_LIMITED_API`` linking with ``python35.dll/python36.dll`` instead
  of ``python3.dll``.

* Small documentation improvements.


v1.11.2
-------

* Fix Windows issue with managing the thread-state on CPython 3.0 to 3.5


v1.11.1
-------

* Fix tests, remove deprecated C API usage

* Fix (hack) for 3.6.0/3.6.1/3.6.2 giving incompatible binary extensions
  (cpython issue `#29943`_)

* Fix for 3.7.0a1+

.. _`#29943`: https://bugs.python.org/issue29943


v1.11
-----

* Support the modern standard types ``char16_t`` and ``char32_t``.
  These work like ``wchar_t``: they represent one unicode character, or
  when used as ``charN_t *`` or ``charN_t[]`` they represent a unicode
  string.  The difference with ``wchar_t`` is that they have a known,
  fixed size.  They should work at all places that used to work with
  ``wchar_t`` (please report an issue if I missed something).  Note
  that with ``set_source()``, you need to make sure that these types are
  actually defined by the C source you provide (if used in ``cdef()``).

* Support the C99 types ``float _Complex`` and ``double _Complex``.
  Note that libffi doesn't support them, which means that in the ABI
  mode you still cannot call C functions that take complex numbers
  directly as arguments or return type.

* Fixed a rare race condition when creating multiple ``FFI`` instances
  from multiple threads.  (Note that you aren't meant to create many
  ``FFI`` instances: in inline mode, you should write ``ffi =
  cffi.FFI()`` at module level just after ``import cffi``; and in
  out-of-line mode you don't instantiate ``FFI`` explicitly at all.)

* Windows: using callbacks can be messy because the CFFI internal error
  messages show up to stderr---but stderr goes nowhere in many
  applications.  This makes it particularly hard to get started with the
  embedding mode.  (Once you get started, you can at least use
  ``@ffi.def_extern(onerror=...)`` and send the error logs where it
  makes sense for your application, or record them in log files, and so
  on.)  So what is new in CFFI is that now, on Windows CFFI will try to
  open a non-modal MessageBox (in addition to sending raw messages to
  stderr).  The MessageBox is only visible if the process stays alive:
  typically, console applications that crash close immediately, but that
  is also the situation where stderr should be visible anyway.

* Progress on support for `callbacks in NetBSD`__.

* Functions returning booleans would in some case still return 0 or 1
  instead of False or True.  Fixed.

* `ffi.gc()`__ now takes an optional third parameter, which gives an
  estimate of the size (in bytes) of the object.  So far, this is only
  used by PyPy, to make the next GC occur more quickly (`issue #320`__).
  In the future, this might have an effect on CPython too (provided
  the CPython `issue 31105`__ is addressed).

* Add a note to the documentation: the ABI mode gives function objects
  that are *slower* to call than the API mode does.  For some reason it
  is often thought to be faster.  It is not!

.. __: https://bitbucket.org/cffi/cffi/issues/321/cffi-191-segmentation-fault-during-self
.. __: ref.html#ffi-gc
.. __: https://bitbucket.org/cffi/cffi/issues/320/improve-memory_pressure-management
.. __: http://bugs.python.org/issue31105


v1.10.1
-------

(only released inside PyPy 5.8.0)

* Fixed the line numbers reported in case of ``cdef()`` errors.
  Also, I just noticed, but pycparser always supported the preprocessor
  directive ``# 42 "foo.h"`` to mean "from the next line, we're in file
  foo.h starting from line 42", which it puts in the error messages.


v1.10
-----

* Issue #295: use calloc() directly instead of
  PyObject_Malloc()+memset() to handle ffi.new() with a default
  allocator.  Speeds up ``ffi.new(large-array)`` where most of the time
  you never touch most of the array.

* Some OS/X build fixes ("only with Xcode but without CLT").

* Improve a couple of error messages: when getting mismatched versions
  of cffi and its backend; and when calling functions which cannot be
  called with libffi because an argument is a struct that is "too
  complicated" (and not a struct *pointer*, which always works).

* Add support for some unusual compilers (non-msvc, non-gcc, non-icc,
  non-clang)

* Implemented the remaining cases for ``ffi.from_buffer``.  Now all
  buffer/memoryview objects can be passed.  The one remaining check is
  against passing unicode strings in Python 2.  (They support the buffer
  interface, but that gives the raw bytes behind the UTF16/UCS4 storage,
  which is most of the times not what you expect.  In Python 3 this has
  been fixed and the unicode strings don't support the memoryview
  interface any more.)

* The C type ``_Bool`` or ``bool`` now converts to a Python boolean
  when reading, instead of the content of the byte as an integer.  The
  potential incompatibility here is what occurs if the byte contains a
  value different from 0 and 1.  Previously, it would just return it;
  with this change, CFFI raises an exception in this case.  But this
  case means "undefined behavior" in C; if you really have to interface
  with a library relying on this, don't use ``bool`` in the CFFI side.
  Also, it is still valid to use a byte string as initializer for a
  ``bool[]``, but now it must only contain ``\x00`` or ``\x01``.  As an
  aside, ``ffi.string()`` no longer works on ``bool[]`` (but it never
  made much sense, as this function stops at the first zero).

* ``ffi.buffer`` is now the name of cffi's buffer type, and
  ``ffi.buffer()`` works like before but is the constructor of that type.

* ``ffi.addressof(lib, "name")``  now works also in in-line mode, not
  only in out-of-line mode.  This is useful for taking the address of
  global variables.

* Issue #255: ``cdata`` objects of a primitive type (integers, floats,
  char) are now compared and ordered by value.  For example, ``<cdata
  'int' 42>`` compares equal to ``42`` and ``<cdata 'char' b'A'>``
  compares equal to ``b'A'``.  Unlike C, ``<cdata 'int' -1>`` does not
  compare equal to ``ffi.cast("unsigned int", -1)``: it compares
  smaller, because ``-1 < 4294967295``.

* PyPy: ``ffi.new()`` and ``ffi.new_allocator()()`` did not record
  "memory pressure", causing the GC to run too infrequently if you call
  ``ffi.new()`` very often and/or with large arrays.  Fixed in PyPy 5.7.

* Support in ``ffi.cdef()`` for numeric expressions with ``+`` or
  ``-``.  Assumes that there is no overflow; it should be fixed first
  before we add more general support for arbitrary arithmetic on
  constants.


v1.9
----

* Structs with variable-sized arrays as their last field: now we track
  the length of the array after ``ffi.new()`` is called, just like we
  always tracked the length of ``ffi.new("int[]", 42)``.  This lets us
  detect out-of-range accesses to array items.  This also lets us
  display a better ``repr()``, and have the total size returned by
  ``ffi.sizeof()`` and ``ffi.buffer()``.  Previously both functions
  would return a result based on the size of the declared structure
  type, with an assumed empty array.  (Thanks andrew for starting this
  refactoring.)

* Add support in ``cdef()/set_source()`` for unspecified-length arrays
  in typedefs: ``typedef int foo_t[...];``.  It was already supported
  for global variables or structure fields.

* I turned in v1.8 a warning from ``cffi/model.py`` into an error:
  ``'enum xxx' has no values explicitly defined: refusing to guess which
  integer type it is meant to be (unsigned/signed, int/long)``.  Now I'm
  turning it back to a warning again; it seems that guessing that the
  enum has size ``int`` is a 99%-safe bet.  (But not 100%, so it stays
  as a warning.)

* Fix leaks in the code handling ``FILE *`` arguments.  In CPython 3
  there is a remaining issue that is hard to fix: if you pass a Python
  file object to a ``FILE *`` argument, then ``os.dup()`` is used and
  the new file descriptor is only closed when the GC reclaims the Python
  file object---and not at the earlier time when you call ``close()``,
  which only closes the original file descriptor.  If this is an issue,
  you should avoid this automatic convertion of Python file objects:
  instead, explicitly manipulate file descriptors and call ``fdopen()``
  from C (...via cffi).


v1.8.3
------

* When passing a ``void *`` argument to a function with a different
  pointer type, or vice-versa, the cast occurs automatically, like in C.
  The same occurs for initialization with ``ffi.new()`` and a few other
  places.  However, I thought that ``char *`` had the same
  property---but I was mistaken.  In C you get the usual warning if you
  try to give a ``char *`` to a ``char **`` argument, for example.
  Sorry about the confusion.  This has been fixed in CFFI by giving for
  now a warning, too.  It will turn into an error in a future version.


v1.8.2
------

* Issue #283: fixed ``ffi.new()`` on structures/unions with nested
  anonymous structures/unions, when there is at least one union in
  the mix.  When initialized with a list or a dict, it should now
  behave more closely like the ``{ }`` syntax does in GCC.


v1.8.1
------

* CPython 3.x: experimental: the generated C extension modules now use
  the "limited API", which means that, as a compiled .so/.dll, it should
  work directly on any version of CPython >= 3.2.  The name produced by
  distutils is still version-specific.  To get the version-independent
  name, you can rename it manually to ``NAME.abi3.so``, or use the very
  recent setuptools 26.

* Added ``ffi.compile(debug=...)``, similar to ``python setup.py build
  --debug`` but defaulting to True if we are running a debugging
  version of Python itself.


v1.8
----

* Removed the restriction that ``ffi.from_buffer()`` cannot be used on
  byte strings.  Now you can get a ``char *`` out of a byte string,
  which is valid as long as the string object is kept alive.  (But
  don't use it to *modify* the string object!  If you need this, use
  ``bytearray`` or other official techniques.)

* PyPy 5.4 can now pass a byte string directly to a ``char *``
  argument (in older versions, a copy would be made).  This used to be
  a CPython-only optimization.


v1.7
----

* ``ffi.gc(p, None)`` removes the destructor on an object previously
  created by another call to ``ffi.gc()``

* ``bool(ffi.cast("primitive type", x))`` now returns False if the
  value is zero (including ``-0.0``), and True otherwise.  Previously
  this would only return False for cdata objects of a pointer type when
  the pointer is NULL.

* bytearrays: ``ffi.from_buffer(bytearray-object)`` is now supported.
  (The reason it was not supported was that it was hard to do in PyPy,
  but it works since PyPy 5.3.)  To call a C function with a ``char *``
  argument from a buffer object---now including bytearrays---you write
  ``lib.foo(ffi.from_buffer(x))``.  Additionally, this is now supported:
  ``p[0:length] = bytearray-object``.  The problem with this was that a
  iterating over bytearrays gives *numbers* instead of *characters*.
  (Now it is implemented with just a memcpy, of course, not actually
  iterating over the characters.)

* C++: compiling the generated C code with C++ was supposed to work,
  but failed if you make use the ``bool`` type (because that is rendered
  as the C ``_Bool`` type, which doesn't exist in C++).

* ``help(lib)`` and ``help(lib.myfunc)`` now give useful information,
  as well as ``dir(p)`` where ``p`` is a struct or pointer-to-struct.


v1.6
----

* `ffi.list_types()`_

* `ffi.unpack()`_

* `extern "Python+C"`_

* in API mode, ``lib.foo.__doc__`` contains the C signature now.  On
  CPython you can say ``help(lib.foo)``, but for some reason
  ``help(lib)`` (or ``help(lib.foo)`` on PyPy) is still useless; I
  haven't yet figured out the hacks needed to convince ``pydoc`` to
  show more.  (You can use ``dir(lib)`` but it is not most helpful.)

* Yet another attempt at robustness of ``ffi.def_extern()`` against
  CPython's interpreter shutdown logic.

.. _`ffi.list_types()`: ref.html#ffi-list-types
.. _`ffi.unpack()`: ref.html#ffi-unpack
.. _`extern "Python+C"`: using.html#extern-python-c


v1.5.2
------

* Fix 1.5.1 for Python 2.6.


v1.5.1
------

* A few installation-time tweaks (thanks Stefano!)

* Issue #245: Win32: ``__stdcall`` was never generated for
  ``extern "Python"`` functions

* Issue #246: trying to be more robust against CPython's fragile
  interpreter shutdown logic


v1.5.0
------

* Support for `using CFFI for embedding`__.

.. __: embedding.html


v1.4.2
------

Nothing changed from v1.4.1.


v1.4.1
------

* Fix the compilation failure of cffi on CPython 3.5.0.  (3.5.1 works;
  some detail changed that makes some underscore-starting macros
  disappear from view of extension modules, and I worked around it,
  thinking it changed in all 3.5 versions---but no: it was only in
  3.5.1.)


v1.4.0
------

* A `better way to do callbacks`__ has been added (faster and more
  portable, and usually cleaner).  It is a mechanism for the
  out-of-line API mode that replaces the dynamic creation of callback
  objects (i.e. C functions that invoke Python) with the static
  declaration in ``cdef()`` of which callbacks are needed.  This is
  more C-like, in that you have to structure your code around the idea
  that you get a fixed number of function pointers, instead of
  creating them on-the-fly.

* ``ffi.compile()`` now takes an optional ``verbose`` argument.  When
  ``True``, distutils prints the calls to the compiler.

* ``ffi.compile()`` used to fail if given ``sources`` with a path that
  includes ``".."``.  Fixed.

* ``ffi.init_once()`` added.  See docs__.

* ``dir(lib)`` now works on libs returned by ``ffi.dlopen()`` too.

* Cleaned up and modernized the content of the ``demo`` subdirectory
  in the sources (thanks matti!).

* ``ffi.new_handle()`` is now guaranteed to return unique ``void *``
  values, even if called twice on the same object.  Previously, in
  that case, CPython would return two ``cdata`` objects with the same
  ``void *`` value.  This change is useful to add and remove handles
  from a global dict (or set) without worrying about duplicates.
  It already used to work like that on PyPy.
  *This change can break code that used to work on CPython by relying
  on the object to be kept alive by other means than keeping the
  result of ffi.new_handle() alive.*  (The corresponding `warning in
  the docs`__ of ``ffi.new_handle()`` has been here since v0.8!)

.. __: using.html#extern-python
.. __: ref.html#ffi-init-once
.. __: ref.html#ffi-new-handle


v1.3.1
------

* The optional typedefs (``bool``, ``FILE`` and all Windows types) were
  not always available from out-of-line FFI objects.

* Opaque enums are phased out from the cdefs: they now give a warning,
  instead of (possibly wrongly) being assumed equal to ``unsigned int``.
  Please report if you get a reasonable use case for them.

* Some parsing details, notably ``volatile`` is passed along like
  ``const`` and ``restrict``.  Also, older versions of pycparser
  mis-parse some pointer-to-pointer types like ``char * const *``: the
  "const" ends up at the wrong place.  Added a workaround.


v1.3.0
------

* Added `ffi.memmove()`_.

* Pull request #64: out-of-line API mode: we can now declare
  floating-point types with ``typedef float... foo_t;``.  This only
  works if ``foo_t`` is a float or a double, not ``long double``.

* Issue #217: fix possible unaligned pointer manipulation, which crashes
  on some architectures (64-bit, non-x86).

* Issues #64 and #126: when using ``set_source()`` or ``verify()``,
  the ``const`` and ``restrict`` keywords are copied from the cdef
  to the generated C code; this fixes warnings by the C compiler.
  It also fixes corner cases like ``typedef const int T; T a;``
  which would previously not consider ``a`` as a constant.  (The
  cdata objects themselves are never ``const``.)

* Win32: support for ``__stdcall``.  For callbacks and function
  pointers; regular C functions still don't need to have their `calling
  convention`_ declared.

* Windows: CPython 2.7 distutils doesn't work with Microsoft's official
  Visual Studio for Python, and I'm told this is `not a bug`__.  For
  ffi.compile(), we `removed a workaround`__ that was inside cffi but
  which had unwanted side-effects.  Try saying ``import setuptools``
  first, which patches distutils...

.. _`ffi.memmove()`: ref.html#ffi-memmove
.. __: https://bugs.python.org/issue23246
.. __: https://bitbucket.org/cffi/cffi/pull-requests/65/remove-_hack_at_distutils-which-imports/diff
.. _`calling convention`: using.html#windows-calling-conventions


v1.2.1
------

Nothing changed from v1.2.0.


v1.2.0
------

* Out-of-line mode: ``int a[][...];`` can be used to declare a structure
  field or global variable which is, simultaneously, of total length
  unknown to the C compiler (the ``a[]`` part) and each element is
  itself an array of N integers, where the value of N *is* known to the
  C compiler (the ``int`` and ``[...]`` parts around it).  Similarly,
  ``int a[5][...];`` is supported (but probably less useful: remember
  that in C it means ``int (a[5])[...];``).

* PyPy: the ``lib.some_function`` objects were missing the attributes
  ``__name__``, ``__module__`` and ``__doc__`` that are expected e.g. by
  some decorators-management functions from ``functools``.

* Out-of-line API mode: you can now do ``from _example.lib import x``
  to import the name ``x`` from ``_example.lib``, even though the
  ``lib`` object is not a standard module object.  (Also works in ``from
  _example.lib import *``, but this is even more of a hack and will fail
  if ``lib`` happens to declare a name called ``__all__``.  Note that
  ``*`` excludes the global variables; only the functions and constants
  make sense to import like this.)

* ``lib.__dict__`` works again and gives you a copy of the
  dict---assuming that ``lib`` has got no symbol called precisely
  ``__dict__``.  (In general, it is safer to use ``dir(lib)``.)

* Out-of-line API mode: global variables are now fetched on demand at
  every access.  It fixes issue #212 (Windows DLL variables), and also
  allows variables that are defined as dynamic macros (like ``errno``)
  or ``__thread`` -local variables.  (This change might also tighten
  the C compiler's check on the variables' type.)

* Issue #209: dereferencing NULL pointers now raises RuntimeError
  instead of segfaulting.  Meant as a debugging aid.  The check is
  only for NULL: if you dereference random or dead pointers you might
  still get segfaults.

* Issue #152: callbacks__: added an argument ``ffi.callback(...,
  onerror=...)``.  If the main callback function raises an exception
  and ``onerror`` is provided, then ``onerror(exception, exc_value,
  traceback)`` is called.  This is similar to writing a ``try:
  except:`` in the main callback function, but in some cases (e.g. a
  signal) an exception can occur at the very start of the callback
  function---before it had time to enter the ``try: except:`` block.

* Issue #115: added ``ffi.new_allocator()``, which officializes
  support for `alternative allocators`__.

.. __: using.html#callbacks
.. __: ref.html#ffi-new-allocator


v1.1.2
------

* ``ffi.gc()``: fixed a race condition in multithreaded programs
  introduced in 1.1.1


v1.1.1
------

* Out-of-line mode: ``ffi.string()``, ``ffi.buffer()`` and
  ``ffi.getwinerror()`` didn't accept their arguments as keyword
  arguments, unlike their in-line mode equivalent.  (It worked in PyPy.)

* Out-of-line ABI mode: documented a restriction__ of ``ffi.dlopen()``
  when compared to the in-line mode.

* ``ffi.gc()``: when called several times with equal pointers, it was
  accidentally registering only the last destructor, or even none at
  all depending on details.  (It was correctly registering all of them
  only in PyPy, and only with the out-of-line FFIs.)

.. __: cdef.html#dlopen-note


v1.1.0
------

* Out-of-line API mode: we can now declare integer types with
  ``typedef int... foo_t;``.  The exact size and signedness of ``foo_t``
  is figured out by the compiler.

* Out-of-line API mode: we can now declare multidimensional arrays
  (as fields or as globals) with ``int n[...][...]``.  Before, only the
  outermost dimension would support the ``...`` syntax.

* Out-of-line ABI mode: we now support any constant declaration,
  instead of only integers whose value is given in the cdef.  Such "new"
  constants, i.e. either non-integers or without a value given in the
  cdef, must correspond to actual symbols in the lib.  At runtime they
  are looked up the first time we access them.  This is useful if the
  library defines ``extern const sometype somename;``.

* ``ffi.addressof(lib, "func_name")`` now returns a regular cdata object
  of type "pointer to function".  You can use it on any function from a
  library in API mode (in ABI mode, all functions are already regular
  cdata objects).  To support this, you need to recompile your cffi
  modules.

* Issue #198: in API mode, if you declare constants of a ``struct``
  type, what you saw from lib.CONSTANT was corrupted.

* Issue #196: ``ffi.set_source("package._ffi", None)`` would
  incorrectly generate the Python source to ``package._ffi.py`` instead
  of ``package/_ffi.py``.  Also fixed: in some cases, if the C file was
  in ``build/foo.c``, the .o file would be put in ``build/build/foo.o``.


v1.0.3
------

* Same as 1.0.2, apart from doc and test fixes on some platforms.


v1.0.2
------

* Variadic C functions (ending in a "..." argument) were not supported
  in the out-of-line ABI mode.  This was a bug---there was even a
  (non-working) example__ doing exactly that!

.. __: overview.html#out-of-line-abi-level


v1.0.1
------

* ``ffi.set_source()`` crashed if passed a ``sources=[..]`` argument.
  Fixed by chrippa on pull request #60.

* Issue #193: if we use a struct between the first cdef() where it is
  declared and another cdef() where its fields are defined, then this
  definition was ignored.

* Enums were buggy if you used too many "..." in their definition.


v1.0.0
------

* The main news item is out-of-line module generation:

  * `for ABI level`_, with ``ffi.dlopen()``

  * `for API level`_, which used to be with ``ffi.verify()``, now deprecated

* (this page will list what is new from all versions from 1.0.0
  forward.)

.. _`for ABI level`: overview.html#out-of-line-abi-level
.. _`for API level`: overview.html#out-of-line-api-level
