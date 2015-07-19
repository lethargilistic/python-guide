Speed
=====

CPython, the most commonly used implementation of Python, is slow for CPU bound
tasks. `PyPy`_ is fast.

Using a slightly modified version of `David Beazleys`_ CPU bound test code
(added loop for multiple tests), you can see the difference between CPython
and PyPy's processing.

.. code-block:: console

   # PyPy
   $ ./pypy -V
   Python 2.7.1 (7773f8fc4223, Nov 18 2011, 18:47:10)
   [PyPy 1.7.0 with GCC 4.4.3]
   $ ./pypy measure2.py
   0.0683999061584
   0.0483210086823
   0.0388588905334
   0.0440690517426
   0.0695300102234

.. code-block:: console

   # CPython
   $ ./python -V
   Python 2.7.1
   $ ./python measure2.py
   1.06774401665
   1.45412397385
   1.51485204697
   1.54693889618
   1.60109114647

Context
:::::::


The GIL
-------

`The GIL`_ (Global Interpreter Lock) is how Python allows multiple threads to
operate at the same time. Python's memory management isn't entirely thread-safe,
so the GIL is required to prevent multiple threads from running the same
Python code at once.

David Beazley has a great `guide`_ on how the GIL operates. He also covers the
`new GIL`_ in Python 3.2. His results show that maximizing performance in a
Python application requires a strong understanding of the GIL, how it affects
your specific application, how many cores you have, and where your application
bottlenecks are.

C Extensions
------------


The GIL
-------

`Special care`_ must be taken when writing C extensions to make sure you
register your threads with the interpreter.

C Extensions
::::::::::::


Cython
------

`Cython <http://cython.org/>`_ implements a superset of the Python language
with which you are able to write C and C++ modules for Python. Cython also
allows you to call functions from compiled C libraries. Using Cython allows
you to take advantage of Python's strong typing of variables and operations.

Here's an example of strong typing with Cython:

.. code-block:: cython

    def primes(int kmax):
    """Calculation of prime numbers with additional
    Cython keywords"""

        cdef int n, k, i
        cdef int p[1000]
        result = []
        if kmax > 1000:
            kmax = 1000
        k = 0
        n = 2
        while k < kmax:
            i = 0
            while i < k and n % p[i] != 0:
                i = i + 1
            if i == k:
                p[k] = n
                k = k + 1
                result.append(n)
            n = n + 1
        return result


This implementation of an algorithm to find prime numbers has some additional
keywords compared to the next one, which is implemented in pure Python:

.. code-block:: python

    def primes(kmax):
    """Calculation of prime numbers in standard Python syntax"""

        p= range(1000)
        result = []
        if kmax > 1000:
            kmax = 1000
        k = 0
        n = 2
        while k < kmax:
            i = 0
            while i < k and n % p[i] != 0:
                i = i + 1
            if i == k:
                p[k] = n
                k = k + 1
                result.append(n)
            n = n + 1
        return result

Notice that in the Cython version you declare integers and integer arrays
to be compiled into C types while also creating a Python list:


.. code-block:: cython

    def primes(int kmax):
        """Calculation of prime numbers with additional
        Cython keywords"""

        cdef int n, k, i
        cdef int p[1000]
        result = []


.. code-block:: python

    def primes(kmax):
        """Calculation of prime numbers in standard Python syntax"""

        p= range(1000)
        result = []

What is the difference? In the upper Cython version you can see the
declaration of the variable types and the integer array in a similar way as
in standard C. For example `cdef int n,k,i` in line 3. This additional type
declaration (i.e. integer) allows the Cython compiler to generate more
efficient C code from the second version. While standard Python code is saved
in :file:`*.py` files, Cython code is saved in :file:`*.pyx` files.

What's the difference in speed? Let's try it!

.. code-block:: python

	import time
	#activate pyx compiler
	import pyximport
	pyximport.install()
	#primes implemented with Cython
	import primesCy
	#primes implemented with Python
	import primes

	print "Cython:"
	t1= time.time()
	print primesCy.primes(500)
	t2= time.time()
	print "Cython time: %s" %(t2-t1)
	print ""
	print "Python"
	t1= time.time()
	print primes.primes(500)
	t2= time.time()
	print "Python time: %s" %(t2-t1)


These lines both need a remark:

.. code-block:: python

    import pyximport
    pyximport.install()


The `pyximport` module allows you to import :file:`*.pyx` files (e.g.,
:file:`primesCy.pyx`) with the Cython-compiled version of the `primes`
function. The `pyximport.install()` command allows the Python interpreter to
start the Cython compiler directly to generate C-code, which is automatically
compiled to a :file:`*.so` C-library. Cython is then able to import this
library for you in your Python code, easily and efficiently. With the
`time.time()` function you are able to compare the time between these 2
different calls to find 500 prime numbers. On a standard notebook (dual core
AMD E-450 1.6 GHz), the measured values are:

.. code-block:: console

    Cython time: 0.0054 seconds

    Python time: 0.0566 seconds


And here the output of an embedded `ARM beaglebone <http://beagleboard.org/Products/BeagleBone>`_ machine:

.. code-block:: console

    Cython time: 0.0196 seconds

    Python time: 0.3302 seconds


Pyrex
-----


Shedskin?
---------

Numba
-----
.. todo:: Write about Numba and the autojit compiler for NumPy

Threading
:::::::::


Threading
---------
Most physical problems, such as moving every chair in a fifty floor building to
the equivalent building across the street or writing a million-chapter story,
can be conceived as tasks performed by a single person. Programs often aren't
so different. 

Surely, it's possible for a sequential program to go through the full source
of a Wikipedia article, then, after finishing that, go to each of those links 
individually and return the HTML of each of those pages in sequence, looking
for links in them and visiting them recursively until it's discovered how many
of Wikipedia's millions of articles are connected to that source.

Is it possible for your sequential program to do that within the day?

You might recall a time you called a friend over to help you move 
furniture and how each of you taking chairs separately made the whole task go
faster. You may recall when each person on your development team claimed some
part of your project to write documentation for, sharing responsibility. In 
programming, the equivalent concept is threading.

Threading and parallel programming are increasingly important as the 
increase of computer hardware speed slows down. Programmers can no longer
rely on poorly optimized programs to be overlooked and count on the 
next processor generation to doubled in speed. With that reality, and more 
cores than ever appearing in computers, programmers cannot afford to ignore 
the fact that modern machines have more than one core.
 
Spawning Processes
------------------
Every program you write is associated with at least one process, a task or job
undertaken by a computer. Within that process, the program does some work in 
which there may be points where if two processes did parts of the job they 
would not interfere with each other. You do not even need to write a separate
program; your one program could create threads to share the task, akin to 
hiring workers. The threads complete their function, may return the result,
and then can be destroyed.

In Python, this can be done with a ``Process`` from the ``threading`` library.

You should know that you can't do this infinitely. Creating a thread is really
assigning one of the computer's processing cores to the thread's task instead
of running the entire process on one core. Modern machines often have several
such workers you can hire, called cores (e.g. dual-core, quad-core), and 
hiring more cores than a computer has will provide no extra benefits.

Locks
-----
A useful feature of threads is that they share an address space within memory.
They have access to the same data and can affect each other's work.
However, since the order of execution of two threads cannot be guaranteed, it
is possible for them to try to access the same value at the same time, which
may be very bad. Consider if a person went to deposit money in a bank account
they share with their spouse. At the very same instant, their spouse accesses
the account. They both see $10,000. The person deposits $10,000, bringing the
total to $20,000 and exits. The spouse extracts $10,000, bringing the total to
$0 and exits, after the person exited. The bank account will have $0 in the 
end unless this case is handled properly, with locks.

Multiprocessing
---------------

.. _`PyPy`: http://pypy.org
.. _`The GIL`: http://wiki.python.org/moin/GlobalInterpreterLock
.. _`guide`: http://www.dabeaz.com/python/UnderstandingGIL.pdf
.. _`New GIL`: http://www.dabeaz.com/python/NewGIL.pdf
.. _`Special care`: http://docs.python.org/c-api/init.html#threads
.. _`David Beazleys`: http://www.dabeaz.com/GIL/gilvis/measure2.py
