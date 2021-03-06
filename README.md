# Calling Python functions from the Julia language

This package provides a `pycall` function (similar in spirit to
Julia's `ccall` function) to call Python functions from the Julia
language, automatically converting types etcetera.

## Work in Progress

PyCall is currently a proof-of-concept and work in progress.  Much
basic functionality works, but major TODO items are:

* Conversions for many more types (set, range, xrange, etc.).  Callback
  functions.

* Support for keyword arguments.

* Syntactic sugar.

**Note**: In order to load Python modules such as NumPy that are partly
implemented in shared libraries (i.e., which are not purely written in Python),
you must patch Julia with the fix in [Julia issue #2317](https://github.com/JuliaLang/julia/pull/2317).  This is required for multidimensional array conversions, which rely on NumPy.

## Installation

Until this package stabilizes and is added to Julia's global
[METADATA.jl](https://github.com/JuliaLang/METADATA.jl) database, you should
do

    cd ~/.julia
    git clone https://github.com/stevengj/PyCall.jl PyCall

to fetch the latest version of the package and install it in your
`.julia` directory (or somewhere else if you prefer).

## Usage

Here is a simple example to call Python's `math.sin` function:

    using PyCall
    math = pyimport("math") # import the Python math module
    pycall(math["sin"], Float64, 3.0) - sin(3.0) # returns 0.0

Note that `math["sin"]` looks up the `sin` function in the Python
`math` module, and is the equivalent of `math.sin` in Python.

Like `ccall`, we must tell `pycall` the return type we wish in Julia.
In principle, this could be determined dynamically at runtime but that
is not currently implemented, in part because that would defeat
efficient compilation in Julia (since the Julia compiler would be
unable to determine the return type at compile-time).  On the other
hand, the argument types need not be specified explicitly; they will
be determined from the types of the Julia arguments, which will be
converted into the corresponding Python types.  For example:

    pycall(math["sin"], Float64, 3) - sin(3.0)

also works and also returns `0.0` since Python's `math.sin` function accepts
integer arguments as well as floating-point arguments.

Currently, numeric, boolean, and string types, along with tuples and
arrays/lists thereof, are supported, with more planned.

You can also look up other names in a module, and use `convert` to
convert them to Julia types, e.g.

    convert(Float64, math["pi"])

returns the numeric value of &pi; from Python's `math.pi`.

## Reference

The PyCall module supplies several subroutines, types, and conversion routines
to simplify calling Python code.

### Calling Python

* `pyimport(s)`: Import the Python module `s` (a string or symbol) and
  return a pointer to it (a `PyObject`).   Functions or other symbols
  in the module may then be looked up by `s[name]` where `name` is a string
  or symbol (`s[name]` also returns a `PyObject`).

* `pycall(function::PyObject, returntype::Type, args...)`.   Call the given 
  Python `function` (typically looked up from a module) with the given
  `args...` (of standard Julia types which are converted automatically to
  the corresponding Python types if possible), converting the return value
  to `returntype` (use `returntype = PyObject` to return the unconverted
  Python object reference).

* `pybuiltin(s)`: Look up `s` (a string or symbol) among the global Python
  builtins.

### Types

#### PyObject

The PyCall module also provides a new type `PyObject` (a wrapper around
`PyObject*` in Python's C API) representing a reference to a Python object.

Constructors `PyObject(o)` are provided for a number of Julia types,
and PyCall also supplies `convert(T, o::PyObject)` to convert
PyObjects back into Julia types `T`.  Currently, the only types
supported are numbers (integer, real, and complex), booleans, and
strings, along with tuples and arrays/lists thereof, but more are planned.

#### PyArray

Multidimensional NumPy arrays (`ndarray`) are supported and can be
converted to the native Julia `Array` type, which makes a copy of the data.

Alternatively, the PyCall module also provides a new type `PyArray` (a
subclass of `AbstractArray`) which implements a no-copy wrapper around
a NumPy array (currently of numeric types or objects only).  Just use
`PyArray` as the return type of a `pycall` returning an `ndarray`, or
call `PyArray(o::PyObject)` on an `ndarray` object `o`.  (Technically,
a `PyArray` works for any Python object that uses the NumPy array
interface to provide a data pointer and shape information.)

Conversely, when passing arrays *to* Python, Julia `Array` types are
converted to `PyObject` types *without* making a copy via NumPy,
e.g. when passed as `pycall` arguments. **Warning:** If Python creates
a new reference to an `Array` object and returns it from `pycall`, you
*must* ensure that the original `Array` object still exists (i.e., is not
garbage collected) as long as any such "hidden" Python references
exist.

#### PyAny

The `PyAny` type is used in conversions to tell PyCall to detect the
Python type at runtime and convert to the corresponding native Julia
type.  That is, `pycall(func, PyAny, ...)` and `convert(PyAny,
o::PyObject)` both automatically convert their result to a native
Julia type (if possible).   This is convenient, but will lead
to slightly worse performance (due to the overhead of runtime type-checking
and the fact that the Julia JIT compiler can no longer infer the type).

### Initialization

By default, whenever you call any of the high-level PyCall routines
above, the Python interpreter (corresponding to the `python`
executable name) is initialized and remains in memory until Julia
exits.  However, you may want to modify this behavior to change the
default Python version, to call low-level Python functions directly
via `ccall`, or to free the memory consumed by Python.  This can be
accomplished using:

* `pyinitialize(s::String)`: Initialize the Python interpreter using
  the Python libraries corresponding to the `python` executable given
  by the argument `s`.  Calling `pyinitialize()` defaults to
  `pyinitialize("python")`, but you may need to change this to use a
  different Python version.  The `pyinitialize` function *must* be
  called before you can call any low-level Python functions (via
  `ccall`), but it is called automatically as needed when you use the
  higher-level functions above.  It is safe to call this function more
  than once; subsequent calls will do nothing (until `pyfinalize` is
  called).

* `pyfinalize()`: End the Python interpreter and free all associated memory.
  After this function is called, you may restart the Python interpreter
  by calling `pyinitialize` again.  It is safe to call `pyfinalize` more
  than once (subsequent calls do nothing).   You must *not* have any
  remaining variables referencing `PyObject` types when `pyfinalize` runs!

* The Python version number is returned by `pyversion()`, which returns
  Julia's native `VersionNumber` type.

### Low-level Python API access

If you want to call low-level functions in the Python C API, you can
do so using `ccall`.  Just remember to call `pyinitialize` first, and:

* Use `pyfunc(func::Symbol)` to get a function pointer to pass to `ccall`
  given a symbol `func` in the Python API.  e.g. you can call `int Py_IsInitialized()` by `ccall(pyfunc(:Py_IsInitialized), Int32, ())`.

* PyCall defines the typealias `PyPtr` for `PythonObject*` argument types,
  and `PythonObject` (see above) arguments are correctly converted to this
  type.

* Use `PythonObject` and the `convert` routines mentioned above to convert
  Julia types to/from `PythonObject*` references.

* If a new reference is returned by a Python function, immediately
  convert the `PyPtr` return values to `PythonObject` objects in order to
  have their Python reference counts decremented when the object is
  garbage collected in Julia.  i.e. `PythonObject(ccall(func, PyPtr, ...))`.

* You can call `pyincref(o::PyObject)` and `pydecref(o::PyObject)` to
  manually increment/decrement the reference count.  This is sometimes
  needed when low-level functions steal a reference or return a borrowed one.

* The function `pyerr_check(msg::String)` can be used to check if a
  Python exception was thrown, and throw a Julia exception (which includes
  both `msg` and the Python exception object) if so.  The Python 
  exception status may be cleared by calling `pyerr_clear()`.

## Author

This package was written by [Steven G. Johnson](http://math.mit.edu/~stevenj/).
