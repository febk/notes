# Wrapping C/C++ code using SWIG
The Simplified Wrapper and Interface Generator (SWIG) is a tool for wrapping C/C++ code into a variety of higher level languages, including JS, Python, Ruby, etc., and is used alongside interface files which help instruct the SWIG pre-compiler which methods and functions should be exposed.

<!--BEGIN TOC-->
## Table of Contents
1. [SWIG setups](#toc-sub-tag-0)
	1. [Python](#toc-sub-tag-1)
		1. [Useful command line options](#toc-sub-tag-2)
2. [Python Packages](#toc-sub-tag-3)
3. [Binding abstractions](#toc-sub-tag-4)
	1. [Pointers](#toc-sub-tag-5)
	2. [Structures](#toc-sub-tag-6)
	3. [Classes](#toc-sub-tag-7)
4. [A worked example: Python](#toc-sub-tag-8)
	1. [SWIG CLI](#toc-sub-tag-9)
	2. [Distutils](#toc-sub-tag-10)
<!--END TOC-->

The official [SWIG Tutorial](http://swig.org/tutorial.html) covers how to get started quickly with SWIG, however my notes will elucidate a few more applied scenarios for SWIG which I will document as I continue to learn the tool myself.

A note is that SWIG does not support the full C++ syntax ([see here](http://swig.org/Doc3.0/SWIGPlus.html#SWIGPlus_nn4)), but still provides enough of the language support, and implicit conversion, that make most of the use cases straight forward to use.

The main difference when using C++ over C with SWIG is to include the relevant flag:

```bash
swig -c++ -python example.i 
```

## SWIG setups <a name="toc-sub-tag-0"></a>
In this section are the interface changes required for compiling to different higher level languages.

### Python <a name="toc-sub-tag-1"></a>
In the interface file, we must define
```C
#define SWIG_FILE_WITH_INIT
```
during the include sections, such that SWIG compiles the code into a Python compatible module.

The `setup.py` basic configuration follows the usual idiom (see the worked example below). To compile the external package to another name we use the setup keyword
```py
ext_package='some_other_name'
```
*NOTE:* I can't fully figure out an elegant solution for doing the above when not using explicit packaging (see Python Packages below). Instead, changing the module name in the interface file, and *prefixing* the python Extension with an underscore achieves the same result.

#### Useful command line options <a name="toc-sub-tag-2"></a>
- `-builtin`: create python built-in types rather than proxy classes for better performance
- `-doxygen`: convert C++ doxygen comments to pydoc comments in proxy classes
- `-interface <mod>`: set low-level C/C++ module name to `<mod>` (default: module name prefixed by an underscore)
- `-py3`: generate code with python3 specific features
- `-O`: enable the `-fastdispatch`, `-fastproxy`, `-fvirtual` optimizations

## Python Packages <a name="toc-sub-tag-3"></a>

To mimic the package hierarchy of python, SWIG provides the `package` keyword in the module definition
```C
%module(package="some_package") module_name
```
which would map to an import
```py
import some_package.module_name
```

Consider the following example project structure:
```
hello/
    world.c
    world.h
    world.i

setup.py
```
where the contents of `world.i` defines the module `world` under the package `hello`, along the lines of
```C
// hello/world.i
%module(package="hello") world

%{
    #define SWIG_FILE_WITH_INIT
    #include "example.h"
%}

// functions-to-wrap declerations, e.g.
int foo();
```

We then define a `setup.py` file to build the extension with
```py
from distutils.core import setup, Extension

example_module = Extension(
    '_world',
    sources=[
        'hello/example_wrap.c',
        'hello/example.c'
    ],
    language='c'
)

setup(
    name='example',
    version='0.1',
    ext_package='hello', # !important
    ext_modules=[
        example_module
    ]
)
```

The full installation procedure is then the usual
```bash
swig -python -py3 hello/world.i \
    && python setup.py build_ext --inplace
```
where *in theory* after the call to swig has been made, generating the relevant `*_wrap.c` and `*.py` files, we could bundle and ship the files and allow someone without swig to build the package with only the call to `setup.py`.

It can then be used with
```py
>>> import hello.world 
>>> hello.world.foo()
0
```

This recipe could then be extended for arbitrary package structure and/or modules, where we define a single interface `.i` file *for each* module in the package.

## Binding abstractions <a name="toc-sub-tag-4"></a>
The SWIG documentation lists how many higher-level classes and types in python are mapped to C/C++ types and classes.

### Pointers <a name="toc-sub-tag-5"></a>
Pointer bindings [here](http://swig.org/Doc3.0/Python.html#Python_nn18).
### Structures <a name="toc-sub-tag-6"></a>
Structure bindings [here](http://swig.org/Doc3.0/Python.html#Python_nn19).
### Classes <a name="toc-sub-tag-7"></a>
Class bindings [here](http://swig.org/Doc3.0/Python.html#Python_nn20).
How SWIG generates shadow classes and mappings [here](http://swig.org/Doc3.0/Python.html#Python_nn28).


## A worked example: Python <a name="toc-sub-tag-8"></a>
Following from [the documentation](http://swig.org/Doc3.0/Python.html#Python).

We create three files
```cpp
// example.cpp

#include "example.hpp"

int fact(int n) {
    if (n < 0){ /* This should probably return an error, but this is simpler */
        return 0;
    }
    if (n == 0) {
        return 1;
    }
    else {
        /* testing for overflow would be a good idea here */
        return n * fact(n-1);
    }
}
```
with header
```cpp
// example.hpp

#ifndef EXAMPLE_HPP
#define EXAMPLE_HPP

int fact(int n);

#endif
```
And then the SWIG interface file:
```C
// example.i
%module example

%{
    #define SWIG_FILE_WITH_INIT
    #include "example.hpp"
%}

int fact(int n);
```

### SWIG CLI <a name="toc-sub-tag-9"></a>

We can then compile the wrapped C++ code for Python with 
```bash
swig -python -c++ example.i
```
which generates an `example_wrap.cxx`, and an `example.py`, i.e. the low level C/C++ code, and the high level wrapping support.

The module name defined by the line
```C
%module example
```
is used as a prefix for the output files.

### Distutils <a name="toc-sub-tag-10"></a>
We can use Distutils to compile and build the extension modules generated by the SWIG CLI. Disutils, in short,
> takes care of making sure that your extension is built with all the correct flags, headers, etc. for the version of Python it is run with.

To use it, we define a `setup.py` file
```py
# setup.py

from distutils.core import setup, Extension

example_module = Extension(
    '_example', # convention to prefix with underscore
    sources=[
        'example_wrap.c',
        'example.c'
    ],
    language='c++'
)

setup(
    name='example',
    version='0.1',
    ext_modules=[example_module],
    py_modules=['example']
)
```
We invoke the build with
```bash
python setup.py build_ext --inplace
```
where the `--inplace` flag specifies to compile the binaries into the current working directory, instead of within the build hierarchy.