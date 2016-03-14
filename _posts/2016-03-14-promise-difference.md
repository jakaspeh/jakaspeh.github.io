---
layout: post
title:  "Difference between promise, packaged task and async"
date:   2016-03-14
categories: concurrency
---

We know three ways of associating a task with a
`{%raw%}std::future{%endraw%}`. They are `{%raw%}std::async{%endraw%}`,
`{%raw%}std::packaged_task{%endraw%}` and `{%raw%}std::promise{%endraw%}`.

The benefit of the packaged task over `{%raw%}std::async{%endraw%}` is to
decouple the creation of the future with the execution of the task.

The benefit of the promise over `{%raw%}std::async{%endraw%}` and
`{%raw%}std::packaged_task{%endraw%}` is that the promise can communicate a
value to the future at a different time than at the end of a function call. Let
me explain what I mean with this.

Differences
-----------

### *std::async* ###

The `{%raw%}std::async{%endraw%}` takes a function as an argument and returns a
future. The future has the same template parameter as the return type of the
function. When the function returns, its return value sets the
future. Therefore, the `{%raw%}std::async{%endraw%}` communicates the value at
the end of the function call. For example:

{% highlight c++ %}
int function(double);

int main()
{
    ...
    std::future<int> future = std::async(function, 2.5);
    ...
}
{% endhighlight %}

The `{%raw%}std::async{%endraw%}` communicates the value to the future at the end of
`{%raw%}function{%endraw%}` execution.

### *std::packaged_task* ###


The constructor of the `{%raw%}std::packaged_task{%endraw%}` takes a function as
an argument and creates a packaged task object. The template parameter of the
packaged task is the signature of the function.

The `{%raw%}.get_future(){%endraw%}` member function of the object creates a
future. The future has the same template parameter as the return type of the
function

The execution of a packaged task, which forwards arguments to the function
and runs it, sets the return value of the function to the future. Therefore
again, the `{%raw%}std::packaged_task{%endraw%}` communicates the value at the
end of the function call. For example:

{% highlight c++ %}
int function(double);

int main()
{
    ...
    std::packaged_task< int(double) > task(function);
    std::future<int> future = task.get_future();
    task(2.5);
    ...
}
{% endhighlight %}

`{%raw%}task(2.5){%endraw%}` executes `{%raw%}function(2.5){%endraw%}` and sets
the return value of the latter call to the future. The
`{%raw%}std::packaged_task{%endraw%}` communicates the value to the future at
the end of `{%raw%}function{%endraw%}` execution.

### *std::promise* ###

There are no such restrictions with `{%raw%}std::promise{%endraw%}`. The promise
can explicitly set a value to a future anytime and not only at the end of a
function call. Let's look at an example of such behavior. 

#### Manager and mechanic ####

Imagine a car repair shop. A customer would like to buy a car. She would like to
be convinced that the engine of the car is OK. The manager of the shop instructs
the mechanic to inspect the engine. As soon as it is clear that the engine is
OK, the manager can negotiate a contract with the customer.

Let's model this story with a program. We have a class which represents a car. 

{% highlight c++ %}
class Car 
{
public:
    Car() 
        :hood(true), electricity(true),
         exhaust(true), engine(true)
    { }
    
    void remove_the_hood()
    { hood = false; }
    
    void add_the_hood()
    { hood = true; }

    void disconnect_electricity()
    { electricity = false; }

    void connect_electricity()
    { electricity = true; }

    void remove_exhaust_system()
    { exhaust = false; }
    
    void add_exhaust_system()
    { exhaust = true; }
    
    bool is_engine_ok() const
    { return engine; }

private:
    bool hood;
    
    bool electricity;
    
    bool exhaust;
    
    bool engine;
};
{% endhighlight %}

The `{%raw%}Car{%endraw%}` has four data members: `{%raw%}hood{%endraw%}`,
`{%raw%}electricity{%endraw%}`, `{%raw%}exhaust{%endraw%}` and
`{%raw%}engine{%endraw%}`. They are all represented by a boolean variable. If a
data member is `{%raw%}true{%endraw%}` than everything is OK, otherwise
something is wrong (the part is broken or missing). 

Additionally, there are data functions which can remove/add certain part of the
car. The `{%raw%}.is_engine_ok(){%endraw%}` tells us whether the engine is OK. 

The next part describes the behavior of the mechanic.

{% highlight c++ %}
void make_break(int millisec)
{
    std::this_thread::sleep_for(std::chrono::milliseconds(millisec));
}
        
void mechanic(Car& car, std::promise<bool> engine_ok)
{
    car.remove_the_hood();
    car.disconnect_electricity();
    car.remove_exhaust_system();
    
    std::cout << "Mechanic: took a car apart." << std::endl;
    
    engine_ok.set_value(car.is_engine_ok());
    
    std::cout << "Mechanic: engine is ok." << std::endl;
    
    if (car.is_engine_ok())
    {
        make_break(10); 
        car.add_exhaust_system();
        car.connect_electricity();
        car.add_the_hood();
    
        std::cout << "Mechanic: put car back together." << std::endl;
    }
}
{% endhighlight %}

The mechanic first removes the hood, electricity and the exhaust system in
order to get an access to the engine. Then he communicates the status of the
engine to the manager via the promise. If the engine is OK, the mechanic
puts the car back together.

The point is that the promise communicates the status of the
engine in the middle of the function. This is not possible with
`{%raw%}std::async{%endraw%}` or `{%raw%}std::packaged_task{%endraw%}`, because
they communicate a value only at the end of the function call.

The last part describes the manager. 

{% highlight c++ %}
void manager()
{
    Car car;
    
    std::cout << "Manager: I would need to know if the car engine is ok."
              << std::endl;
    
    std::promise<bool> promise;
    std::future<bool> answer = promise.get_future();
    
    std::thread thread (mechanic, std::ref(car), std::move(promise));
    thread.detach();
    
    if (answer.get())
    {
        std::cout << "Manager: ensures the client that the engine is ok.\n"
                  << "Manager: negotiates the contract for selling the car."
                  << std::endl;
    }
    else
    {
        std::cout << "Manager: buys a new engine." << std::endl;
    }
    
    make_break(10);
}

int main()
{
    manager();
    return 0;
}
{% endhighlight %}

The manager would like to know if the car engine is OK. Therefore, he instructs
the mechanic to check the engine. This is done by creating a new thread which
represents the mechanic:

{% highlight c++ %}
std::thread thread (mechanic, std::ref(car), std::move(promise));   
{% endhighlight %}

The function `{%raw%}mechanic(...){%endraw%}` requires two arguments: a car and a
promise. The `{%raw%}std::ref{%endraw%}` correctly passes the
`{%raw%}car{%endraw%}` as a reference to the `{%raw%}mechanic(...){%endraw%}`. Without
`{%raw%}std::ref{%endraw%}` a copy of the `{%raw%}car{%endraw%}` would be
passed.  Look at [Passing arguments to a thread](/blog/2016/01/arguments.html)
for a detailed explanation.

The `{%raw%}promise{%endraw%}` is not copyable, therefore
`{%raw%}std::move{%endraw%}` moves it to the `{%raw%}mechanic(...){%endraw%}`. 

After creating and detaching the thread, the manager waits for the answer from
the mechanic. If the answer is `{%raw%}true{%endraw%}` -- the car engine is OK
-- then the manager negotiates the contract for selling the car. Otherwise, he
buys a new engine.

If you are not familiar with `{%raw%}std::promise{%endraw%}`, look at
[Promise](/blog/2016/03/promise.html).

#### Running the example ####

The output of the example is:

{% highlight c++ %}
$ ./promise2
Manager: I would need to know if the car engine is ok.
Mechanic: took a car apart.
Mechanic: engine is ok.
Manager: ensures the client that the engine is ok.
Manager: negotiates the contract for selling the car.
Mechanic: put car back together.
{% endhighlight %}

The figure below describes the timeline of the program.

![Car and promise](/pics/car_promise.png)

This example shows the difference between the promise and other synchronization
mechanisms. The promise communicates the status of the engine in the middle of
the function and not at the end.

Summary
-------

The difference between the promise and other synchronization mechanisms is that
the promise can communicate the value at a different time than at the end of the
function call. We illustrated this difference with the example.

* [Source code](https://github.com/jakaspeh/concurrency/blob/master/promise2.cpp)
* [Jaka's Corner: Futures](/blog/2016/02/futures.html)
* [Jaka's Corner: Packaged task](/blog/2016/02/packaged-task.html)
* [Jaka's Corner: Promise](/blog/2016/03/promise.html)


