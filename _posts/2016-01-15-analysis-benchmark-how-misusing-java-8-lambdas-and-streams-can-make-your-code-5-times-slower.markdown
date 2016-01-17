---
layout: post
title:  "Analysis Takipi Benchmark 'How Misusing Streams Can Make Your Code 5 Times Slower'"
date:   2016-01-15 10:06:01
categories: benchmark
comments: true
---

The Takipi benchmark ['How Misusing Streams Can Make Your Code 5 Times Slower'](http://blog.takipi.com/benchmark-how-java-8-lambdas-and-streams-can-make-your-code-5-times-slower/) contains interesting
but unexplained results:

* The first one is the autoboxing/unboxing issue as stated by [Sergey Kuksenko comment](http://blog.takipi.com/benchmark-how-java-8-lambdas-and-streams-can-make-your-code-5-times-slower/#comment-2377268130).
* The second one is the difference between the "lambda" and "stream" benchmark: the first one is 5 times slower than the last one whereas the code is quite similar.


<!--more-->


# First the results on my environment
 
## The original code

The code is available on [Takipi github](https://github.com/takipi/loops-jmh-playground/). I have taken the code from both the `fixes` branch (optimized version) 
and the `master` branch (with the boxing issue). The modified code of this article is available [on github](https://github.com/nithril/loops-jmh-playground)   
  
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
   

## The results

Core i5, 3Ghz, JDK 8u66

| Benchmark  | ms/op   |
|:-----------|:---------|
| forMax2Integer          | 0.094 ±  0.002  |
| lambdaBoxingMaxInteger  | 0.500 ±  0.018  |
| lambdaMaxInteger        | 0.494 ±  0.253  |
| streamBoxingMaxInteger  | 0.503 ±  0.005  |
| streamMaxInteger        | 0.107 ±  0.001  |
  


# Autoboxing / Unboxing

`static int Integer#max(int a, int b)` takes `int` primitive arguments and return a `int` primitive. 
On the other side, the stream consumes a list of Integer. Thus the result of `Integer#max` is boxed from an `int` to an `Integer`, ie. an implicit 
`Integer#valueOf` is inserted by the compiler.  

{% highlight java linenos %}
    LINENUMBER 191 L0
    ALOAD 0
    INVOKEVIRTUAL java/lang/Integer.intValue ()I
    ALOAD 1
    INVOKEVIRTUAL java/lang/Integer.intValue ()I
    INVOKESTATIC java/lang/Integer.max (II)I
    INVOKESTATIC java/lang/Integer.valueOf (I)Ljava/lang/Integer;
{% endhighlight %} 


As suggested by [Sergey Kuksenko comment](http://blog.takipi.com/benchmark-how-java-8-lambdas-and-streams-can-make-your-code-5-times-slower/#comment-2377268130), to get rid of the
 autoboxing, the max function must operate on `Integer`.

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

Results are far better as soon as you are aware of the autoboxing issue:

| Benchmark  | Before ms/op   | After ms/op |
|:-----------|:---------|:---------|
| lambdaBoxingMaxInteger  | 0.500 ±  0.018 | 0.266 ± 0.268 |
| streamBoxingMaxInteger  | 0.503 ±  0.005 | 0.085 ± 0.013 |


When working with boxed primitive and to avoid this issue, it may be better to use an `IntStream` using for example the `mapToInt` transformation.

> IntStream Stream#mapToInt(ToIntFunction<? super T> mapper)
> Returns an IntStream consisting of the results of applying the given function to the elements of this stream.

# Difference between the "lambda" and "stream" benchmark

## Analysis

These two codes are very similar once compiled:

* streamMaxInteger 

{% highlight java linenos %}
 integers.stream().mapToInt(Integer::intValue).reduce(Integer.MIN_VALUE, Integer::max);
{% endhighlight %} 
* lambdaMaxInteger
 
{% highlight java linenos %}
 integers.stream().mapToInt(Integer::intValue).reduce(Integer.MIN_VALUE, (a, b) -> Integer.max(a, b));
{% endhighlight %} 


The difference is in the `reduce` function:
 
* `streamMaxInteger` contains a reference to a static method. The compiled code use an `INVOKESTATIC` to the `Integer#max` method. 
* `lambdaMaxInteger` uses a lambda. Once compiled this lambda is converted to a static method similar to:

{% highlight java linenos %}
private static Integer lambda$lambdaMaxInteger$0(Integer a , Integer b){
    return Integer.max(a,b);
}
 integers.stream().mapToInt(Integer::intValue).reduce(Integer.MIN_VALUE, LoopBenchmarkMain::lambda$lambdaMaxInteger$0);
{% endhighlight %} 


Once compiled both use `INVOKESTATIC` but `lambdaMaxInteger` has a call depth deeper by 1 than `streamMaxInteger`. The performance difference
 could be explained if the call to `lambda$lambdaMaxInteger$0` is not inlined


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

Gotcha,  **`@ 2   java.lang.Integer::max (6 bytes)   inlining too deep`**, the issue and the performance difference is not correlated to lambda/stream misuse. 
[Yan Bonnel is true](http://blog.takipi.com/benchmark-how-java-8-lambdas-and-streams-can-make-your-code-5-times-slower/#comment-2379774095).

> For lambda, try change Integer.max by Math.max. I think you hit a limit of jit so it doesn't inline code?

## Results

Let increase the max inlining level to 10 from 9 `-XX:MaxInlineLevel=10`

Results are again far better:

| Benchmark  | Before ms/op   | After ms/op |
|:-----------|:---------|
| lambdaMaxInteger        | 0.494 ±  0.253 | 0.109 ± 0.025 | 
| streamMaxInteger        | 0.107 ±  0.001 | 0.107 ± 0.001 |




# Conclusion


| Benchmark  | Before ms/op   | After ms/op |
|:-----------|:---------|
| forMax2Integer          | 0.094 ±  0.002  | 0.094 ±  0.002 |
| lambdaBoxingMaxInteger  | 0.500 ±  0.018  | <span style="color: green;">**0.080 ±  0.001**</span> |
| lambdaMaxInteger        | 0.494 ±  0.253  | <span style="color: green;">**0.108 ± 0.003**</span>  | 
| streamBoxingMaxInteger  | 0.503 ±  0.005  | <span style="color: green;">**0.080 ±  0.001**</span> |
| streamMaxInteger        | 0.107 ±  0.001  | 0.107 ± 0.001  |


The title `How Misusing Streams` is in my PoV wrong. These issues are not related to lambda/stream but to some Java subtlety. 
The autoboxing performance issue could have also occurred with a for statement:

{% highlight java linenos %}
    public int forMaxInteger() {
        Integer max = Integer.MIN_VALUE;
        for (int i = 0; i < size; i++) {
            max = Integer.max(max, integers.get(i));
        }
        return max;
    }
{% endhighlight %} 

In my opinion a better title would be `Benchmark: How Misusing Autoboxing Can Make Your Code 5 Times Slower`.

Java is a subtle language and optimizing could be complex and benchmarking could be hard. 
Java library does not implement for wrapper classes the counterpart methods which operate on primitives. 
It relies on autoboxing and unboxing and the Java compiler and JIT could not be able to fully optimize the code. 

Anyway for this use case (max of a list), stream and lambda are not slower than a `for` statement.
 On the contrary they seem faster. I do not have yet analyse why, maybe in a part 2.
 
 
 
 
 
 
 
 