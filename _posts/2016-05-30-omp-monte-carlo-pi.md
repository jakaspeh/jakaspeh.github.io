---
layout: post
title:  "OpenMP: Monte Carlo method for Pi"
date:   2016-05-30
categories: cpp, OpenMP
---

In [the previous article](/blog/2016/05/monte-carlo-pi.html) we studied a
sequential Monte Carlo algorithm for approximating Pi (`{%raw%}π{%endraw%}`).

In this article we briefly repeat the idea of the sequential algorithm. Then,
we parallelize the sequential version with the help of OpenMP.

The sequential  algorithm
-------------------------

A Monte Carlo algorithm for approximating `{%raw%}π{%endraw%}`
uniformly generates the points in the square `{%raw%}[-1, 1] x [-1
1]{%endraw%}`. Then it counts the points which lie in the inside of the unit
circle. 

![Points in the square](/pics/generate_points_pi.png)

An approximation of `{%raw%}π{%endraw%}` is then computed by the following formula: 

![Approximation of Pi](/pics/approximation_pi.png)

For more details about the formula look in [the previous
article](/blog/2016/05/monte-carlo-pi.html).

The source code for the approximation algorithm is 

{% highlight c++ %}
double approximatePi(const int numSamples)
{
    UniformDistribution distribution;
    
    int counter = 0;
    for (int s = 0; s != numSamples; s++)
    {
        auto x = distribution.sample();
        auto y = distribution.sample();
        
        if (x * x + y * y < 1)
        {
            counter++;
        }
    }

    return 4.0 * counter / numSamples;
}
{% endhighlight %}

It uses `{%raw%}UniformDistribution{%endraw%}` to generate samples in
`{%raw%}[-1, 1]{%endraw%}`. Then it counts the number of points in the inside of
the circle and computes an approximation of `{%raw%}π{%endraw%}` by the
approximation formula.

Parallelization with OpenMP
---------------------------

The task is to parallelize the approximation algorithm with the help of OpenMP. 

Looking at the function `{%raw%}approximatePi{%endraw%}`, we might be tempted to
simply parallelize the for loop in the function with `{%raw%}#pragma omp
parallel for{%endraw%}`. But unfortunately, this approach has some problems. It
introduces data races in the program. 

The reasons for the data races are the variables `{%raw%}distribution{%endraw%}`
and `{%raw%}counter{%endraw%}`. They are updated by different iterations. If we
would parallelize the for loop, then the variables might be updated
by different threads at the same time.


Note that the `{%raw%}distribution{%endraw%}` is updated in every iteration,
because the sampling updates the internal state of the procedure which generates
the random samples.

Since a simple parallelization of the `{%raw%}approximatePi{%endraw%}` is not
possible, we slightly change the existing code. 

Modifications
=============

Our strategy is to divide the sampling into chunks. This strategy introduces
two nested loops

* an outer loop over the chunks and
* an inner loop over the samples for each chunk. 

The code for the outer loop is presented below

{% highlight c++ %}
double parallelPi()
{
    const int numTotalSamples = 1000000000; 
    int numChunks = 8;
    int chunk = N / numChunks;
    
    int counter = 0;
    
    for (int i = 0; i < numChunks; i++)
    {
        counter += samplesInsideCircle(chunk);
    }

    return approxPi = 4.0 * counter / numTotalSamples;
}
{% endhighlight %}

The function `{%raw%}parallelPi{%endraw%}` divides the number of total samples
into eight chunks. The `{%raw%}chunk{%endraw%}` variable holds the number of
samples for each chunk. Then the function loops over each chunk. In the for
loop, the function `{%raw%}samplesInsideCircle{%endraw%}` generates samples and
counts the number of samples in the inside of the unit circle. In the end, the
`{%raw%}parallelPi{%endraw%}` computes an approximation of the
`{%raw%}π{%endraw%}` with the help of the known formula.

The missing piece is the function `{%raw%}samplesInsideCircle{%endraw%}`. It is
presented below.

{% highlight c++ %}
int samplesInsideCircle(const int numSamples)
{
    UniformDistribution distribution;
    
    int counter = 0;
    for (int s = 0; s != numSamples; s++)
    {
        auto x = distribution.sample();
        auto y = distribution.sample();;
        
        if (x * x + y * y < 1)
        {
            counter++;
        }
    }
    
    return counter;
}
{% endhighlight %}

This function is very similar to the `{%raw%}approximatePi{%endraw%}`. The
`{%raw%}samplesInsideCircle{%endraw%}` generates `{%raw%}numSamples{%endraw%}`
uniform samples and counts how many of them are in the inside of the circle.

Parallelization
===============

Now, we are ready for a parallelization of the algorithm. We choose to
parallelize the for loop over the chunks, since it is the outer for loop.

In this for loop

{% highlight c++ %}
for (int i = 0; i < numChunks; i++)
{
    counter += samplesInsideCircle(chunk); 
}
{% endhighlight %}

we have three shared variables: `{%raw%}numChunks{%endraw%}`,
`{%raw%}counter{%endraw%}` and `{%raw%}chunk{%endraw%}`. We do not modify the
`{%raw%}numChunks{%endraw%}` and `{%raw%}chunk{%endraw%}`. They are read-only
variables. Therefore, this is not a problem. On the other hand, we modify the
`{%raw%}counter{%endraw%}`. We must deal with the problem of data races for this
variable.

Before we continue, let us remember the sequential version of the program. There
we have problems (data races) with two variables:
`{%raw%}distribution{%endraw%}` and `{%raw%}counter{%endraw%}`. The
`{%raw%}counter{%endraw%}` is still problematic but we do not have a problem
with the `{%raw%}distribution{%endraw%}`. We avoided this problem by having a
different `{%raw%}distribution{%endraw%}` for each chunk. With other words, the
sequential version initializes one object of
`{%raw%}UniformDistribution{%endraw%}` and the parallel version initializes
`{%raw%}numChunks{%endraw%}` objects of `{%raw%}UniformDistribution{%endraw%}`.

Now, we continue with the problem of `{%raw%}counter{%endraw%}`. 

{% highlight c++ %}
for (int i = 0; i < numChunks; i++)
{
    counter += samplesInsideCircle(chunk); 
}
{% endhighlight %}

OpenMP has a special clause of the loop construct which solves this kind of
problems. This is the `{%raw%}reduction{%endraw%}` clause which can specify a
reduction. An example of a reduction is

{% highlight c++ %}
sum = a0 + a1 + a2 + a3 + a4 + a5 + a6 + a7 + a8 + a9
{% endhighlight %}

Here, `{%raw%}+{%endraw%}` is the operation which reduces `{%raw%}a0{%endraw%}`,
`{%raw%}a1{%endraw%}`, ..., `{%raw%}a9{%endraw%}` into the
`{%raw%}sum{%endraw%}`. 

The reduction clause requires two things to specify a reduction

* the reduction operation and
* the variable, which will hold the result of the reduction.

In the for loop, `{%raw%}counter{%endraw%}` is the variable, which will hold the
result and `{%raw%}+{%endraw%}` is the reduction operation. The loop construct
then looks like

{% highlight c++ %}
#pragma omp parallel for shared(numChunks, chunk) reduction(+:counter)
for (int i = 0; i < numChunks; i++)
{
    counter += samplesInsideCircle(chunk); 
}
{% endhighlight %}

This loop construct correctly parallelizes the for loop. OpenMP increments the
shared variable `{%raw%}counter{%endraw%}` without introducing a data race. 

Example
-------

We run the sequential and parallel versions of the Monte Carlo algorithm for
approximating `{%raw%}π{%endraw%}`. We measure the execution time of the both
versions. The source code is available
[here](https://github.com/jakaspeh/concurrency/blob/master/ompMonteCarloPi.cpp).

{% highlight c++ %}
$ ./ompMonteCarloPi 
Sequential version: 
number of samples: 1000000000
real Pi: 3.141592653589...
approx.: 3.141582136
Time: 20017 milliseconds.

Parallel version: 
number of samples: 1000000000
real Pi: 3.141592653589...
approx.: 3.141563188
Time: 3643 milliseconds.
{% endhighlight %}

We can see that the execution time of the parallel version (3.6 seconds) is
lower than the execution time of the sequential version (20 seconds). Both
results are accurate up to four decimal places. I ran the computations on my 
machine which has 4 two-threaded cores.

In the next article, we will look in more detail at the reduction clause.

Summary
-------

We parallelized a Monte Carlo method for approximating
`{%raw%}π{%endraw%}`. We saw that the parallelization can decrease the execution
time.

Links:

* [Source code](https://github.com/jakaspeh/concurrency/blob/master/ompMonteCarloPi.cpp)
* [Monte Carlo method for Pi](/blog/2016/05/monte-carlo-pi.html)

Jaka's Corner OpenMP series:

* [OpenMP: Introduction](/blog/2016/04/omp-introduction.html)
* [OpenMP: For](/blog/2016/05/omp-for.html)
* [OpenMP: Sections](/blog/2016/05/omp-sections.html)

