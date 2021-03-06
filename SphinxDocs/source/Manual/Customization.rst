Customization Features
=========================

In many cases, it is desirable to change the default wrapping of
particular declarations in an interface. For example, you might want to
provide hooks for catching C++ exceptions, add assertions, or provide
hints to the underlying code generator. This chapter describes some of
these customization techniques. First, a discussion of exception
handling is presented. Then, a more general-purpose customization
mechanism known as "features" is described.

Exception handling with %exception
---------------------------------------

The ``%exception`` directive allows you to define a general purpose
exception handler. For example, you can specify the following:

.. container:: code

   ::

      %exception {
        try {
          $action
        }
        catch (RangeError) {
          ... handle error ...
        }
      }

How the exception is handled depends on the target language, for
example, Python:

.. container:: code

   ::

      %exception {
        try {
          $action
        }
        catch (RangeError) {
          PyErr_SetString(PyExc_IndexError, "index out-of-bounds");
          SWIG_fail;
        }
      }

When defined, the code enclosed in braces is inserted directly into the
low-level wrapper functions. The special variable ``$action`` is one of
a few `%exception special
variables <Customization.html#Customization_exception_special_variables>`__
supported and gets replaced with the actual operation to be performed (a
function call, method invocation, attribute access, etc.). An exception
handler remains in effect until it is explicitly deleted. This is done
by using either ``%exception`` or ``%noexception`` with no code. For
example:

.. container:: code

   ::

      %exception;   // Deletes any previously defined handler

**Compatibility note:** Previous versions of SWIG used a special
directive ``%except`` for exception handling. That directive is
deprecated--``%exception`` provides the same functionality, but is
substantially more flexible.

Handling exceptions in C code
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

C has no formal exception handling mechanism so there are several
approaches that might be used. A somewhat common technique is to simply
set a special error code. For example:

.. container:: code

   ::

      /* File : except.c */

      static char error_message[256];
      static int error_status = 0;

      void throw_exception(char *msg) {
        strncpy(error_message, msg, 256);
        error_status = 1;
      }

      void clear_exception() {
        error_status = 0;
      }
      char *check_exception() {
        if (error_status)
          return error_message;
        else
          return NULL;
      }

To use these functions, functions simply call ``throw_exception()`` to
indicate an error occurred. For example :

.. container:: code

   ::

      double inv(double x) {
        if (x != 0)
          return 1.0/x;
        else {
          throw_exception("Division by zero");
          return 0;
        }
      }

To catch the exception, you can write a simple exception handler such as
the following (shown for Perl5) :

.. container:: code

   ::

      %exception {
        char *err;
        clear_exception();
        $action
        if ((err = check_exception())) {
          croak(err);
        }
      }

In this case, when an error occurs, it is translated into a Perl error.
Each target language has its own approach to creating a runtime
error/exception in and for Perl it is the ``croak`` method shown above.

Exception handling with longjmp()
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Exception handling can also be added to C code using the ``<setjmp.h>``
library. Here is a minimalistic implementation that relies on the C
preprocessor :

.. container:: code

   ::

      /* File : except.c
         Just the declaration of a few global variables we're going to use */

      #include <setjmp.h>
      jmp_buf exception_buffer;
      int exception_status;

      /* File : except.h */
      #include <setjmp.h>
      extern jmp_buf exception_buffer;
      extern int exception_status;

      #define try if ((exception_status = setjmp(exception_buffer)) == 0)
      #define catch(val) else if (exception_status == val)
      #define throw(val) longjmp(exception_buffer, val)
      #define finally else

      /* Exception codes */

      #define RangeError     1
      #define DivisionByZero 2
      #define OutOfMemory    3

Now, within a C program, you can do the following :

.. container:: code

   ::

      double inv(double x) {
        if (x)
          return 1.0/x;
        else
          throw(DivisionByZero);
      }

Finally, to create a SWIG exception handler, write the following :

.. container:: code

   ::

      %{
      #include "except.h"
      %}

      %exception {
        try {
          $action
        } catch(RangeError) {
          croak("Range Error");
        } catch(DivisionByZero) {
          croak("Division by zero");
        } catch(OutOfMemory) {
          croak("Out of memory");
        } finally {
          croak("Unknown exception");
        }
      }

Note: This implementation is only intended to illustrate the general
idea. To make it work better, you'll need to modify it to handle nested
``try`` declarations.

Handling C++ exceptions
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Handling C++ exceptions is also straightforward. For example:

.. container:: code

   ::

      %exception {
        try {
          $action
        } catch(RangeError) {
          croak("Range Error");
        } catch(DivisionByZero) {
          croak("Division by zero");
        } catch(OutOfMemory) {
          croak("Out of memory");
        } catch(...) {
          croak("Unknown exception");
        }
      }

The exception types need to be declared as classes elsewhere, possibly
in a header file :

.. container:: code

   ::

      class RangeError {};
      class DivisionByZero {};
      class OutOfMemory {};

Exception handlers for variables
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

By default all variables will ignore ``%exception``, so it is
effectively turned off for all variables wrappers. This applies to
global variables, member variables and static member variables. The
approach is certainly a logical one when wrapping variables in C.
However, in C++, it is quite possible for an exception to be thrown
while the variable is being assigned. To ensure ``%exception`` is used
when wrapping variables, it needs to be 'turned on' using the
``%allowexception`` feature. Note that ``%allowexception`` is just a
macro for ``%feature("allowexcept")``, that is, it is a feature called
"allowexcept". Any variable which has this feature attached to it, will
then use the ``%exception`` feature, but of course, only if there is a
``%exception`` attached to the variable in the first place. The
``%allowexception`` feature works like any other feature and so can be
used globally or for selective variables.

.. container:: code

   ::

      %allowexception;                // turn on globally
      %allowexception Klass::MyVar;   // turn on for a specific variable

      %noallowexception Klass::MyVar; // turn off for a specific variable
      %noallowexception;              // turn off globally

Defining different exception handlers
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

By default, the ``%exception`` directive creates an exception handler
that is used for all wrapper functions that follow it. Unless there is a
well-defined (and simple) error handling mechanism in place, defining
one universal exception handler may be unwieldy and result in excessive
code bloat since the handler is inlined into each wrapper function.

To fix this, you can be more selective about how you use the
``%exception`` directive. One approach is to only place it around
critical pieces of code. For example:

.. container:: code

   ::

      %exception {
        ... your exception handler ...
      }
      /* Define critical operations that can throw exceptions here */

      %exception;

      /* Define non-critical operations that don't throw exceptions */

More precise control over exception handling can be obtained by
attaching an exception handler to specific declaration name. For
example:

.. container:: code

   ::

      %exception allocate {
        try {
          $action
        }
        catch (MemoryError) {
          croak("Out of memory");
        }
      }

In this case, the exception handler is only attached to declarations
named "allocate". This would include both global and member functions.
The names supplied to ``%exception`` follow the same rules as for
``%rename`` described in the section on `Renaming and ambiguity
resolution <SWIGPlus.html#SWIGPlus_ambiguity_resolution_renaming>`__.
For example, if you wanted to define an exception handler for a specific
class, you might write this:

.. container:: code

   ::

      %exception Object::allocate {
        try {
          $action
        }
        catch (MemoryError) {
          croak("Out of memory");
        }
      }

When a class prefix is supplied, the exception handler is applied to the
corresponding declaration in the specified class as well as for
identically named functions appearing in derived classes.

``%exception`` can even be used to pinpoint a precise declaration when
overloading is used. For example:

.. container:: code

   ::

      %exception Object::allocate(int) {
        try {
          $action
        }
        catch (MemoryError) {
          croak("Out of memory");
        }
      }

Attaching exceptions to specific declarations is a good way to reduce
code bloat. It can also be a useful way to attach exceptions to specific
parts of a header file. For example:

.. container:: code

   ::

      %module example
      %{
      #include "someheader.h"
      %}

      // Define a few exception handlers for specific declarations
      %exception Object::allocate(int) {
        try {
          $action
        }
        catch (MemoryError) {
          croak("Out of memory");
        }
      }

      %exception Object::getitem {
        try {
          $action
        }
        catch (RangeError) {
          croak("Index out of range");
        }
      }
      ...
      // Read a raw header file
      %include "someheader.h"

**Compatibility note:** The ``%exception`` directive replaces the
functionality provided by the deprecated "except" typemap. The typemap
would allow exceptions to be thrown in the target language based on the
return type of a function and was intended to be a mechanism for
pinpointing specific declarations. However, it never really worked that
well and the new %exception directive is much better.

Special variables for %exception
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The %exception directive supports a few special variables which are
placeholders for code substitution. The following table shows the
available special variables and details what the special variables are
replaced with.

+---------------------+-----------------------------------------------+
| $action             | The actual operation to be performed (a       |
|                     | function call, method invocation, variable    |
|                     | access, etc.)                                 |
+---------------------+-----------------------------------------------+
| $name               | The C/C++ symbol name for the function.       |
+---------------------+-----------------------------------------------+
| $symname            | The symbol name used internally by SWIG       |
+---------------------+-----------------------------------------------+
| $overname           | The extra mangling used in the symbol name    |
|                     | for overloaded method. Expands to nothing if  |
|                     | the wrapped method is not overloaded.         |
+---------------------+-----------------------------------------------+
| $wrapname           | The language specific wrapper name (usually a |
|                     | C function name exported from the shared      |
|                     | object/dll)                                   |
+---------------------+-----------------------------------------------+
| $decl               | The fully qualified C/C++ declaration of the  |
|                     | method being wrapped without the return type  |
+---------------------+-----------------------------------------------+
| $fulldecl           | The fully qualified C/C++ declaration of the  |
|                     | method being wrapped including the return     |
|                     | type                                          |
+---------------------+-----------------------------------------------+
| $parentclassname    | The parent class name (if any) for a method.  |
+---------------------+-----------------------------------------------+
| $parentclasssymname | The target language parent class name (if     |
|                     | any) for a method.                            |
+---------------------+-----------------------------------------------+

The special variables are often used in situations where method calls
are logged. Exactly which form of the method call needs logging is up to
individual requirements, but the example code below shows all the
possible expansions, plus how an exception message could be tailored to
show the C++ method declaration:

.. container:: code

   ::

      %exception Special::something {
        log("symname: $symname");
        log("overname: $overname");
        log("wrapname: $wrapname");
        log("decl: $decl");
        log("fulldecl: $fulldecl");
        try {
          $action
        } 
        catch (MemoryError) {
          croak("Out of memory in $decl");
        }
      }
      void log(const char *message);
      struct Special {
        void something(const char *c);
        void something(int i);
      };

Below shows the expansions for the 1st of the overloaded ``something``
wrapper methods for Perl:

.. container:: code

   ::

        log("symname: Special_something");
        log("overname: __SWIG_0");
        log("wrapname: _wrap_Special_something__SWIG_0");
        log("decl: Special::something(char const *)");
        log("fulldecl: void Special::something(char const *)");
        try {
          (arg1)->something((char const *)arg2);
        } 
        catch (MemoryError) {
          croak("Out of memory in Special::something(char const *)");
        }

Using The SWIG exception library
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

The ``exception.i`` library file provides support for creating language
independent exceptions in your interfaces. To use it, simply put an
"``%include exception.i``" in your interface file. This provides a
function ``SWIG_exception()`` that can be used to raise common scripting
language exceptions in a portable manner. For example :

.. container:: code

   ::

      // Language independent exception handler
      %include exception.i       

      %exception {
        try {
          $action
        } catch(RangeError) {
          SWIG_exception(SWIG_ValueError, "Range Error");
        } catch(DivisionByZero) {
          SWIG_exception(SWIG_DivisionByZero, "Division by zero");
        } catch(OutOfMemory) {
          SWIG_exception(SWIG_MemoryError, "Out of memory");
        } catch(...) {
          SWIG_exception(SWIG_RuntimeError, "Unknown exception");
        }
      }

As arguments, ``SWIG_exception()`` takes an error type code (an integer)
and an error message string. The currently supported error types are :

.. container:: diagram

   ::

      SWIG_UnknownError
      SWIG_IOError
      SWIG_RuntimeError
      SWIG_IndexError
      SWIG_TypeError
      SWIG_DivisionByZero
      SWIG_OverflowError
      SWIG_SyntaxError
      SWIG_ValueError
      SWIG_SystemError
      SWIG_AttributeError
      SWIG_MemoryError
      SWIG_NullReferenceError

The ``SWIG_exception()`` function can also be used in typemaps.

Object ownership and %newobject
------------------------------------

A common problem in some applications is managing proper ownership of
objects. For example, consider a function like this:

.. container:: code

   ::

      Foo *blah() {
        Foo *f = new Foo();
        return f;
      }

If you wrap the function ``blah()``, SWIG has no idea that the return
value is a newly allocated object. As a result, the resulting extension
module may produce a memory leak (SWIG is conservative and will never
delete objects unless it knows for certain that the returned object was
newly created).

To fix this, you can provide an extra hint to the code generator using
the ``%newobject`` directive. For example:

.. container:: code

   ::

      %newobject blah;
      Foo *blah();

``%newobject`` works exactly like ``%rename`` and ``%exception``. In
other words, you can attach it to class members and parameterized
declarations as before. For example:

.. container:: code

   ::

      %newobject ::blah();                   // Only applies to global blah
      %newobject Object::blah(int, double);  // Only blah(int, double) in Object
      %newobject *::copy;                    // Copy method in all classes
      ...

When ``%newobject`` is supplied, many language modules will arrange to
take ownership of the return value. This allows the value to be
automatically garbage-collected when it is no longer in use. However,
this depends entirely on the target language (a language module may also
choose to ignore the ``%newobject`` directive).

Closely related to ``%newobject`` is a special typemap. The "newfree"
typemap can be used to deallocate a newly allocated return value. It is
only available on methods for which ``%newobject`` has been applied and
is commonly used to clean-up string results. For example:

.. container:: code

   ::

      %typemap(newfree) char * "free($1);";
      ...
      %newobject strdup;
      ...
      char *strdup(const char *s);

In this case, the result of the function is a string in the target
language. Since this string is a copy of the original result, the data
returned by ``strdup()`` is no longer needed. The "newfree" typemap in
the example simply releases this memory.

As a complement to the ``%newobject``, from SWIG 1.3.28, you can use the
``%delobject`` directive. For example, if you have two methods, one to
create objects and one to destroy them, you can use:

.. container:: code

   ::

      %newobject create_foo;
      %delobject destroy_foo;
      ...
      Foo *create_foo();
      void destroy_foo(Foo *foo);

or in a member method as:

.. container:: code

   ::

      %delobject Foo::destroy;

      class Foo {
      public:
        void destroy() { delete this;}

      private:
        ~Foo();
      };

``%delobject`` instructs SWIG that the first argument passed to the
method will be destroyed, and therefore, the target language should not
attempt to deallocate it twice. This is similar to use the DISOWN
typemap in the first method argument, and in fact, it also depends on
the target language on implementing the 'disown' mechanism properly.

The use of ``%newobject`` is also integrated with reference counting and
is covered in the `C++ reference counted
objects <SWIGPlus.html#SWIGPlus_ref_unref>`__ section.

**Compatibility note:** Previous versions of SWIG had a special ``%new``
directive. However, unlike ``%newobject``, it only applied to the next
declaration. For example:

.. container:: code

   ::

      %new char *strdup(const char *s);

For now this is still supported but is deprecated.

**How to shoot yourself in the foot:** The ``%newobject`` directive is
not a declaration modifier like the old ``%new`` directive. Don't write
code like this:

.. container:: code

   ::

      %newobject
      char *strdup(const char *s);

The results might not be what you expect.

Features and the %feature directive
----------------------------------------

Both ``%exception`` and ``%newobject`` are examples of a more general
purpose customization mechanism known as "features." A feature is simply
a user-definable property that is attached to specific declarations.
Features are attached using the ``%feature`` directive. For example:

.. container:: code

   ::

      %feature("except") Object::allocate {
        try {
          $action
        }
        catch (MemoryError) {
          croak("Out of memory");
        }
      }

      %feature("new", "1") *::copy;

In fact, the ``%exception`` and ``%newobject`` directives are really
nothing more than macros involving ``%feature``:

.. container:: code

   ::

      #define %exception %feature("except")
      #define %newobject %feature("new", "1")

The name matching rules outlined in the `Renaming and ambiguity
resolution <SWIGPlus.html#SWIGPlus_ambiguity_resolution_renaming>`__
section applies to all ``%feature`` directives. In fact the ``%rename``
directive is just a special form of ``%feature``. The matching rules
mean that features are very flexible and can be applied with pinpoint
accuracy to specific declarations if needed. Additionally, if no
declaration name is given, a global feature is said to be defined. This
feature is then attached to *every* declaration that follows. This is
how global exception handlers are defined. For example:

.. container:: code

   ::

      /* Define a global exception handler */
      %feature("except") {
        try {
          $action
        }
        ...
      }

      ... bunch of declarations ...

The ``%feature`` directive can be used with different syntax. The
following are all equivalent:

.. container:: code

   ::

      %feature("except") Object::method { $action };
      %feature("except") Object::method %{ $action %};
      %feature("except") Object::method " $action ";
      %feature("except", "$action") Object::method;

The syntax in the first variation will generate the ``{ }`` delimiters
used whereas the other variations will not.

Feature attributes
~~~~~~~~~~~~~~~~~~~~~~~~~

The ``%feature`` directive also accepts XML style attributes in the same
way that typemaps do. Any number of attributes can be specified. The
following is the generic syntax for features:

.. container:: code

   ::

      %feature("name", "value", attribute1="AttributeValue1") symbol;
      %feature("name", attribute1="AttributeValue1") symbol {value};
      %feature("name", attribute1="AttributeValue1") symbol %{value%};
      %feature("name", attribute1="AttributeValue1") symbol "value";

More than one attribute can be specified using a comma separated list.
The Java module is an example that uses attributes in
``%feature("except")``. The ``throws`` attribute specifies the name of a
Java class to add to a proxy method's throws clause. In the following
example, ``MyExceptionClass`` is the name of the Java class for adding
to the throws clause.

.. container:: code

   ::

      %feature("except", throws="MyExceptionClass") Object::method { 
        try {
          $action
        } catch (...) {
          ... code to throw a MyExceptionClass Java exception ...
        }
      };

Further details can be obtained from the `Java exception
handling <Java.html#Java_exception_handling>`__ section.

Feature flags
~~~~~~~~~~~~~~~~~~~~

Feature flags are used to enable or disable a particular feature.
Feature flags are a common but simple usage of ``%feature`` and the
feature value should be either ``1`` to enable or ``0`` to disable the
feature.

.. container:: code

   ::

      %feature("featurename")          // enables feature
      %feature("featurename", "1")     // enables feature
      %feature("featurename", "x")     // enables feature
      %feature("featurename", "0")     // disables feature
      %feature("featurename", "")      // clears feature

Actually any value other than zero will enable the feature. Note that if
the value is omitted completely, the default value becomes ``1``,
thereby enabling the feature. A feature is cleared by specifying no
value, see `Clearing features <#Customization_clearing_features>`__. The
``%immutable`` directive described in the `Creating read-only
variables <SWIG.html#SWIG_readonly_variables>`__ section, is just a
macro for ``%feature("immutable")``, and can be used to demonstrates
feature flags:

.. container:: code

   ::

                                      // features are disabled by default
      int red;                        // mutable

      %feature("immutable");          // global enable
      int orange;                     // immutable

      %feature("immutable", "0");     // global disable
      int yellow;                     // mutable

      %feature("immutable", "1");     // another form of global enable
      int green;                      // immutable

      %feature("immutable", "");      // clears the global feature
      int blue;                       // mutable

Note that features are disabled by default and must be explicitly
enabled either globally or by specifying a targeted declaration. The
above intersperses SWIG directives with C code. Of course you can target
features explicitly, so the above could also be rewritten as:

.. container:: code

   ::

      %feature("immutable", "1") orange;
      %feature("immutable", "1") green;
      int red;                        // mutable
      int orange;                     // immutable
      int yellow;                     // mutable
      int green;                      // immutable
      int blue;                       // mutable

The above approach allows for the C declarations to be separated from
the SWIG directives for when the C declarations are parsed from a C
header file. The logic above can of course be inverted and rewritten as:

.. container:: code

   ::

      %feature("immutable", "1");
      %feature("immutable", "0") red;
      %feature("immutable", "0") yellow;
      %feature("immutable", "0") blue;
      int red;                        // mutable
      int orange;                     // immutable
      int yellow;                     // mutable
      int green;                      // immutable
      int blue;                       // mutable

As hinted above for ``%immutable``, most feature flags can also be
specified via alternative syntax. The alternative syntax is just a macro
in the ``swig.swg`` Library file. The following shows the alternative
syntax for the imaginary ``featurename`` feature:

.. container:: code

   ::

      %featurename       // equivalent to %feature("featurename", "1") ie enables feature
      %nofeaturename     // equivalent to %feature("featurename", "0") ie disables feature
      %clearfeaturename  // equivalent to %feature("featurename", "")  ie clears feature

The concept of clearing features is discussed next.

Clearing features
~~~~~~~~~~~~~~~~~~~~~~~~

A feature stays in effect until it is explicitly cleared. A feature is
cleared by supplying a ``%feature`` directive with no value. For example
``%feature("name", "")``. A cleared feature means that any feature
exactly matching any previously defined feature is no longer used in the
name matching rules. So if a feature is cleared, it might mean that
another name matching rule will apply. To clarify, let's consider the
``except`` feature again (``%exception``):

.. container:: code

   ::

      // Define global exception handler
      %feature("except") {
        try {
          $action
        } catch (...) {
          croak("Unknown C++ exception");
        }
      }

      // Define exception handler for all clone methods to log the method calls
      %feature("except") *::clone() {
        try {
          logger.info("$action");
          $action
        } catch (...) {
          croak("Unknown C++ exception");
        }
      }

      ... initial set of class declarations with clone methods ...

      // clear the previously defined feature
      %feature("except", "") *::clone();

      ... final set of class declarations with clone methods ...

In the above scenario, the initial set of clone methods will log all
method invocations from the target language. This specific feature is
cleared for the final set of clone methods. However, these clone methods
will still have an exception handler (without logging) as the next best
feature match for them is the global exception handler.

Note that clearing a feature is not always the same as disabling it.
Clearing the feature above with ``%feature("except", "") *::clone()`` is
not the same as specifying ``%feature("except", "0") *::clone()``. The
former will disable the feature for clone methods - the feature is still
a better match than the global feature. If on the other hand, no global
exception handler had been defined at all, then clearing the feature
would be the same as disabling it as no other feature would have
matched.

Note that the feature must match exactly for it to be cleared by any
previously defined feature. For example the following attempt to clear
the initial feature will not work:

.. container:: code

   ::

      %feature("except") clone() { logger.info("$action"); $action }
      %feature("except", "") *::clone();

but this will:

.. container:: code

   ::

      %feature("except") clone() { logger.info("$action"); $action }
      %feature("except", "") clone();

SWIG provides macros for disabling and clearing features. Many of these
can be found in the ``swig.swg`` library file. The typical pattern is to
define three macros; one to define the feature itself, one to disable
the feature and one to clear the feature. The three macros below show
this for the "except" feature:

.. container:: code

   ::

      #define %exception      %feature("except")
      #define %noexception    %feature("except", "0")
      #define %clearexception %feature("except", "")

Features and default arguments
~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~~

SWIG treats methods with default arguments as separate overloaded
methods as detailed in the `default
arguments <SWIGPlus.html#SWIGPlus_default_args>`__ section. Any
``%feature`` targeting a method with default arguments will apply to all
the extra overloaded methods that SWIG generates if the default
arguments are specified in the feature. If the default arguments are not
specified in the feature, then the feature will match that exact wrapper
method only and not the extra overloaded methods that SWIG generates.
For example:

.. container:: code

   ::

      %feature("except") hello(int i=0, double d=0.0) { ... }
      void hello(int i=0, double d=0.0);

will apply the feature to all three wrapper methods, that is:

.. container:: code

   ::

      void hello(int i, double d);
      void hello(int i);
      void hello();

If the default arguments are not specified in the feature:

.. container:: code

   ::

      %feature("except") hello(int i, double d) { ... }
      void hello(int i=0, double d=0.0);

then the feature will only apply to this wrapper method:

.. container:: code

   ::

      void hello(int i, double d);

and not these wrapper methods:

.. container:: code

   ::

      void hello(int i);
      void hello();

If `compactdefaultargs <SWIGPlus.html#SWIGPlus_default_args>`__ are
being used, then the difference between specifying or not specifying
default arguments in a feature is not applicable as just one wrapper is
generated.

**Compatibility note:** The different behaviour of features specified
with or without default arguments was introduced in SWIG-1.3.23 when the
approach to wrapping methods with default arguments was changed.

Feature example
~~~~~~~~~~~~~~~~~~~~~~

As has been shown earlier, the intended use for the ``%feature``
directive is as a highly flexible customization mechanism that can be
used to annotate declarations with additional information for use by
specific target language modules. Another example is in the Python
module. You might use ``%feature`` to rewrite proxy/shadow class code as
follows:

.. container:: code

   ::

      %module example
      %rename(bar_id) bar(int, double);

      // Rewrite bar() to allow some nice overloading

      %feature("shadow") Foo::bar(int) %{
      def bar(*args):
          if len(args) == 3:
              return apply(examplec.Foo_bar_id, args)
          return apply(examplec.Foo_bar, args)
      %}
          
      class Foo {
      public:
        int bar(int x);
        int bar(int x, double y);
      }

Further details of ``%feature`` usage is described in the documentation
for specific language modules.
