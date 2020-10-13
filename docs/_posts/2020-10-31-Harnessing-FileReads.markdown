---
layout: post
title:  "Using Hooks for Harnessing"
date:   2020-10-13 16:00:17 +1000
---
One problem I've encountered during fuzzing is how to best fuzz an application that performs multiple read operations on an input file. For example, say an application takes in an input file path from a user and parses it, if the application loads
the entire file into a single buffer to parse, this is simple to fuzz (we can modify the buffer in-memory and resume), however if the target does multiple read operations of various sizes, how do we best fuzz this target?

Of course if we're fuzzing by replacing the file on disk for each fuzz case this isn't an issue, but for performance if we're fuzzing entirely in-memory, we need to ensure each read operation the target performs on our input does not retrieve the original contents from disk, but instead recieves our
mutated input.

One method would be to break on each ReadFile operation and modify the returned contents, but then we have to handle the different file offsets it may read from, the read length, etc, alongside handling this for each individual ReadFile on our target.

I decided to go with the method of hooking different file operations (including ReadFile) and implementing my own custom handlers for these functions, this has multiple benefits:
    
* We eliminate syscalls, as lots of file operations are syscalls I hook and replace them with a user-land implementation, this is good for perf
* We keep track of different file operations but it all operates on a memory-mapped version of our input file, this means we can mutate this entire mem-mapped file once and guarantee all ReadFiles will be on our mutated mapped file

So in other words, the normal operation is as below:
    
* CreateFile on target
* ReadFile on target into buf 1 
    * (syscalls, reads from disk)
* Parse buf 1
* ReadFile on target into buf 2 
    * (syscalls, reads from disk)
* Parse buf 2

![Normal Operations](/assets/FileHook/oldop.png)

With our hooks, the operation turns into:
    
* CreateFile on target
    * Our hook memory maps the target once and keeps track of multiple handles to the file
* ReadFile on target into buf 1
    * We intercept this, no syscalls, reads from memory-mapped file, no disk IO
* Parse buf 1
* ReadFile on target into buf 2
    * Same as above
* Parse buf 2

![Hooked Operations](/assets/FileHook/newop.png)

This greatly simplifies what to mutate and eliminates some syscalls.

The implementation of this was pretty simple, MSDN has good documentation on how the APIs perform so we can essentially emulate it, alongside writing a tester application we can run without hooks, then with hooks to identify any
operations we are incorrectly handling. 

The code for this can be found on my github: [FileHook](https://github.com/Kharos102/FileHook)



