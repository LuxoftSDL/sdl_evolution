# Replace thread::Thread implementation with std::thread from C++11

* Proposal: [SDL-NNNN](NNNN-filename.md)
* Author: [SDL Developer](https://github.com/smartdevicelink)
* Status: **Awaiting review**
* Impacted Platforms: [Core / iOS / Java Suite / JavaScript Suite / HMI / Policy Server / SHAID / RPC / Protocol]

## Introduction

Replacing pthreds with std::thread will make the code more platform-independent. That will simplify build, support and porting of the code to various platforms


## Motivation

Currently we using pthread (in POSIX systems) for thread works. It is a C library, and was not designed with some issues critical to C++ in mind, most importantly object lifetimes and exceptions.

C++11 standart library provides some classes, functions and primitives having a single interface for work with threads:

 - class thread as an abstraction for a thread of execution.
 - several classes and class templates for mutexes and locks, intending RAII2 to be used for their management and locking strategies, e.g., std::recursive_mutex, std::timed_mutex; std::unique_lock; std::defer_lock, std::try_to_lock.
 - function and class templates to create callable objects which are integrated into the thread facilities.
 - сondition variables (some uses of POSIX condition variables are better replaced by std::future).
 - atomic types and functions on atomic types.
 - memory fence functions to for memory-ordering between operations.
 - variadic templates (which enable the simple means of passing arguments to a thread function)
 - lambda expressions (anonymous closure objects), which can be used in place of functions.
 - rvalue references, which enable perfect forwarding to a thread function

By replacing the thread code with the features listed above, we can make it simpler, more reliable, and portable.


## Proposed solution

The current implementation of threads is a class - wrapper over a platform-specific thread.
Existing code can be simplified by introducing standardized С++ std::thread, which having a single interface and using the platform functionality in their implementation.

First of all we could change type of member handle_:


class Thread {                                   class Thread {
...                                              	...

	PlatformThreadHandle handle_;      --->   		std::thread handle_;


std::thread is a standard class that allows to do all the work with threads regardless of the target platform.
So in this case we could also remove from our thread.h this:

	#if defined(OS_POSIX)
	typedef pthread_t PlatformThreadHandle;
	#else
	#error Please implement thread for your OS
	#endif

If in some place we need a system specific handler then we could use std::thread::native_handle which will give us an implementation-defined base thread descriptor.

File name thread_posix.cc can be simplify to thread.cc

The interface of Thread API will remain practically the same, with the exception of some methods:
Specifically, I would suggest dropping the Thread stack resizing functionality, because it's extremely rare to actually need a non-default stack-size.
Some refactor of methods implementation in class Thread will be required:

  public methods:

  bool Start(); 									- need some refactor 
  bool Stop();  									- almost does not change
  void Join(const ThreadJoinOption join_option); 	- almost does not change
  ThreadDelegate* GetDelegate()                     - does not change
  void SetDelegate(ThreadDelegate* delegate)        - does not change
  static void SchedYield();                         - can be implemented with std::this_thread::yield
  static PlatformThreadHandle CurrentId();          - only the signature changes
  static void SetNameForId(const PlatformThreadHandle& thread_id,  std::string name); - does not change
  const std::string& GetThreadName()                - does not change
  bool IsRunning()                                  - does not change
  bool IsJoinable()                                 - almost does not change                      
  size_t StackSize()                                - redundant (on my opinion)
  PlatformThreadHandleThreadHandle() const          - only the signature changes
  bool IsCurrentThread() const;                     - refactor with using std::this_thread...
  const ThreadOptions& GetThreadOptions() const     - does not change

  private methods:
  Thread(const char* name, ThreadDelegate* delegate); - need some refactor 
  virtual ~Thread();                                  - need some refactor
  static void* threadFunc(void* arg);                 - need refactor probably we could move this functionality to some thread manager                
  static void cleanup(void* arg);                     - also can be move to thread manager  
  pthread_attr_t SetThreadCreationAttributes(ThreadOptions* thread_options); - does not change
  
  bool StopDelegate(sync_primitives::AutoLock& auto_lock);  - this methods probably should be renamed because current names are poorly representative of the implementation
  bool StopSoft(sync_primitives::AutoLock& auto_lock);        also need some refactor  
  void StopForce(sync_primitives::AutoLock& auto_lock);

  void JoinDelegate(sync_primitives::AutoLock& auto_lock); - does not change

Static methods of Thread class makes sense to move to ThreadManager class.
Also in ThreadManager could be moved functionsl of global functions CreateThread() and DeleteThread()

Based on the features listed in the Motivation section we can replace the contents of the files with c++11 facilities:
  - utils/atomic.h
  - utils/callable.h
  - utils/conditional_variable.h
  - memory_barrier.h


## Potential downsides

C++11 provides no equivalent to pthread_cancel(pthread_t thread).
As variant to solve this issue we can add some cancellation_token to thread attributes which contains a sign that thread has cancelled.

Pthreads provides control over the size of the stack of created threads, C++11 does not address this issue.
Most likely we can do without this functionality, but if not, then we can turn to std::thread::native_handle 



## Impact on existing code



## Alternatives considered

