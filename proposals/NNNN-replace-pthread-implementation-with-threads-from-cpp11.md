# Replace thread::Thread implementation with std::thread from C++11

* Proposal: [SDL-NNNN](NNNN-filename.md)
* Author: [SDL Developer](https://github.com/smartdevicelink)
* Status: **Awaiting review**
* Impacted Platforms: [Core]

## Introduction

Replacing [pthreads](https://man7.org/linux/man-pages/man7/pthreads.7.html) with [std::thread](https://www.cplusplus.com/reference/thread/thread/) will make the code more platform-independent. That will simplify build, support and porting of the code to various platforms.

## Motivation

Currently we are using pthread library (in POSIX systems) for working with threads. It is a C library, and was not designed with some issues critical to C++ in mind like an object lifetimes and exceptions.

Implementation of class-wrapper for working with threads used in sdl looks overloaded. 
There is can note that we had faced with the need to add some multithreaded code when porting SDL to Android, this would not have been necessary if using a standardized thread.

C++11 standard library provides some classes, functions and primitives having a single interface for work with threads:
 - class thread as an abstraction for a thread of execution.
 - several classes and class templates for mutexes and locks, intending RAII2 to be used for their management and locking strategies, e.g., `std::recursive_mutex`, `std::timed_mutex`; `std::unique_lock`; `std::defer_lock`, `std::try_to_lock`.
 - function and class templates to create callable objects which are integrated into the thread facilities.
 - сondition variables (some uses of POSIX condition variables are better replaced by `std::future`).
 - memory fence functions to for memory-ordering between operations.
 - variadic templates (which enable the simple means of passing arguments to a thread function)
 - lambda expressions (anonymous closure objects), which can be used in place of functions.
 - rvalue references, which enable perfect forwarding to a thread function

By replacing the thread code with the features listed above, we can make it simpler, more reliable, and portable.


## Proposed solution

We could keep all functionality of the current implementation of multi-threaded code where it is required and at the same time be able to use the proposed standardized approach. For this the interface of Thread API will remain practically the same, with the exception of some methods:

 - some of the methods should be made as virtual
 - type of field handle_ can be defined as a template parameter.
 - static methods of Thread class makes sense to move to ThreadManager class.
 - also in ThreadManager could be moved functional from global functions `CreateThread()` and `DeleteThread()`

Below is an approximate class diagram for working with multithreaded code:

![GitHub Logo](https://raw.githubusercontent.com/LuxoftSDL/sdl_evolution/refactor/Replace_Thread_implementation_with_std_thread_fromC%2B%2B11/assets/proposals/Replace-Thread-implementation/threads_uml.png)

We could also remove from our thread.h this:

	#if defined(OS_POSIX)
	typedef pthread_t PlatformThreadHandle;
	#else
	#error Please implement thread for your OS
	#endif


When using standard C++11 threads, if in some place we need a system specific handler then we could use `std::thread::native_handle` which will give us an implementation-defined base thread descriptor.

Specifically, there is a suggestion to drop the Thread stack resizing functionality, because it's extremely rare to actually need a non-default stack-size.

Based on the features listed in the Motivation section we can replace the contents of the files with c++11 facilities:
  - utils/callable.h
  - utils/conditional_variable.h
  - memory_barrier.h

We can also refuse to use mutexes from the boost library, since it is as part of the standard library

## Potential downsides

C++11 provides no equivalent to pthread_cancel(pthread_t thread).
As variant to solve this issue we can add some cancellation_token to thread attributes which contains a sign that thread has cancelled.

Pthreads provides control over the size of the stack of created threads, C++11 does not address this issue.
Most likely we can do without this functionality, but if not, then we can turn to `std::thread::native_handle` 



## Impact on existing code
This proposal will impact SDL Core only.


## Alternatives considered
Only leave the multithread code as it is.
