---
layout: post
title:  "Passing arguments to a thread"
date:   2016-01-04
categories: concurrency
---

We know how to construct a thread without any input
arguments. But a function `{%raw%}printNTimes(int n, const std::string&
msg){%endraw%}` requires two arguments. How to run it in a separate
thread is described below.

{% highlight c++ %}
std::string msg = "Today is a beautiful day.";
std::thread t(printNTimes, 5, msg);
{% endhighlight %}

This looks straightforward, but something unexpected happens if you don't have a
const reference. Let us consider a `{%raw%}referenceArguments.cpp{%endraw%}`
below

{% highlight c++ %}
#include <iostream>
#include <thread>
#include <algorithm>

void toUpper(std::string& str)
{
    std::transform(str.begin(), str.end(), str.begin(), ::toupper);
}

int main()
{
    std::string str = "Today is a beautiful day.";
    
    std::cout << "Before: " << str << std::endl;
    
    std::thread t(toUpper, str);
    t1.join();
    
    std::cout << "After: " << str << std::endl;

    return 0;
}
{% endhighlight %}

This looks perfectly reasonable, at least to me. But it doesn't compile.
 
{% highlight bash %}
$ g++ referenceArguments.cpp -std=c++11
In file included from /usr/include/c++/4.8/thread:39:0,
                 from referenceArguments.cpp:2:
/usr/include/c++/4.8/functional: In instantiation of ‘struct std::_Bind_simple<void (*(std::basic_string<char>))(std::basic_string<char>&)>’:
/usr/include/c++/4.8/thread:137:47:   required from ‘std::thread::thread(_Callable&&, _Args&& ...) [with _Callable = void (&)(std::basic_string<char>&); _Args = {std::basic_string<char, std::char_traits<char>, std::allocator<char> >&}]’
referenceArguments.cpp:16:32:   required from here
/usr/include/c++/4.8/functional:1697:61: error: no type named ‘type’ in ‘class std::result_of<void (*(std::basic_string<char>))(std::basic_string<char>&)>’
       typedef typename result_of<_Callable(_Args...)>::type result_type;
                                                             ^
/usr/include/c++/4.8/functional:1727:9: error: no type named ‘type’ in ‘class std::result_of<void (*(std::basic_string<char>))(std::basic_string<char>&)>’
         _M_invoke(_Index_tuple<_Indices...>)
{% endhighlight %}

The error message is quite cryptic. The solution is to add
`{%raw%}std::ref{%endraw%}` around `{%raw%}str{%endraw%}`:

{% highlight c++ %}
std::thread t(toUpper, std::ref(str));
{% endhighlight %}

Explanation
===========

The reasoning behind it is that a thread, during a construction, takes a
function and copies all of the arguments to its internal storage. Then,
accordingly to the function signature, thread passes arguments to the function
by values or references. If the function expects a reference, it
gets a reference to an internal copy of the object. 

In the example

{% highlight c++ %}
std::thread t(printNTimes, 5, msg);
{% endhighlight %}

the thread `{%raw%}t{%endraw%}` copies arguments `{%raw%}5{%endraw%}` and
`{%raw%}msg{%endraw%}` to its internal storage. Then, the function
`{%raw%}printNTimes{%endraw%}` recieves a reference to the internal copy of the
`{%raw%}msg{%endraw%}` not a reference to the original `{%raw%}msg{%endraw%}`.

If we use `{%raw%}std::ref{%endraw%}`, the function correctly gets a reference
to the original object, like in the example

{% highlight c++ %}
std::thread t(toUpper, std::ref(str));
{% endhighlight %}

In the case of non const reference, we get a compiler error if we don't use the
`{%raw%}std::ref{%endraw%}`, but in the case of const reference we don't get an
error . Therefore, I recommend using `{%raw%}std::ref{%endraw%}` every time when
we have references.

Another example
===============

Here is `{%raw%}arguments.cpp{%endraw%}`, which uses the
`{%raw%}printNTimes{%endraw%}` function from above.

{% highlight c++ %}
#include <iostream>
#include <thread>

void printNTimes(int n, const std::string& msg)
{
    for (int i = 0; i != n; i++)
    {
        std::cout << msg << std::endl;
    }
}

int main()
{
    std::string msg1 = "Today is a beautiful day.";
    std::thread t1(printNTimes, 6, std::ref(msg1));
    
    std::string msg2 = "And sun is shining.";
    std::thread t2(printNTimes, 6, std::ref(msg2));
    
    t1.join();
    t2.join();
    
    return 0;
}
{% endhighlight %}

The output might be:

{% highlight bash %}
$ ./arguments 
Today is a beautiful day.And sun is shining.

Today is a beautiful day.And sun is shining.
And sun is shining.
And sun is shining.
And sun is shining.
And sun is shining.

Today is a beautiful day.
Today is a beautiful day.
Today is a beautiful day.
Today is a beautiful day.
{% endhighlight %}

It looks a bit strange. This is because of so called **data races**.

Summary
-------

We learned how to pass an argument to a function, which will run in a separate
thread. The `{%raw%}std::ref{%endraw%}` is used for passing a reference to a
variable.

In the next blog post, we will investigate why is the output of the final
program strange -- in concurrent language: we will explore data races.

Links:

* Source code: [arguments.cpp](https://github.com/jakaspeh/concurrency/blob/master/arguments.cpp) and [referenceArguments.cpp](https://github.com/jakaspeh/concurrency/blob/master/referenceArguments.cpp)

* [Examples: constructors of std::thread](http://en.cppreference.com/w/cpp/thread/thread/thread#Example)


