NUMCY   -  A Better Numpy
=========================


### Project not started yet

[Python](https://www.python.org/) is my favorite language, and using it everyday I came to consider that one of the thing in python I don't like to work with was [numpy](https://numpy.org/). How is that possible ?! on can say, numpy is so useful ! it is one of the best tool in order to get efficiency in computation ! 

And indeed it is ...

.

### Mainly, here is what I don't like in numpy:


- Not polymorphic

	this is the most impotant point here: there is only one data type in numpy: `array`,  it is intended to represent any type of array regardless on its internal structure, and it is not subclassable. 
        
	This can lead to headaches as lib developers when we want to retrieve data from a numpy array, and as software developper when we are looking for a class well suited for our application


- Difficult to extend

	That is comming with the previous, and its important too.

	As the internal structure of a `np.array` is complex, it is impossibe to create specialized versions of arrays and it is complex to add custom dtypes (just look at what is needed to get [quaternions as dtype](https://github.com/moble/quaternion))

- Inefficient for python data access

	That's the cost of the great flexibility of the array type: accessing a `np.array` element by element from python is less efficient than using a list or even a dict containing the same data (performance is lost in checks). This could be solved if arrays could be specialized.

- Slow for import

	That can be a problem for small python scripts that load/operate and display datas on command line
        
- The operators are all put in the global scope, and there is plenty.
    
- Too many redundant functions

	every array has a `__add__` method that provide the syntaxic `'+'` operator, but also exists `np.add`. likewise, `stack` exists, but there is also `hstack`, `vstack`. we can do `len(a.shape)` but there is also a `np.shape` and a `np.ndim`.
	
	all these functions can be usefull indeed, but it lacks of simplicity.

### goals

This project is a proposal to rebuild a new numpy-like library from scratch. The idea is to keep all the current advantages of numpy, but to fix the previous stated issues. 

I think that [Cython](https://cython.org/) would be great to write this module (hence the proposal name), as it's very close to python and allows to write very optimized code without making it hard to include python interactions.
The [proposal](proposal.md) explains the main lines of the new proposed API.

If in a way or an other you think that refactoring the numpy is a good idea, please come to discuss it, open an issue or mail me at [jimy.byerley@gmail.com](jimy.byerley@gmail.com)


