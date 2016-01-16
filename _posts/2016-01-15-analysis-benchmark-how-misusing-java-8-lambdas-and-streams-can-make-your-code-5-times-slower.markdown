---
layout: post
title:  "Analysis Takipi Benchmark How Misusing Streams Can Make Your Code 5 Times Slower"
date:   2016-01-15 10:06:01
categories: benchmark
comments: true
---

The Takipi benchmark 'How Java 8 lambdas and streams can make your code 5 times slower' renamed to 'How Misusing Streams Can Make Your Code 5 Times Slower' contains interesting
but unexplained results:

* The first one is the boxing/unboxing issue as stated by [Sergey Kuksenko comment](http://blog.takipi.com/benchmark-how-java-8-lambdas-and-streams-can-make-your-code-5-times-slower/#comment-2377268130).
* The second one is the difference between the "lambda" and "stream" benchmark: the first one is 5 times slower than the last one whereas the code is quite similar.



<!--more-->


# First the results on my environment
 

| Benchmark  | ms/op   |
|:-----------|:---------|
| lambdaBoxingMaxInteger  | 0.513 ± 0.094  |
| lambdaMaxInteger        | 0.555 ± 0.062  |
| streamBoxingMaxInteger  | 0.506 ± 0.010  |
| streamMaxInteger        | 0.112 ± 0.037  |

 
## The original code
  
{% highlight java linenos %}
public int streamBoxingMaxInteger() {
    return integers.stream().reduce(Integer.MIN_VALUE, Integer::max);
}

public int lambdaBoxingMaxInteger() {
    return integers.stream().reduce(Integer.MIN_VALUE, (a, b) -> Integer.max(a, b));
}

public int streamMaxInteger() {
    return integers.stream().mapToInt(Integer::intValue).reduce(Integer.MIN_VALUE, Integer::max);
}
	
public int lambdaMaxInteger() {
    return integers.stream().mapToInt(Integer::intValue).reduce(Integer.MIN_VALUE, (a, b) -> Integer.max(a, b));
}
{% endhighlight %} 
   

  


# Boxing / Unboxing

`static int Integer#max(int a, int b)` takes primitive arguments and return a primitive. 
On the other side, the stream consumes a list of Integer. Thus the primitive result of `Integer#max` is boxed to an `Integer`, ie. an implicit 
`Integer#valueOf` is inserted by the compiler.  

{% highlight bytecode linenos %}
    LINENUMBER 191 L0
    ALOAD 0
    INVOKEVIRTUAL java/lang/Integer.intValue ()I
    ALOAD 1
    INVOKEVIRTUAL java/lang/Integer.intValue ()I
    INVOKESTATIC java/lang/Integer.max (II)I
    INVOKESTATIC java/lang/Integer.valueOf (I)Ljava/lang/Integer;
{% endhighlight %} 


As suggested by [Sergey Kuksenko comment](http://blog.takipi.com/benchmark-how-java-8-lambdas-and-streams-can-make-your-code-5-times-slower/#comment-2377268130), to get rid of the
 boxing, the max function must operate on `Integer`.
 
streamBoxingMaxInteger 
{% highlight java linenos %}
private static Integer max(Integer a , Integer b){
    return (a >= b) ? a : b;
}

public int streamBoxingMaxInteger() {
    return integers.stream().reduce(Integer.MIN_VALUE, LoopBenchmarkMain::max);
}

public int lambdaBoxingMaxInteger() {
    return integers.stream().reduce(Integer.MIN_VALUE, (a, b) -> max(a, b));
}
{% endhighlight %} 


## Results

Results are far better as soon as you are aware of the boxing issue:

| Benchmark  | Before ms/op   | After ms/op |
|:-----------|:---------|
| lambdaBoxingMaxInteger  | 0.527 ± 0.098 | 0.266 ± 0.268 |
| lambdaMaxInteger        | 0.563 ± 0.035 | TBD | 
| streamBoxingMaxInteger  | 0.617 ± 0.489  | 0.085 ± 0.013 |
| streamMaxInteger        | 0.115 ± 0.020  | TBD |

When working with boxed primitive it is better to use an `IntStream` using for example the `mapToInt` transformation.

> IntStream Stream#mapToInt(ToIntFunction<? super T> mapper)
> Returns an IntStream consisting of the results of applying the given function to the elements of this stream.

# Difference between the "lambda" and "stream" benchmark


This two codes are once compiled very similar:

* streamMaxInteger 

{% highlight java linenos %}
 integers.stream().mapToInt(Integer::intValue).reduce(Integer.MIN_VALUE, Integer::max);
{% endhighlight %} 
* lambdaMaxInteger
 
{% highlight java linenos %}
 integers.stream().mapToInt(Integer::intValue).reduce(Integer.MIN_VALUE, (a, b) -> Integer.max(a, b));
{% endhighlight %} 


The difference is in the reduce function. `streamMaxInteger` contains a reference to a static method. The compiled code use an `INVOKESTATIC` to the `Integer#max` method. 
`lambdaMaxInteger` uses a lambda. Once compiled this lambda is converted to a static method similar to:
{% highlight java linenos %}
private static Integer lambda$lambdaMaxInteger$0(Integer a , Integer b){
    return Integer.max(a,b);
}
 integers.stream().mapToInt(Integer::intValue).reduce(Integer.MIN_VALUE, LoopBenchmarkMain::lambda$lambdaMaxInteger$0);
{% endhighlight %} 

And the call site is too an `INVOKESTATIC`.

Once compiled both use `INVOKESTATIC` but `lambdaMaxInteger` has a call depth deeper by 1 than `streamMaxInteger`. The performance difference
 could be explained if the call to `lambda$lambdaMaxInteger$0` is not inlined


## Inlining

In order to analyse the JVM inlining, the bench is launched using [`-XX:+UnlockDiagnosticVMOptions`](http://stas-blogspot.blogspot.fr/2011/07/most-complete-list-of-xx-options-for.html#UnlockDiagnosticVMOptions)
 and [`-XX:+PrintInlining`](http://stas-blogspot.blogspot.fr/2011/07/most-complete-list-of-xx-options-for.html#PrintInlining) arguments. 

{% highlight java linenos %}
@ 32   java.util.ArrayList$ArrayListSpliterator::forEachRemaining (129 bytes)   inline (hot)
  @ 51   java.util.ArrayList::access$100 (5 bytes)   accessor
  @ 99   java.util.stream.ReferencePipeline$4$1::accept (23 bytes)   inline (hot)
    @ 12   com.takipi.oss.benchmarks.jmh.loops.OptimizedLoopBenchmarkMain$$Lambda$1/1699549388::applyAsInt (8 bytes)   inline (hot)
     \-> TypeProfile (12880/12880 counts) = com/takipi/oss/benchmarks/jmh/loops/OptimizedLoopBenchmarkMain$$Lambda$1
      @ 4   java.lang.Integer::intValue (5 bytes)   accessor
    @ 17   java.util.stream.ReduceOps$5ReducingSink::accept (19 bytes)   inline (hot)
     \-> TypeProfile (12880/12880 counts) = java/util/stream/ReduceOps$5ReducingSink
      @ 10   com.takipi.oss.benchmarks.jmh.loops.OptimizedLoopBenchmarkMain$$Lambda$2/813872125::applyAsInt (6 bytes)   inline (hot)
       \-> TypeProfile (13602/13602 counts) = com/takipi/oss/benchmarks/jmh/loops/OptimizedLoopBenchmarkMain$$Lambda$2
        @ 2   com.takipi.oss.benchmarks.jmh.loops.OptimizedLoopBenchmarkMain::lambda$lambdaMaxInteger$0 (6 bytes)   inline (hot)
          @ 2   java.lang.Integer::max (6 bytes)   inlining too deep
{% endhighlight %} 




