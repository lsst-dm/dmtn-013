:tocdepth: 1

All LSST code currently uses `SWIG <http://www.swig.org>`_ to generate Python wrappers around C++ code. This document investigates using `Cython <www.cython.org>`_ as an alternative.
The prime motivation for this is that `AstroPy <www.astropy.org>`_ uses Cython and a closer collaboration and code sharing with AstroPy is currently being evaluated.
To start the investigation Jim Bosch has written a `C++/Python Bindings Challenge <https://github.com/TallJimbo/python-cpp-challenge>`_. 
It consists of "a small suite of C++ classes designed to highlight any of the most common challenges involved in providing Python bindings ot a C++ library, as well as a set of Python unit tests that attempt to measure the quality of the resulting bindings".
Concretely, this document then describes the (partial) Cython solution to this challenge.

About the scope of this document
================================

Although this document contains some examples of how to wrap C++ with Cython and might help to get you started it is not meant to be a full tutorial by any means. The examples are just there to get some context when describing the various things that I encountered. If you are looking for comprehensive information about Cython and C++ please have a look at some of the documents linked below instead.

About Cython
============

Unlike most C/C++ Python wrapping solutions, Cython is not just a wrapper tool. Instead it is a full featured programming language with a Python like syntax but with optional added static typing. Additionally Cython provides a compiler that translates Cython code into C or C++ code which can subsequently be compiled to a Python extension module by a standard C/C++ compiler.

In my opinion this is both Cythons main strength as well as one of its weaknesses. On the plus side having a full blown programming language available that closely matches Python enables the resulting wrapper modules to be very Pythonic. Probably more so than any other wrapper tool allows. On the other hand it means a lot of wrapping code has to be written manually. Because the wrapping language is closer to Python then it is to C++ it is sometimes a little difficult to figure out what exactly is going on behind the scenes and not everything translates one-to-one.

A quick example
===============

Given a simple C++ class:

.. code-block:: cpp
    :name: basics.cpp

    namespace basics {

    class Doodad {
    public:
        Doodad(std::string const & name_, int value_);
        
        std::string name;
        int value;
    };
    
    } // namespace basics

a basic Cython wrapper for this would look like this.

.. code-block:: cython
    :name: basics.pyx

    from cython.operator cimport dereference as deref

    from libcpp.memory cimport unique_ptr
    from libcpp.string cimport string
    
    cdef extern from "basics.hpp" namespace "basics":
        cdef cppclass Doodad:
            Doodad(string, int)
    
            string name
            int value
    
    cdef class PyDoodad:
        cdef unique_ptr[Doodad] thisptr
    
        def __init__(self, name, value):
            self.thisptr.reset(new Doodad(name, value))
    
        property name:
            def __get__(self):
                return deref(self.thisptr).name
            def __set__(self, _name):
                deref(self.thisptr).name = _name
    
        property value:
            def __get__(self):
                return deref(self.thisptr).value
            def __set__(self, _value):
                deref(self.thisptr).value = _value

This can then be compiled into a standard (C)Python extension module with the following "setup.py" file.
Note that there are other ways to compile the code, some of which even work on the fly while importing, but for purposes of distribution this is probably best.

.. code-block:: python

    import sys
    from distutils.core import setup, Extension
    from Cython.Build import cythonize
    
    compile_args = ['-g', '-std=c++11', '-stdlib=libc++']
    
    if sys.platform == 'darwin':
        compile_args.append('-mmacosx-version-min=10.7')
    
    basics_module = Extension('example.basics',
                    sources=['example/basics.pyx', 'example/basics.cpp'],
                    extra_compile_args=compile_args,
                    language='c++')
    
    setup(
        name='example',
        packages=['example'],
        ext_modules=cythonize(basics_module)
    )

Which can be built with:

.. code-block:: bash

    python setup.py build_ext --inplace

Quick example step-by-step
--------------------------

Now let's examine the wrapper code step by step.

.. code-block:: cython

    from cython.operator cimport dereference as deref

This line brings in the dereference "operator". In Cython a pointer dereference, ``*p`` can be written either as ``p[0]`` or ``deref(p)``. I prefer the latter since it seems to work in more contexts. If it is considered too verbose just change the import line to ``... cimport dereference as d``.

The next two lines:

.. code-block:: cython
    
    from libcpp.memory cimport unique_ptr
    from libcpp.string cimport string

bring in ``unique_ptr`` and ``string`` from the C++ standard library. Cython provides very elegant wrappers around the most frequently used standard library types (e.g. ``shared_ptr``, ``vector``, ``map``, ``unordered_map``, etc.).

The block starting with:

.. code-block:: cython

    cdef extern from "basics.hpp" namespace "basics":
        ...

declares the C++ types (and functions) to be usable from Cython. The only thing this does is place the declarations in the resulting C/C++ file with an ``extern`` modifier. Because of this it is sometimes confusingly the users responsibility of ensuring that the declarations here match those on the C++ side. Otherwise this is only discovered at linking. This tasks is further complicated because C++ declarations cannot always be copied entirely verbatim to Cython, which doesn't allow ``*``, ``&`` or qualifiers such as ``const`` in all places. But this is a minor nuisance which decreases with increasing understanding. Note that Cython also supports nested namespaces, but only one namespace can be used per extern block.

Now let's move on to the class definition.

.. code-block:: cython

    cdef class PyDoodad:
        cdef unique_ptr[Doodad] thisptr
    
        def __init__(self, name, value):
            self.thisptr.reset(new Doodad(name, value))

        ...

This now is the class that is going to be available from Python (the Doodad itself is strictly C++). This class structure follows a standard approach with Cython. A ``thisptr`` member contains a pointer to an instance of the underlying C++ class. In this case we use a ``std::unique_ptr<Doodad>`` (template types use square brackets in Cython) to represent ownership and ensure proper lifetime. Most examples online use raw pointers with ``new`` in a ``__cinit__`` constructor and ``delete`` in a corresponding ``__dealoc__`` (which are guaranteed by Cython to be called before and after the Python constructor and destructor respectively). However, I would recommend using either `unique_ptr`` or ``shared_ptr`` instead, reserving raw pointers for non-owners, in keeping with modern C++ convention.

The remaining code:

.. code-block:: cython

    property name:
        def __get__(self):
            return deref(self.thisptr).name
        def __set__(self, _name):
            deref(self.thisptr).name = _name
    ...

deals with Python attributes and should be obvious.

Solving the C++/Python bindings challenge with Cython
=====================================================

The previous section gave a quick overview of wrapping a C++ class with Cython. This section describes some of the issues encountered while wrapping the C++/Python bindings challenge code. This code was designed to be more representative of the type of code encountered when porting larger swaths of LSST library code.

It contains four C++ source files which are to be compiled into three different Python modules (with interdependencies).

* ``basics`` contains a class ``Doodad``, a class ``Secret`` and a struct ``WhatsIt``. The class ``WhatsIt`` should be visible to Python only as a tuple and ``Secret`` can only be constructed by ``Doodad``, it is to be passed around in Python as an opaque object. ``Doodad`` is the main class to be wrapped.

* ``extensions`` contains a templated class ``Thingamajig`` that inherits from ``Doodad``.

* ``containers`` defines a ``DoodadSet`` class (backed by a ``std::vector``) that contains ``Doodads`` and ``Thingamajigs``.

* ``conversions`` contains various tests for SWIG compatibility (more on this later).

The current solution passes all unit tests for ``basics``, ``containers`` and ``conversions`` but does not (yet) wrap ``extensions``. This is only due to time constraints and I do not yet foresee any major problems with it.

Diving in
---------

Class name clashes
^^^^^^^^^^^^^^^^^^

The first problem encountered is that the unit tests expect the Python classes to be available under the same name as they have in C++.
As can be seen in the example above the extern block brings in the C++ classes under their own name (this is required) and therefore doesn't allow the Python class to be given the same name when declared in the same file.

One can get around this by placing the extern block in a separate ``.pxd`` file and then adding

.. code-block:: cython

    from _basics.pxd cimport Doodad as _Doodad

to the top of the ``.pxd`` file. Then ``_Doodad`` refers to the C++ class while ``Doodad`` can be used for Python.

Dealing with const
^^^^^^^^^^^^^^^^^^

The second interesting problem pops up when dealing with ``const`` objects. How does one represent them on the Python side? And how can it be stored internally?
One solution involves keeping two pointers in the object, one ``unique_ptr[Doodad]`` and one ``unique_ptr[const Doodad]`` and then raising errors if a write operation is accessed for a const-backed object.
This has the advantage of presenting one type to the Python user.
But then the behaviour of the object is dependent on the backing object.
This doesn't feel very Pythonic.
The solution taken instead was to have two different classes on the Python side. A regular writable ``Doodad`` (backed by a ``unique_ptr[Doodad]`` and a read-only ``ImmutableDoodad`` (backed by a ``unique_ptr[const Doodad]``. The latter simply lacks the methods for writing.
This approach is more Pythonic IMHO but does require some duplicate code (although one could probably get away with that with some clever subclassing on the Cython side).

Getting a const object
^^^^^^^^^^^^^^^^^^^^^^

In the challenge the previously mentioned ``ImmutableDoodad`` is obtained from a ``Doodad`` instance by calling its ``.get_const()`` method. In C++ this returns a ``shared_ptr<const Doodad>``. The easiest way of dealing with this is to simply change the backing smart pointer type in the Python object to a ``shared_ptr`` as well. This simply follows the standard C++ rule of using ``shared_ptr`` for things that you know are going to be shared.

Note that if this method didn't exist on the C++ level we should stick to ``unique_ptr``.

Cloning
^^^^^^^

The C++ class ``Doodad`` also has a method called ``clone()`` that returns a ``unique_ptr`` to a newly copied object. To pass the unit tests the returned object has to be given back to Python without making any additional copies. This is achieved in Cython using:

.. code-block:: cython

    def clone(self):
        d = Doodad(init=False)

        d.thisptr = move(deref(self.thisptr).clone())

        return d

which also requires ``std::move`` to be declared in the ``.pxd`` file.

.. code-block:: cython
    cdef extern from "<utility>" namespace "std" nogil:
        cdef shared_ptr[Doodad] move(unique_ptr[Doodad])
        cdef shared_ptr[Doodad] move(shared_ptr[Doodad])

There are a few things annoying about this. One is that a separate specialization is to be declared for every type of move and the other is that Cython doesn't like it if two specializations have the same arguments but different return types. The following is thus not allowed.

.. code-block:: cython
    cdef extern from "<utility>" namespace "std" nogil:
        cdef unique_ptr[Doodad] move(unique_ptr[Doodad])
        cdef shared_ptr[Doodad] move(unique_ptr[Doodad]) # error!
        cdef shared_ptr[Doodad] move(shared_ptr[Doodad])

Which was fortunately not needed in this case but can be really annoying when it is. That this is necessary at all is the result of Cython not knowing about rvalue references. It is a `known bug <https://groups.google.com/forum/#!topic/cython-users/-U8r0Lc_fU4>`_, with so far no solution.

But hey, in this case it works!

Notice also the ``init=False``. This, rather ugly, thing is needed because:

* C++ ``Doodad`` has no default constructor, and
* ``__init__`` has two arguments with a default value.

You need some way to tell Cython to make a Python ``Doodad`` with an uninitialized ``shared_ptr``.
Ideally one would want to use a factory ``Doodad.__new__(Doodad)`` thing here, but for some reason this doesn't play well with Cython (specifically, when called like that it doesn't seem to add a ``thisptr``).

Comparison operators
^^^^^^^^^^^^^^^^^^^^

Unlike Python, Cython has only one special method to implement all comparison operators called ``__richcmp__``.
The current solution, which has to support custom equality and inequality for ``Doodad`` and ``ImmutableDoodad`` looks like:

.. code-block:: cython

    def __richcmp__(self, other, int op):
        if op == Py_EQ and isinstance(other, Doodad):
            return isEqualDD(self, other)
        elif op == Py_EQ and isinstance(other, ImmutableDoodad):
            return isEqualDI(self, other)
        elif op == Py_NE and isinstance(other, Doodad):
            return isNotEqualDD(self, other)
        elif op == Py_NE and isinstance(other, ImmutableDoodad):
            return isNotEqualDI(self, other)
        else:
            raise NotImplementedError

where the functions that are called look like:

.. code-block:: cython

    cdef isEqualDD(Doodad a, Doodad b):
        return a.thisptr.get() == b.thisptr.get()

those functions are simply for convenience since the type of ``other`` is not known (and thus not guaranteed to have a ``thisptr``). An alternative is casting at runtime (see SWIG example below).

Containers and iterators
^^^^^^^^^^^^^^^^^^^^^^^^

Because of the nice availability of ``vector`` and ``map`` in Cython writing conversion methods to Python ``list`` and ``dict``, given the methods ``as_vector()`` and ``as_map()`` is easy.

.. code-block:: cython

    cpdef as_list(self):
        cdef vector[shared_ptr[_Doodad]] v = self.inst.as_vector()

        results = []
        for item in v:
            d = Doodad(init=False)
            d.thisptr = move(item)
            results.append(d)

        return results

    cpdef as_dict(self):
        cdef map[string, shared_ptr[_Doodad]] m = self.inst.as_map()

        results = {}
        for k in m:
            d = Doodad(init=False)
            d.thisptr = move(k.second)

            results[k.first] = d

        return results

It would have been even easier if the ``vector`` or ``map`` only included items that were already known to Cython. In that case we could simply return the result directly, without having to build up a new list or dict.
In this case however Cython does not know what Python type to put in for the elements. Perhaps this can be fixed somehow?

Inter module dependencies
^^^^^^^^^^^^^^^^^^^^^^^^^

Note that in the previous example something funny is going on.
In the methods ``as_list()`` and ``as_dict()`` we need both the C++ type ``_Doodad`` and the Python type ``Doodad``.

Well we can get those with a simple import right?

.. code-block:: cython

    from _basics cimport _Doodad
    from basics import Doodad

Wrong! The imported Python type doesn't give access to the ``thisptr`` member. To solve this we need to split up ``basics`` into three modules.

* ``_basics.pxd`` containing the C++ declarations.
* ``basics.pxd`` containing the Cython class declarations.
* ``basics.pyx`` containing the Cython class definitions.

So why can't we just stick the Cython class declarations in ``_basics.pxd``? Because we need the classes to be named the same!

So in the end we get ``_basics.pxd``:

.. code-block:: cython

    cdef extern from "basics.hpp" namespace "basics":
        cdef cppclass Doodad:
            ...

and ``basics.pxd``:

.. code-block:: cython

    from _basics cimport Doodad as _Doodad

    cdef class Doodad:
        cdef shared_ptr[Doodad] thisptr

        ...

and ``basics.pyx``:

.. code-block:: cython

    from _basics cimport Doodad as _Doodad
    from basics cimport Doodad

    cdef class Doodad:
        def __init__(self):
            ...

and finally in ``containers.pyx``:

.. code-block:: cython

    from basics import Doodad
    from basics cimport Doodad
    from _basics cimport Doodad as _Doodad

    ...

Granted, the naming could have been nicer...

Iterators
^^^^^^^^^

Implementing iterators in Cython is roughly the same as in Python.

.. code-block:: cython

    cdef class DoodadSet:
        ...

        def __iter__(self):
            self.it = self.inst.begin()
            return self

        def __next__(self):
            if self.it == self.inst.end():
                raise StopIteration()

            d = Doodad(init=False)
            d.thisptr = deref(self.it)

            incr(self.it)

            return d

Note that in this case we use ``inst`` instead of ``thisptr``. If a C++ object has a default constructor you can do this. This also allows the objects to be put on the stack (when used from within Cython).
This is just to show how it can be done.

Of course ``begin()`` and ``end()`` have to be declared as well.

.. code-block:: cython
    cdef extern from "containers.hpp" namespace "containers":
        cdef cppclass DoodadSet:
            vector[shared_ptr[Doodad]].const_iterator begin() const
            vector[shared_ptr[Doodad]].const_iterator end() const

SWIG interoperability
^^^^^^^^^^^^^^^^^^^^^

The final thing that is needed is to pass all unit test for conversions to and from SWIG.

The SWIG wrapped extension module ``converters`` contains functions like.

.. code-block:: cpp

    std::shared_ptr<basics::Doodad> make_sptr(std::string const & name, int value) {
        return std::shared_ptr<basics::Doodad>(new basics::Doodad(name, value));
    }

These then use typemaps declared in ``basics_typemaps.i`` such as.

.. code-block:: cpp

    %typemap(out) std::shared_ptr<basics::Doodad> {
        Py_Initialize();
        initbasics();
        $result = newDoodadFromSptr($1);
    }

    %typemap(in) std::shared_ptr<basics::Doodad> {
    Py_Initialize();
    initbasics();
    if (!sptrFromDoodad($input, &$1)) {
        return nullptr;
    }
}

The key to Cython / SWIG interoperability is of course in the functions ``newDoodadFromSptr`` (that should take a ``shared_ptr<Doodad>`` and return a Python object) and ``sptrFromDoodad`` (which does the opposite).

These are implemented in ``basics.pyx`` and look like this.

.. code-block:: cython

    cdef public newDoodadFromSptr(shared_ptr[_Doodad] _d):
        d = Doodad(init=False)
        d.thisptr = move(_d)
    
        return d
    
    cdef public bool sptrFromDoodad(object _d, shared_ptr[_Doodad] *ptr) except + :
        d = <Doodad?> _d
        ptr[0] = d.thisptr
    
        return True # cannot catch exception here

The mysterious ``<Doodad?>`` is a Cython style cast. It tries to do a conversion and raises an exception if if fails. The ``except +`` is needed to allow this exception to be translated and propagated to Python. Cython knows about some default exception types to map to standard Python ones. In this case a ``TypeError`` is raised if ``_d`` is not a (subclass of) ``Doodad``.

You may also notice ``move`` which is declared as described above (in case you skipped to this section immediately).

Now how is this stuff actually called by the C++ SWIG code?
Cython has a nice ``public`` keyword as written above. It causes the Cython compiler to generate
a C++ header file (``basics.h``) for with these function declarations and places that in the build directory. Then it simply gets included by the SWIG build.

A gotcha here is that when calling these functions from C++, the Python module needs to be initialized (or segfaults and other madness ensue). This is what the ``PyInitialize()`` and ``initbasics()`` calls are for. If you ask me it would also need a ``PyFinalize()`` but putting that in breaks everything. Probably these should instead be moved to a place where SWIG does module initialization and finalization.

Another problem is linking. In particular, linking on OSX. Since OSX has the concept of bundles (i.e. ``.so``) and dynamic libraries (i.e. ``.dylib``) things get interesting.
By default Cython builds bundles. Which is the sensible thing to do because extension modules is what bundles are meant for.
However, now ``basics`` is not only an extension module but also a library, that our SWIG module wants to link to.
Thus we need to tell Cython that on OSX we want a dynamic library instead (while still calling the resulting thing ``basics.so``). This is done by adding the following to ``setup.py``.

.. code-block:: python

    if sys.platform == 'darwin':
        from distutils import sysconfig
        vars = sysconfig.get_config_vars()
        vars['LDSHARED'] = vars['LDSHARED'].replace('-bundle', '-dynamiclib')
        compile_args.append('-mmacosx-version-min=10.7')

The SWIG module can now be built with.

.. code-block:: python

    converters_module = Extension(
        'challenge.converters',
        sources=[
            os.path.join('challenge', 'converters.i'),
        ],
        include_dirs=[
            os.path.join('..', 'include'),
            os.path.join('include')
        ],
        swig_opts = ['-modern', '-c++', '-Iinclude', '-noproxy'],
        extra_compile_args=compile_args,
        extra_link_args=[
            os.path.join('challenge', 'basics.so')
        ]
    )

See also
========

* The full implementation of the Cython solution to the C++/Python bindings challenge is available in the ``cython`` branch of my `fork on github <https://github.com/pschella/python-cpp-challenge>`_.

* A great book on Cython is "Cython - A guide for Python programmers" by Kurt W. Smith.

* Another excellent source is the online `Cython documentation <http://docs.cython.org/index.html>`_.

Relevant JIRA tickets
=====================

* `DM-5470 <https://jira.lsstcorp.org/browse/DM-5470>`_: Develop C++ code for experimenting with Python binding
* `DM-5471 <https://jira.lsstcorp.org/browse/DM-5471>`_: Wrap example C++ code with Cython

