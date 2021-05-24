# Replace thread::Thread implementation with std::thread from C++11

* Proposal: [SDL-NNNN](NNNN-filename.md)
* Author: [SDL Developer](https://github.com/smartdevicelink)
* Status: **Awaiting review**
* Impacted Platforms: [Core]

## Introduction

This proposal is called to replacing [pthreads](https://man7.org/linux/man-pages/man7/pthreads.7.html) with [std::thread](https://www.cplusplus.com/reference/thread/thread/) will make the code more platform-independent. That will simplify build, support and porting of the code to various platforms.

## Motivation

The current implementation uses the pthread library (on POSIX systems) to work with threads. It is a C library which does not consider some OOP aspects critical to C++ like an object lifetimes and exceptions handling. With the transition to С++11 standard thread, we will be able to make the code simpler, more reliable, and also more easily portable to other platforms.

Implementation of class-wrapper for working with threads used in SDL Core looks overloaded. 
There is can note that we we already had some difficulties with threads when porting the code, this would not have been necessary if using a standardized thread.

C++11 standard library provides some classes, functions and primitives having a single interface for work with threads:
 - class thread as an abstraction for a thread of execution.
 - several classes and class templates for mutexes and locks, intending RAII2 to be used for their management and locking strategies, e.g., `std::recursive_mutex`, `std::timed_mutex`; `std::unique_lock`; `std::defer_lock`, `std::try_to_lock`.
 - function and class templates to create callable objects which are integrated into the thread facilities.
 - сondition variables (some uses of POSIX condition variables are better replaced by `std::future`).
 - memory fence functions to for memory-ordering between operations.
 - variadic templates (that enable the simple means of passing arguments to a thread function)
 - lambda expressions (anonymous closure objects), that can be used in place of functions.
 - rvalue references, which enable perfect forwarding to a thread function


## Proposed solution


The proposed solution is not aimed to change the functionality of the existing code. The changes will affect only the design of the code by generalizing the approach, removing obsolete parts of the code and introducing new approaches proposed by the C ++ 11 standard

The interface of Thread API will remain practically the same, with the exception of some methods:

 - some of the methods should be made as virtual
 - type of field handle_ changes to std::thread
 - static methods of Thread class makes sense to move to ThreadManager.
 - also in ThreadManager could be moved functional from global functions `CreateThread()` and `DeleteThread()`

In order to approximately show how the code will be changed, there can bring the class UsbHandler:

In current implementation we should create some delegate class (UsbHandlerDelegate) to wrap thread function.
For creation and binding Thread with delegate we use arbitrary function `Thread* CreateThread(const char* name, ThreadDelegate* delegate)`.

```
class UsbHandler {
    public:
	UsbHandler::UsbHandler() {
	  thread_ = threads::CreateThread("UsbHandler", new UsbHandlerDelegate(this));
	}
	
   private:

    void Thread() {...}

  	class UsbHandlerDelegate : public threads::ThreadDelegate {
   	public:
    void threadMain() {
  		handler_->Thread();
	}
    void exitThreadMain() OVERRIDE;

   	private:
    UsbHandler* handler_;
  	};

    threads::Thread* thread_;
  };
  ```
In this case using an extra wrapper for Thread() function is redundant: we could pass standard wrapper(std::function) directly to std::thread.
Also we keep Thread* thread_ by raw pointer and must do in destructor:
```  
  delete thread_->GetDelegate();
  threads::DeleteThread(thread_);
```

which is an additional reason for potential memory leaks.

With new approach we could rewrite this code to:
```
class UsbHandler {
    public:
	UsbHandler::UsbHandler() {
	  thread = std::thread([](){Thread();});
	}

   private:
    void Thread() {...}

    std::thread thread_;
  };
  ```

In other classes with multithreading behavior situation the same.

When using standard C++11 threads, if in some place we need a system specific handler (for example: stack resizing) then we could use `std::thread::native_handle` which will give us an implementation-defined base thread descriptor.

Based on the features listed in the Motivation section we can replace the contents of the files with c++11 facilities:
  - utils/callable.h
  - utils/conditional_variable.h
  - memory_barrier.h

We can also stop using mutexes from the boost library since it is a part of the standard library



## Potential downsides

C++11 provides no equivalent to pthread_cancel(pthread_t thread).
As an option to resolve this issue we can add some `cancellation_token` to thread attributes which contains a sign that thread has cancelled.

Pthreads provides control over the size of the stack of created threads, C++11 does not address this issue.
`std::thread::native_handle` can be used if needed 



## Impact on existing code
The thread approach of SDL Core will be impacted.


## Alternatives considered
Alternatively, we can use platform specific approaches (pthread on POSIX and win thread on windows) or numerous wrappers like Boost.Thread, Dlib, OpenMP, OpenThreads etc
