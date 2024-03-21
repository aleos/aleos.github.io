---
title: C++ Initialization Lists
date: 2012-01-06T05:32:00Z
tags: [c++, dev]
---

All C++ programmers know that class members can be initialized in the initialization list in the constructor, but not everyone understands well when class members should receive their values in the initialization list and when in the constructor's body. Moreover, using initialization lists is associated with some implementation subtleties, neglecting which can lead to hard-to-understand errors.

To grasp the difference in the operation mechanism of the initialization list and the constructor's body, one must first turn to the concepts of initialization and assignment. _Initialization_ is the moment an object receives a value for the first time. The acquisition of a new value by an already initialized object is _assignment_. Functionally, the difference between these concepts is manifested in that a constructor is called during initialization (including the copy constructor), and `operator=` is executed during assignment. An important point is that by the time of entering the constructor's body, all class members are always initialized, even if it is not explicitly stated.

Such class members as references and constants must receive their values in the initialization lists, as they cannot change their value after initialization. Objects of classes (not to be confused with pointers to objects), whose constructors need to be supplied with parameters, also need to be specified in the initialization list. Using the initialization list can also provide a significant performance gain if used for large objects instead of copying in the constructor's body.

However, if a class has a large number of simple built-in type members (`int`, `double`, etc.), it might be more sensible to create a private method that would assign the needed values to all such members and be called in all constructors where necessary. The performance loss in this case will be negligible, but this will improve the readability of the code.

Another very important feature is that the order of initialization of class members is determined _only_ by the order of their declaration in the class. This feature is due to the need to destroy objects in the destructor in the correct (reverse) sequence, but in different constructors, the initialization lists can contain class members in different orders, so to simplify, a convention on the order of initialization identical to the order in the class description was adopted.
