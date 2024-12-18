.. _writing_stubs:

**********************************
Writing and Maintaining Stub Files
**********************************

Stub files are a means of providing type information for Python modules.
For a full reference, refer to :ref:`stub-files`.

Maintaining stubs can be a little cumbersome because they are separated from the
implementation. This page lists some tools that make writing and maintaining
stubs less painful, as well as some best practices on stub contents and style.

Tools for generating stubs
==========================

stubgen
-------

stubgen is a tool bundled with `mypy <https://github.com/python/mypy>`__
that can be used to generate basic stubs. These stubs serve as a
basic starting point; most types will default to ``Any``.

.. code-block:: console

    stubgen -p my_great_package

For more details, see `stubgen docs <https://mypy.readthedocs.io/en/stable/stubgen.html>`__.

pyright
-------

pyright contains a tool that generates basic stubs. Like stubgen, these generated
stubs serve more as a starting point.

.. code-block:: console

    pyright --createstub my_great_package

For more details, see `pyright docs <https://github.com/microsoft/pyright/blob/main/docs/type-stubs.md#generating-type-stubs-from-command-line>`__.

monkeytype
----------

monkeytype takes a slightly different approach — you run your code (perhaps via
your tests) and monkeytype collects the types it observes at runtime to generate
stubs.

.. code-block:: console

    monkeytype run script.py
    monkeytype stub my_great_package

For more details, see `monkeytype docs <https://monkeytype.readthedocs.io/en/latest/>`__.

Tools for maintaining stubs
===========================

stubtest
--------

stubtest is a tool bundled with `mypy <https://github.com/python/mypy>`__.

stubtest finds inconsistencies between stub files and the implementation. It
does this by comparing stub definitions to what it finds from importing your
code and using runtime introspection (via the ``inspect`` module).

.. code-block:: console

    stubtest my_great_package

For more details, see `stubtest docs <https://mypy.readthedocs.io/en/stable/stubtest.html>`__.

flake8-pyi
----------

flake8-pyi is a `flake8 <https://flake8.pycqa.org/en/latest/>`__ plugin that
lints common issues in stub files.

.. code-block:: console

    flake8 my_great_package

For more details, see `flake8-pyi docs <https://github.com/PyCQA/flake8-pyi>`__.

Running a type checker on the stubs
-----------------------------------

Simply running a type checker on the stubs can catch several issues, from simple
things like detecting missing annotations to more complex things like ensuring
Liskov substitutability or detecting problematic overloads.

It may be instructive to examine `typeshed <https://github.com/python/typeshed/>`__'s
`setup for testing stubs <https://github.com/python/typeshed/blob/main/tests/README.md>`__.

..
   TODO: consider adding examples and configurations for specific type checkers

Type checking usage of your package
-----------------------------------

If you have access to a codebase that uses your package — perhaps tests for your
package — running a type checker against it can help you detect issues,
particularly with false positives.

If your package has some particularly complex aspects, you could even consider
writing dedicated typing tests for tricky definitions. For more details, see
:ref:`testing`.

Stub Content
============

This section documents best practices on what elements to include or
leave out of stub files.

Modules excluded from stubs
---------------------------

Not all modules should be included in stubs.

It is recommended to exclude:

1. Implementation details, with `multiprocessing/popen_spawn_win32.py <https://github.com/python/cpython/blob/main/Lib/multiprocessing/popen_spawn_win32.py>`_ as a notable example
2. Modules that are not supposed to be imported, such as ``__main__.py``
3. Protected modules that start with a single ``_`` char. However, when needed protected modules can still be added (see :ref:`undocumented-objects` section below)
4. Tests

Public Interface
----------------

Stubs should include the complete public interface (classes, functions,
constants, etc.) of the module they cover, but it is not always
clear exactly what is part of the interface.

The following should always be included:

* All objects listed in the module's documentation.
* All objects included in ``__all__`` (if present).

Other objects may be included if they are not prefixed with an underscore
or if they are being used in practice. (See the next section.)

.. _undocumented-objects:

Undocumented Objects
--------------------

Undocumented objects may be included as long as they are marked with a comment
of the form ``# undocumented``.

Example::

    def list2cmdline(seq: Sequence[str]) -> str: ...  # undocumented

Such undocumented objects are allowed because omitting objects can confuse
users. Users who see an error like "module X has no attribute Y" will
not know whether the error appeared because their code had a bug or
because the stub is wrong. Although it may also be helpful for a type
checker to point out usage of private objects, false negatives (no errors for
wrong code) are preferable over false positives (type errors
for correct code). In addition, even for private objects a type checker
can be helpful in pointing out that an incorrect type was used.

``__all__``
------------

A stub file should contain an ``__all__`` variable if and only if it is also
present at runtime. In that case, the contents of ``__all__`` should be
identical in the stub and at runtime. If the runtime dynamically adds
or removes elements (for example if certain functions are only available on
some system configurations), include all possible elements in the stubs.

Stub-Only Objects
-----------------

Definitions that do not exist at runtime may be included in stubs to aid in
expressing types. Unless intentionally exposed to users (see below), such
definitions should be marked as private by prefixing their names with an
underscore.

Yes::

    _T = TypeVar("_T")
    _DictList: TypeAlias = dict[str, list[int | None]]

No::

    T = TypeVar("T")
    DictList: TypeAlias = dict[str, list[int | None]]

Sometimes, it is desirable to make a stub-only class available
to a stub's users — for example, to allow them to type the return value of a
public method for which a library does not provided a usable runtime type. Use
the ``typing.type_check_only`` decorator to mark such objects::

  from typing import Protocol, type_check_only

  @type_check_only
  class Readable(Protocol):
      def read(self) -> str: ...

  def get_reader() -> Readable: ...

Structural Types
----------------

As seen in the example with ``Readable`` in the previous section, a common use
of stub-only objects is to model types that are best described by their
structure. These objects are called protocols (:pep:`544`), and it is encouraged
to use them freely to describe simple structural types.

Incomplete Stubs
----------------

Partial stubs can be useful, especially for larger packages, but they should
follow the following guidelines:

* Included functions and methods should list all arguments, but the arguments
  can be left unannotated.
* Do not use ``Any`` to mark unannotated or partially annotated values. Leave
  function parameters and return values unannotated. In all other cases, use
  ``_typeshed.Incomplete``
  (`documentation <https://github.com/python/typeshed/blob/main/stdlib/_typeshed/README.md>`_)::

    from _typeshed import Incomplete

    field1: Incomplete
    field2: dict[str, Incomplete]

    def foo(x): ...

* Partial classes should include a ``__getattr__()`` method marked with
  ``_typeshed.Incomplete`` (see example below).
* Partial modules (i.e. modules that are missing some or all classes,
  functions, or attributes) should include a top-level ``__getattr__()``
  function marked with ``_typeshed.Incomplete`` (see example below).
* Partial packages (i.e. packages that are missing one or more sub-modules)
  should have a ``__init__.pyi`` stub that is marked as incomplete (see above).
  A better alternative is to create empty stubs for all sub-modules and
  mark them as incomplete individually.

Example of a partial module with a partial class ``Foo`` and a partially
annotated function ``bar()``::

    from _typeshed import Incomplete

    def __getattr__(name: str) -> Incomplete: ...

    class Foo:
        def __getattr__(self, name: str) -> Incomplete: ...
        x: int
        y: str

    def bar(x: str, y, *, z=...): ...

Attribute Access
----------------

Python has several methods for customizing attribute access: ``__getattr__``,
``__getattribute__``, ``__setattr__``, and ``__delattr__``. Of these,
``__getattr__`` and ``__setattr___`` should sometimes be included in stubs.

In addition to marking incomplete definitions, ``__getattr__`` should be
included when a class or module allows any name to be accessed. For example, consider
the following class::

  class Foo:
      def __getattribute__(self, name):
          return self.__dict__.setdefault(name)

An appropriate stub definition is::

  from typing import Any

  class Foo:
      def __getattr__(self, name: str) -> Any | None: ...

Note that only ``__getattr__``, not ``__getattribute__``, is guaranteed to be
supported in stubs.

On the other hand, ``__getattr__`` should be omitted even if the source code
includes it, if only limited names are allowed. For example, consider this class::

  class ComplexNumber:
      def __init__(self, n):
          self._n = n
      def __getattr__(self, name):
          if name in ("real", "imag"):
              return getattr(self._n, name)
          raise AttributeError(name)

In this case, the stub should list the attributes individually::

  class ComplexNumber:
      @property
      def real(self) -> float: ...
      @property
      def imag(self) -> float: ...
      def __init__(self, n: complex) -> None: ...

``__setattr___`` should be included when a class allows any name to be set and
restricts the type. For example::

  class IntHolder:
      def __setattr__(self, name, value):
          if isinstance(value, int):
              return super().__setattr__(name, value)
          raise ValueError(value)

A good stub definition would be::

  class IntHolder:
      def __setattr__(self, name: str, value: int) -> None: ...

``__delattr__`` should not be included in stubs.

Finally, even in the presence of ``__getattr__`` and ``__setattr__``, it is
still recommended to separately define known attributes.

Constants
---------

When the value of a constant is important,  mark it as ``Final`` and assign it
to its value.

Yes::

    TEL_LANDLINE: Final = "landline"
    TEL_MOBILE: Final = "mobile"
    DAY_FLAG: Final = 0x01
    NIGHT_FLAG: Final = 0x02

No::

    TEL_LANDLINE: str
    TEL_MOBILE: str
    DAY_FLAG: int
    NIGHT_FLAG: int

Overloads
---------

All variants of overloaded functions and methods must have an ``@overload``
decorator. Do not include the implementation's final non-`@overload`-decorated
definition.

Yes::

  @overload
  def foo(x: str) -> str: ...
  @overload
  def foo(x: float) -> int: ...

No::

  @overload
  def foo(x: str) -> str: ...
  @overload
  def foo(x: float) -> int: ...
  def foo(x: str | float) -> Any: ...

Decorators
----------

Include only the decorators listed :ref:`here <stub-decorators>`, whose effects
are understood by all of the major type checkers. The behavior of other
decorators should instead be incorporated into the types. For example, for the
following function::

  import contextlib
  @contextlib.contextmanager
  def f():
      yield 42

the stub definition should be::

  from contextlib import AbstractContextManager
  def f() -> AbstractContextManager[int]: ...

Documentation or Implementation
-------------------------------

Sometimes a library's documented types will differ from the actual types in the
code. In such cases, stub authors should use their best judgment. Consider these
two examples::

  def print_elements(x):
      """Print every element of list x."""
      for y in x:
          print(y)

  def maybe_raise(x):
      """Raise an error if x (a boolean) is true."""
      if x:
          raise ValueError()

The implementation of ``print_elements`` takes any iterable, despite the
documented type of ``list``. In this case, annotate the argument as
``Iterable[object]``, to follow the :ref:`best practice<argument-return-practices>`
of preferring abstract types for arguments.

For ``maybe_raise``, on the other hand, it is better to annotate the argument as
``bool`` even though the implementation accepts any object. This guards against
common mistakes like unintentionally passing in ``None``.

If in doubt, consider asking the library maintainers about their intent.

Common Patterns
===============

.. _stub-patterns:

This section documents common patterns that are useful in stub files.

Overloads and Flags
-------------------

.. _overloads-and-flags:

Sometimes a function or method has a flag argument that changes the return type
or other accepted argument types. For example, take the following function::

  def open(name: str, mode: Literal["r", "w"] = "r") -> Reader | Writer:
      ...

We can express this case easily with two overloads::

  @overload
  def open(name: str, mode: Literal["r"] = "r") -> Reader: ...
  @overload
  def open(name: str, mode: Literal["w"]) -> Writer: ...

The first overload is picked when the mode is ``"r"`` or not given, and the
second overload is picked when the mode is ``"w"``. But what if the first
argument is optional?

::

  def open(name: str | None = None, mode: Literal["r", "w"] = "r") -> Reader | Writer:
      ...

Ideally we would be able to use the following overloads::

  @overload
  def open(name: str | None = None, mode: Literal["r"] = "r") -> Reader: ...
  @overload
  def open(name: str | None = None, mode: Literal["w"]) -> Writer: ...

And while the first overload is fine, the second is a syntax error in Python,
because non-default arguments cannot follow default arguments. To work around
this, we need an extra overload::

  @overload
  def open(name: str | None = None, mode: Literal["r"] = "r") -> Reader: ...
  @overload
  def open(name: str | None, mode: Literal["w"]) -> Writer: ...
  @overload
  def open(*, mode: Literal["w"]) -> Writer: ...

As before, the first overload is picked when the mode is ``"r"`` or not given.
Otherwise, the second overload is used when ``open`` is called with an explicit
``name``, e.g. ``open("file.txt", "w")`` or ``open(None, "w")``. The third
overload is used when ``open`` is called without a name , e.g.
``open(mode="w")``.

Style Guide
===========

The recommendations in this section are aimed at stub authors who wish to
provide a consistent style for stubs. Type checkers should not reject stubs that
do not follow these recommendations, but linters can warn about them.

Stub files should generally follow the Style Guide for Python Code (:pep:`8`)
and the :ref:`best-practices`. There are a few exceptions, outlined below, that take the
different structure of stub files into account and aim to create
more concise files.

Maximum Line Length
-------------------

Stub files should be limited to 130 characters per line.

Blank Lines
-----------

Do not use empty lines between functions, methods, and fields, except to
group them with one empty line. Use one empty line around classes with non-empty
bodies. Do not use empty lines between body-less classes, except for grouping.

Yes::

    def time_func() -> None: ...
    def date_func() -> None: ...

    def ip_func() -> None: ...

    class Foo:
        x: int
        y: int
        def __init__(self) -> None: ...

    class MyError(Exception): ...
    class AnotherError(Exception): ...

No::

    def time_func() -> None: ...

    def date_func() -> None: ...  # do no leave unnecessary empty lines

    def ip_func() -> None: ...


    class Foo:  # leave only one empty line above
        x: int
    class MyError(Exception): ...  # leave an empty line between the classes

Module Level Attributes
-----------------------

Do not unnecessarily use an assignment for module-level attributes.

Yes::

    CONST: Literal["const"]
    x: int
    y: Final = 0  # this assignment conveys additional type information

No::

    CONST = "const"
    x: int = 0
    y: float = ...
    z = 0  # type: int
    a = ...  # type: int

.. _stub-style-classes:

Classes
-------

Classes without bodies should use the ellipsis literal ``...`` in place
of the body on the same line as the class definition.

Yes::

    class MyError(Exception): ...

No::

    class MyError(Exception):
        ...
    class AnotherError(Exception): pass

Instance attributes and class variables follow the same recommendations as
module level attributes:

Yes::

    class Foo:
        c: ClassVar[str]
        x: int

    class Color(Enum):
        # An assignment with no type annotation is a convention used to indicate
	# an enum member.
        RED = 1

No::

    class Foo:
        c: ClassVar[str] = ""
        d: ClassVar[int] = ...
        x = 4
        y: int = ...

Functions and Methods
---------------------

For keyword-only and positional-or-keyword arguments, use the same
argument names as in the implementation, because otherwise using
keyword arguments will fail.

For default values, use the literal values of "simple" default values (``None``,
bools, ints, bytes, strings, and floats). Use the ellipsis literal ``...`` in
place of more complex default values. Use an explicit ``X | None`` annotation
when the default is ``None``.

Yes::

    def foo(x: int = 0) -> None: ...
    def bar(y: str | None = None) -> None: ...

No::

    def foo(x: X = X()) -> None: ...
    def bar(y: str = None) -> None: ...

Do not annotate ``self`` and ``cls`` in method definitions, except when
referencing a type variable.

Yes::

    _T = TypeVar("_T")

    class Foo:
        def bar(self) -> None: ...
        @classmethod
        def create(cls: type[_T]) -> _T: ...

No::

    class Foo:
        def bar(self: Foo) -> None: ...
        @classmethod
        def baz(cls: type[Foo]) -> int: ...

The bodies of functions and methods should consist of only the ellipsis
literal ``...`` on the same line as the closing parenthesis and colon.

Yes::

    def to_int1(x: str) -> int: ...
    def to_int2(
        x: str,
    ) -> int: ...

No::

    def to_int1(x: str) -> int:
        return int(x)
    def to_int2(x: str) -> int:
        ...
    def to_int3(x: str) -> int: pass

Language Features
-----------------

Use the latest language features available, even for stubs targeting older
Python versions. For example, Python 3.7 added the ``async`` keyword (see
:pep:`492`). Stubs should use it to mark coroutines, even if the implementation
still uses the ``@coroutine`` decorator. On the other hand, the ``type`` soft
keyword from :pep:`695`, introduced in Python 3.12, should not be used in stubs
until Python 3.11 reaches end-of-life in October 2027.

Do not use quotes around forward references and do not use ``__future__``
imports. See :ref:`stub-file-syntax` for more information.

Yes::

    class Py35Class:
        x: int
        forward_reference: OtherClass

    class OtherClass: ...

No::

    class Py35Class:
        x = 0  # type: int
        forward_reference: 'OtherClass'

    class OtherClass: ...

NamedTuple and TypedDict
------------------------

Use the class-based syntax for ``typing.NamedTuple`` and
``typing.TypedDict``, following the :ref:`stub-style-classes` section of this style guide.

Yes::

    from typing import NamedTuple, TypedDict

    class Point(NamedTuple):
        x: float
        y: float

    class Thing(TypedDict):
        stuff: str
        index: int

No::

    from typing import NamedTuple, TypedDict
    Point = NamedTuple("Point", [('x', float), ('y', float)])
    Thing = TypedDict("Thing", {'stuff': str, 'index': int})

Built-in Generics
-----------------

:pep:`585` built-in generics are supported and should be used instead
of the corresponding types from ``typing``::

    from collections import defaultdict

    def foo(t: type[MyClass]) -> list[int]: ...
    x: defaultdict[int]

Using imports from ``collections.abc`` instead of ``typing`` is
generally possible and recommended::

    from collections.abc import Iterable

    def foo(iter: Iterable[int]) -> None: ...

Unions
------

Declaring unions with the shorthand `|` syntax is recommended and supported by
all type checkers::

  def foo(x: int | str) -> int | None: ...  # recommended
  def foo(x: Union[int, str]) -> Optional[int]: ...  # ok
