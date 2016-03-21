---
layout: post
title:  "Variadic number of arguments"
date:   2016-03-21
categories: concurrency, cpp
---

The constructor of the `{%raw%}std::thread{%endraw%}` is able to accept
different number of arguments. It accepts a function and additionally
all the arguments for this function. If the function `{%raw%}f{%endraw%}`
has n arguments, then the constructor looks like

{% highlight c++ %}
std::thread thread(f, arg1, arg2, ..., argn);
{% endhighlight %}

If we would like to write concurrent code, we need to know how to handle
functions (constructors) with variadic number of arguments. We need to know how
to write such functions and we also need to know how to pass the arguments
around. Let's see how this is done.

Functions with variadic number of arguments
-------------------------------------------

We would like to write a function which prints all of its arguments. The
function should be able to handle any number of arguments. With the C++11
standard, this is not so hard to do.

{% highlight c++ %}
template <typename T>
void print(T value)
{
    std::cout << value << " ";
}

template <typename T, typename... Args>
void print(T first, Args... args)
{
    std::cout << first << " ";
    print(args...);
}
{% endhighlight %}

First function is simple. It receives a value of arbitrary type and then prints
the value.

Second function is more interesting. It receives two arguments
`{%raw%}first{%endraw%}` and `{%raw%}args{%endraw%}`. The
`{%raw%}first{%endraw%}` argument of type `{%raw%}T{%endraw%}` represents the
first argument of the function. The `{%raw%}args{%endraw%}` argument of the type
`{%raw%}Args...{%endraw%}` represents all of the other arguments. The function
prints the first argument and calls the function `{%raw%}print{%endraw%}` with
all the other arguments.

The `{%raw%}main{%endraw%}` function contains some examples. 

{% highlight c++ %}
int main()
{
    print(1);
    std::cout << std::endl;
    
    print(1, 2, 3, 4, 5);
    std::cout << std::endl;
    
    print("one", 2, "three", 4, "five");
    std::cout << std::endl;

    return 0;
}
{% endhighlight %}

Running the program gives us the following output.

{% highlight bash %}
$ ./variadicTemplates 
1 
1 2 3 4 5 
one 2 three 4 five 
{% endhighlight %}

Let's examine the `{%raw%}main(){%endraw%}` function. 

The `{%raw%}print(1){%endraw%}` calls the `{%raw%}print(T value){%endraw%}`. It
prints `{%raw%}1{%endraw%}` and then returns.

The `{%raw%}print(1, 2, 3, 4, 5){%endraw%}` calls the `{%raw%}print(T first,
Args... arg){%endraw%}` function. The `{%raw%}first{%endraw%}` then equals to
`{%raw%}1{%endraw%}` and the `{%raw%}args...{%endraw%}` equals to `{%raw%}2, 3,
4, 5{%endraw%}`. The function prints `{%raw%}1{%endraw%}` and calls 
`{%raw%}print(2, 3, 4, 5){%endraw%}`. The procedure continues until
`{%raw%}print(5){%endraw%}` is called. After this call the
`{%raw%}print(1, 2, 3, 4, 5){%endraw%}` returns and the main function proceeds
with the next line. 

You can visualize the `{%raw%}print(1, 2, 3, 4, 5){%endraw%}` like this:

![print(1, 2, 3, 4, 5)](/pics/variadic_print.png)

The `{%raw%}print("one", 2, "three", 4, "five"){%endraw%}` demonstrates that we
can also pass arguments of different types. 

Programming the functions with variadic number of arguments feels like
programming the recursive functions. Basically this is what we did: If the
number of arguments is equal to one, we print the argument. If the number of
arguments is greater than 1, we print the first argument and invoke
`{%raw%}print{%endraw%}` with the other arguments.

Technically, it is not a recursion, because every time different compiler
generated function is called. But conceptually, I would call it a recursion. 

The code does not look very clean. We recognize some repeating patterns. The
main function calls `{%raw%}print{%endraw%}` and then prints a new line. Our
objective is to write such function which will print arbitrary number of
arguments followed by a new line. 

Forwarding variadic number of arguments
---------------------------------------

Let's write a function `{%raw%}println{%endraw%}` which will forward the
arguments to `{%raw%}print{%endraw%}` and then it will additionally print a new
line. The `{%raw%}println{%endraw%}` should also accept variadic number of
arguments.

{% highlight c++ %}
template <typename... Args>
void println(Args&&... args)
{
    print(std::forward<Args>(args)..).;
    std::cout << std::endl;
}
{% endhighlight %}

The `{%raw%}Args&&...{%endraw%}` and `{%raw%}std::forward<Args>...{%endraw%}` are
necessary for perfect forwarding of the arguments. Roughly speaking, the code
ensures the following behavior: If the arguments support the move semantics they
will be moved. If the arguments does not support the move semantics they will be
passed by reference. For additional details about forwarding and move semantics
I recommend the [series of
articles](http://thbecker.net/articles/rvalue_references/section_01.html) by
Thomas Becker.

The main function now simplifies to 

{% highlight c++ %}
int main()
{
    println(1);
    
    println(1, 2, 3, 4, 5);
    
    println("one", 2, "three", 4, "five");

    return 0;
}
{% endhighlight %}

and it produces the same output as before.

Summary
-------

We learned how to handle a variadic number of arguments. This is specially
important in concurrent programming, because we might want to write a code which
could handle arbitrary number of arguments.

Links:

* [Source code](https://github.com/jakaspeh/concurrency/blob/master/variadicTemplates.cpp)
* Eli Bendersky wrote 
  [a nice article](http://eli.thegreenplace.net/2014/variadic-templates-in-c/) 
  about handling arbitrary number of arguments. He also covers 
  variadic data structures. 
* Thomas Becker's 
  [series of articles](http://thbecker.net/articles/rvalue_references/section_01.html) 
  in depth covers moving semantics and forwarding. 




