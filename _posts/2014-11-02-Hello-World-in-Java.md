---
layout: post
title: Hello World in Java
categories: [java, programming, c]
tags: [hello world, java]
description: How Hello World actually works
comments: true
fullview: true
---

The hello world program is something that every developer is familiar with, and for some, it was their first foray into computer programming. Well, I want to return to ground zero and dive head-first into the rabbit hole to explain what that deceptively simple HelloWorld.java program is actually doing behind the scenes. To do this, we will be looking at the implementation of various features of the JVM, the JDK, and the Java Stadard Library. The source code for many of the classes present in the 1.8 version of the standard library is viewable at [grepcode](http://grepcode.com/).

To begin our journey, let's look at the source code for a bare bones hello world implementation in Java:

```java
public class HelloWorld {
    public static void main( String[] args ) {
        System.out.println("Hello World"); // Simple, right?
    }
}
```

The first place to look would appear to be the [System](http://docs.oracle.com/javase/8/docs/api/java/lang/System.html) class in the Java API. In the source code for System.java, we will find that ```System.out``` is declared like so:

```java
public final static PrintStream out = null;
```

Wait a minute! What do you mean ```System.out``` is null? If ```System.out``` is null, how could I possibly call ```println``` without having Java fall apart and spit out the ever-so-helpful NullPointerException? Well hold your horses and let's have a look. The System class is one of the first classes that is loaded by the JVM when it starts program execution, and when it is initialized the ```initializeSystemClass``` method is called. This method does many important things, including setting up System.out. The lines that properly initializes it are:

```java
FileOutputStream fdOut = new FileOutputStream(FileDescriptor.out);
setOut0(newPrintStream(fdOut,
    props.getProperty("sun.stdout.encoding")));
```
The first thing that you might notice is that we're opening up a [FileOutputStream](http://docs.oracle.com/javase/8/docs/api/java/io/FileOutputStream.html). Why? Well, we're opening a FileOutputStream on a [FileDescriptor](http://docs.oracle.com/javase/8/docs/api/java/io/FileDescriptor.html). Because stdout is not handled in the same way on all platforms, Java created the FileDescriptor class, whose sole responsibility is to handle setting up the input and output file handles. After setting up a FileOutputStream for stdout, the ```setOut0``` method is called, which has the following signature:

```java
private static native void setOut0(PrintStream out);
```

This is a Java native method, which means that its implementation is backed by native code defined using the JNI (Java Native Interface). As you delve deeper and deeper into the Java standard library, you will come to discover that many of the base operations in the language ultimately fall down to native methods. This is especially true for file IO and operations that involve system calls. The standard template for a native method that will be executing C code looks something like this:

```C
JNIEXPORT void JNICALL Java_ClassName_MethodName
  (JNIEnv *env, jobject obj)
{
    /*
     * The env pointer, at its most basic, is a reference to the JVM,
     * which allows it to perform actions that are necessary for
     * Java/C interop.
     *
     * The obj argument refers to the current class that this method
     * is being executed in.
     */

    /* Do some fancy C stuff here */
}
```


This native method in particular is defined in [System.c](http://hg.openjdk.java.net/jdk8/jdk8/jdk/file/687fd7c7986d/src/share/native/java/lang/System.c#l437) and sets the static ```out``` variable within the System class to be the FileOutputStream that was created.

```C
JNIEXPORT void JNICALL
Java_java_lang_System_setOut0(JNIEnv *env, jclass cla, jobject stream)
{
    jfieldID fid =
        (*env)->GetStaticFieldID(env,cla,"out",
            "Ljava/io/PrintStream;");
    if (fid == 0)
        return;
    (*env)->SetStaticObjectField(env,cla,fid,stream);
}
```

So, this is all well and good, but what actually happens when we call println? Well, when we're calling System.out.println, we're actually calling the println function defined in [PrintStream](http://docs.oracle.com/javase/8/docs/api/java/io/PrintStream.html), so let's have a look at that:

```java
public void println(String x) {
    synchronized (this) {
        print(x);
        newLine();
    }
}

public void print(String s) {
    if (s == null) {
        s = "null";
    }
    write(s);
}
```

For now, we'll ignore the call to the newline function, and focus instead on the print function. Aside from calling write, the only interesting thing that the write function does is perform a null check and replace your string with the default "null" text. Since print doesn't really seem to be supplying us with any answers, let's take a look at write:

```java
private void write(String s) {
    try {
        synchronized (this) {
            ensureOpen();
            textOut.write(s);
            textOut.flushBuffer();
            charOut.flushBuffer();
            if (autoFlush && (s.indexOf('\n') >= 0))
                out.flush();
        }
    }
    catch (InterruptedIOException x) {
        Thread.currentThread().interrupt();
    }
    catch (IOException x) {
        trouble = true;
    }
}
```

The next steps in this seemingly endless stream of operations involves a call to the write method, which eventually results in a call to an abstract write method as declared in the abstract [Writer](http://docs.oracle.com/javase/8/docs/api/java/io/Writer.html) class and implemented, in our case, in [BufferedWriter](http://docs.oracle.com/javase/8/docs/api/java/io/BufferedWriter.html).

```java
public void write(char cbuf[], int off, int len)
    throws IOException {
    synchronized (lock) {
        ensureOpen();
        if ((off < 0) || (off > cbuf.length) || (len < 0) ||
            ((off + len) > cbuf.length) || ((off + len) < 0)) {
                throw new IndexOutOfBoundsException();
        } else if (len == 0) {
            return;
        }
        if (len >= nChars) {
            /* If the request length exceeds the size of the output
               buffer, flush the buffer and then write the data
               directly. In this way buffered streams will cascade
               harmlessly. */
            flushBuffer();
            out.write(cbuf, off, len);
            return;
        }
        int b = off, t = off + len;
        while (b < t) {
            int d = min(nChars - nextChar, t - b);
            System.arraycopy(cbuf, b, cb, nextChar, d);
            b += d;
            nextChar += d;
            if (nextChar >= nChars)
                flushBuffer();
        }
    }
}
```

As if the number of hoops that we've jumped through hadn't quite hit critical mass, we can see that the write function in BufferedWriter calls the write function on _another_ stream, which in this case is an [OutputStreamWriter](http://docs.oracle.com/javase/8/docs/api/java/io/OutputStreamWriter.html):

```java
public void write(char cbuf[], int off, int len) throws IOException {
    se.write(cbuf, off, len);
}
```

_sigh_

As you can see, the write function in OutputStreamWriter calls yet another write function. This one happens to be the write function located in StreamEncoder, which is a lower level Java class located in the sun.nio.cs package. Outside of doing some bounds checking on the char array that gets passed to it, the actual meat of the operation is delegated to implWrite method which, after copying the characters into a buffer, calls the writeBytes method:

```java
private void writeBytes() throws IOException {
    bb.flip();
    int lim = bb.limit();
    int pos = bb.position();
    assert (pos <= lim);
    int rem = (pos <= lim ? lim - pos : 0);

    if (rem > 0) {
    if (ch != null) {
        if (ch.write(bb) != rem)
            assert false : rem;
        } else {
            out.write(bb.array(), bb.arrayOffset() + pos, rem);
        }
    }
    bb.clear();
}
```

There are two possible exit points for this method that we will be focusing on. The first calls the write function on a [WritableByteStream](http://docs.oracle.com/javase/8/docs/api/java/nio/channels/WritableByteChannel.html) whereas the second calls the write function on a FileOutputStream and passes it an array of bytes. However, we will be ignoring the first because at the time of writing this blog post, it is a disabled feature in the StreamEncoder as shown below:

```java
// This path disabled until direct buffers are faster
if (false && out instanceof FileOutputStream) {
    ch = ((FileOutputStream)out).getChannel();
```

The secondary exit point eventually results in a call to writeBytes in the FileOutputStream class.

```java
private native void writeBytes(byte b[], int off, int len,
    boolean append) throws IOException;
```

Holy native methods Batman! Beyond this point, the implementations differ based on the operating system that the JVM is running on - meaning that this is our exit. We shall leave continuing down the rabbit hole beyond this as an exercise to the reader.

In summary:

```java
System.out.println("Hello World");
```

is vastly more complicated under the hood than it first appears to be.
