---
layout: post
title:  "Return type - Part 1"
date:   2016-03-28
categories: concurrency, cpp
---

Templates in C++ allows us to write generic concurrent programs. It is not
unusual to have a template argument which represent a function. For example

{% highlight c++ %}
template <typename Fun>
void function(Fun f, ...)
{% endhighlight %}

the `{%raw%}function{%endraw%}`'s argument is a function
`{%raw%}f{%endraw%}` of type `{%raw%}Fun{%endraw%}`. 

If we use functions as template arguments, then we don't know what is the return
type of the function. (For example, we don't know the return type of
`{%raw%}f{%endraw%}`.) There are some cases, when we would need to know the
return type of a function. For example, making a future with
`{%raw%}std::async{%endraw%}` requires such knowledge.

Fortunately there exist a solution for this problem. The solution is 
`{%raw%}std::result_of{%endraw%}`. This class can figure out what is the return
type of the function at the compile time.

std::result_of
--------------

Let's look at an example

{% highlight c++ %}
template <typename Fun, typename... Args>
void checkType(Fun f, Args... args)
{
    using return_t = typename std::result_of< Fun(Args...) >::type;

    std::cout << "int:    " << std::is_same<int, return_t>::value << "\n";
    std::cout << "double: " << std::is_same<double, return_t>::value << "\n";
    std::cout << "float:  " << std::is_same<float, return_t>::value << "\n";
    std::cout << std::endl;
}
{% endhighlight %}

The `{%raw%}checkType{%endraw%}` accepts two arguments. First one is a function
`{%raw%}f{%endraw%}` of type `{%raw%}Fun{%endraw%}` and the second one
represents variadic number of arguments. The assumption is that 
`{%raw%}args{%endraw%}` are the arguments of `{%raw%}f{%endraw%}`. If you are
not familiar with functions, which have variadic number of arguments, look at
[Variadic number of arguments](/blog/2016/03/variadic-number-of-arguments.html).

The next line 

{% highlight c++ %}
using return_t = typename std::result_of< Fun(Args...) >::type;
{% endhighlight %}

determines the return type of `{%raw%}f{%endraw%}`. This is done with
`{%raw%}std::result_of{%endraw%}`. The template parameter of
`{%raw%}std::result_of{%endraw%}` must be a signature of a function. Then, the
`{%raw%}std::result_of< >::type{%endraw%}` represents the return type of the
function, which is determined by the signature. The deduced
return type is stored in `{%raw%}return_t{%endraw%}`.

In the next lines, the program prints information about
`{%raw%}return_t{%endraw%}`. The statement

{% highlight c++ %}
std::is_same<T, U>::value
{% endhighlight %}

is equal to `{%raw%}true{%endraw%}` if `{%raw%}T{%endraw%}` and
`{%raw%}U{%endraw%}` are the same type. Otherwise, the statement is
`{%raw%}false{%endraw%}`. 

Using `{%raw%}std::is_same< >{%endraw%}`, the program checks if the
`{%raw%}return_t{%endraw%}` is equal to `{%raw%}int{%endraw%}` or
`{%raw%}double{%endraw%}` or `{%raw%}float{%endraw%}`.

Examples
========

Let's look at the behavior of the function `{%raw%}checkType{%endraw%}`.

{% highlight c++ %}
int fun1(int i)
{
    return i;
}

double fun2(double d)
{
    return d;
}

float fun3(int i1, int i2)
{
    return static_cast<float>(i1 + i2);
}

int main()
{
    int i = 1;
    double d = 1.0;

    std::cout << "Function1: " << std::endl;
    checkType(fun1, i);

    std::cout << "Function2: " << std::endl;
    checkType(fun2, d);

    std::cout << "Function3: " << std::endl;
    checkType(fun3, i, i);
}
{% endhighlight %}

We have three functions `{%raw%}fun1{%endraw%}`, `{%raw%}fun2{%endraw%}` and
`{%raw%}fun3{%endraw%}`, which return `{%raw%}int{%endraw%}`,
`{%raw%}double{%endraw%}` and `{%raw%}float{%endraw%}`, respectively. In
`{%raw%}main{%endraw%}` function, `{%raw%}checkType{%endraw%}` prints
information about the return types of `{%raw%}fun1{%endraw%}`,
`{%raw%}fun2{%endraw%}` and `{%raw%}fun3{%endraw%}` using
`{%raw%}std::result_of< >{%endraw%}`.

The output of the program is:

{% highlight c++ %}
Function1: 
int:    1
double: 0
float:  0

Function2: 
int:    0
double: 1
float:  0

Function3: 
int:    0
double: 0
float:  1
{% endhighlight %}

Summary
-------

We learned how to deduce the return type of a function with
`{%raw%}std::result_of<>{%endraw%}`.


Links:

* [Source code](https://github.com/jakaspeh/concurrency/blob/master/resultOfPart1.cpp)
* [Functions with variadic number of arguments](/blog/2016/03/variadic-number-of-arguments.html)
* [std::result_of](http://en.cppreference.com/w/cpp/types/result_of)
* [std::is_same](http://en.cppreference.com/w/cpp/types/is_same)
