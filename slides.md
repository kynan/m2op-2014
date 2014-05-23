class: center, middle, inverse

# Firedrake: a High-level, Portable Finite Element Computation Framework

### **Florian Rathgeber**<sup>1</sup>, Lawrence Mitchell<sup>1</sup>, David Ham<sup>1,2</sup>, Andrew McRae<sup>2</sup>, Fabio Luporini<sup>1</sup>, Gheorghe-teodor Bercea<sup>1</sup>, Paul Kelly<sup>1</sup>

Slides: http://kynan.github.io/m2op-2014

.footnote[<sup>1</sup> Department of Computing, Imperial College London
<sup>2</sup> Department of Mathematics, Imperial College London]

---

.left-column[
  ## Computational Science is hard

  Unless you break it down with the right abstractions

  Many-core hardware has brought a paradigm shift to CSE, scientific software needs to keep up
]
.right-column[
## The Solution

### High-level structure

  * Goal: producing high level interfaces to numerical computing
  * PyOP2: a high-level interface to unstructured mesh based methods  
    *Efficiently execute kernels over an unstructured grid in parallel*
  * Firedrake: a performance-portable finite-element computation framework  
    *Drive FE computations from a high-level problem specification*

### Low-level operations

  * Separating the low-level implementation from the high-level problem specification
  * Generate platform-specific implementations from a common source instead of hand-coding them
  * Runtime code generation and JIT compilation open space for compiler-driven optimizations and performance portability
]

---

class: center, middle
# Parallel computations on unstructured meshes with PyOP2

---

## Structure of scientific computations on unstructured meshes

* Independent *local operations* for each element of the mesh described as a *kernel*.
* *Reductions* aggregating results from local operations to produce the final result.

## PyOP2

A domain-specific language embedded in Python for parallel computations on unstructured meshes or graphs.

### Unstructured meshes
Meshes are described by *sets* and *maps* between them and can have data associated with them:

![PyOP2 mesh](images/op2_mesh.svg)

---

## PyOP2 Data Model

### Mesh topology
* Sets – cells, vertices, etc
* Maps – connectivity between entities in different sets

### Data
* Dats – Defined on sets (hold pressure, temperature, etc)

### Kernels
* Executed in parallel on a set through a parallel loop
* Read / write / increment data accessed via maps

### Linear algebra
* Matrix sparsities defined by mappings
* Matrix data on sparsities
* Kernels compute a local matrix – PyOP2 handles global assembly

---

.left-column[
## PyOP2 Kernels and Parallel Loops

Performance portability for *any* unstructured mesh computations
]
.right-column[
### Parallel loop syntax
```python
op2.par_loop(kernel, iteration_set,
             kernel_arg1(access_mode, mapping[index]),
             ...,
             kernel_argN(access_mode, mapping[index]))
```

### PyOP2 program for computing the midpoint of a triangle
```python
from pyop2 import op2
op2.init()

vertices = op2.Set(num_vertices)
cells = op2.Set(num_cells)

cell2vertex = op2.Map(cells, vertices, 3, [...])

coordinates = op2.Dat(vertices ** 2, [...], dtype=float)
midpoints = op2.Dat(cells ** 2, dtype=float)

midpoint = op2.Kernel("""
void midpoint(double p[2], double *coords[2]) {
  p[0] = (coords[0][0] + coords[1][0] + coords[2][0]) / 3.0;
  p[1] = (coords[0][1] + coords[1][1] + coords[2][1]) / 3.0;
}""", "midpoint")

op2.par_loop(midpoint, cells,
             midpoints(op2.WRITE),
             coordinates(op2.READ, cell2vertex))
```
Future work: define kernels as Python functions
]

---

## PyOP2 Architecture
* Parallel scheduling: partitioning, staging and coloring
* Runtime code generation and JIT compilation

![PyOP2 implementation](images/pyop2_architecture.svg)

---

## Generated sequential code calling the midpoint kernel

```c
// Kernel provided by the user
static inline void midpoint(double p[2], double *coords[2]) {
  p[0] = (coords[0][0] + coords[1][0] + coords[2][0]) / 3.0;
  p[1] = (coords[0][1] + coords[1][1] + coords[2][1]) / 3.0;
}

// Generated marshaling code executing the sequential loop
void wrap_midpoint(int start, int end,
                   double *arg0_0, double *arg1_0, int *arg1_0_map0_0) {
  double *arg1_0_vec[3];
  for ( int n = start; n < end; n++ ) {
    int i = n;
    arg1_0_vec[0] = arg1_0 + (arg1_0_map0_0[i * 3 + 0])* 2;
    arg1_0_vec[1] = arg1_0 + (arg1_0_map0_0[i * 3 + 1])* 2;
    arg1_0_vec[2] = arg1_0 + (arg1_0_map0_0[i * 3 + 2])* 2;
*   midpoint(arg0_0 + i * 2, arg1_0_vec);
  }
}
```

---

## Generated OpenMP code calling the midpoint kernel

```c
// Kernel provided by the user
static inline void midpoint(double p[2], double *coords[2]) {
  p[0] = (coords[0][0] + coords[1][0] + coords[2][0]) / 3.0;
  p[1] = (coords[0][1] + coords[1][1] + coords[2][1]) / 3.0;
}

// Generated marshaling code executing the parallel loop
void wrap_midpoint(int boffset, int nblocks,
                   int *blkmap, int *offset, int *nelems,
                   double *arg0_0, double *arg1_0, int *arg1_0_map0_0) {
  #pragma omp parallel shared(boffset, nblocks, nelems, blkmap) {
    int tid = omp_get_thread_num();
    double *arg1_0_vec[3];
    #pragma omp for schedule(static)
    for ( int __b = boffset; __b < boffset + nblocks; __b++ ) {
      int bid = blkmap[__b];
      int nelem = nelems[bid];
      int efirst = offset[bid];
      for (int n = efirst; n < efirst+ nelem; n++ ) {
        int i = n;
        arg1_0_vec[0] = arg1_0 + (arg1_0_map0_0[i * 3 + 0])* 2;
        arg1_0_vec[1] = arg1_0 + (arg1_0_map0_0[i * 3 + 1])* 2;
        arg1_0_vec[2] = arg1_0 + (arg1_0_map0_0[i * 3 + 2])* 2;
*       midpoint(arg0_0 + i * 2, arg1_0_vec);
      }
    }
  }
}
```

---

## PyOP2 Partitioning, Staging & Coloring

### Key optimizations performed by PyOP2 runtime core
* *Partitioning* for on-chip memory (shared memory / cache)
* *Coloring* to avoid data races on updates to the same memory location

### Example

Parallel computation executing a kernel over the edges of the mesh:
```python
# Sets of nodes and edges
nodes = op2.Set(N) # N = number of nodes
edges = op2.Set(M) # M = number of edges

# Mapping from edges to nodes
edge_to_node_map = op2.Map(edges, nodes, 2, ...)

# Data defined on nodes
u = op2.Dat(nodes, ...)

# Kernel executing over set of edges, computing on nodal data
op2.par_loop(kernel, edges,
             u(egde_to_node_map[:], op2.INC))
```

---

background-image:url(images/partitioning.svg)

---

background-image:url(images/staging.svg)

---

background-image:url(images/coloring.svg)

---

class: center, middle

# Finite-element computations with Firedrake

---

background-image:url(images/fem.svg)

---

.left-column[
## UFL: High-level definition of finite-element forms

UFL is the [Unified Form Language](https://bitbucket.org/fenics-project/ufl) from the [FEniCS project](http://fenicsproject.org).
]
.right-column[
### The weak form of the Helmholtz equation

`$$\int_\Omega \nabla v \cdot \nabla u - \lambda v u ~dV = \int_\Omega v f ~dV$$`

### And its (almost) literal translation to Python with UFL

UFL: embedded domain-specific language (eDSL) for weak forms of partial differential equations (PDEs)
```python
e = FiniteElement('CG', 'triangle', 1)

v = TestFunction(e)
u = TrialFunction(e)
f = Coefficient(e)

lmbda = 1
a = (dot(grad(v), grad(u)) - lmbda * v * u) * dx

L = v * f * dx
```
]

---

.left-column[
## Helmholtz local assembly kernel generated by FFC

The [FEniCS Form Compiler](https://bitbucket.org/fenics-project/ffc) FFC compiles UFL forms to low-level code.
]
.right-column[
### Helmholtz equation
`$$\int_\Omega \nabla v \cdot \nabla u - \lambda v u ~dV = \int_\Omega v f ~dV$$`

### UFL expression
```python
a = (dot(grad(v), grad(u)) - lmbda * v * u) * dx
```

### Generated C code
```c
// A - local tensor to assemble
// x - local coordinates
// j, k - 2D indices into the local assembly matrix
void kernel(double A[1][1], double *x[2],
            int j, int k) {
  // FE0 - Shape functions
  // Dij - Shape function derivatives
  // Kij - Jacobian inverse / determinant
  // W3  - Quadrature weights
  // det - Jacobian determinant
  for (unsigned int ip = 0; ip < 3; ip++) {
    A[0][0] += (FE0[ip][j] * FE0[ip][k] * (-1.0)
      + (((K00 * D10[ip][j] + K10 * D01[ip][j]))
        *((K00 * D10[ip][k] + K10 * D01[ip][k]))
      +  ((K01 * D10[ip][j] + K11 * D01[ip][j]))
        *((K01 * D10[ip][k] + K11 * D01[ip][k])))) * W3[ip] * det;
  }
}
```
]

---

## The Firedrake/PyOP2 tool chain

![Firedrake](images/firedrake_toolchain.svg)

---

## Two-layered abstraction: Separation of concerns

![Separation of concerns](images/firedrake_toolchain_users.svg)

---

## Driving Finite-element Computations in Firedrake

Solving the Helmholtz equation in Python using Firedrake:
`$$\int_\Omega \nabla v \cdot \nabla u - \lambda v u ~dV = \int_\Omega v f ~dV$$`

```python
from firedrake import *

# Read a mesh and define a function space
mesh = Mesh('filename')
V = FunctionSpace(mesh, "Lagrange", 1)

# Define forcing function for right-hand side
f = Expression("- (lmbda + 2*(n**2)*pi**2) * sin(X[0]*pi*n) * sin(X[1]*pi*n)",
               lmbda=1, n=8)

# Set up the Finite-element weak forms
u = TrialFunction(V)
v = TestFunction(V)

lmbda = 1
a = (dot(grad(v), grad(u)) - lmbda * v * u) * dx
L = v * f * dx

# Solve the resulting finite-element equation
p = Function(V)
solve(a == L, p)
```

---

## Finite element assembly and solve in Firedrake

What goes on behind the scenes of the `solve` call (simplified example!):
```python
from pyop2 import op2, ffc_interface

def solve(equation, x):
    # Invoke FFC to generate kernels for matrix and rhs assembly
    lhs = ffc_interface.compile_form(equation.lhs, "lhs")[0]
    rhs = ffc_interface.compile_form(equation.rhs, "rhs")[0]

    # Omitted: extract coordinates (coords), connectivity (elem_node)
    # and coefficients (field f)

    # Construct OP2 matrix to assemble into
    sparsity = op2.Sparsity((elem_node, elem_node), sparsity_dim)
    mat = op2.Mat(sparsity, numpy.float64)
    b = op2.Dat(nodes, np.zeros(nodes.size))

    # Assemble lhs, rhs and solve linear system
    op2.par_loop(lhs, elements,
                 mat(op2.INC, (elem_node[op2.i[0]], elem_node[op2.i[1]])),
                 coords(op2.READ, elem_node))
    op2.par_loop(rhs, elements,
                 b(op2.INC, elem_node[op2.i[0]]),
                 coords(op2.READ, elem_node),
                 f(op2.READ, elem_node))

    # Solve the assembled sparse linear system of equations
    op2.solve(mat, x, b)
```

---

class: center, middle

# Recap

---

## Conclusions and Future Work

### Conclusions
* PyOP2: a high-level interface to unstructured mesh based methods  
  *Efficiently execute kernels over an unstructured grid in parallel*
* Firedrake: a performance-portable finite-element computation framework  
  *Drive FE computations from a high-level problem specification*
* Two-layer abstraction for FEM computation from UFL sources
* Decoupling of UFL (FEM) and PyOP2 (parallelisation) layers
* Target-specific runtime code generation and JIT compilation
* Performance portability for unstructured grid applications: FEM, non-FEM or combinations

### Future Work
* Auto-tuning of optimization parameters (e.g. iteration space)
* Support for curved (isoparametric) finite elements
* Kernel fusion, lazy evaluation
* Alternative code generation / JIT approaches (LLVM)

---

## Thank you!

Contact: Florian Rathgeber, [@frathgeber](https://twitter.com/frathgeber), f.rathgeber@imperial.ac.uk

### Resources

  * **PyOP2** https://github.com/OP2/PyOP2
    * *[PyOP2: A High-Level Framework for Performance-Portable Simulations on Unstructured Meshes](http://dx.doi.org/10.1109/SC.Companion.2012.134)*
      Florian Rathgeber, Graham R. Markall, Lawrence Mitchell, Nicholas Loriant, David A. Ham, Carlo Bertolli, Paul H.J. Kelly,
      WOLFHPC 2012
    * *[Performance-Portable Finite Element Assembly Using PyOP2 and FEniCS](http://link.springer.com/chapter/10.1007/978-3-642-38750-0_21)*
       Graham R. Markall, Florian Rathgeber, Lawrence Mitchell, Nicolas Loriant, Carlo Bertolli, David A. Ham, Paul H. J. Kelly ,
       ISC 2013
  * **Firedrake** https://github.com/firedrakeproject/firedrake
    * *COFFEE: an Optimizing Compiler for Finite Element Local Assembly*
      Fabio Luporini, Ana Lucia Varbanescu, Florian Rathgeber, Gheorghe-Teodor Bercea, J. Ramanujam, David A. Ham, Paul H. J. Kelly,
      submitted
  * **UFL** https://bitbucket.org/mapdes/ufl
  * **FFC** https://bitbucket.org/mapdes/ffc

**This talk** is available at http://kynan.github.io/m2op-2014

Slides created with [remark](http://remarkjs.com)