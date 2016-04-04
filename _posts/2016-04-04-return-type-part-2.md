---
layout: post
title:  "Return type - Part 2"
date:   2016-04-04
categories: concurrency, cpp
---

Using `{%raw%}auto{%endraw%}` as a return type is another feature which is new
in C++11. The feature has a bit unusual syntax at first sight. 

If there is a function
with return type `{%raw%}T{%endraw%}`

{% highlight c++ %}
T function();
{% endhighlight %}

it is equivalent to write

{% highlight c++ %}
auto function() -> T;
{% endhighlight %}

This is just another way of saying that the return type of
`{%raw%}function{%endraw%}` is `{%raw%}T{%endraw%}`.

But why should we use this notation? It looks more complicated and we need to
type more. Well, there is a reason. But before I tell you the reason, we should
look at another C++11 feature: `{%raw%}decltype{%endraw%}`.

Decltype
--------

Rule of thumb is that `{%raw%}decltype(argument){%endraw%}` tells you the type of
`{%raw%}argument{%endraw%}`. For example, the program

{% highlight c++ %}
int main()
{
    int i = 1;
    double d = 1.0;

    std::cout << std::is_same<int, decltype(i + i)>::value << "\n"
              << std::is_same<double, decltype(d + i)>::value << "\n"
              << std::is_same<int, decltype(d + i)>::value << "\n"
              << std::endl;
    return 0;
}
{% endhighlight %}

outputs 

{% highlight c++ %}
1
1
0
{% endhighlight %}

The result of expression `{%raw%}i + i{%endraw%}` is `{%raw%}int{%endraw%}`,
thus `{%raw%}std::is_same<int, decltype(i + i)>::value{%endraw%}` is
`{%raw%}true{%endraw%}`. A similar argument tells us the types of other
expressions.

Usually `{%raw%}decltype{%endraw%}` tells us what we want, but there are some
corner cases which give us a result we might not expect. For example,
running this program

{% highlight c++ %}
int main()
{
    int i = 1;
    std::cout << std::is_same<int, decltype(i)>::value << "\n"
              << std::is_same<int, decltype((i))>::value << "\n"
              << std::is_same<int&, decltype((i))>::value << "\n"
              << std::endl;
    return 0;
}
{% endhighlight %}

produces the following output

{% highlight c++ %}
1
0
1
{% endhighlight %}

We notice that `{%raw%}decltype(i){%endraw%}` deduces to `{%raw%}int{%endraw%}`
and `{%raw%}decltype((i)){%endraw%}` deduces to `{%raw%}int&{%endraw%}`.

Therefore, when you use `{%raw%}decltype{%endraw%}`, you should check its rules
for deducing types.

Using decltype
--------------

Now, we are going back to the reason for 

{% highlight c++ %}
auto function() -> T
{% endhighlight %}

Using `{%raw%}decltype{%endraw%}`, we can define the following function

{% highlight c++ %}
auto fun(int i) -> decltype(i + i)
{
    return i + i;
}
{% endhighlight %}

The return type of `{%raw%}fun{%endraw%}` is `{%raw%}int{%endraw%}`, because
the type of the expression `{%raw%}i + i{%endraw%}` is equal to
`{%raw%}int{%endraw%}`. 

But in this case it is not possible to write something like

{% highlight c++ %}
decltype(i + i) fun(int i)
{
    return i + i;
}
{% endhighlight %}

Because when compiler reaches `{%raw%}decltype(i + i){%endraw%}` it does not
know what `{%raw%}i{%endraw%}` is. Consequently, the compiler is not able to
figure out the return type of `{%raw%}fun{%endraw%}`. Therefore, we have to use
the former version of the return type declaration.

C++14
-----

We should note that for C++14 the following 

{% highlight c++ %}
auto fun(int i) -> decltype(i + i)
{
    return i + i;
}
{% endhighlight %}

is equivalent to 

{% highlight c++ %}
decltype(auto) fun(int i)
{
    return i + i;
}
{% endhighlight %}

In this case, `{%raw%}decltype(auto){%endraw%}` indicates that the return type
of `{%raw%}fun{%endraw%}` is equal to `{%raw%}decltype{%endraw%}` of the return
expression.

Summary
-------

We learned a new feature: how to declare a return value of a function using
`{%raw%}auto{%endraw%}` keyword. We also looked at the
`{%raw%}decltype{%endraw%}` specifier.

Links:

* [Source code](https://github.com/jakaspeh/concurrency/blob/master/resultOfPart2.cpp)
* [decltype](http://en.cppreference.com/w/cpp/language/decltype)
* [Jaka's Corner: Return type - Part 1](/blog/2016/03/return-type-part-1.html)
* [Stack Overflow: arrow operator (->) in function heading](http://stackoverflow.com/questions/22514855/arrow-operator-in-function-heading)

