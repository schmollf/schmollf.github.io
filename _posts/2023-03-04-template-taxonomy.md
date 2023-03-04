---
layout: post
title: Primer on Template Taxonomy
---

As a C++ programmer templates are important to know, but my personal experience is that I rarely get to write template libraries as part of my day job and only get to use them. As such, it requires some dedicated effort to get a better understanding of why certain design decisions are made. In this article, I'll list up a bunch of key words that are important when learning about templates. I find that knowing the taxonomy about a new subject gives me a skeleton to "attach" knowledge on and helps me prioritize what to learn first.

This article is a cop-out from writing a longer article on how templates work in depth or even why and when they are useful, but there are already a variety of resources about that out there. I'll link some of them at the end of this article and might add on the topic in a future sprint.

### Function Templates

The simplest way to use templates is to write a template function:

```c++
template<typename T>
void my_func(T t) {
    // do something
}
```

### Class Templates

Class templates let you generalize classes for various types:

```c++
template<typename T>
class MyClass
{
    void my_func(T t) {
        // do something
    }
}
```

### Full Specialization

Full specializations allow you to override the template class or template function for a specific type.

```c++
template<>
class MyClass<int>
{
    void my_func(int t) {
        // do something special for integers
    }
}
```

### Partial Specialization

Only class templates can be partially specialized. That means that you provide an override for a type that is stil not fully specificed, but more specific than just type T. An example would be an override for an array of artibrary type N `MyClass<T[N]>`;

### Type Parameters

Type parameters are the parameters specifid in the template declaration. In the following example that would be T and N: `template<typename T, typename N>`

### Variadic Templates

If a template takes an arbitrary number of parameters, we are speaking of variadic templates. This is indicated via ellipses (`...`), as in `template<typename... T>`

### Template Instantiation

Template instantiation refers to the process in which the compiler generates a class or function from the template.

## Closing Words

Once you get your hands dirty with actual code you'll also run across shorthands like SFINAE and practical patterns such as CRTP. The elephant in the room will be template metaprogramming. Templates are quite an extensive topic :)

## References
- if you're into books, I can recommend "C++ Templates - The Complete Guide" by David Vandevoorde et al.
- the [Microsoft guide](https://learn.microsoft.com/en-us/cpp/cpp/templates-cpp) is actually a pretty decent resource to start out with
