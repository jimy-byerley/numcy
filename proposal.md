 The API proposal
=================

This document describes shortly the API, interfaces and protocols of the new library. This is not a complete reference, nor a complete architecture description, but the main lines are there. 

The class names and some method names may need to be reworked to get more coherence.



## dtypes

Can be any of the following:

- **built from a string**, defining the memory layout of each element
	
	```python
	'3f4 5x 12u1'
	```

- **a builtin type**, (be careful: the array operations will copy the content without consideration for any object constructor/destructor, so be sure you use only basic static data objects with a fixed size here)

	```python
	<builtin type 'vec3'>
	```

- **a pure python class**, defining methods `tobytes` and `frombytes`, optionally defining its memory layout in the dumped bytes using a member `packlayout`

	typical pure python dtype:

	```python
	class quaternion:
		packlayout = '4f8'
		vectorized = { ... }
		def tobytes(self):  ...
		def frombytes(bytes): ...
	```
	
No registeration of the dtype is necessary to use it in one of the arrays defined here. in order to improve the efficiency of operations, you may declare some vectorized implementations (see below).

#### dtype layout strings

The syntax is a simple sequence of this kind: `'[N]TP[A] [N]TP[A] ...'`

- N is the number of repeated elements of this primitive type
- T is the primitive type, that gives a meaning to the data (`u`, `i`, `f`)
- P is the precision (number of bytes used)
- A is alignment specification (`'<'` or `'>'`)

example 
`'f8'` means 64bits precison floating point number
`'13i4'` means 13 successive 32 bytes integers

table of possible types

| precision | u  | i  | f  |  C-equivalent         
|-----------|----|----|----|-----------------------
| 1         | u1 | i1 | f1 | unsigned char, char
| 2         | u2 | i2 | f2 | unsigned short int, short int 
| 4         | u4 | i4 | f4 | unsigned short int, float
| 8         | u8 | i8 | f8 | unsigned long, long, double

It can happens that you want to keep unused spaces in a dtype, then replace the unused bytes by `x`:  
	`'3f4 5x 2u1'` keeps 5 bytes between a group of floats and a group of unsigned.


## optimized operations

to get the maximum efficiency on repeated operations in an array, np.array uses ufuncs (functions defined both for one only element and for an array), the same would apply here

the module will declare dictionnary as global variable to associate each operation to an optimized function:

```python
nc.vectorized = {('__iadd__', dtype, dtype): func}
```

for custom dtypes, it would be better to define a member `vectorize` using the same format (the module global dictionnary defaults to the member dictionnary).



## class array

Strict equivalent of np.ndarray, for unstructured arrays.

#### Readonly members:

+ `dtype`
+ `ptr` - int

	pointer to the referenced memory

+ `size` - int

	the byte size of the allocated area pointed

+ `strides` - compiled tuple
	
	the number of bytes between elements in each dimension

+ `owner` - object

	the python object that own the pointed data, making this memory view compatible with counted references and readonly buffers and so on ...

#### Constructors:

no copy of the internal data is made, it just ensures that the result is an array

- `array(memoryview, dtype=None)`

	build from any object that implements the buffer protocol
	by default uses the dtype of the retreived memoryview

- `array(iterable, dtype=None)`

	build from python objects, the objects must be dtype instances (see above) or iterables of dtype instances

- `array(other, dtype=None)`

	simple reinterpretation of the buffer of the other array, no copy is made even if the provided dtype doesn't match

#### Item get/set

This kind of array is multidimensional (given the shape), the first sub items are used to take square regions of the matrix. The next indices will be used as sub items to index the layout fields.
Slices are working on the same way, but a returns an `array` instead of a `buffer`.

- `__getitem__`  bytes are retreived from the memory and used to create an instance of the matching dtype. If there is no sub index, the dtype is `self.dtype` else its a part of the dtype layout.

- `__setitem__` the value is dumped to bytes and copied to the matching memory zone.

#### Methods:

- `swap(other)`

	memory content swap between this array and the other

- `reshape(shape, fit=True)`

	Return an array pointing the same memory, but using the new shape.
	If fit is True, will raise an exception if the new is smaller than the memory size.

- `cast(dtype) -> array`

	return an array of the same shape, but source elements will be converted into the new dtype. At contrary to the array constructor that only retinterpret the memory.

- `zeros() -> self`

	fill with zeros. Works for custom dtypes if they defines `packlayout`.
	if shape is a tuple, the result is an `array` else a `buffer`.
	
- `full(element) -> self`

	fill with byte copies of the given element.

- `map(func) -> array`

	apply a function to each element

- `imap(func)`

	apply a function to each element, storing the content inplace

- `find(x)`

	return the index/location of the first occurence of x (byte match)

- `__add__`, `__mul__`, etc

	for syntaxic operations

- `__matmul__`

	to provide the `A @ B` syntax for matrix multiplication, the current shape is used and elements should support `__add__` and `__mul__`

- `__len__`

	gives the total number of elements
	



## class buffer

Sort of ndarray, but 1-dimension only to allow append capabilities.

#### Readonly members:

+ `dtype`
+ `ptr` - int

	pointer to the allocated memory

+ `size` - int

	size of the used space

+ `allocated` - int

	size of the allocated memory

The constructors signatures of array applies here

#### Item get/set

This kind of array is one dimension only, sub items can be used to index the layout fields.
Slices are working on the same way, but a returns an `array` instead of a `buffer`.

#### Methods:

list-like methods
- `append(x)`
- `insert(i, x)`
- `pop(i)`
- `reserve(n)`

	ensure that n additional elements can be stored in without reallocation, reallocate if necessary.

- `shrink()`

	reduce the allocated area to only the used space

array-like methods
- `swap(other)`
- `cast(dtype) -> buffer`
- `zeros()`
- `full(element)`
- `map(func) -> buffer`
- `imap(func)`
- `find(x, start=0, end=-1) -> int`

	find the first occurence of x (byte match)

- `__add__`, `__mul__`, etc
- `__matmul__`

	to provide the `A @ B` syntax for matrix multiplication, considering that the current buffer is a column array, elements should support `__add__` and `__mul__`

- `__len__`

#### Notes

As you can see, buffers are sharing their pointer to data through the buffer protocol and to any array class used a a view. Which can lead to memory corruption because the `buffer`'s pointer can change with its size. I doesn't happend here, because the storage used for a `buffer` is a python object `bytearray` and is ref-counted (its shared through the property `owner` to view arrays). So when the `buffer` reallocates to grow, the old memory lasts for as long as someone is using it.



## class zipped

Proxy to harvest multiple arrays, and access it by keys or indices as if they were one only array

#### Readonly members:

+ `arrays`- [numcy arrays]
+ `names` - {key/index: index}
+ `dtype`

	type used to access this array at an element: the dtype to use when concatenating byte elements from the sub arrays

#### Constructors

- `zipped(*arrays, dtype=None)`
- `zipped([('name': array)], dtype=None)`
- `zipped({'name': array}, dtype=None)`

#### item get/set

The array is indexed by the keys and by associated integer index (the same as for self.arrays).
The sub-indexing is like buffer's

- `__getitem__` bytes parts are retreived from the sub arrays, then a dtype instance is created onto it and returned.

- `__setitem__` the assigned item is transformed to bytes, then distributed across sub arrays.

Slicing is possible even with sub indices, it provides a zipped on slices of the subarrays.

#### Methods:

array-like methods, reproducing approximately the same behaviors as `buffer` methods
- `cast(dtype) -> buffer`
- `swap(other)`
- `zeros()`
- `full(element)`
- `map(func) -> buffer`
- `imap(func)`
- `find(x) -> int`
- `find(x, start=0, end=0) -> int`

- `__add__`, `__mul__`, etc
- `__matmul__`

	to provide the `A @ B` syntax for matrix multiplication, considering that the current buffer is a column array, elements should support `__add__` and `__mul__`

- `__len__`

	the common length to the sub arrays


## class readonly

this is a proxy on the `array` class, but with only the read abilities, no inplace or setitem operations. It only integrates a ref on a normal array.



## class sparse

sparse array, the memory is not a buffer here so it's not exposed, but instead there is a dictionnary of non-null positions.
TODO

## class bits

array specialized for boolean values only, better in memory usage.
TODO

## class io

array using a file as memory, usefull for arrays too large to be loaded to RAM (append and pop methods).
TODO


## functions at module level

- `stack(*arrays, dim=0)`

	Equivalent of `[ ... ] + [ ... ]` but for buffers and arrays instead of lists (the addition syntax is already in use for the per-element operations).
	Note that `dim` specifies the stack dimension, but is limited by the number of dimensions. using `buffer` for instance, only 0 is valid.

	The return type depends on the input arrays types.

- `merge(*arrays)`

	Merges the dtypes and create an array that concatenate element by element the given arrays.

	The return type depends on the input arrays types.

- `sizeof(dtype) -> int`, `sizeof(array) -> int`

	The byte len of an instance of this dtype, or the current byte len of the array content.

- `empty(len/shape, dtype) -> buffer/array`
	
	array with uninitialized memory 
	
	NOTE: the dtype must allow this

	

## sub-module math

math functions working on arrays are placed here, and are just shorthands.

These math functions are defined as follow:
```python
import math  # the python math module
def tan(x: array):
	return x.apply(math.tan)
```
In this example, `x.apply` is getting the right optimized function to do the task. Its parameter is used as key to find an optimized function in the `vectorized` dictionnaries, if nothing is found, then the basic element by element operation is executed (with no efficiency gain).
