---
layout: post
title: Why you should be using Java 8
---

Java 8 was recently released by Oracle and included a large number of improvements and new features that are poised to have the biggest impact on the platform since the introduction of [generics](http://docs.oracle.com/javase/tutorial/java/generics/) in Java 5.

## When did this become a thing?

Java 8 was released to the general public on March 18th, 2014, although the features included in the version had been in development for several years after the specification for the update ([JSR 337](https://jcp.org/en/jsr/detail?id=337)) was being worked on in the later months of 2010.

## So what's included in this update?

As with every major version update, a large number of changes were made to the Java platform. The biggest changes that were announced, and the one's that we'll be discussing today, include:

* Functional interfaces and default methods
* Lambda expressions
* Method references
* The Java Streaming API

## Functional Interfaces

Interfaces allow the encapuslation of a specific behavior into something a class must implement. This encapsulation will allow a programmer to uniformly handle any class that implements that interface without having to worry about type errors.

In Java 8, a new type annotation has been added:

{% highlight java %}
@FunctionalInterface
{% endhighlight %}

A functional interface must contain **exactly one** abstract method. This means that every other method declared in the interface must be given a *default* implementation. Enter default methods:

{% highlight java linenos %}
@FunctionalInterface public interface Predicate<T> {

    /**
     * This is our functional interface. It will take an input
     * and determine if that input matches the conditions
     * defined in the predicate.
     *
     * @param t The object to test against the predicate
     * @return True if the object satisfies the predicate and
     * returns false otherwise
     */
    public boolean test( T t);

    /**
     * This is a default method. Implementing the interface will
     * give you access to the methods with a default
     * implementation defined in that interface. This method will
     * return a Predicate<T> that returns the logical negation of
     * what this predicate returns.
     *
     * @return The predicate that returns the logical negation of
     * this predicate.
     */
    public default Predicate<T> negate() {
        Predicate<T> me = this;
        Predicate<T> predicate = new Predicate<T>() {
            public boolean test(T t) {
                return ! me.test(t);
            }
        };
        return predicate;
    }
}
{% endhighlight %}

An interface marked with the <code>@FunctionalInterface</code> annotation can have an unlimited number of default methods, but must have exactly one abstract method.

But is there any way that we can trim down on the verbosity?

## Lambda Expressions

In Predicate example for functional interfaces, we could have also written our negate method as follows:

{% highlight java linenos %}
    public default Predicate<T> negate() {
        Predicate<T> me = this;
        return new Predicate<T>() {
            public boolean test(T t) {
                return ! me.test(t);
            }
        }
    }
{% endhighlight %}

This is called an anonymous class definition. It allows you to define a class without specifically assigning it to a variable. It is likely one of the ugliest and most verbose constructs in the Java programming langauge. It also doesn't have any performance benefits because every time you declare an anonymous class, the JVM must jump through the same hoops for object instantiation.

Wouldn't it be nice if we could clean this up a bit? This is where lambda expressions come in. Lambda expressions are essentially anonymous functions and they can be used to great effect to gain not only readability benefits, but performance increases as well. The standard form for a lambda expression is as follows:

<code>(argument list) -> {function body}</code>

In the context of the Java platform, these can be used on *any functional interface* and the code in the <code>{function body}</code> will be the implementation for that functional interface's abstract method.

Using this, let's see if we can't clean up that negate code a bit:

{% highlight java linenos %}
    public default Predicate<T> negate() {
        Predicate<T> me = this;
        Predicate<T> negated = (t) -> ( ! me.test(t) );
        return negated;
    }
{% endhighlight %}

Now, there are a few moving parts here so let's break it down:

1. <code>(t)</code>: This is the argument list for the Predicate interface's abstract method, test. That method takes one argument, of generic type <code>\<T\></code>. Java's type system is smart enough to figure out the types of the items inside the argument list, so it is unnecessary to explicitly give them types.

2. <code>-\></code>: This separates the argument list from the function body. The arrow operator is something that is seen in many languages, especially in the context of functions and lambda expressions. You may, albeit rarely, hear this arrow referred to as the "burger arrow".

3. <code>(! me.test(t) );</code>: This defines the function body for the Predicates abstract method (test). In this case, it simply returns the negation of evaluating the input on ourselves.

Now what about those performance increases that were vaguely alluded to earlier? Well in [Java 7](http://jcp.org/en/jsr/detail?id=292), they introduced a new bytecode instruction called <code>invokedynamic</code>. In a nutshell, the <code>invokedynamic</code> instruction allows for symbols to be bound at runtime which, in this case, allows us to execute the lambda's function body without spinning up a new class instance. This saves the JVM the time that it would have spent during the object instantiation phase and sped the process up. For a more detailed explanation of lambda expression compilation on the JVM, see [this excellently detailed blog post](http://www.takipiblog.com/2014/01/16/compiling-lambda-expressions-scala-vs-java-8/) by Tal Weiss which discusses how Java 8 handles lambdas versus how Scala (another JVM language) handles them.

## Method References

So what should I do if I want to pass a method to another method so that I don't have to reinvent the wheel in my code? Well, then you pass a pass method to another method! In Java 8, they introduced method references, which allows for methods to be passed as an argument to functions. In most cases, this is generally syntactic sugar for a lambda expression. Suppose we are sorting a list of strings:

{% highlight java linenos %}
Collections.sort( strings, (a,b) -> a.compareTo(b) );
{% endhighlight %}

This could be rewritten using a method reference to the <code>compareTo</code> defined in the String class.

{% highlight java linenos %}
Collections.sort( strings, String::compareTo );
{% endhighlight %}

## The Streaming API

The addition of the streaming API in Java 8 is arguably one of the most useful additions in the release. As defined by the Java 8 API, a stream is:

>"A sequence of elements supporting sequential and parallel aggregate operations."

Much in the same way that you can use a for each loop to iterate over an iterable collection, you can now use streams to perform operations on the data as you go, possibly creating another stream in the process and eventually reducing into some desired result.

Let's tackle the problem of creating an array of all primes numbers in a list. For simplicity, we'll assume the existence of a method <code>isPrime(int)</code> that will return true iff the supplied integer is prime. Here is what a standard solution to this problem might look like:

{% highlight java linenos %}
public List<Integer> primesUnder( List<Integer> numbers ) {
    List<Integer> primes = new ArrayList<>();
    for( Integer i : numbers ) {
        if( isPrime(i) ) {
            primes.add(i);
        }
    }

    return primes.toArray( new int[primes.size()] );
}
{% endhighlight %}

Now let's see how we could do this by using Streams:

{% highlight java linenos %}
int[] primes = IntStream.rangeClosed(2, n)
                        .filter( i -> isPrime(i) )
                        .toArray();
{% endhighlight %}

This solution is much easier to read and as opposed to the standard iterative solution. However, this can still be improved. In the definition of a stream, it said that it supposed sequential **and** parallel operations - so which is this?

Well, by default, the creation of a stream will yield a stream that executes operations in a sequential fashion (non-parallel). However, creating a parallel stream is very simple. If we were to change our primes example so that it would execute in parallel, it would look like this.

{% highlight java linenos %}
int[] primes = IntStream.rangeClosed(2, n)
                        .parallel()
                        .filter( i -> isPrime(i) )
                        .toArray();
{% endhighlight %}

And it is as simple as that! However, be warned - for there be dragons! Parallel streams are **not** the solution to every problem and in some cases will actually cause performance to decrease as a result of the threading overhead. Try to reserve parallel streams for computationally intensive tasks that will actually benefit from parallelization.

As a final example, let's write a method that will take a list of files, load the images, rescale and rename those images, and then save them back to the disk. To shorten the length of this example, we're going to assume the existence of a few methods:

1. <code>loadImage(File f)</code>: Loads the image at the file or returns null if it is not an image.

2. <code>saveImage(BufferedImage i)</code>: Saves the image to the disk with a name and ID.

3. <code>getScaledImage(BufferedImage image, int width, int height)</code>: Rescales the image to the specified height and width.

The full implementation of these methods can be found [here](http://github.com/gizmo385/Java-8-Experiments/blob/master/Image%20Rescaler/src/ImageRescaler.java).

{% highlight java linenos %}
public void rescaleImages(Collection<File> files, final int width,
            final int height) {
    files.parallelStream()
         .map( f -> loadImage(f) )
         .filter(Objects::nonNull)
         .map( f -> getScaledImage( f, width, height )
         .forEach( f -> saveImage(f) );
}
{% endhighlight %}

As a final exercise, let's examine each statement and see what is going on in more detail:

1. <code>files.parallelStream()</code>: This creates a parallel stream of the files in the collection. This means that any operation carried out on this stream (and its derivatives) will be executed in parallel using a [fork/join](http://docs.oracle.com/javase/tutorial/essential/concurrency/forkjoin.html).

2. <code>map( f -> loadImage(f) )</code>: This makes use of a [higher-order function](http://en.wikipedia.org/wiki/Higher-order_function) known as [map](http://en.wikipedia.org/wiki/Map_\(higher-order_function\)) which applies a function to every item in the stream.

3. <code>filter(Objects::nonNull)</code>: nonNull is a method defined in the [Objects class](http://docs.oracle.com/javase/8/docs/api/index.html?java/lang/Object.html). This statement removes any objects from the list that are null.

4. <code>map( f -> getScaledImage( f, width, height )</code>: This scales all of the images in the stream to the specified height and width. This operations is a type of [Affine Transformation](http://en.wikipedia.org/wiki/Affine_transformation).

5. <code>forEach( f -> saveImage(f) )</code>: forEach performs an operation on every element in the stream. In this case, it saves the images to the disk.

## Conclusion

Java 8 added tons of features, a few of which were discussed here, that have the potential to make large changes to the mainstream Java development stack. The conciseness (in terms of Java) that is possible allows for seemingly complex operations to be distilled down to their base parts and expressed effectively to developers who are reading your code. On top of the readability improvements, many of these changes also carry performance increases alongside them - which is difficult to pass up.
