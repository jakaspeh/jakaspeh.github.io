---
layout: post
title:  "Packaged task with a restaurant"
date:   2016-02-29
categories: concurrency
---

The `{%raw%}std::packaged_task{%endraw%}` is one of the possible ways of
associating a task with a `{%raw%}std::future{%endraw%}`. The benefit of the
packaged task is to decouple the creation of the future with the execution of
the task.

In the following, we will look how to exploit this decoupling. 

Restaurant
----------

Imagine a restaurant. There are a waiter and a cook. The procedure usually goes
like this

- a customer orders a meal,
- the waiter records the order and passes it to the cook,
- the cook receives the order, cooks the meal and sends 
  it back to the waiter,
- the waiter serves the meal.

In our particular example, we have a moody cook. He worked in a fast food
restaurant before and he doesn't want to make hamburgers any
more. Unfortunately, our waiter is not aware of this issue.

We would like to model the restaurant with our code. The cook and the waiter
will be represented as two threads. The tasks
(`{%raw%}std::packaged_task{%endraw%}`) will be orders. The waiter will create
the task, hold its `{%raw%}std::future{%endraw%}`  and push the task to the cook. 
The cook will execute the task: make a meal out of the order. 

Since we now know the general idea of the program, let's look at the code. 

{% highlight c++ %}
#include <iostream>
#include <string>
#include <chrono>
#include <queue>

#include <thread>
#include <future>
#include <utility>


std::mutex MUT;
std::queue < std::packaged_task<bool()> > ORDERS;
std::condition_variable CV;

bool CLOSED = false;

void make_break(const int milisec)
{
    std::this_thread::sleep_for(std::chrono::milliseconds(milisec));
}
{% endhighlight %}

The code begins with include statements. We declare global mutex, queue and
condition variable. This variables will be used for passing the tasks between
the threads. You should review the [Passing data with condition
variable](/blog/2016/02/passing-data-condition-variable.html) if you are not
comfortable passing data (in our case the package tasks) with condition
variable.

The boolean variable `{%raw%}CLOSED{%endraw%}` marks whether the restaurant is
open or closed.

The `{%raw%}make_break{%endraw%}` function is familiar to us. It was a building
block of many previous examples. We use it to make the output nicer. 

Next follows the code for the cook. 

{% highlight c++ %}
void cooking()
{
    while (!CLOSED)
    {
        std::packaged_task< bool() > cooking_order;
        {
            std::unique_lock< std::mutex > guard(MUT);
            CV.wait(guard, []{return !ORDERS.empty();});
            cooking_order = std::move(ORDERS.front());
            ORDERS.pop();
        }
        cooking_order();
    }
}

bool cook_the_meal(const std::string meal)
{
    if (meal == "hamburger")
    {
        std::cout << "Cook: I don't make hamburgers!" << std::endl;
        return false;
    }
    else
    {
        std::cout << "Cook: making the " << meal << std::endl;
        make_break(2);
        return true;
    }
}
{% endhighlight %}

The `{%raw%}cooking(){%endraw%}` function loops until the restaurant closes. It
gets a cooking order from the queue with the help of the condition variable.
[Passing data with condition
variable](/blog/2016/02/passing-data-condition-variable.html) explains how this
works. Then, the function executes the cooking order -- the cook prepares the
meal.

The function `{%raw%}cook_the_meal(...){%endraw%}` also models the cook.  Here, the
 moody behavior of the cook is defined. If the meal is hamburger, the cook
 doesn't want to prepare it, therefore the function returns
 `{%raw%}false{%endraw%}`. Otherwise, the cook prepares the meal and the
 function returns `{%raw%}true{%endraw%}`.

Now follows the code which describes the waiter. 

{% highlight c++ %}
std::future<bool> add_order(const std::string meal)
{
    std::cout << "Adding order: " << meal << std::endl;
    
    std::packaged_task<bool()> order(std::bind(cook_the_meal, meal));
    std::future< bool > result = order.get_future();

    std::lock_guard< std::mutex > guard(MUT);
    ORDERS.push(std::move(order));
    CV.notify_one();
    
    return result;
}

void serve(std::future<bool> meal, 
           const std::string mealName)
{
    if (meal.get())
    {
        std::cout << "Waiter: serving " << mealName << std::endl;
    }
    else
    {
        std::cout << "Waiter: Unfortunately we don't have " 
                  << mealName << ". Would you mind to order again?" 
                  << std::endl;
    }
}

void serve_orders()
{
    std::string mealOrder1 = "steak";
    std::string mealOrder2 = "hamburger";
    std::string mealOrder3 = "cheesecake";
    
    std::future<bool> meal1 = add_order(mealOrder1);
    std::future<bool> meal2 = add_order(mealOrder2);
    std::future<bool> meal3 = add_order(mealOrder3);
    
    serve(std::move(meal1), mealOrder1);
    serve(std::move(meal2), mealOrder2);
    serve(std::move(meal3), mealOrder3);
}
{% endhighlight %}

Let's explain it from the bottom up.

#### *serve_orders* ####

There are three orders: steak, hamburger and cheesecake. The waiter first adds
all three orders to the queue by calling the `{%raw%}add_order{%endraw%}`
function. The function returns the future which represents the boolean --
`{%raw%}true{%endraw%}` if the meal is cooked or `{%raw%}false{%endraw%}` if
something went wrong.

Later, the waiter serves the meals via `{%raw%}serve{%endraw%}` function. The
`{%raw%}std::future{%endraw%}` is not copyable, therefore we need to use
`{%raw%}std::move{%endraw%}` when passing the future as an argument.

#### *serve* ####

The waiter checks the results of the cooking by calling the
`{%raw%}.get(){%endraw%}` member function of the future. If it returns
`{%raw%}true{%endraw%}`, everything went well and the waiter serves the meal.
Otherwise, the waiter apologies to the customer and kindly asks her/him to order
again.

#### *add_order* ####

The `{%raw%}add_order{%endraw%}` function passes the orders between the threads
using the same technique as in [Passing data with condition
variable](/blog/2016/02/passing-data-condition-variable.html).

It creates the package task from the string which represents the meal.
The `{%raw%}std::bind{%endraw%}` binds the function
`{%raw%}cook_the_meal{%endraw%}` with the `{%raw%}std::string{%endraw%}`
argument. If we have

{% highlight c++ %}
auto make_cake = std::bind(cook_the_meal, "cake");
{% endhighlight %}

then invoking `{%raw%}make_cake(){%endraw%}` is the same as 

{% highlight c++ %}
cook_the_meal("cake");
{% endhighlight %}

The advantage of using `{%raw%}std::bind{%endraw%}` is that we
pass only the `{%raw%}std::packaged_task< bool() >{%endraw%}` to the other
thread instead of passing `{%raw%}std::packaged_task< bool(std::string)
>{%endraw%}` together with all the arguments for the packaged tasks.

After the creation of packaged task, we get the future from the packaged task,
push the packaged task to the queue and notify the condition variable. (This
technique is the same as in [Passing data with condition
variable](/blog/2016/02/passing-data-condition-variable.html).) Note that
`{%raw%}std::packaged_task{%endraw%}` is not copyable, therefore
`{%raw%}std::move{%endraw%}` moves it to the queue. At the end, the function
returns the future.

We decoupled the creation of the future with the execution of the task. The
future is created in `{%raw%}add_order{%endraw%}`. The task is executed
in `{%raw%}cooking{%endraw%}`, which will run in a different thread than
`{%raw%}add_order{%endraw%}`.

{% highlight c++ %}
int main()
{
    std::thread cook(cooking);
    cook.detach();
    
    std::thread waiter(serve_orders);
    waiter.detach();
    
    
    make_break(100);
    CLOSED = true;
    
    return 0;
}
{% endhighlight %}

The main function creates two threads: `{%raw%}cook{%endraw%}` and
`{%raw%}waiter{%endraw%}` with appropriate functions. After some time, we close
the restaurant and the main function ends.

The entire source code is available [here](https://github.com/jakaspeh/concurrency/blob/master/packagedTaskRestaurant.cpp).

Running the restaurant
----------------------

This is one possible output of the program. 

{% highlight c++ %}
$ ./packagedTaskRestaurant 
Adding order: steak
Adding order: hamburger
Adding order: cheesecake
Cook: making the steak
Cook: I don't make hamburgers!
Waiter: serving steak
Waiter: Unfortunately we don't have hamburger. Would you mind to order again?
Cook: making the cheesecake
Waiter: serving cheesecake
{% endhighlight %}

The picture below describes the flow of the program. 

![Packaged task and restaurant](/pics/packagedRestaurant.png)

At the beginning, the waiter sends three orders to the cook. The cook picks up
the orders and starts making the meals. When the cook finishes the meal
(successfully or unsuccessfully), he informs the waiter about it (via
`{%raw%}std::future{%endraw%}`). Once the waiter gets the prepared meal, he serves
it.

The figure nicely describes the decoupling. The waiter's thread is responsible
for the creation of the packaged tasks and the cook's thread is responsible for
the execution of the tasks.


Summary
------- 

The `{%raw%}std::packaged_task{%endraw%}` decouples the creation of the future
with the execution of the task. We learned how to benefit from this
decoupling. We created tasks in one thread and we executed them in the other
thread.

Links:

* [Source code](https://github.com/jakaspeh/concurrency/blob/master/packagedTaskRestaurant.cpp)
* [std::bind](http://en.cppreference.com/w/cpp/utility/functional/bind)
* [std::packaged_task](http://en.cppreference.com/w/cpp/thread/packaged_task/packaged_task)

