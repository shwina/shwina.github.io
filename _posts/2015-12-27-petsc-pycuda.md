---
layout: post
title: "Using PETSc for Python with PyCUDA"
---

This post is about how to use the
[petsc4py](https://bitbucket.org/petsc/petsc4py)
and [PyCUDA](https://mathema.tician.de/software/pycuda/)
together to write applications that use several GPUs
in parallel.

PETSc supports the use of CUDA GPUs via the
[CUSP](https://developer.nvidia.com/cusp) C++ library.
The PETSc provided `VECCUSP` and `AIJCUSP` classes
are used to store vectors and matrices respectively on GPUs.
Vector operations and matrix vector products
with these classes are performed on the GPU.
All the PETSc linear solvers (except BiCG)
are thus able to run entirely on the GPU.

Sometimes, you'll want to perform computations with vectors
that can't be expressed using the available vector operations
or matrix-vector products.
In this case, you'll need access to the
underlying GPU (device) buffer,
which you pass to your own CUDA kernel that performs the computation.

In C++, this is done using the PETSc functions
`VecCUSPGetArrayRead` and `VecCUSPGetArrayWrite`
that expose the underlying CUSP vectors.
From the CUSP vector,
you can get a pointer to the raw device memory
using the `thrust::raw_pointer_cast` function.
This device pointer can be passed to your own custom kernels,
functions from other GPU libraries, whatever.
See a few examples of this [here](https://www.mcs.anl.gov/petsc/petsc-current/src/vec/vec/impls/seq/seqcusp/veccusp.cu#VecCUSPGetCUDAArray).

Here's how to do it with `petsc4py` and PyCUDA.
We'll multiply two vectors (elementwise)
using a custom kernel.
The following bits of code all go in a single Python script `gpumult.py`.
First, we'll create the input and output vectors:

```python
from petsc4py import PETSc
from pycuda import autoinit
import pycuda.driver as cuda
import pycuda.compiler as compiler
import pycuda.gpuarray as gpuarray
import numpy as np

N = 8

# create input vectors
a = PETSc.Vec().create()
a.setType('cusp')
a.setSizes(N)

b = PETSc.Vec().create()
b.setType('cusp')
b.setSizes(N)

# create output vectors
c = PETSc.Vec().create()
c.setType('cusp')
c.setSizes(N)

# set initial values:
a.set(3.0)
b.set(2.0)
c.set(0.0)
```

Next, we'll use the `getCUDAHandle` method
to get the raw CUDA pointers
of the underlying GPU buffers:

```python
a_ptr = a.getCUDAHandle()
b_ptr = b.getCUDAHandle()
c_ptr = c.getCUDAHandle()
```

Next, we'll write a CUDA kernel implementing
the elementwise product, and use PyCUDA to interface with it:

```python
kernel_text = '''
__global__ void gpuElemwiseProduct(double* a_d,
    double* b_d, double* c_d, int n) {
    int i = threadIdx.x + blockIdx.x*blockDim.x;
    c_d[i] = a_d[i]*b_d[i];
}
'''

mult = compiler.SourceModule(kernel_text,
        options=['-O2']).get_function('gpuElemwiseProduct')
mult.prepare('PPPi')
```

Now, we'll perform the multiplication:

```python
mult.prepared_call((1, 1, 1), (N, 1, 1),
    a_ptr, b_ptr, c_ptr, N)
```

The API requires us to "restore" the CUDA handles.

```python
a.restoreCUDAHandle(a_ptr)
b.restoreCUDAHandle(b_ptr)
c.restoreCUDAHandle(c_ptr)
```

Actually, it doesn't matter what the input is to
the `restoreCUDAHandle` function - but whatever.

Now look at the output vector:

```
c.view()
```

Here's a run of the above program on 2 processes:

```bash
$ mpiexec -n 2 python gpumult.py 
Vec Object: 2 MPI processes
  type: mpicusp
Process [0]
6.
6.
6.
6.
Process [1]
6.
6.
6.
6.
```

You can also use the
raw pointer to construct a PyCUDA GPUArray,
or pass it to a function from the
`scikits.cuda` library, or wherever else.
