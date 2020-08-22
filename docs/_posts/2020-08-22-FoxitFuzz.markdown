---
layout: post
title:  "Creating a fuzzing harness for FoxitReader 9.7's ConvertToPDF Functionality"
date:   2020-08-22 20:15:17 +1000
---
Inspiration to create a fuzzing harness for the FoxitReader's ConvertToPDF function (targeting FoxitReader 9.7) came from discovering Richard Johnson's fuzzer for a previous version on FoxitReader (found here: https://www.cnblogs.com/st404/p/9384704.html).

Multiple changes have since been introduced into the way FoxitReader converts an image to a PDF, including the introduction of new vtable entries, the necessity to load in the main FoxitReader.exe binary (including fixing the IAT and modiyfing data sections to contain valid handles to the current process heap) + more.

The source for my version of the fuzzing harness targeting version 9.7 can be found on my github: https://github.com/Kharos102/FoxitFuzz9.7

Below is a quick walktrhough of the reversing and coding I had to do to get this harness working.

First, based on the existing work from the previous fuzzers available we know that most of the calls for the conversion of an image to a PDF occur via vtable function calls from an object returned from ConvertToPDF_x86!CreateFXPDFConvertor, however this could also be gleamed manually by breaking on file read access to the image we supply as a parameter to the conversion function and walking the call stack.

So first thing I decided to do is analyse how the actual FoxitReader.exe process sets up objects required for the conversion function, first I set a breakpoint for the CreateFXPDFConvertor function.

Next I step out and set a breakpoint on all the vtable function pointers for the returned object, this way I can discover in what order these functions are called along with their parameters as this will be necessary to setup the object before calling the actual conversion routine.

![Debugging FoxitReader](/assets/FoxitFuzz/vtable_analysis.png)

We know how to view the vtable as the pointer to the vtable is the first 4-bytes (32bit) when dumping the object.

During this process we can notice multiple differences compared to the older versions of FoxitReader, including changes to existing function prototypes and the introduction of new vtable functions that require to be called.

After executing and noting the details of execution, we hit the main conversion function from the vtable of our object, here we can analyse the main parameter (some sort of conversion buffer structure) by viewing its memory and noting its contents.

First we see the initial 4-bytes are a pointer to an offset within the FoxitReader.exe image:

![Debugging FoxitReader Cont.](/assets/FoxitFuzz/foxit_pointer_inbuf.png)

This means our harness will have to load the FoxitReader image in-memory to also supply a valid pointer (we also ahve to fix its IAT and modify the image too, as we discover after testing the harness).

Then we continue noting down the buffer's contents, including the input file path at offset +0x1624, the output file path at offset +0x182c, and more (including a version string and other little bits).

Finally after the conversion the object is released and the buffer is freed.

After noting all the above we can make a hrness from the information discovered and give it a test.

Issues arose shortly afterward, including exceptions in FoxitReader.exe that I loaded into memory, this was due to import table addresses not being resolved, this was fixed by manually patching the import table of the loaded FoxitReader binary.

Additionall, calls to HeapAlloc were occurring where the heap handle was obtained via an offset into the FoxitReader binary, this was fixed by writing the current process heap handle into the loaded FoxitReader image.

Overall the process was not long, and the resulting harness allowed fuzzing of the ConvertToPDF functionality in-memory with decent perf :).