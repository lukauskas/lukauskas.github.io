---
title: Why Python is Slow
layout: post
published: false
category: articles
tags: '[python, pypy, cython, numba, optimisation]'
---
In Waza 2013, almost a year ago, Alex Gaynor has delivered an excellent talk on ["Why Python, Ruby and Javascript are slow"](http://vimeo.com/61044810).  The key of his talk was, as he emphasised, the fact  that these languages *are* slow at present, indicating the situation will improve later on.

When one asks a question on why Python is slower than C, dynamic typing is the first thing to get thrown around. And it is true, dynamic typing makes a programming language harder to optimise, however it is not the major factor contributing to the fact that your Python seems to run slower. In most cases, it all boils down to the data structures you use.

You Code Differently in Python
----------------------------------------

Any reasonable python developer would probably not hesitate to use the following code to create a point in python:

{% highlight python %}
point = {'x': 0, 'y': 0}
{% endhighlight %}

It is elegant, easy to code and read.

In C, however, one would probably use a `struct` for a similar code:

{% highlight python %}
struct Point {
   int x;
   int y;
};
{% endhighlight %}

Again, it is elegant, and easy to read and would not be frowned upon by the code reviewers.

The problem is that the  C structure (leftmost) allows the compiler to know where in memory each of it's fields (`x` and `y`) are exactly just by knowing where the whole `Point` structure is (as these x and y will always be at the same offset from it).  

Now Python users use a hash table to perform a similar task.  In order to access the key `‘y’` in this structure, the compiler would need to compute the hash of `‘y’`, which takes some short, but nonzero amount of time, and then access the memory at offset returned by this hash function. 

While hash functions are usually efficient, it is still a noticeable overhead to compute these values compared to the C structure that always "just knows" where these fields are.

