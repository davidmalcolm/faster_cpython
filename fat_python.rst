.. _fat-python:

**********
FAT Python
**********

.. image:: fat_python.jpg
   :alt: Three-year-old Cambodian boy Oeun Sambat hugs his best friend, a four-metre long female python named Chamreun
   :align: right
   :target: http://pictures.reuters.com/archive/CAMBODIA-PYTHONBOY-RP3DRIMPKQAA.html

.. Source of the photo:
   Three-year-old befriends python
   Sit Tbow (Cambodia) May 22
   Cambodians are flocking to see a three-year-old boy they believe was the son
   of a dragon in his previous life because his best friend is a
   four-metre-long python.
   Curled up for an afternoon snooze inside the coils of his companion, the
   child, Oeun Sambath, attracts regular visits from villagers anxious to make
   use of what they believe are his supernatural powers. "He has been playing
   with the python ever since he could first crawl," said his mother Kim
   Kannara. Reuters

Intro
=====

The FAT Python project was started by Victor Stinner in October 2015 to try to
solve issues of previous attempts of "static optimizers" for Python. The main
feature are efficient guards using versionned dictionaries to check if
something was modified. Guards are used to decide if the specialized bytecode
of a function can be used or not.

Python FAT is expected to be FAT... maybe FAST if we are lucky. FAT because
it will use two versions of some functions where one version is specialised to
specific argument types, a specific environment, optimized when builtins are
not mocked, etc.

Announcements and status reports:

* [Python-Dev] `Third milestone of FAT Python
  <https://mail.python.org/pipermail/python-dev/2015-December/142397.html>`_
* `Status of the FAT Python project, November 26, 2015
  <https://haypo.github.io/fat-python-status-nov26-2015.html>`_
* [python-dev] `Second milestone of FAT Python
  <https://mail.python.org/pipermail/python-dev/2015-November/142113.html>`_
  (Nov 2015)
* [python-ideas] `Add specialized bytecode with guards to functions
  <https://mail.python.org/pipermail/python-ideas/2015-October/036908.html>`_
  (Oct 2015)

The project was created in October 2015.

See also the :ref:`AST optimizer <new-ast-optimizer>`.


Test FAT Python
===============

Download FAT Python with::

    hg clone http://hg.python.org/sandbox/fatpython

Compile it and run tests::

    ./configure && make && ./python -m test test_astoptimizer test_fat

Benchmark::

    ./python bench.py
    ./python -F bench.py

Example
=======

::

    >>> def func():
    ...     return len("abc")
    ...
    >>> import dis
    >>> dis.dis(func)
      2           0 LOAD_GLOBAL              0 (len)
                  3 LOAD_CONST               1 ('abc')
                  6 CALL_FUNCTION            1 (1 positional, 0 keyword pair)
                  9 RETURN_VALUE

    >>> len(func.get_specialized())
    1
    >>> specialized=func.get_specialized()[0]
    >>> dis.dis(specialized['code'])
      2           0 LOAD_CONST               1 (3)
                  3 RETURN_VALUE
    >>> len(specialized['guards'])
    2

    >>> func()
    3

    >>> len=lambda obj: "mock"
    >>> func()
    'mock'
    >>> func.get_specialized()
    []

The function ``func()`` has specialized bytecode which returns directly 3
instead of calling ``len("abc")``. The specialized bytecode has two guards
dictionary keys: ``builtins.__dict__['len']`` and ``globals()['len']``. If one
of these keys is modified, the specialized bytecode is simply removed (when the
function is called) and the original bytecode is executed.


Optimizations
=============

Call pure builtins
------------------

Call pure builtin functions at compilation: replace the call with the result in
the specialized bytecode, add guards on the called builtin functions.

The optimization is disabled when the builtin function is modified or if
a variable with the same name is added to the global namespace of the function.

The optimization on the builtin ``NAME`` requires two guards:

* ``NAME`` key in builtin namespace
* ``NAME`` key in global namespace

Example:

+------------------------+---------------+
| Original               | Specialized   |
+========================+===============+
| ::                     | ::            |
|                        |               |
|  def func():           |  def func():  |
|      return len("abc") |      return 3 |
+------------------------+---------------+

Bytecode of the specialized function::

    0 LOAD_CONST               1 (3)
    3 RETURN_VALUE


.. _fat-loop-unroll:

Loop unrolling
--------------

``for i in range(3): ...`` and ``for i in (1, 2, 3): ...`` are unrolled.
By default, only loops with 16 iterations or less are optimized.

.. note::
   If ``break`` and/or ``continue`` instructions are used in the loop body,
   the loop is not unrolled.

See also the :ref:`loop unrolling optimization <loop-unroll>`.

tuple example
^^^^^^^^^^^^^

Example with a tuple.

+---------------------------+--------------------------+
| Original                  | Loop unrolled            |
+===========================+==========================+
| ::                        | ::                       |
|                           |                          |
|  def func():              |  def func():             |
|      for i in ("a", "b"): |      i = "a"             |
|          print(i)         |      print(i)            |
|                           |                          |
|                           |      i = "b"             |
|                           |      print(i)            |
+---------------------------+--------------------------+

No guard is required. The function has no specialized bytecode, the
optimization is done directly on the function.

Original bytecode::

    .     0 SETUP_LOOP              14 (to 17)
          3 LOAD_CONST               3 (('hello', 'world'))
          6 GET_ITER

    >>    7 FOR_ITER                 6 (to 16)
         10 STORE_FAST               0 (i)

         13 JUMP_ABSOLUTE            7
    >>   16 POP_BLOCK

    >>   17 LOAD_CONST               0 (None)
         20 RETURN_VALUE

FAT Python bytecode::

    LOAD_CONST   1 ("hello")
    STORE_FAST   0 (i)

    LOAD_CONST   2 ("world")
    STORE_FAST   0 (i)

    LOAD_CONST   0 (None)
    RETURN_VALUE


range example
^^^^^^^^^^^^^

Example of a loop using ``range()``.

+--------------------------+--------------------------+
| Original                 | Loop unrolled            |
+==========================+==========================+
| ::                       | ::                       |
|                          |                          |
|  def func():             |  def func():             |
|      for i in range(2):  |      i = 0               |
|          print(i)        |      print(i)            |
|                          |                          |
|                          |      i = 1               |
|                          |      print(i)            |
+--------------------------+--------------------------+

The specialized bytecode requires two :ref:`guards <fat-guard>`:

* ``range`` builtin variable
* ``range`` global variable

Combined with :ref:`constant propagation <fat-const-prop>`, the code becomes
even more interesting::

    def func():
        i = 0
        print(0)

        i = 1
        print(1)


.. _fat-const-prop:

Constant propagation
--------------------

Propagate constant values of variables.

+----------------+----------------------+
| Original       | Constant propagation |
+================+======================+
| ::             | ::                   |
|                |                      |
|   def func()   |   def func()         |
|       x = 1    |       x = 1          |
|       y = x    |       y = 1          |
|       return y |       return 1       |
+----------------+----------------------+

See also the :ref:`constant folding <const-fold>` optimization.

.. _fat-const-fold:

Constant folding
----------------

Compute simple operations at the compilation:

* arithmetic operations: a+b, a-b, a*b, a/b, a//b, a%b, a**b, a<<b, a>>b, a&b,
  a|b and a^b for int, float and complex types
* str: str + str, str * int
* bytes: bytes + bytes, bytes * int
* tuple: tuple + tuple, tuple * int

Example:

+--------------------+------------------+
| Original           | Constant folding |
+====================+==================+
| ::                 | ::               |
|                    |                  |
|   def func()       |   def func()     |
|       return 1 + 1 |       return 2   |
+--------------------+------------------+

See also the :ref:`constant propagation <const-prop>` optimization.

.. _fat-copy-builtin-to-constant:

Copy builtin functions to constants
-----------------------------------

Opt-in optimization (disabled by default) to copy builtin functions to
constants.

Example with a function simple::

    def log(message):
        print(message)

+--------------------------------------------------+----------------------------------------------------+
| Bytecode                                         | Specialized bytecode                               |
+==================================================+====================================================+
| ::                                               | ::                                                 |
|                                                  |                                                    |
|   LOAD_GLOBAL   0 (print)                        |   LOAD_CONST      1 (<built-in function print>)    |
|   LOAD_FAST     0 (message)                      |   LOAD_FAST       0 (message)                      |
|   CALL_FUNCTION 1 (1 positional, 0 keyword pair) |   CALL_FUNCTION   1 (1 positional, 0 keyword pair) |
|   POP_TOP                                        |   POP_TOP                                          |
|   LOAD_CONST    0 (None)                         |   LOAD_CONST      0 (None)                         |
|   RETURN_VALUE                                   |   RETURN_VALUE                                     |
+--------------------------------------------------+----------------------------------------------------+

The first ``LOAD_GLOBAL`` instruction is replaced with ``LOAD_CONST``.
``LOAD_GLOBAL`` requires to lookup in the global namespace and then in the
builtin namespaces, two dictionary lookups. ``LOAD_CONST`` gets the value from
a C array, O(1) lookup.

The specialized bytecode requires two :ref:`guards <fat-guard>`:

* ``print`` builtin variable
* ``print`` global variable

The ``print()`` function is injected in the constants with the
``func.patch_constants()`` method.

The optimization on the builtin ``NAME`` requires two guards:

* ``NAME`` key in builtin namespace
* ``NAME`` key in global namespace

This optimization is disabled by default because it changes the :ref:`Python
semantic <fat-python-semantic>`: if the copied builtin function is replacd in
the middle of the function, the specialized bytecode still uses the old builtin
function.

See also the :ref:`load globals and builtins when the module is loaded
<load-global-optim>` optimization.

.. note::
   Currently, astoptimizer is unable to guess if an instruction can modify
   builtins functions or not. For example, the optimization changes the
   behaviour of the following function.

::

    def func():
        x = range(3)
        print(len(x))   # expect: 3

        _len = builtins.len
        try:
            builtins.len = lambda x: "mock"
            print(len(x))   # expect: mock
        finally:
            builtins.len = _len


Limitations and Python semantic
===============================

FAT Python bets that the Python code is not modified when modules are loaded,
but only later, when functions and classes are executed. If this assumption is
wrong, FAT Python changes the semantic of Python.

.. _fat-python-semantic:

Python semantic
---------------

It is very hard, to not say impossible, to implementation and keep the exact
behaviour of regular CPython. CPython implementation is used as the Python
"standard". Since CPython is the most popular implementation, a Python
implementation must do its best to mimic CPython behaviour. We will call it the
Python semantic.

FAT Python should not change the Python semantic with the default
configuration.  Optimizations obvisouly the Python semantic must be disabled by
default: opt-in options.

As written above, it's really hard to mimic exactly CPython behaviour. For
example, in CPython, it's technically possible to modify local variables of a
function from anywhere, a function can modify its caller, or a thread B can
modify a thread A (just for fun). See :ref:`Everything in Python is mutable
<mutable>` for more information. It's also hard to support all introspections
features like ``locals()`` (``vars()``), ``globals()`` and ``sys._getframe()``.

Builtin functions replaced in the middle of a function
------------------------------------------------------

FAT Python uses :ref:`guards <fat-guard>` to disable specialized function when
assumptions made to optimize the function are no more true. The problem is that
guard are only called at the entry of a function. For example, if a specialized
function ensures that the builtin function ``chr()`` was not modified, but
``chr()`` is modified during the call of the function, the specialized function
will continue to call the old ``chr()`` function.

The :ref:`copy builtin functions to constants <fat-copy-builtin-to-constant>`
optimization changes the Python semantic. If a builtin function is replaced
while the specialized function is optimized, the specialized function will
continue to use the old builtin function. For this reason, the optimization
is disabled by default.

Example::

    def func(arg):
        x = chr(arg)

        with unittest.mock.patch('builtins.chr', result='mock'):
            y = chr(arg)

        return (x == y)

If the :ref:`copy builtin functions to constants
<fat-copy-builtin-to-constant>` optimization is used on this function, the
specialized function returns ``True``, whereas the original function returns
``False``.


Guards on builtin functions
---------------------------

When a function is specialized, the specialization is ignored if a builtin
function was replaced after the end of the Python initialization. Typically,
the end of the Python initialization occurs just after the execution of the
``site`` module. It means that if a builtin is replaced during Python
initialization, a function will be specialized even if the builtin is not the
expected builtin function.

Example::

    import builtins

    builtins.chr = lambda: mock

    def func():
        return len("abc")

In this example, the ``func()`` is optimized, but the function is *not*
specialize. The internal call to ``func.specialize()`` is ignored because the
``chr()`` function was replaced after the end of the Python initialization.


Guards on type dictionary and global namespace
-----------------------------------------------

For other guards on dictionaries (type dictionary, global namespace), the guard
uses the current value of the mapping. It doesn't check if the dictionary value
was "modified".


Tracing and profiling
---------------------

Tracing and profiling works in FAT mode, but the exact control flow and traces
are different in regular and FAT mode. For example, :ref:`loop unrolling
<fat-loop-unroll>` removes the call to ``range(n)``.

See ``sys.settrace()`` and ``sys.setprofiling()`` functions.

Expected limitations
--------------------

Inlining makes debugging more complex:

* sys.getframe()
* locals()
* pdb
* etc.
* don't work as expected anymore

Bugs, shit happens:

* Missing guard: specialized function is called even if the "environment"
  was modified

FAT python! Memory vs CPU, fight!

* Memory footprint: loading two versions of a function is memory uses more
  memory
* Disk usage: .pyc will be more larger

Possible worse performance:

* guards adds an overhead higher than the optimization of the specialized code
* specialized code may be slower than the original bytecode


Goals
=====

Goals:

* *no* overhead when FAT mode is disabled (default). The FAT mode must remain
  optional.
* Faster than current CPython on real applications like Django or Mercurial.
  5% faster would be nice, 10% would be better.
* 100% compatible with CPython and the Python language: everything must be kept
  mutable. Optimizations are disabled when the environment is modified.
* 100% compatible with the CPython C API: ABI and C structures must not be
  modified.
* Add a generic API to support "specialized" functions.

Non-goal:

* FAT Python doesn't modify the Python C API: don't expect better memory
  footprint with specialized types, like PyPy list of integers stored
  as a real array of C int in memory.
* FAT Python is not a JIT. Don't expected crazy performances as PyPy, Numba or
  Pyston. PyPy must remain the fastest implementation of Python, 100%
  compatible with CPython!


Roadmap
=======

Milestone 1: DONE
-----------------

* guard and specialized PoC in Python: DONE
* optimize guards with modification in CPython internals:

  - add version to dictionaries: DONE
  - add version to functions: DONE

* implement guards and specialized function in C: DONE

Expected speedup: 10% on specific microbenchmarks, but require to modify the
source code manually to specialize functions.

Milestone 2: DONE
-----------------

* write an AST optimizer:

  - call pure builtin functions at compilation: DONE
  - generate guards: DONE

* enable the optimizer by default in FAT mode in the site module and
  ensure that *no* test fails in the Python test suite, running
  the test suite with -j0 to isolate processes

Expected speedup: no speedup, it's just a milestone to validate the
implementation. (It's still possible to optimize *manually* code to specialize
functions, to implement better optimizations.)


Milestone 3 (faster)
--------------------

DONE:

* add a configuration to astoptimizer
* opt-in optimization "copy global to locals", currently used to load builtin
  functions

TODO:

* configuration to manually help the optimizer:

  - give a whitelist of "constants": app.DEBUG, app.enum.BLUE, ...
  - type hint with strict types: x is Python int in range [3; 10]
  - expect platform values to be constant: sys.version_info, sys.maxunicode,
    os.name, sys.platform, os.linesep, etc.
  - declare pure functions
  - see astoptimizer for more ideas

* implement more optimizations:

  - constant folding
  - detect pure functions in AST and call them at the compilation
  - function inlining
  - move invariants out of the loop

* implement code to detect the exact type of function parameters and function
  locals and save it into an annotation file
* implement profiling directed optimization: benchmark guards at runtime
  to decide if it's worth to use a specialized function. Measure maybe also
  the memory footprint using tracemalloc?
* implement basic stategy to decide if specialized function must be emitted
  or not using raw estimation, like the size of the bytecode in bytes

Milestone 4 (goal)
------------------

* move the optimizer into a new third-party project. Only keep the API for
  specialized functions with guards


Guards on specialized functions
===============================

To decide if we can use the specialized version of a function, we have to
ensure that the environment was not modified. We will call these checks
"guards.

First attempt: "modified" and "readonly" flags
----------------------------------------------

See :ref:`read-only Python <readonly>`.


New try: versionned dictionary
------------------------------

To reduce the cost of dictionary lookup when checking guards, a subclass of
dict is added: verdict(), versionned dictionary. A verdict has a global version
incremented each time that the dict is modified and each mapping (key) has
a version too, modified when a key is modified. Example::

    >>> import fat
    >>> d = fat.verdict()
    >>> d.__version__
    1
    >>> d.getversion('a')
    >>> print(d.getversion('a'))
    None

    >>> d['a'] = 1
    >>> d.__version__
    2
    >>> print(d.getversion('a'))
    2

    >>> d['a'] = 2
    >>> d.__version__
    3
    >>> print(d.getversion('a'))
    3

    >>> del d['a']
    >>> d.__version__
    4
    >>> print(d.getversion('a'))
    None

A guard only has to lookup for the watched key if the global version is
modified. Currently, the specialized function is disabled when the value was
modified, even if the key is modified and restored before the guard is checked.
The reason for this is that keeping a reference to the watched value can create
reference leaks and may keep objects alive longer than expected.

For the same reason, the guard doesn't keep a strong reference to the
dictionary, but a *weak* reference. It's not possible to create a weak
reference to a dict, but it's possible to create a weak reference to a verdict.

.. _fat-guard:

Guards
------

Guards:

* FuncGuard: check if a function was modified (currently only __code__ is
  checked)
* DictGuard: check if a dictionary key is created (if it didn't exist) or
  modified
* ArgTypeGuard: check the type of function arguments

Example: Guard on a builtin function
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Example of function::

    def use_builtin_len():
        return len("abc")

To replace ``len("abc")``, we have to ensure that:

* the builtin ``len()`` function was not overriden
  with ``builtins.len = mock_len``
* the ``len`` symbol was not added to the function globals which are the module
  globals

Example: Guard to inline a function
^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^^

Example of function::

    def is_python(filename):
        return filename.endswith('.py')

    def filter_python(filenames):
        return [filename for filename in filenames
                if is_python(filename)]

To replace ``is_python(filename)`` with ``filename.endswith('.py')`` in
``filter_python()``, we have to ensure that:

* the ``is_python`` symbol was not modified in the namespace (module globals)
* the ``is_python()`` function was not modified


Implementation
==============

FAT python:

* new builtins.__fat__ variable (bool)
* Object/dictobject.c: add __version__
* Modules/fat.c: specialized functions with guards
* Tests

  - Lib/test/test_fat.py
  - Lib/test/fattester.py
  - Lib/test/fattesterast.py
  - Lib/test/fattesterast2.py

Other changes:

* Python/bltinmodule.c: add __fat__ builtin symbol
* Python/ceval.c: bugfixes when builtins is not a dict type
* Python/sysmodule.c: add sys.flags.fat
* Modules/main.c: add -F command line option

See also the :ref:`AST optimizer <new-ast-optimizer>`.


Possible optimizations
======================

Short term:

* Function func2() calls func1() if func1() is pure: inline func1()
  into func2()
* Call builtin pure functions during compilation. Example: replace len("abc")
  with 3 or range(3) with (0, 1, 2).
* Constant folding: replace a variable with its value. We may do that for
  optimal parameters with default value if these parameters are not set.
  Example: replace app.DEBUG with False.

Using types:

* Detect the exact type of parameters and function local variables
* Specialized code relying on the types. For example, move invariant out of
  loops (ex: obj.append for list).
* x + 0 gives a TypeError for str, but can be replaced with x for int and
  float. Same optimization for x*0.
* See astoptimizer for more ideas.

Longer term:

* Compile to machine code using Cython, Numba, PyPy, etc. Maybe only for
  numeric types at the beginning? Release the GIL if possible, but check
  "sometimes" if we got UNIX signals.


Pure functions
==============

A "pure" function is a function with no side effect.

Example of pure operators:

* x+y, x-y, x*y, x/y, x//y, x**y for types int, float, complex, bytes, str,
  and also tuple and list for x+y

Example of instructions with side effect:

* "global var"

Example of pure function::

    def mysum(x, y):
        return x + y

Example of function with side effect::

    global _last_sum

    def mysum(x, y):
        global _last_sum
        s = x + y
        _last_sum = s
        return s


FAT Python API
==============

* func.specialize(bytecode[, guards: list]): add a specialized bytecode.
  If bytecode is a function, uses its __code__ attribute.
  Guards a list of dict, syntax of one guard:

  - ``{'guard_type': 'func', 'func': func2}``:
    guard on func2.__code__
  - ``{'guard_type': 'dict', 'dict': ns, 'key': key}``:
    guard on the versionned dictionary ns[key]
  - ``{'guard_type': 'builtins', 'name': 'len'}``:
    guard on builtins.__dict__['len'] and globals()['len']. The specialization
    is ignored if builtins.__dict__['len'] was replaced after the end of Python
    initialization or if globals()['len'] already exists.
  - ``{'guard_type': 'globals', 'name': 'obj'}``:
    guard on globals()['obj']
  - ``{'guard_type': 'type_dict', 'type': MyClass, 'key': attr}``:
    guard on MyClass.__dict__[key]
  - ``{'guard_type': 'type', 'type': MyClass, 'key': 'attr'}``:
    guard on MyClass.__dict__['attr']
  - ``{'guard_type': 'arg_type', 'arg_index': 0, 'type': str}``:
    type of the function argument 0 must be str. As isinstance, *type* accepts
    an iterable of types, ex: ``{..., 'type': (list, tuple)}``.

* func.get_specialized()

For dictionary and function guards: specialized functions are removed if the
guards fail:

* Broken weak-reference to the dictionary/function
* The dictionary key was modified (created, modified or removed depending on
  the initial state)
* The function was modified
* An error occurred when getting the dictionary entry to get the key version


Origins of FAT Python
=====================

* :ref:`Old AST optimizer project <old-ast-optimizer>`
* :ref:`read-only Python <readonly>`
* Dave Malcolm wrote a patch modifying Python/eval.c to support specialized
  functions. See the http://bugs.python.org/issue10399
