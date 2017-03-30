
Developer documentation
=======================

Main highlights
---------------

* **Python 3 only**!
* Fast, most computation are done in C++11 (including multi-threading)
* Interplay between Python and C++ is done using swig


.. contents::
    :depth: 3
    :backlinks: none

.. _BaseClass:

Base class
----------
Most of our objects inherit from `mlpp.base.Base` class. If this class does
not have many public methods, it allow us to benefit from several behaviors.
These behaviors are deducted from a private dictionary set by developer :
`_attrinfos`.

.. testsetup:: *

    from mlpp.base import Base

Read-only attributes
^^^^^^^^^^^^^^^^^^^^

Behavior
~~~~~~~~
The first feature is the ability to set read-only attributes. If you think
end-user should not modify an attribute of your class, you can simply set
writable to False (writable is True by default).

.. testcode:: [readonly]

    class A(Base):
        _attrinfos = {
            'readonly_attr' : {
                'writable' : False
            }
        }

    a = A()

Then if you try to set it

.. doctest:: [readonly]

    >>> a.readonly_attr = 3
    Traceback (most recent call last):
      ...
    AttributeError: readonly_attr is readonly in A

How to set it manually?
~~~~~~~~~~~~~~~~~~~~~~~
First if you set a read-only attribute during the initialization phase (ie.
in `__init__` method or in a method called by it) you will not raise any error.

.. testcode:: [readonly]

    class A(Base):
        _attrinfos = {
            'readonly_attr' : {'writable' : False}
        }

        def __init__(self, readonly_value):
            self.readonly_attr = readonly_value

This code snippet will work as usual.

.. doctest:: [readonly]

    >>> a = A(5)
    >>> a.readonly_attr
    5

But if you need to change this attribute after the initialization phase, you
can force set it by using `_set` method.


.. doctest:: [readonly]

    >>> a._set('readonly_attr', 10)
    >>> a.readonly_attr
    10

Settable attributes
^^^^^^^^^^^^^^^^^^^
Base class also restricts which attributes can be set. Attributes that can be
set are:

* Attributes contained in `_attrinfos` dictionary
* Attributes documented in class docstring (with numpydoc style)
* Attributes passed as argument to `__init__`

.. warning::

    When you document an attribute with numpydoc style, do not forget the
    space before the colon that follow its name.

Hence, if an attribute was never mentioned in your class before, trying to
set it will raise an exception.

.. testcode:: [settable]

    class A(Base):
        """This is an awesome class that inherits from Base

        Parameters
        ----------
        documented_parameter : `int`
            This is a documented parameter of my class

        Attributes
        ----------
        documented_attribute : `string`
            This is a documented attribute of my class
        """
        _attrinfos = {
            'attr_in_attrinfos' : {}
        }
        def __init__(self, documented_parameter, undocumented_parameter):
            pass

The following will work as expected

.. doctest:: [settable]

    >>> a = A(10, 12)
    >>> a.documented_parameter = 32
    >>> a.documented_attribute = 'bananas'
    >>> a.undocumented_parameter = 'are too many'
    >>> a.documented_parameter, a.documented_attribute, a.undocumented_parameter
    (32, 'bananas', 'are too many')

But this raises an error

.. doctest:: [settable]

    >>> a = A(10, 12)
    >>> a.unexisting_attr = 25
    Traceback (most recent call last):
     ...
    AttributeError: 'A' object has no settable attribute 'unexisting_attr'


Link with C++ setter
^^^^^^^^^^^^^^^^^^^^
Another useful feature is the possibility to add a direct linking between a
Python attribute and its C++ equivalent.

In many cases our code consists in a Python object which encompasses a C++
object used for intense computations. Find more details in the SWIG part
of this documentation. In this setting we might want to update our C++ object
each time our Python object is. We can do so by specifying which setter to
call when an attribute is modified in Python.

For this example, let's suppose we have a C++ class (named `_A`) that has a int
attribute associated to a setter (`set_cpp_int`) and a getter (`get_cpp_int`).
In order to enable the linking we must specify:

* What is the C++ object's name, through `_cpp_obj_name` attribute of the class
* What is the C++ method that sets attribute `cpp_int`, through `cpp_setter`
  in `_attrinfos` dictionary

.. testsetup:: [cpp_setter]

    from mlpp.base.utils.build.utils import A0 as _A

.. testcode:: [cpp_setter]

    class A(Base):
        _attrinfos = {
            'cpp_int': {'cpp_setter': 'set_cpp_int'},
            '_a' : {'writable' : False}
        }
        _cpp_obj_name = "_a"

        def __init__(self):
            self._a = _A()
            self.cpp_int = 0

Now each time we will modify `cpp_int` attribute of an instance of the class
`A`, `set_cpp_int` method of the C++ object will be called and modify the
value of the C++ int.

.. doctest:: [cpp_setter]

    >>> a = A()
    >>> a.cpp_int, a._a.get_cpp_int()
    (0, 0)
    >>> a.cpp_int = -4
    >>> a.cpp_int, a._a.get_cpp_int()
    (-4, -4)

.. note::
    If the reader wants to run this example, he might find the corresponding
    class by importing it `from mlpp.base.utils.build.utils import A0 as _A`.

How does this work?
^^^^^^^^^^^^^^^^^^^
This class behavior is obtained thanks to Python metaclasses. A metaclass is
the object that is called to create the class object itself. For example, it
allow us to automatize property creation. For more information, please report
to `Python documentation`_.

What we do is creating a property for each attribute. This property is linked
to a hidden attribute, stored with the same name of the property with a
double underscore before

.. _Python documentation:
    https://docs.python.org/3/reference/datamodel.html#
    customizing-class-creation

If we create the following class `A`:

.. testcode:: [how]

    class A(Base):
        def __init__(self, attr):
            self.attr = attr

We have access to the property `attr` and its linked attribute `__attr`:

.. doctest:: [how]

    >>> a = A(15)
    >>> a.attr, a.__attr
    (15, 15)

Two good practises to avoid unexpected behaviors:

* Do not define an attribute that starts with a double underscore
* Add property documentation in class docstring instead of property getter


How to add a new model, solver or prox
--------------------------------------

Many of our models, prox and solvers are Python classes that wraps a C++ class
which handles the heavy computations. This allows us to have a code that runs
fast.

Let's see what we should do if we want to add prox L2. Adding a model or a
solver is basically identical.

Create C++ class
^^^^^^^^^^^^^^^^

First we need to create the C++ class that will be wrapped by our Python
class later. We want our prox to be able to give the value of the
penalization at a given point and call the proximal operator on a given vector.

Here is what our .h file should look like

.. code-block:: cpp

    class ProxL2Sq {

    protected:
        double strength;

    public:
        ProxL2Sq(double strength);
        double value(ArrayDouble &coeffs) const;
        void call(ArrayDouble &coeffs, double step, ArrayDouble &out) const;
        inline void set_strength(double strength){
            this->strength = strength;
        }
    };

Basically we have one constructor that set the only one parameter strength
(usually denoted by lambda), and the two methods we described above.

Our .cpp implementation looks like:

.. code-block:: cpp

    #include "prox_l2sq.h"

    ProxL2Sq::ProxL2Sq(double strength) {
        this->strength = strength;
    }

    double ProxL2Sq::value(ArrayDouble &coeffs) const {
        return 0.5 * coeffs.normsq();
    }

    void ProxL2Sq::call(ArrayDouble &coeffs, double step, ArrayDouble &out) const {
        for (unsigned long i; i < coeffs.size; ++i)
            out[i] = coeffs[i] / (1 + step * strength);
    }

In mlpp these files are stored in /src folder

Link it with Python
^^^^^^^^^^^^^^^^^^^

Create SWIG file
~~~~~~~~~~~~~~~~

Now that our proximal operator is defined in C++ we need to make it available
in Python. We do it thanks to `SWIG <http://www.swig.org/Doc3.0/>`_.
Hence we have to create a .i file. In mlpp we store them in /swig folder.

This .i file looks a lot like our .h file.

.. code-block:: cpp

    %include <std_shared_ptr.i>
    %shared_ptr(ProxL2Sq);

    %{
    #include "prox_l2sq.h"
    %}

    class ProxL2Sq {
    public:
        ProxL2Sq(double strength);
        double value(ArrayDouble &coeffs) const;
        void call(ArrayDouble &coeffs, double step, ArrayDouble &out) const;
        virtual void set_strength(double strength);
    };

In this file our goal is to explain to Python what it can do with this class. In
our example it will be able to instantiate it by calling its constructor
with a double, and call three methods, `value`, `call` and `set_strength`.

.. note::
  * There is no interest in mentioning here any private method or attribute of
    the class as this is what Python see and Python would not be able to call
    them.
  * We need to include the file in which is declared the class we are talking
    about in the .i file. This what we do with `#include "prox_l2sq.h"`.
  * Finally, as we will want to share our proximal operator and as it might be
    used by several objects, we wrap it in class from the standard library: the
    shared pointer. To make SWIG aware that this class will be used with shared
    pointers we must add `%shared_ptr(ProxL2Sq);` which must be done after
    `%include \<std_shared_ptr.i\>`.

.. note::
  In mlpp our ProxL2Sq class is not really identical as it inherits
  from Prox abstract class. Hence some of this logic might not be present in
  the exact same file. Everything that concerns prox, is imported through
  `prox_module.i`.

Reference it in Python build
~~~~~~~~~~~~~~~~~~~~~~~~~~~~

Now that we have written our .i file we should add our files to our python
script that builds the extension : `setup.py`.

Most of the time, we add a file that belongs to a module that has already
been created. In this case we only need to add its source files at the right
place in `setup.py`.

Let's supposed we already had two prox in our module (abstract class Prox and
prox L1), we need to add `prox_l2sq.cpp` and `prox_l2sq.h` that we have just
created at the following place.

.. code-block:: python
    :emphasize-lines: 4, 7

    prox_core_info = {
        "cpp_files": ["prox.cpp",
                      "prox_l1.cpp",
                      "prox_l2sq.cpp"],
        "h_files": ["prox.h",
                    "prox_l1.h",
                    "prox_l2sq.h"],
        "swig_files": ["prox_module.i", ],
        "module_dir": "./mlpp/optim/prox/",
        "extension_name": "prox",
        "include_modules": base_modules
    }

.. note::
  We do not need to add `prox_l2sq.i` file here as it is imported
  in `prox_module.i` with a `%include` operator. This operator works like a
  copy/paste of the code of the included file.

Use class in Python
^^^^^^^^^^^^^^^^^^^

Now that our C++ class is linked with Python we can import it and use its
methods that we have declared in the .i file.

In mlpp we always wrap C++ classes in a Python class that will call C++
object methods when it needs to perform the computations. Hence here is the
Python class we might create:

.. code-block:: python

    import numpy as np
    from .build.prox import ProxL2Sq as _ProxL2sq

    class ProxL2Sq:
        _attrinfos = {
            "strength": {
                "cpp_setter": "set_strength"
            }
        }
        _cpp_obj_name = "_prox"

        def __init__(self, strength: float):
            self._prox = _ProxL2sq(strength)
            self.strength = strength

        def value(self, coeffs: np.ndarray):
            return self._prox.value(coeffs)

You might have seen that we instantiate a dictionary called `_attrinfos` in
the class declaration. This dictionary is useful in many ways and you should
refer to BaseClass_. Here we use one of
its functionalities: automatic set of C++ attributes. Each time `strength` of
our prox will be modified, the `set_strength` method of the object stored in
`_prox` attribute (as specified by `_cpp_obj_name` will be called with the
new value passed as argument). This allow us to have strength values of
Python and C++ that are always linked.
