=======================================================
安装和状态
=======================================================

CPython快速安装 (cffi随PyPy一起发布):

* ``pip install cffi``

* 或者通过 `Python Package Index`__ 获取源代码。

.. __: http://pypi.python.org/pypi/cffi

更多细节:

此代码是在Linux上开发的，但应该适用于任何POSIX平台以及Windows 32位和64位。  (它偶尔会依赖于libffi，所以这取决于libffi有没有bug; 在一些更奇特的平台上，情况可能并非如此。)

CFFI 支持CPython 2.6，2.7，3.x (用3.2到3.4进行测试); CFFI并与PyPy一起发布 (CFFI 1.0 需要随PyPy 2.6一起发布).

CFFI的核心速度优于ctypes，如果使用1.0之后的功能，则意味着时间更短，或者更高，除非你不这样做。  您通常需要围绕原始CFFI接口编写的包装Python代码会减慢CPython的速度，但并非不合理。  在PyPy上，由于JIT编译器，这个包装器代码的影响很小。 这使得CFFI成为PyPy与C库接口的推荐方式。

要求:

* CPython 2.6 或 2.7 或 3.x，或 PyPy 
  (PyPy 2.0是提供给CFFI最早的版本; 或PyPy 2.6提供CFFI 1.0).

* 在某些情况下，您需要能够编译C语言的扩展模块。
  在非Windows平台上，一般方法安装 ``python-dev`` 包。 请参阅适用于您的操作系统的相应文档。

* 在CPython，非Windows平台上，
  你还需要安装 ``libffi-dev`` 才能编译CFFI本身。

* pycparser >= 2.06: https://github.com/eliben/pycparser 
  ( ``pip install cffi`` 自动跟踪).

* CFFI 自身 需要 `py.test`_ 来运行测试。

.. _`py.test`: http://pypi.python.org/pypi/pytest

下载和安装:

* https://pypi.python.org/pypi/cffi

* 校验 "source" 包 version 1.12.3:

   - MD5: 35ad1f9e1003cac9404c1493eb10d7f5

   - SHA: ccc49cf31bc3f4248f45b9ec83685e4e8090a9fa

   - SHA256: 041c81822e9f84b1d9c401182e174996f0bae9991f33725d059b771744290774

* 或者从 `Bitbucket page`_ 页面获取最新版本:
  ``hg clone https://bitbucket.org/cffi/cffi``

* ``python setup.py install`` 或 ``python setup_base.py install``
  (在Linux 或 Windows上应该开箱即用; 请参阅
  `MacOS X`_ 或 `Windows 64`_.)

* 运行测试: ``py.test  c/  testing/`` (如果你还没有安装cffi，首先你需要 ``python setup_base.py build_ext -f
  -i``)

.. _`Bitbucket page`: https://bitbucket.org/cffi/cffi

演示:

* `demo`_ 目录包含许多使用 ``cffi`` 的小型和大型的演示。

* 下面的文档可能在细节上是简略的; 目前测试给出了最终的参考，特别是
  `testing/cffi1/test_verify1.py`_ 和 `testing/cffi0/backend_tests.py`_.

.. _`demo`: https://bitbucket.org/cffi/cffi/src/default/demo
.. _`testing/cffi1/test_verify1.py`: https://bitbucket.org/cffi/cffi/src/default/testing/cffi1/test_verify1.py
.. _`testing/cffi0/backend_tests.py`: https://bitbucket.org/cffi/cffi/src/default/testing/cffi0/backend_tests.py


特定于平台的说明
------------------------------

``libffi`` 的安装和使用非常混乱 - 以至于CPython包含自己的副本以避免依赖外部软件包。
CFFI对Windows也是如此，但对其他平台则没有 (哪一个应该具有他们自己的加工的 libffi)。
感谢 ``pkg-config`` 当前Linux开箱即用。 以下是（用户提供的）其他平台的一些说明。


MacOS X
+++++++

**Homebrew** (感谢David Griffin)

1) 安装 homebrew: http://brew.sh

2) 在终端中运行以下命令

::

    brew install pkg-config libffi
    PKG_CONFIG_PATH=/usr/local/opt/libffi/lib/pkgconfig pip install cffi


可选，**on OS/X 10.6** (感谢Juraj Sukop)

要构建libffi，您可以使用默认安装路径，但是然后，在
``setup.py`` 中你需要改变::

    include_dirs = []

为::

    include_dirs = ['/usr/local/lib/libffi-3.0.11/include']

然后运行 ``python setup.py build`` 报出 "fatal error: error writing to -: Broken pipe"，可以通过运行一下来修复::

    ARCHFLAGS="-arch i386 -arch x86_64" python setup.py build

如 此处_ 所述。

.. _此处: http://superuser.com/questions/259278/python-2-6-1-pycrypto-2-3-pypi-package-broken-pipe-during-build


Windows (常规 32-bit)
++++++++++++++++++++++++

Win32工作并至少在每个正式版本中进行测试。

与Python 2.7兼容的推荐C编译器是这个:
http://www.microsoft.com/en-us/download/details.aspx?id=44266
Python 2.7上存在distutils的已知问题，如中所述 https://bugs.python.org/issue23246，当你想运行compile()来使用这个特定编译器套件下载来构建的一个dll时，同样的问题也适用。 
``import setuptools`` 可能有帮助，但是 YMMV

适用于Python 3.4及更高版本:
https://www.visualstudio.com/en-us/downloads/visual-studio-2015-ctp-vs


Windows 64
++++++++++

Win64接受了非常基本的测试，我们在cffi 0.7中应用了一些基本的修复。 上面的评论也适用于Windows 64上的Python 2.7。 请报告任何其他问题。

请注意，这只是关于在64位操作系统上运行64位版本的Python。 如果您正在使用32位版本 (显然是常见的情况)，你是使用Win32那么就我们关心Win32。

.. _`issue 9`: https://bitbucket.org/cffi/cffi/issue/9
.. _`Python issue 7546`: http://bugs.python.org/issue7546


Linux and OS/X: UCS2 与 UCS4
++++++++++++++++++++++++++++++++

这是关于使用像 ``Symbol not found: _PyUnicodeUCS2_AsASCIIString`` 这样的消息来获取关于 ``_cffi_backend.so`` 的ImportError。 只要混合使用Python的“ucs2”和“ucs4”版本，就会在Python 2中发生此错误。  这意味着您现在正在运行使用“ucs4”编译的Python，但是扩展模块 ``_cffi_backend.so`` 是由不同的Python编译的: 一个正在运行“ucs2”。 (如果出现相反的问题，则会收到有关 ``_PyUnicodeUCS4_AsASCIIString`` 的错误。)

如果您正在使用 ``pyenv`` ，请参阅
https://github.com/yyuu/pyenv/issues/257.

更一般地说，应该始终有效的解决方案是下载CFFI的源代码(而不是预先构建的二进制代码)并确保使用与使用它相同版本的Python构建它。
例如，使用virtualenv:

* ``virtualenv ~/venv``

* ``cd ~/path/to/sources/of/cffi``

* ``~/venv/bin/python setup.py build --force`` # forcing a rebuild to
  make sure

* ``~/venv/bin/python setup.py install``

这将使用virtualenv中的Python在这个virtualenv中编译和安装CFFI。


NetBSD
++++++

您需要确保拥有最新版本的libffi，它修复了一些错误。
