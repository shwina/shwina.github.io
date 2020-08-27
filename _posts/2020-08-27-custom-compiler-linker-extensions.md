---
layout: post
title: Customizing the compiler and linker when building C/C++ extensions for Python
---

This post is about how to customize the compiler and linker used
when building a C/C++ extension.


Sometimes, you want to build an extension module using a different compiler than
`gcc` or whatever default compiler is used by setuptools to build extensions.
Additionally, you may want to pass custom flags to the compiler/linker.
Importantly, you may want to prevent the "default" flags from being passed to your custom compiler,
perhaps because it doesn't support them. This post shows one way of achieving just that.


As an example, here is a simple `setup.py` for building an extension from a Cython file `foo.pyx`

```python
from setuptools import setup
from Cython.Build import cythonize

setup(name='foo', ext_modules=cythonize("foo.pyx"), zip_safe=False)
```

Invoking `setup.py`, we can see the compiler and flags used for building the extension:

```bash
$ python setup.py build_ext --inplace

...

gcc -pthread -B /home/ashwin/miniconda3/envs/cython-dev/compiler_compat -Wl,--sysroot=/ -Wsign-compare -DNDEBUG -g -fwrapv -O3 -Wall -Wstrict-prototypes -fPIC -I/home/ashwin/miniconda3/envs/cython-dev/include/python3.8 -c foo.c -o build/temp.linux-x86_64-3.8/foo.o

gcc -pthread -shared -B /home/ashwin/miniconda3/envs/cython-dev/compiler_compat -L/home/ashwin/miniconda3/envs/cython-dev/lib -Wl,-rpath=/home/ashwin/miniconda3/envs/cython-dev/lib -Wl,--no-as-needed -Wl,--sysroot=/ build/temp.linux-x86_64-3.8/foo.o -o build/lib.linux-x86_64-3.8/foo.cpython-38-x86_64-linux-gnu.so
copying build/lib.linux-x86_64-3.8/foo.cpython-38-x86_64-linux-gnu.so ->
```

Using a different compiler than `gcc` is easy -- just set the value of the `CC` environment variable.
For illustration, we'll consider changing the compiler from `gcc` to `g++`:


```bash
$ CC=g++ cythonize -i foo.pyx

...

g++ -pthread -B /path/to/compiler_compat -Wl,--sysroot=/ -Wsign-compare -DNDEBUG -g -fwrapv -O3 -Wall -Wstrict-prototypes -fPIC -c foo.c foo.o

gcc -pthread -shared -B -Wl,-rpath=/path/to/lib -Wl,--no-as-needed -Wl,--sysroot=/ foo.o -o foo.cpython-38-x86_64-linux-gnu.so
```

However, we notice two things when we do this:

1. The linking step still uses `gcc`
2. The arguments to the compiler remain the same as before. This can be a problem if the
   compiler you want to use doesn't support one or more of the default arguments.

One solution to these problems is to override the executables directly in `distutils` using the
following approach:

```python
from setuptools import setup, Extension
from Cython.Build import cythonize

from setuptools.command.build_ext import build_ext


class custom_build_ext(build_ext):
    def build_extensions(self):
        # Override the compiler executables. Importantly, this
        # removes the "default" compiler flags that would
        # otherwise get passed on to to the compiler, i.e.,
        # distutils.sysconfig.get_var("CFLAGS").
        self.compiler.set_executable("compiler_so", "g++")
        self.compiler.set_executable("compiler_cxx", "g++")
        self.compiler.set_executable("linker_so", "g++")
        build_ext.build_extensions(self)


extensions = [
    Extension(
        "*",
        sources=["foo.pyx"],
        extra_compile_args=["-fPIC", "-std=c++17"],
        extra_link_args=["-shared"]
    )
]

setup(
    name='foo',
    ext_modules=cythonize(extensions),
    zip_safe=False,
    cmdclass={"build_ext": custom_build_ext}
)
```

Doing this erases *all* the compiler flags passed to the compiler and linker,
so any flags you wish to pass must be done so via the
`extra_compiler_args` and `extra_link_args` options.

Now we see that we have full control over the flags passed to the compiler and linker commands:

```bash
$ python setup.py build_ext --inplace

...

building 'foo' extension
g++ -I/home/ashwin/miniconda3/envs/cython-dev/include/python3.8 -c foo.c -o build/temp.linux-x86_64-3.8/foo.o -fPIC -std=c++17
g++ build/temp.linux-x86_64-3.8/foo.o -o build/lib.linux-x86_64-3.8/foo.cpython-38-x86_64-linux-gnu.so -shared
copying build/lib.linux-x86_64-3.8/foo.cpython-38-x86_64-linux-gnu.so ->
```
