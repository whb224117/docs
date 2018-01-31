==================
S2E Python support
==================

This document outlines the design behind S2E's support for Python plugins, and how to extend this support to other S2E
features. Note that this document is **not** intended as a guide/tutorial on *how* to write Python plugins - for more
detail on this see `here <Howtos/WritingPythonPlugins.rst>`_.

Background
----------

Traditionally, S2E plugins are written in C++. This means that any time a new plugin is created and/or modified, libs2e
must be recompiled. This can be a frustrating process. To alleviate this, the code in this directory adds support for
writing S2E plugins in the Python programming language. Python has both many similarities and differences to C++: they
both allow OOP, however Python does not have pointers or strong types in the same way that C++ does. This creates
challenges when trying to expose the existing S2E API in a way that allows plugin developers to (a) have access to
S2E's many capabilities and (b) write their plugins in a "Pythonic" way.

Embedding a Python interpreter in S2E
-------------------------------------

The `Boost Python <http://www.boost.org/doc/libs/1_66_0/libs/python/doc/html/index.html>`_ library is used to "embed" a
Python interpreter within S2E. At the core of this embedding is the `Interpreter` class. This class initializes the
Python interpreter, loads and maintains Python plugins during runtime.

Unlike a C++ plugin, we do not know anything about a particular Python plugin until runtime (in contrast, a C++
plugin's info is available when S2E is compiled). Therefore, Python plugins are created differently to C++ plugins.
When a C++ plugin is created (in ``PluginsFactory::createPlugin``), the ``PluginInfo`` object is already available from
the static ``CompiledPlugin``. The ``PluginInfo``'s ``instanceCreator`` can then be used to instantiate the plugin at
runtime.

In contrast, there is no static information available for Python plugins (because we don't even know the location of
the Python plugins until runtime). A Python plugin's ``PluginInfo`` object is therefore created dynamically based on
`class attributes <https://docs.python.org/2/tutorial/classes.html#class-objects>`_ that contain a plugin's
description, dependencies, etc.. Once this information has been retrieved from the Python plugin class (in
``Interpreter::loadPlugin``), a ``PluginInfo`` object can be created. This ``PluginInfo`` is then used to configure a
C++ ``Plugin`` object, which can be registered with S2E's ``PluginFactory``.

Exposing the S2E API
--------------------

S2E has a complex API which uses elements from QEMU and KLEE, in addition to its own functionality. Exposing these
different APIs to a Python plugin author is similarly complex. We must tell the Python interpreter about the different
classes that we wish to expose, which in turn involves telling the Python interpreter about any classes that are used
by the original class we wished to expose. Due to the complexity involved in this, not all of S2E's API is usable by a
Python plugin. The following sections document some of the exposed API.

Signals and callbacks
=====================

S2E uses signals to inform plugins of particular events during runtime. A plugin author must write a callback method
that can then be connected to a signal and called when that signal is emitted. Python plugins follow a very similar
approach.

S2E's core signals (defined in ``CorePlugin.h``) are each wrapped in a ``PythonSignal`` class that make them accessible
from a Python plugin (see ``Signals.h``). However, we must introduce an additional layer of abstraction when connecting
these signals to a Python callback function. This is achieved via the ``CallbackFunctor`` class. This class wraps the
Python function written by the plugin author and provides a call operator that can be connected to an S2E signal. Why
is this additional layer of indirection required? Why can't we connect a wrapped S2E signal directly to a Python
function? Because often signals take parameters, and these parameters may not be directly passable to Python code. For
example, we may need to wrap a parameter in a class known to the Python interpreter, the parameter may be a pointer
which must be handled differently by Boost Python, etc. We can do any required manipulation in the ``CallbackFunctor``
class before we call the Python function.

Making S2E structs/classes usable in Python
===========================================

We previously mentioned that the Python interpreter must be "told" about any C++ class that we wish to use in a Python
plugin. We do this in the ``export_*.cpp`` source files. Each of these files follows a similar pattern - a Python
module is defined for where we wish to export a class from (e.g. ``s2e.core``, ``s2e.core.signals``, etc.). The
``boost::python::class_`` template is then used to expose the C++ class to the Python interpreter. We have tried to
make this exposure as Pythonic as possible - e.g. using properties, snake-case propety and method names, etc.

Unfortunately, due to the sheer size of the S2E (which itself is made up of a number of different APIs, each containing
their own C++ structs and classes), only a small subsection of the S2E API is currently exposed. To expose more of the
API, a similar approach taken in the existing ``export_*.cpp`` files is required.

Making existing C++ plugins usable in Python
============================================

S2E C++ plugins are themselves C++ classes. Therefore, if you wish to make use of existing C++ plugins in your Python
plugins, you must export these plugins also. If these plugins emit their own signals, you must use the same level of
indirection described above when connecting to these signals. S2E C++ plugins should be exposed under the
``s2e.core.plugins`` module (see ``export_plugins.cpp`` for details).
