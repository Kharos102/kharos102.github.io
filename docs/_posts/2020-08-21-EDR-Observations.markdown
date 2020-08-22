---
layout: post
title:  "EDR Observations"
date:   2020-08-21 20:15:17 +1000
---
Recently I've had the opportunity to assess multiple EDR & AV solutions including: CrowdStrike, Defender ATP, McAfee, Carbon Black Defence, and more.

As a result I've decided to write about a few observations including general/design weaknesses in these products.

## EDR Primer

Generally EDRs will contain the following components:

* Self-Protection
* Hooking engine
* Virtualization/Sandbox/Emulation
* Log/Alert generation
* Network comms

### Quick Primer: Kernel Callbacks (Self-Protection & Hooking Engine)

The windows NT kernel exports multiple callback routines, including but not limited to:

* PsSetCreateProcessNotifyRoutine
* PsSetLoadImageNotifyRoutine
* PsSetThreadCreateNotifyRoutine
* ObRegisterCallbacks
* CmRegisterCallbacks

![Kernel Callbacks](/assets/EDR Observations/kernel_callbacks.png)
Exported callback routines in ntoskrnl.exe

These callbacks may be used by kernel drivers such that when an event happens (process creation, registry modifications, handle creations, etc) the kernel driver is notified pre/post op and may interfere with the operation.

A common usage of this is for EDRs to be notified of process creations and inject their own userland DLLs (usually to hook NTDLL) into newly created processes.

Additionally EDRs may intercept handle creation events and block and handle creations that occur on their processes (an example of their self-protection capabilities)

### QuickPrimer: Disassembling Callbacks

Callbacks can be enumerated and disassembled on windows via kernel debugging (or in-kernel disassembling, e.g. by compiling a kernel driver with capstone).

If using kd/windbg in local/remote, we can leverage public symbols to first disassemble function "PsSetCreateProcessNotifyRoutine" with the command `Nt!PsSetCreateProcessNotifyRoutine`.

![Disassebled Nt!PsSetCreateProcessNotifyRoutine](/assets/EDR Observations/dis_processnotify.png)

We then follow any initial JMP (depending on the version of ntoskrnl.exe) to the main implementation of the function (e.g. nt!PspSetCreateProcessNotifyRoutine).

Continue disassembling the function and look for a "LEA" command on the callback array. Callbacks are stored in arrays of an undocumented "EX_CALLBACK" structure from which we can discover the function pointer that points to the actual callback registered for a particular driver.

![Locating Callback Array](/assets/EDR Observations/dis_callbackarr.png)

As shown above, the callback array should also have the symbol `nt!PspCreateProcessNotifyRoutine` and is loaded into r13.

Next, we can dump the contents of the callback array:

![Dumping Callback Array](/assets/EDR Observations/dump_callbackarr.png)

Leveraging the values dumped above (each representing a callback registration), we can resolve the callback function registered for each of these callbacks by changing the last byte of an entry on the array from "f" to "8", this will contain a pointer to the function registered to the callback, e.g. :

![Disassembled Callback Function](/assets/EDR Observations/dis_callbackfunc.png)

As shown above, we have chosen an entry from the array and modified the last byte to "8" and dissassembled it, we then discovered this callback belongs to the function `nt!ViCreateProcessCalback`.

### Hooking

Hooking techniques are commonly used by EDRs to hook userland functions for API monitoring. The following demonstrates a potential use for hooking whereby an EDR registers for process callback notifications and injects a DLL into each newly created process, this DLL then hooks NTDLL.DLL functions to potentially block/alert of malicious behavior (like calling NtReadVirtualMemory on a handle representing the LSASS process).

![NTDLL Hooking](/assets/EDR Observations/ntdll_hooks.png)

![Callbacks and Hooking](/assets/EDR Observations/callbacks_hooks.png)

Note: EDRs may also leverage sandbox/emulation to run a binary in isolation and log API usage.


## Common Weaknesses

The following list represents common weaknesses identified in multiple EDR solutions.

### Binary Padding

Scanning and emulation of a binary may be used to detect malicious behavior, however many EDRs (and AVs) have file size limitations on the file to analyze.

As a result, by appending junk to the end of a binary until it is roughly 100mb in size may be enough to prevent the EDR/AV from analyzing it.

This is effective against product that heavily rely on an emulation and scanning layer to detect threats.

### Unmonitored APIs

Typical APIs used for malicious activity (e.g. combinations of VirtualAllocEx, WriteProcessMemory & CreateRemoteThread) may be alerted on by EDRs for process injection.

However, perfoming the same or similiar actions with different sets of APIs may prevent EDRs from detecting such behavior.

For example, in the case of dumping sensitive process memory (like LSASS), EDRs may not alert on handle creation of the target process, but may instead alert when an api like MiniDumpWriteDump or ReadProcessMemory is called on the target.

However, if we clone the target process with PssCaptureSnapshot and dump the memory of the cloned process, we may bypass such detections. This stems from the following main factors:

1. Simple handle creations on a target process are permitted; and
2. Dumping memory of non-sensitive processes are permitted

Therefore dumping LSASS will be detected, but cloning LSASS and dumping the clones memory is permitted.

This can be manually coded, or you can simply leverage the "-r" flag in ProcDump.exe.

Another example is DLL injection via windows hooks (E.g. leveraging SetWindowsHookEx API), this method of process injection does not rely on typicall behavior of opening a process, writing into the process memory and then creating a thread. 

As a result, leveraging windows hooks also typically goes undetected.

### Breaking Process Trees

EDRs also leverage process trees for detecting malicious behavior (e.g. alerting if WORD.exe spawns CMD.exe), however we can leverage COM objects such as "C08AFD90-F2A1-11D1-8455-00A0C91F3880" that exposes the "ShellExecute" function to spawn arbitrary processes under the "explorer.exe" process, even from within VBScript running under WORD.exe.

There are other techniques too (e.g. leveraging RPC) that may also be applicable to break process-tree based detections.

### Attacking EDRs

EDR's weaknesses also include their design that make them susceptable to subversion.

For example, as shown above, userland hooking may be key to an EDR's detection capabilities (such that without it, the product may be rendered useless).

EDRs that hook userland APIs via hooking ntdll.dll may be subverted by loading up a fresh copy of ntdll.dll into the process and redirecting (via hooks) our API calls to the freshly loaded (and unhooked by EDR) ntdll.

This technique alone for bypassing EDR hooks may be enough to then perform LSASS dumping and other malicious actions without detections.

EDRs also expose a lot of attack surface due to their massive codebase (drivers, IPC, support for various file formats) that may make them full of privesc/RCE bugs too.