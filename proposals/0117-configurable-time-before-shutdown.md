# Configurable time before shutdown

* Proposal: [SDL-0117](0117-configurable-time-before-shutdown.md)
* Author: [Alexander Kutsan](https://github.com/LuxoftAKutsan), [Andrii Kalinich](https://github.com/AKalinich-Luxoft)
* Status: **Accepted**
* Impacted Platforms: [Core]

## Introduction

To prevent missing logs after SDL shutdown, additional ini file options should be added : 
 - Write all logs to file system before shutdown 
 - Maximum time of SDL shutting down
 
## Motivation

In some use cases (like video streaming or big put file) SDL can produce a ton (up to 1 gigabyte) of logs in asynchronous mode. 
SDL collects these logs in the queue and writes them to a file system in a separate thread.
Writing to the file system may require a big amount of time (sometimes up to 5 - 10 minutes).

In the current implementation, after receiving IGNITION_OFF signal SDL dumps all logs into the file system before shutdown. Such behavior is useful for debugging purposes while working with the ATF scripts. However, such behavior from another side may cause a bad user experience when the system is trying to shut down several minutes after the video streaming on the production build.

To avoid such behavior, SDL should have a possibility to configure time before shutdown as well as the possibility to control the saving of the log messages.

## Proposed solution

 - Add option to flush log messages before shutdown.
 - Add option that specifies maximum time before shutdown.

```
// Write all logs in queue to file system before shutdown 
FlushLogMessagesBeforeShutdown = true

// Maximum time to wait for writing all data before exit SDL in seconds
MaxTimeBeforeShutdown = 30
```

By default `FlushLogMessagesBeforeShutdown` should be `true` to keep existing SDL behavior unchanged, so SDL will be able to dump all the log messages into the file system as it currently doing. This flag can be changed to `false` in the case when missing log messages before the shutdown are not necessary (for example in the production builds). In this case, SDL can be closed much faster.

`MaxTimeBeforeShutdown` would be used in case if `FlushLogMessagesBeforeShutdown` is `true`. It should measure the time taken by SDL since the start of the logger's `Flush()` call. If writing logs to the file system takes more time than specified, SDL should terminate `Flush()` call and exit from the logger process. Note, that time since `OnExitAllApplications` notification received can't be considered as a start point because it is just a happy path for SDL to shut down, however, SDL can also be stopped abnormally for example when the unhandled exception was thrown or system signal from another process received. Mentioned timeout should be applicable for described scenarios as well.

## Potential downsides

N/A

## Impact on existing code

Impacts ignition off process of SDL core.

## Alternatives considered
 1. Do not write all logs to SDL before shutdown ( some logs may  be missed)
 2. Write all logs before shutdown ( shutdown may take a long time)
 

