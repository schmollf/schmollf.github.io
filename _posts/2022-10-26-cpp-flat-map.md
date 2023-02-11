---
layout: post
title: std::flat_map in C++ 
---

`flat_maps` are a new container adaptor in C++23. Like std::map, it is an associative ordered container, meaning that it allows you to insert key-value-pairs and look up keys later on. While std::map is implemented using balanced binary trees, `std::flat_map` maintains a pair of sorted vectors, one first for keys and a second for values. That means that std::map has better asymptotic complexity, but `std::flat_map` might still perform better due to cache-friendliness of having contiguous memory. To understand that better, lets look at how exactly `std::flat_map` is implemented.

## How is `std::flat_map` implemented exactly?

`std::flat_map` is implemented using two sequence containers (by default std::vector). One container contains the keys in sorted order and the second container maintains all the values in the corresponding order.

This means that inserting is O(n) because if we insert at the front, we will have to shift all previous elements to the back. If you insert at the back, insertion will be O(1) if there is no allocation. That makes it particularly performant if your input is already sorted. Similar logic applies for deletion.

This helps us understand the following asymptotic complexities:

```
+-----------+----------+---------------+
| Operation | std::map | std::flat_map  |
+-----------+----------+---------------+
| Create    | O(nlgn)  | O(nlgn)       |
| Insert    | O(lgn)   | O(n)          |
| Delete    | O(lgn)   | O(n)          |
| Lookup    | O(lgn)   | O(lgn)        |
+-----------+----------+---------------+
```

## When is std::flat_map useful?

`std::flat_map` was introduced because it is more cache friendly to search across keys in contiguous memory than to do the many memory dereferences required by std::map when traversing the binary tree. However, we can see in the table on complexities above that random insertion and deletion are expensive operations, at least for large sizes.

Rule of thumb is to use flat_maps when insertions and deletions are rare and mostly lookups and iterating are required. Those are the use cases where std::flat_map shines and you can profit from its cache-friendliness.

## How do I use std::flat_map?

Flat map is mostly used like std::map. There are some caveats around iterator stability and node extraction is not supported. You can look in the standard for more details.

```c++
std::flat_map<int, int> my_map;
map[0] = 500;
map[4] = 3;
```

## Closing comments

Together with `std::flat_map` the committee also introduced the `std::flat_multimap`, `std::flat_set` and `std::flat_multiset` in cpp23. As you can imagine, the multi-versions allow duplicate keys and sets only maintain a single array of keys without values. Note also that its fairly easy to roll your own version of `flat_map` using std::pair and a single vector. Having two separate vectors is more cache-friendly, which is why its used in the standard.

If you've enjoyed this article and you are interested in writing high-performance software scaling across dozens of CPUs consider joining our team at Pure Storage. We have offices in Mountain View, Prague and Bangalore and are always looking for top notch talent to help us shape the future of data storage.

## Literature References

- Definition of `std::flat_map` in the C++ standard: https://eel.is/c++draft/flat.map
- None of the standard libraries have implemented `std::flat_map` as of October 2023, but you can find a standard-conforming implementation to play around with in the github repository of the responsible working group of the C++ committee https://github.com/WG21-SG14/SG14/blob/master/SG14/flat_map.h.
https://en.cppreference.com/w/cpp/header/flat_map
- Interesting talk by Arthur O'Dwyer which mentions flat_maps and exception guarantees. It helps to understand why it took so long to standardize std::flat_map. https://www.youtube.com/watch?v=b9ZYM0d6htg
- Summary of issues the committee had with an earlier proposal for `std::flat_map` https://www.open-std.org/jtc1/sc22/wg21/docs/papers/2019/p1727r0.pdf
- Boost has had its own version of flat_map for a while https://www.boost.org/doc/libs/1_80_0/doc/html/boost/container/flat_map.html
