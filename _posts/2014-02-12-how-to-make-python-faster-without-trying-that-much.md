---
title: How to Make Python Faster Without Trying That Much
layout: post
published: true
category: articles
tags: Python, PyPy, Cython, CPython, profiling, optimisation, programming
comments: true
---
<section id="table-of-contents" class="toc">
  <header>
    <h3>Contents</h3>
  </header>
<div id="drawer" markdown="1">
*  Auto generated table of contents
{:toc}
</div>
</section><!-- /#table-of-contents -->


This article is based on a demo I gave in a session at the [Theoretical Systems Biology Group](http://www.theosysbio.bio.ic.ac.uk/) on 4th of February.

The example focuses on optimising a small program that computes the best alignment for two protein sequences.

Starting code
-----------------
The code for this demo is available [directly from GitHub](https://github.com/sauliusl/python-optimisation-example/).
Please checkout the  `starting-code` tag to begin with:

{% highlight bash %}
git clone git@github.com:sauliusl/python-optimisation-example.git
git checkout starting-code
{% endhighlight %}

Alternatively, you can browse the code [here](https://github.com/sauliusl/python-optimisation-example/tree/starting-code/alignment).

Take a look at the directory alignment, there are three files:

*  `align.py` -- the file will be the target of our optimisation today, as it contains the routine to compute the dynamic programming matrix;
* `alignment.py`  -- the main executable that calls this function and prints the output.
* `sequences.py` --the file that provide a convenient way to get the sequences.

Baseline
-----------
Before starting any optimisations, let's verify whether the code works at all:

{% highlight bash %}
> python alignment.py
Score: 315

Alignment:

MEE-PQSD-PS-VE-PPLSQETFS-DLWKLL-PE--NNVL-SP--LPSQAMDD-LMLSP-
MEES-QSDI-SL-EL-PLSQETFSG-LWKLLPPEDI---LPSPHC-----MDDLL-L-PQ

<... snip ...>

KE-PG-GSRAHSS-HLK-SKKGQSTSRHKK-LM--FK-TEGPDSD
-ES-GD-SRAHSSY-LKT-KKGQSTSRHKKT-MVK-KV--GPDSD
{% endhighlight %}

As you can see the code aligns the sequences in some way.
The output seems reasonable given the simplicity of the alignment scores, which is great news for us.
We can now continue to establishing a baseline on how fast the code runs.
Unix command `time` is perfect for  this task.

{% highlight bash %}
> time python alignment.py
{% endhighlight %}

Typing this command into the terminal reports the baseline to be around 0.502s.

At this point it is curious to measure the runtime using an optimising python interpreter, [PyPy](http://pypy.org).
Download and install PyPy according to instructions in [their download page](http://pypy.org/download.html ).
For those lucky enough to use OSX, [Homebrew](http://brew.sh/) provides a simple interface to install PyPy:

{% highlight bash %}
> brew install pypy
{% endhighlight %}

Since we know PyPy is an optimising Python compiler, we would expect the code to run faster than on it than on CPython.
To test this assumption, we can again use `time`:

{% highlight bash %}
> time pypy alingment.py
{% endhighlight %}

This function reports the code to run for 0.255s -- almost twice as fast as the under the standard python.
Thinking about this, it is quite impressive that we were able to nearly halve the runtime without any significant effort on our side (besides installing PyPy, of course).

Profiling the Baseline
----------------------------

Now let's see if we can speed things up even further.
Let's fire up [`cProfile`](http://docs.python.org/2/library/profile.html#module-cProfile) and investigate where the program spends majority of it's time.
The standard way to run `cProfile` is given below:

{% highlight bash %}
> python -m cProfile -o profile.pstats alignment.py
{% endhighlight %}

The `-o profile.pstats` option tells the profiler to save the stats in `profile.pstats` file, which we are going to view using [`gprof2dot`](https://code.google.com/p/jrfonseca/wiki/Gprof2Dot) script and [`graphviz`](http://www.graphviz.org/).

If you do not yet have these packages, you should be able to install `gprof2dot` using `pip` directly:

{% highlight bash %}
> pip install gprof2dot
{% endhighlight %}

If your computer does not recognise `pip` command, try to install it first using [these instructions](http://www.pip-installer.org/en/latest/installing.html).

Since `graphviz` is not a python package it cannot be installed via `pip`, therefore follow the instructions in the [download page](http://www.graphviz.org/Download.php) for guidance on how to obtain it. Alternatively, you can use your  distribution's package manager. For instance, the aforementioned Homebrew on OSX can do this (`brew install graphviz`).

Once you have both modules, run the following two-part command:

{% highlight bash %}
> gprof2dot -f pstats profile.pstats -n 0 -e 0 | dot -Tpdf -o profile.pdf
{% endhighlight %}

Here`-n 0 -e 0` just tells the software to draw all nodes (by default it cuts a few unimportant ones off for visual clarity). The piped `dot` command generates `profile.pdf` with the profile of function calls for the program.

Open this `profile.pdf` file with your favourite PDF viewer to see the results.

![Contents of profile.pdf]({{ site.url }}/assets/post-images/2014-02-12-how-to-make-python-faster-without-trying-that-much/profile-1.png)

Improving the code based on profile results
------------------------------------------------------
From the profile above we can see that the majority of the time is spent in the `align` function, just as we would expect, 35% of which in function `max`.
Out of this time, we see that we spend 9.75% of time in the `lambda` function, namely the lambda function `lambda x: x[0]` that just tells the function to get the first value and use that for comparison.


Judging from these results, one must ask whether using `max` function for three elements only is an overkill. M
aybe we were too smart here and could actually get away with simpler code? Lets replace the call for max function with a block of nested `if` statements:

{% highlight python %}
            if match_score >= deletion and match_score >= insertion:
                max_score = match_score
                traceback_pos = i-1, j-1
            elif deletion > match_score and deletion >= insertion:
                max_score = deletion
                traceback_pos = i-1, j
            else:
                max_score = insertion
                traceback_pos = i, j-1

            matrix[i, j] = max_score
            traceback[i, j] = traceback_pos
  {% endhighlight %}

 To get this code just checkout `simplified-initial` tag:

{% highlight bash %}
  > git checkout simplified-initial
{% endhighlight %}

Alternatively, browse it [here](https://github.com/sauliusl/python-optimisation-example/tree/simplified-initial).

Let's `time` this new code:

{% highlight bash %}
> time python alignment.py
0m0.327s
> time pypy alignment.py
0m0.249s
{% endhighlight %}

We can see that CPython version of the code became much faster (remember, it was around 0.5s before), whereas PyPy runtimes stayed roughly the same, suggesting that PyPy has already thought of this optimisation on it's own.

Running cProfile again, we can see that align function is still the bottleneck. Unfortunately, it does not tell us what is making it slow any more:

![Contents of profile.pdf]({{ site.url }}/assets/post-images/2014-02-12-how-to-make-python-faster-without-trying-that-much/profile-2.png)

Profiling line-by-line
--------------------------

We will use a line profiler to get more granularity for our profiles.

First install it if you haven't already:

{% highlight bash %}
> pip install line_profiler
{% endhighlight %}

Now in order for the profiler to work, we need to tell it which functions we want to profile. This is done by decorating said functions with `@profile` decorator.
Open `align.py` in your favourite text editor and add the line `@profile` on top of the `def align ...` line, i.e.:

{% highlight python %}
{% raw %}
# align.py

# < snip ... >

@profile
def align(left, right):

    matrix = {}

#< snip ... >
{% endraw %}
{% endhighlight %}

Don't worry about importing this line. Profiler will make sure to import it for you automagically.

The actual profiling is done by running

{% highlight bash %}
> kernprof.py --line-by-line alignment.py
{% endhighlight %}

This would store the results in `alignment.py.lprof` file which can then be viewed by

{% highlight bash %}
> python -m line_profiler alignment.py.lprof
{% endhighlight %}

The first command in this set generates the profile, and the second one prints it in a readable format.
This is the output:

~~~
{% raw %}
Timer unit: 1e-06 s

File: align.py
Function: align at line 6
Total time: 1.38775 s

Line #      Hits         Time  Per Hit   % Time  Line Contents
==============================================================
     6                                           @profile
     7                                           def align(left, right):
     8
     9         1            3      3.0      0.0      matrix = {}
    10
    11                                               # Initialise
    12         1            1      1.0      0.0      matrix[0, 0] = 0
    13         1            1      1.0      0.0      traceback = {}
    14       394          369      0.9      0.0      for i in xrange(len(left)):
    15       393          525      1.3      0.0          matrix[i+1, 0] = (i+1) * DELETION_SCORE
    16       393          624      1.6      0.0          traceback[i+1, 0] = i, 0
    17
    18       388          405      1.0      0.0      for i in xrange(len(right)):
    19       387          477      1.2      0.0          matrix[0, i+1] = (i+1) * INSERTION_SCORE
    20       387          532      1.4      0.0          traceback[0, i+1] = 0, i
    21
    22
    23
    24       394          338      0.9      0.0      for i, symbol_left in enumerate(left):
    25    152484       140777      0.9     10.1          for j, symbol_right in enumerate(right):
    26    152091       125126      0.8      9.0              match_score = matrix[i, j] + MATCH_SCORE if symbol_left == symbol_right else MISMATCH_SCORE
    27    152091       155781      1.0     11.2              deletion = matrix[i, j+1] + DELETION_SCORE
    28    152091       146020      1.0     10.5              insertion = matrix[i+1, j] + INSERTION_SCORE
    29    152091       111502      0.7      8.0              if match_score >= deletion and match_score >= insertion:
    30      9563         6786      0.7      0.5                  max_score = match_score
    31      9563         7496      0.8      0.5                  traceback_pos = i, j
    32    142528       114515      0.8      8.3              elif deletion > match_score and deletion >= insertion:
    33     87713        62775      0.7      4.5                  max_score = deletion
    34     87713        72069      0.8      5.2                  traceback_pos = i, j+1
    35                                                       else:
    36     54815        38668      0.7      2.8                  max_score = insertion
    37     54815        44198      0.8      3.2                  traceback_pos = i+1, j
    38
    39    152091       171714      1.1     12.4              matrix[i+1, j+1] = max_score
    40    152091       187046      1.2     13.5              traceback[i+1, j+1] = traceback_pos
    41
    42         1            2      2.0      0.0      return matrix[len(left), len(right)], traceback
{% endraw %}
~~~

We can see we spend a significant amount of time in lines that access the matrix, i.e. lines 26-28, 39.
Similarly, a significant amount of time is spent in the comparisons and other parts of the loop.
Optimisation of any of these lines will provide the most benefit. Let's see if we can find a way to perform these optimisations.

Cythonising the code
---------------------------
[Cython](http://cython.org/) can be considered to be a compiler for Python.
Generally, it is able to convert the Python code into C, which should provide a significant performance boost for simple statements such as element access in a matrix.
In order to start using it, we would need to install it first. `pip` would be able to handle this for us:

{% highlight bash %}
> pip install Cython
{% endhighlight %}

Once the installation is complete, it is ridiculously easy to convert the code to Cython format:

{% highlight bash %}
> cp align.py align.pyx
{% endhighlight %}

You got it right, we just copied the file to a file with different extension (`.pyx` is Cython file extension).
In order to be able to run it, however, we would need to compile this file. This is done by creating `setup.py`.
The following code provides the minimal skeleton for such file:

{% highlight python linenos %}
{% raw %}
# setup.py
from distutils.core import setup
from distutils.extension import Extension
from Cython.Distutils import build_ext

setup(
    cmdclass = {'build_ext': build_ext},
    ext_modules = [Extension("align_cythonised", ["align.pyx"])]
)
{% endraw %}
{% endhighlight %}

Line eight in this file can roughly be translated to say *compile file `align.pyx` into a library `align_cythonised`*.

Now let's quickly compile our code using this setup file:

{% highlight bash %}
> python setup.py build_ext --inplace
{% endhighlight %}

The code should generate `align.c` and `align_cythonised.so` files in your local directory (do not forget the `--inplace` tag, as otherwise it would compile it into `build/` directory).

We could now use [`timeit`](http://docs.python.org/2/library/timeit.html) module to compare the runtimes of these functions.
It is convenient to use IPython ([http://ipython.org/](http://ipython.org), `pip install ipython`) for `timeit` tests due to it's magic command `%timeit`. I use it here:

{% highlight python %}
In [1]: from sequences import P53_HUMAN, P53_MOUSE
In [2]: from align import align
In [3]: from align_cythonised import align as align_cythonised
In [4]: %timeit align(P53_HUMAN, P53_MOUSE)
1 loops, best of 3: 258 ms per loop
In [5]: %timeit align_cythonised(P53_HUMAN, P53_MOUSE)
1 loops, best of 3: 189 ms per loop
{% endhighlight %}

We can see we got 69 ms improvement by just compiling the code without changing a line of code -- not bad.
Note that PyPy runs this function in 168 ms -- 21 milliseconds faster.
One could overstretch this result and say that Python runs faster than C in this scenario (hehe).

Let's see if we can make Cythonised code beat this though.

Numpy and Cython
-------------------------

Cython collaborates with [`numpy`](http://www.numpy.org/) folks a lot and is able to run `numpy`'s C interface natively. We could exploit this native interface to our advantage.
To do so, we will change our matrix to be a numpy n-dimensional array rather than a dictionary.

First of all, we need to install `numpy`, `pip` could be used to do this yet again:

{% highlight bash %}
> pip install numpy
{% endhighlight %}

You might need a Fortran compiler to do so (you can obtain it from Homebrew on OSX `brew install gfortran`). If you are having trouble with this step consult the [downloads page](http://www.scipy.org/scipylib/download.html) on how to get the precompiled library directly.

Once this is all set, copy `align.pyx` into `align_numpy.pyx` and change it slightly to use `numpy`.

The modified file is available from [GitHub](https://github.com/sauliusl/python-optimisation-example/blob/cython-final/alignment/align_numpy.pyx ).

Git tag `cython-final` could be used to checkout this code:

{% highlight bash %}
> git checkout cython-final
{% endhighlight %}

In the first couple of lines in the `align_numpy.pyx` we import both pythonic and c libraries for numpy:

{% highlight python %}
{% raw %}
import numpy as np # Import python numpy bindings
cimport numpy as np # Import cpython numpy bindings
{% endraw %}
{% endhighlight %}

Then, we define the matrix to be a `numpy` array (rather than a dictionary), and initialise it:

{% highlight python %}
{% raw %}
    cdef np.ndarray[np.float_t, ndim=2] matrix
    matrix = np.empty((len_left+1, len_right+1))
{% endraw %}
{% endhighlight %}

Then, we explicitly define that our iterators, i and j, are unsigned integers:

{% highlight python %}
{% raw %}
    cdef unsigned int i, j
{% endraw %}
{% endhighlight %}

This step is crucial, as numpy array accesses are optimised to native C speeds only for typed indexes.
We add a couple of other `cdef` hints, and are ready to go.

In order to compile this new file, `setup.py` needs to be changed a little bit as well,
the file is available in [GitHub](https://github.com/sauliusl/python-optimisation-example/blob/cython-final/alignment/setup.py).

All we did here was to add another `Extension` line to make the `setup.py` aware of this new file.

{% highlight python %}
Extension("align_cythonised_numpy", ["align_numpy.pyx"],
                           include_dirs=[numpy.get_include()])]
{% endhighlight %}

We also make sure to include `numpy` headers to the compilation, the `include_dirs` parameter takes care of this.

We can now check how fast this code is running by using `%timeit` magic function, after appropriately importing the numpy-cythonised function:

{% highlight python %}
In [7]: %timeit align_cythonised_numpy(P53_HUMAN, P53_MOUSE)
10 loops, best of 3: 101 ms per loop
{% endhighlight %}

As you can see, the total runtime is 101 milliseconds. This is 157ms faster than the original version of the code and all we did was to start using numpy and add a couple of type hints.

This Cythonic code can be further optimised by enabling the potentially dangerous options such as disabling bound checking.
Furthermore, if you want to take optimisation matters into your own hands, one could run *native* C code from Cython.
I refer the reader to Cython documentation and various tutorial videos from SciPy conferences for more information on these advanced topics.

Conclusions
----------------

In conclusion, this guide provides a reference on some quick optimisations you could make for your code, without much significant effort from your side.
Were able to significantly improve runtimes of Python code by profiling where the bottlenecks are and doing simple, yet incremental changes to it.

A word of warning, though, these alternative interpreters/compilers for python should be considered only as a last resort. More often than not, there is a way to obtain sufficient performance boosts from simply changing the bits of the code, as we did in the beginning. However, if in any case this would prove insufficient for your needs, these optimisation tools exist already and will continue to improve over time.
