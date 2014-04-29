---
layout: post
title: Why you should be using Java 8
---

Java 8 was recently released by Oracle and included a large number of improvements and new features that are poised to have the biggest impact on the platform since the introduction of [generics](http://docs.oracle.com/javase/tutorial/java/generics/) in Java 5.

# When did this become a thing?

Java 8 was released to the general public on March 18th, 2014, although the features included in the version had been in development for several years after the specification for the update ([JSR 337](https://jcp.org/en/jsr/detail?id=337)) was being worked on in the later months of 2010.

# So what's included in this update?

As with every major version update, a large number of changes were made to the Java platform. The biggest changes that were announced, and the one's that we'll be discussing today, include:

* Functional interfaces and default methods
* Lambda expressions
* Method references
* The Java Streaming API
* General API improvements
* Nashorn: The newest JavaScript engine

# Functional Interfaces

Interfaces allow the encapuslation of a specific behavior into something a class must implement. This encapsulation will allow a programmer to uniformly handle any class that implements that interface without having to worry about type errors.

In Java 8, a new type annotation has been added:

{% highlight java %}
@FunctionalInterface
{% endhighlight %}

A functional interface must contain **exactly one** abstract method. This means that every other method declared in the interface must be given a *default* implementation. Enter default methods:

{% highlight java %}
@FunctionalInterface public interface Predicate<T> {
    /**
     * This is our functional interface. It will take an input and
     * determine if that input matches the conditions defined in the
     * predicate.
     *
     * @param t The object to test against the predicate
     * @return True if the object satisfies the predicate and
     * returns false otherwise
     */
    public boolean test( T t);

    /**
     * This is a default method. Implementing the interface will
     * give you access to the methods with a default implementation
     * defined in that interface. This method will return a
     * Predicate<T> that returns the logical negation of what this
     * predicate returns.
     *
     * @return The predicate that returns the logical negation of
     * this predicate.
     */
    public default Predicate<T> negate() {
        Predicate<T> me = this;

        Predicate<T> predicate = new Predicate<T> {

            public boolean test(T t) {
                return ! me.test(t);
            }
        }

        return predicate;
    }
}
{% endhighlight %}
