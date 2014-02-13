---
title: 'Why Your Python Runs Slow. Part 1: Data Structures'
layout: post
published: false
category: articles
tags: '[python, pypy, data structures, optimisation]'
---
Almost a year ago, in Waza 2013, Alex Gaynor delivered an excellent talk on ["Why Python, Ruby and Javascript are slow"][1] . The key of his talk, as he emphasised, is in the present tense. In other words, even if these languages *are* slower now, it does not mean that the situation is hopeless and will necessarily stay that way forever.

When one asks a question among the lines of why Python is slower than C, dynamic typing is the first answer to get thrown around. While it is true that dynamically  typed programming languages have some overhead associated with them, it may not be the major factor slowing down *your* Python code. 

Dynamic typing (and other *magic* features of programming languages like Python) make the code harder to optimise. There is a huge difference between "harder to optimise" and "slower". In fact, as Alex mentions in his talk, we have had years of research focusing on what is the best way to perform type checking at runtime. This research has already lead to ways of making this overhead negligible in most cases. 

In reality, the significant runtime differences between Pythonic code and C boils down to different data structures and algorithms used in each of the cases. Sometimes even without the programmers being aware of them.

[1]: http://vimeo.com/61044810

You Code Differently in Python
----------------------------------------

Let's illustrate the point in previous paragraph with an example from Alex's talk.
A Python programmer would be likely to use a similar code snippet to code a container for 2D point in Python:

{% highlight python %}
point = {'x': 0, 'y': 0}
{% endhighlight %}

The solution easy to read, even easier to code and elegant in general.

A C developer, on the other hand, would use a `struct` for 2D point container:

{% highlight python %}
struct Point {
   int x;
   int y;
};
{% endhighlight %}

While this solution is as elegant and as easy to work with as the Pythonic solution, it is completely different data structure.  Here we tell the interpreter that each point would have exactly two fields, `x` and `y`. Knowing the size of these fields, the interpreter could take the hint and allocate  them near each other, as a single contiguous memory block. In other words, the structure would be similar to an array. The interpreter would know exactly where `x` and `y` fields of a given point are at any time. We could also access both of these fields in memory easily, just by looking some constant offset away from where the point itself is. 

Pythonic code uses a hash table-like data structure to perform a similar task.  The interpreter cannot simply preallocate a contiguous block of memory to store both `x` and `y` next to each other to make them easily accessible. This is impossible as we could also have any number of keys present in the dictionary at any time. We could also delete these keys if we wanted to. The interpreter must be able to convert any input to a memory location. Hash functions were designed to provide this mapping. Needless to say, these functions add a some overhead to the calculation. While it is usually small, it may as well be significant enough to slow down your code, especially if you have to do a lot of them.

If someone was to directly translate the pythonic code to C++, they would probably  end up with something similar to:

{% highlight  c++ %}
std::hash_set<std::string, int> point;
point[“x”] = x
point[“y”] = y
{% endhighlight %}

Looking at the code snippet, it looks as if the language designers themselves went to great lengths to make the hash tables as complicated to use as possible, just so nobody would use them by accident. And rightfully so. Because of this *feature*, a newcomer to the C world would be able to spot that there's something ridiculously wrong with this code. This begs to question why is it acceptable in Python?

The answer is in the "dictionaries are lightweight objects" mindset of Python programmers.
Consider the following code, which is the closest thing in Python to the C `struct` above[^1]:

{% highlight python linenos %}
class Point(object):
    x, y = None, None
    def __init__(self, x, y):
         self.x, self.y = x, y
{% endhighlight %}

This representation provides useful hints to the compiler, just like the C `struct` does. For instance, in the second line, we are explicitly telling the interpreter that our point object will always have at least two fields `x` and `y` when it is created, hoping it would be able to deal with them.

Unfortunately, the standard Python interpreter, also called CPython, is not able to make much use of this representation. The following code block takes 186 microseconds to run on my machine:

{% highlight python %}
def sum_(points):
    sum_x, sum_y = 0, 0
    for point in points:
        sum_x += point['x']
        sum_y += point['y']
    return sum_x, sum_y
{% endhighlight %}

A similar code block that replaces `point['x']` with `point.x` takes 201 microseconds to run on the same machine. In other words, using the class `Point` is 8% slower. 

In CPython, `point.x` is often executed in the same way as the statement  `dict(point)['x']`[^2].
This means that the class implementation of a point still uses a dictionary lookup, as  before, with some additional overhead. In this sense, it is easy to see why the dictionaries are thought to be "lightweight objects".

Some Python implementations that were specifically designed to be efficient, such as [PyPy](http://pypy.org), are able to flip the scales for these two approaches around. For instance, the same code snippets run for 21.6 and 3.75 microseconds respectively when  using PyPy instead of Python. While both of these results are significantly faster than their CPython counterparts, using a class is 5.76 *times* faster under this JIT-capable interpreter, than using a dictionary. In other words, PyPy are able to use correct data structure here.

I encourage you to look at the smallest number again -- 3.75 microseconds. This number means that we could perform 266 000 of these calculations in a second, all from Python, with dynamic typing, monkey-patching, etc. still built into it. All of this, by just using a better data structure, both in the code, and in the implementation. Next time, you are writing a line of code in Python, have a thought of what data structure you are using, both explicitly and implicitly, and ask yourself whether there is a better one. This is what you would do in C, is it?

Finally, I like to believe that this article is a clear example why the future is bright for Python (and similar languages). The standard Python implementation, CPython is there  only to act as reference. It has never been designed to be the fastest one. As we have seen today, alternative implementations, like PyPy, are already able to optimise your code to great lengths. These optimisations are possible, even with all the challenges offered by the nature of the programming language. We only had 23 years of Python interpreter development, how would things look like when Python is 42, like C?

[^1]: One would be right to argue that [`collections.namedtuple()`](http://docs.python.org/2/library/collections.html#collections.namedtuple) is even closer to C struct than this. Let's not overcomplicate things, however. The point I'm trying to make is valid regardless.

[^2]: Consult python [documentation](http://docs.python.org/2/reference/datamodel.html#the-standard-type-hierarchy) for a detailed view of how attributes work in python classes.