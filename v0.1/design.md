---
layout: v0.1/docpage
title: 1. Design
prev: Introduction
next: Messages
prev_page: index.html
next_page: messages.html
---

This chapter exposes the main design decisions that were made when developping
nanomsgxx.

Constants
---------

nanomsg uses many constants to represent protocols, socket domains, options,
flags... Since C++ syntax is mostly compatible with C, standard nanomsg constants
(NN\_&#42;) can be simply used wherever constants are expected in the C++ API.
However nanomsgxx ports all constants within the [nnxx](api/nnxx/namespace.html) namespace, removing the
leading NN\_. For example the [NN\_MSG](http://nanomsg.org/v0.3/nn_send.3.html) constant in the C API is ported by the
[nnxx::MSG](api/nnxx/namespace.html#MSG) symbol. This applies to almost all constants, except in a few cases
where it conflicts with some existing symbols (more on that later).

Strings
-------

nanomsgxx functions accept string arguments in some cases, to bind or connect
to a node for example. Everywhere a string is expected the functions are
designed to receive any object that can be converted to a C-string
(null-terminted char sequence), this includes:

- objects of **const char &#42;** type, or implicitly converted to this pointer type
- objects of **std::string** type, or any other specialization of
[std::basic_string](http://en.cppreference.com/w/cpp/string/basic_string)
- any object that has a c_str member function returning a **const char &#42;** 

This feature can be extended by providing a c_str function for new types, for
example:

```c++
namespace X {
  const char *c_str(const other_string_type &s) {
    return s.c_string();
  }
}
```

Objects
-------

The nanomsgxx API provides a few object types that abstracts concepts of the
C API, while trying to stay as close as possible it adds a few extra types to
integrate with the C++ standard library and take advantages of the rich
features of the language.

**Moveable Objects**

All types in the nanomsgxx API provide a move constructor and a move assignment
operator, these are powerful features of C++11 and the library makes heavy use
of it to provide an easy resource management model, as well as protecting the
user from mis-using the library in some places.

**Member vs Non-Member Functions**

Member functions are used of nanomsgxx objects to provide operations that are
part of the object's identity, providing the low-level pieces on which extensions
can be built.  
For example the [nnxx::socket](api/nnxx/socket.html) type will have the recv and send member functions
because receiving and sending messages are the core features of the this type,
but protocol extensions such as turning on or off Nagle's algorithm for TCP
connections will be provided as a non-member function because it's not an
operation that can be performed on any socket type.

Exceptions
----------

The nanomsg library uses the POSIX mechanism to report errors, setting errno
to some error number and returning a nagative number from the function.  
C++ exceptions are used to report these conditions and almost all exceptions
thrown by the nanomsgxx functions are sub-classes of
[std::system_error](http://en.cppreference.com/w/cpp/error/system_error),
so the original error number that was generated by the underlying nanomsg
routine can be accessed through the caught exception.

**Signals**

When a program receives a signal it may cause any ongoing system call to be
cancelled to execute a signal handler instead, in that case the systeam call
terminates with the **EINTR** error code. While nanomsg mirrors this behavior,
nanomsgxx reports these conditions with the [nnxx::signal&#95;error](api/nnxx/signal_error.html) exception.  
This can be deactivated on a per-call basis by passing [nnxx::NO&#95;SIGNAL&#95;ERROR](api/nnxx/namespace.thml#NO_SIGNAL_ERROR)
in the flags argument on any blocking call, you should then refer to the
documentation of each function to understand how to handle this case.

**Timeouts**

nanomsg's API lets the user define a timeout for any blocking operation
(typically receiving or sending messages), and reports any timeout in the
operation with a [nnxx::timeout&#95;error](api/nnxx/timeout_error.html) exception.  
This can be deactivated on a per-call basis by passing [nnxx::NO&#95;TIMEOUT&#95;ERROR](api/nnxx/namespace.html#NO_TIMEOUT_ERROR)
in the flags argument on any blocking call, you should then refer to the
documentation of each function to understand how to handle this case.

**Termination**

When the program is about to exit it is supposed to inform the nanomsg library
by calling
[nn_term](http://nanomsg.org/v0.3/nn_term.3.html),
making any future or pending calls to nanomsg routines
failing and setting errno to **ETERM**. While this is still reported with an
exception in nanomsgxx it's on case where the exception thrown isn't a
sub-class of [std::system_error](http://en.cppreference.com/w/cpp/error/system_error). The [nnxx::term&#95;error](api/nxxx/term_error.html) exception is
instead a sub-class of
[std::logic_error](http://en.cppreference.com/w/cpp/error/logic_error).
That allows the program to handle this condition away from local error handling
logic.

**Memory Allocation**

The nanomsg library uses dynamic memory allocations in some places, if these
fail for whatever reasons the functions report the error with the ENOMEM errno
code.  
This is another case where nanomsgxx will report an error through an exception
that is not a subclass of [std::system_error](http://en.cppreference.com/w/cpp/error/system_error).
The C++ standard library provides the
[std::bad_alloc](http://en.cppreference.com/w/cpp/memory/new/bad_alloc)
exception for this purpose,
in order to easily integrate with programs that are written for standard C++
nanomsgxx also reports memory allocation failures by throwing an instance of
[std::bad_alloc](http://en.cppreference.com/w/cpp/memory/new/bad_alloc). 