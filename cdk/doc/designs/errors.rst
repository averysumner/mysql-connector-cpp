================
 Error handling
================

In CDK errors are reported by throwing exceptions that derive from ``cdk::Error`` class.
Design of ``Error`` class follows that of C++11 ``std::system_error``
(see http://en.cppreference.com/w/cpp/error).

Error description consists of numeric error code and a textual description. Error code
is platform-specific and might depend on the sub-system that generated given error or
the OS on which it was triggered. For example, if we get error because of refused
TCP/IP connection, the error code is assigned by OS TCP/IP stack and will be different
on Windows and on Linux.

Platform-specific error codes are mapped to platform-independent error conditions that
can be used by user-level code to handle errors in platform independent way. For example,
there is ``connection_refused`` error condition and code that wants to detect it could
look as follows::

  try
  {
    conn.connect();
  }
  catch (Error &e)
  {
     if (e.code() == errc::connection_refused)
       // connection was refused
  }

The platform-specific error code returned by ``e.code()`` is compared to
platform-independent ``errc::connection_refused`` error condition using the mapping from
error codes to error conditions. Error conditions have numeric values like error codes,
but these values do not depend on platform specifics.

.. uml::

  !include class.cnf

  !include designs/error.if

  class error_code {
    Platform-dependent numeric error code
    --
    error_code(int, error_category)
    ..
    value()    : int
    category() : error_category
    operator bool()
  }

  class error_category {
    Defines a set of error codes providing
    textual descriptions and mapping to
    platform-independent error conditions
    --
    name() : const char*
    message(int) : string
    default_error_condition(int) : error_condition
    equivalent(int, error_condition) : bool
  }

  class error_condition {
    User-level, platform-independent error value
    --
    value()    : int
    category() : error_category
    operator bool()
  }


Error codes and categories
==========================

Numeric error code is assigned to one of existing error categories. The
category determines how to interpret given error code. For example,
``system`` error code is an error code generated by operating system (such
as from Windows :func:`GetLastError` or Linux function call) while error
codes generated by a third-party library can have its own category to
distinguish them from system errors.

An error category XXX is described by instance of ``error_category`` class.
A reference to this error instance is returned by :func:`XXX_error_category`
function:

.. function:: error_category& cdk::XXX_error_category()

  Return reference to ``error_category`` instance describing error category XXX.

The ``error_category`` object defines textual description for each error code in
this category and the mapping from error codes to platform-independent error conditions.

Each error category has its name. When presenting error codes, they are shown as
"``CCC:NNN``" where ``CCC`` is the name of the category and ``NNN`` is the numeric
error code. For example, ``system:23``.

CDK uses the following error categories:

========= ========== =====================================================
Category  Name       Description
========= ========== =====================================================
generic    "cdk"      errors that do not fall into any other category
system     "system"   OS specific errors
posix      "posix"    errors generated by POSIX function calls (``errno``)
std        "errc"     category for standard error conditions (``errc::XXX``)
io         "cdk-io"   errors from I/O layer (``io_errc::XXX``)
========= ========== =====================================================


Error descriptions
==================

Apart from error code, ``Error`` object can provide more detailed textual description
of the error.

.. function:: void Error::describe(std::ostream &out)

  Write textual description of an error to given output stream. Default implementation
  of this method writes description of the form::

    Error code description (CCC:NNN)

  where "``Error code description``" is the textual description of the error code
  as defined by the error code category, ``CCC`` is the name of the error code category
  and ``NNN`` is the error code numeric value. For example (on Windows):
  "``No connection could be made because the target machine actively refused it
  (system:10061)``"

.. function:: string Error::description()

  Returns description produced by :func:`Error::describe` in a string object. This string
  object is owned by ``Error`` instance.


Method :func:`Error::what` returns error description as defined by :func:`describe`,
prefixed with "``CDK Error:``", as utf-8 string.

There is also convenience operator ``<<`` that allows sending ``Error`` instance to
``std::ostream`` using :func:`Error::describe`.


Error conditions
================

CDK uses two categories of error conditions: standard error conditions as defined
by C++11 (http://en.cppreference.com/w/cpp/error/errc) and CDK specific error
conditions. Standard conditions use standard error category (``std``/"``errc``")
while CDK specific error conditions use generic error category (``generic``/"``cdk``").

Constants for standard error conditions are defined in ``errc`` namespace, for
example ``errc::connection_refused``. Constants for CDK specific error conditions
are defined in ``cdkerrc`` namespace (see below). The following ``error_condition``
constructors are defined:

.. function:: error_condtion::error_condition(cdkerrc::code code)

  Construct CDK specific error condition from ``cdkerrc::XXX`` constant
  (``cdkerrc::code`` is an enumeration type of these constants).

.. function:: error_condition::error_condtion(errc::code code)

  Construct standard error condition from ``errc::XXX`` constant (``errc::code``
  is an enumeration type of these constants).

.. function:: error_condition::error_condition(int code)

  Construct standard error condition with given numeric code. Equivalent to
  ``error_condition(errc(code))``.

Operator ``==`` is overloaded so that error codes are correctly matched to error
conditions using the mappings defined by error categories.


Error conditions specific to CDK
--------------------------------
Constants for these error conditions are defined in ``cdk::foundation::cdkerrc``
namespace.

================================ ===== ====================
 Name                            Code  Description string
================================ ===== ====================
``cdkerrc::generic_error``         1    "Generic CDK error"
``cdkerrc::standard_exception``    2    "Standard exception"
``cdkerrc::unknown_exception``     3    "Unknown exception"
``cdkerrc::boost_error``           4    "Boost error"
================================ ===== ====================

Details:

:generic_error: Error condition for errors that do not have any more specific error
  code. For example errors that are thrown with ``throw_error("Description")`` map
  to this error condition.

:standard_exception: Error condition for CDK errors generated by :func:`rethrow_error`
  that wrap standard exceptions (``std::exception``).

:unknown_exception: Error condition for CDK errors generated by :func:`rethrow_error`
  which wrap an exception object that was unknown to CDK.

:boost_error: Error condition for CDK errors that wrap Boost errors whose error
  code can not be mapped to CDK error code. If mapping is possible, then
  wrapping CDK error will have error code corresponding to the Boost error code
  and will map to the same error conditions as the wrapped Boost error.


Functions for generating errors
===============================

.. function:: void cdk::throw_error(const string &description)

  Throw error with error code ``cdkerrc::generic_error`` in ``cdk`` category
  and with given textual description. Error description generated by :func:`Error::describe`
  will be the ``description`` string given as function argument.


.. function:: void cdk::throw_system_error(const string &prefix)
              void cdk::throw_system_error()

  If last system function call reported error (via system's "last error" mechanism)
  these functions throw corresponding CDK error in ``system`` category. Description
  of this error has the form::

    Prefix: System error description (system:NNN)

  where ``Prefix`` is given by function argument, ``NNN`` is system error code
  and ``System error description`` is this error description as defined by ``system``
  error category. Second variant of this function does not add any prefix to error
  description.

.. function:: void cdk::throw_system_error(int code, const string &prefix)
              void cdk::throw_system_error(int code)

  Like above functions but system error code is given as function argument. If
  error code is 0 then no error is thrown.


.. function:: void cdk::throw_posix_error(const string &prefix)
              void cdk::throw_posix_error()

  If ``errno`` is not 0 then these functions throw corresponding CDK error in
  ``posix`` category. Description of this error has the form::

    Prefix: POSIX error description (posix:NNN)

  where ``Prefix`` is given by function argument, ``NNN`` is the numeric error
  code and ``POSIX error description`` is the corresponding description as defined
  by ``posix`` error category. Second variant of this function does not add any prefix
  to error description.

.. function:: void cdk::throw_posix_error(int code, const string &prefix)
              void cdk::throw_posix_error(int code)

  Like above functions but POSIX error code is given as function argument. If
  error code is 0 then no error is thrown.


.. function:: void cdk::thorw_error(const error_code &ec, const string &prefix)
              void cdk::throw_error(const error_code &ec)
              void cdk::throw_error(int code, const error_category &ec)

  Throw error with given error code and with description of the form::

    Prefix: Error code description (CCC:NNN)

  where ``Prefix`` is given by function argument, ``CCC`` and ``NNN`` are category
  name and numeric value of the error code. Second and third variant do not add
  any prefix.


.. function:: void cdk::rethrow_error(const string &prefix)
              void cdk::rethrow_error()

  This function can be used inside ``catch()`` block. It re-throws current exception
  after wrapping it in ``cdk::Error`` instance. This ``cdk::Error`` will have error
  code appropriate for the exception being re-thrown:

  ============================ =================
  Type of re-thrown exception  Error code
  ============================ =================
  ``cdk::Error``                the same as in the original error
   Boost error                  Boost error code mapped to CDK one
  ``std::exception``           ``cdkerrc::standard_exception``
   other                       ``cdkerrc::unknown_exception``
  ============================ =================

  When re-throwing Boost errors, boost error code is mapped to CDK error code
  if boost error category corresponds to a CDK category (e.g., Boost ``system``
  category corresponds to CDK ``system`` category). If there is no corresponding
  CDK error category then boost error code maps to ``cdkerrc::boost_error``.

  Description of a wrapped exception has the general form::

    Prefix: Exception description

  where ``Prefix`` is given by function argument and ``Exception description`` is
  a description of the wrapped exception: for example, for exceptions of type
  ``std::exception`` it is string "``Standard exception:``"
  followed by result of exception's :func:`what` method. If exception type is not
  recognized, its description will be "``Unknown exception``".

  Second variant of this function does not add any prefix to ``Exception description``.


Cloning errors
--------------

Function :func:`rethrow_error` works like this:

 1. catch current exception/error,
 2. make a copy of it and store it in new ``Extended_error`` object which adds
    a prefix to the base error,
 3. throw this new extended error.

The ``Extended_error`` object can not store a pointer or reference to the base
error object, because this base object goes out of scope when ``Extended_error``
is thrown in step 3. This is why a copy of the base error must be created. For this
``Error`` class defines virtual method:

.. function:: Error* Error::clone() const

  Produce a copy of itself. For error classes derived from base ``Error`` class
  this copy should be of the derived class, not ``Error`` one.

To correctly work with :func:`rethrow_error`, an error class ``EEE`` which derives
from ``Error`` should implement :func:`clone` as follows::

  Error* EEE::clone() const
  {
    return new EEE(*this);
  }

That is, use copy constructor to produce exact copy of itself. Defining :func:`clone`
method can be automated using ``Error_class`` template. It is enough to define
derived error class as follows::

  class EEE : public Error_class<EEE>
  {
    ...
  };

Template ``Error_class`` takes care of defining correct :func:`clone` method. It
also inherits from ``Error`` so that ``EEE`` is also a specialization of ``Error``.


.. _Diagnostics:

Diagnostics interface
=====================
Diagnostics interface is used to examine accumulated diagnostics information.
This information is stored in diagnostic entries, which extend :class:`Error` class but also contain severity level such as ``ERROR``, ``WARNING``, ``INFO`` etc.

.. uml::

  !include class.cnf

  !include designs/iterator.if
  !include designs/error.if!0
  !include designs/diagnostics.if!0
  !include designs/diagnostics.if!1

Default description of a diagnostics entry is of the form ``SSS: Error
description`` where ``SSS`` is the severity level and ``Error description`` is
description as produced by :class:`Error` class. Method :func:`what` adds
``CDK`` prefix to above description. For example::

  CDK Warning: file not found (system:3)

Possible severity levels are defined by ``Diagnostics::Level`` enumeration.

.. function:: uint Diagnostics::entry_count(Level level =ERROR)

   Return number of diagnostic entries with given severity level (defaults to
   ``ERROR``).

.. function:: Iterator Diagnostics::get_entries(Level level =ERROR)

   Get an iterator over diagnostic entries with severity level above or equal
   to the given one (for example, if level is ``WARNING`` then iterates over
   all warnings and errors). By default returns iterator over errors only. The
   :class:`Diagnostics::Iterator` interface extends :class:`Iterator` one with
   single :func:`entry` method that returns the current entry from the
   sequence.

.. function:: Error Diagnostics::get_error()

   Convenience method to return first error entry (if any). Equivalent to
   ``get_entries(ERROR).entry()``. Note that this method can throw exception
   if there is no error available.


Defining new error classes
==========================

New error classes can be defined by deriving from one of existing error classes:

:Error: Base error class with the following constructors:

  .. function:: Error::Error(int code)

    Error with given error code in generic error category. Error description is
    of the form ``Error code description (cdk:NNN)`` where ``NNN`` is the given
    error code. ``Error code description`` is defined by the generic error
    category.

  .. function:: Error::Error(const error_code &ec)

    Error with given error code. Error description is of the form
    ``Error code description (CCC:NNN)`` where ``CCC`` and ``NNN`` are,
    respectively, category and value of the error code. ``Error code description``
    is given by ``ec.message()``.

  .. function:: Error::Error(const error_code &ec, const string &descr)

    Error with given error code and description (note: error code will not
    be added to the description).

:Generic_error: Class of generic errors whose error code is
   cdkerrc::generic_error and description is given during error construction.

   .. function:: Generic_error::Generic_error(const string &descr)

     Generic error with given description.

:Extended_error: An error which wraps another error and adds prefix to its
  description.

  .. function:: Extened_error::Extended_error(const Error &base, \
                                              const string &prefix)

    Extended error whose description will be of the form
    ``Prefix: Description`` where ``Description`` is the description of the
    base error.


When specializing one of existing error classes, ``Error_class`` template
can be used to ensure that :func:`clone` method is correctly defined. Second
template argument specifies the base error class that is being specialized.
For example::

  class My_error : public Error_class<My_error, Extended_error>
  {
    public:

      My_error(const Error &e)
        : Error_base(NULL, e, "My prefix")
      {}
  };

Note that ``Error_class`` template defines ``Error_base`` type to make it
easier to pass arguments to the base constructor. Base constructor invocation
``Error_base(NULL, e, "My prefix")`` will pass ``e`` and ``"My prefix"`` as
arguments for the constructor of base ``Extended_error`` class. The extra
``NULL`` argument is needed to avoid confusion of this form of constructor with
copy constructor.


Defining error category
=======================

CDK contains infrastructure for defining error categories: a list of error codes
with descriptions and defined mapping to platform-independent error conditions.

When defining new category ``foo``, list of errors in this category is defined using
macro of the form::

  #define EC_foo_ERRORS(X) \
    CDK_ERROR(X, first_error, 1, "First error description") \
    ...

Each :func:`CDK_ERROR` line defines one error in the category, with symbolic name,
numeric value and error description. Given such error list, the corresponding
category can be defined using macro::

  CDK_ERROR_CATEGORY(foo, foo_errc)

The first parameter is the name of the category (and should match the name used in
:func:`EC_foo_ERRORS` macro) and second parameter is name of namespace where error
constants will be defined. Thus, with above examples, :func:`CDK_ERROR_CATEGORY` macro
will define ``foo_errc::first_error`` constant corresponding to the first error defined
in the error list. Apart from error constants, :func:`CDK_ERROR_CATEGORY` macro
defines the following:

.. function:: const error_category& foo_error_category()

  Returns instance of ``error_category`` class which defines category ``foo``.

.. function:: error_code foo_error(int code)

  Returns error code in ``foo`` category with given numeric value.

Macro :func:`CDK_ERROR_CATEGORY` also defines ``error_category_foo`` class which
implements the new category. To complete its definition, user must define
:func:`error_category_foo::default_error_condition`
and :func:`error_category_foo::equivalent`
methods that determine mapping to error conditions.

