---
title: x64 calling convention
date: 2020-05-17 10:30:04
tags: ['Assembly Language', 'Registers', 'Calling Convention']
#excerpt: This post demonstrates how the functions are called in 64-bit application, espcially how function parameters are passed from the calling function to the called function.
---

This post demonstrates how the functions are called in 64-bit application, espcially how function parameters are passed from the calling function to the called function.
<!-- [![](/images/x64_calling_convention/D001.png)](http://localhost:4000/2020/05/x64-calling-convention/) -->

<!-- more -->

X64 application uses a 'fast call' calling conversion. The first 4 arguments are passed through registers. The rest goes onto the stack if the arguments are more than 4, much like x86. Typically you will see mov instructions rather than push instructions. And mov instructions here are used to mov some values or arguments into registers and that happens right prior to the call. Push instructions are pushing values or arguments onto the stack.

The first 4 arguments could also be pushed into Register Home Space, which is nothing more than an area on the stack that is setup by the calling function(the caller). We usually see this happens for applicatoin compiled in debug mode. Since the first 4 parameters are passed via registers but will possibly pushed onto stack, is it wasteful? Not really, this can be used for other purposes. The compiler will make certain decisions and make good use of the home space. It may put values back into home space or it may not. And if it doesn't, it will use the home space for other purpose.

Below is the demo code. The funtion _tmain will call Func who has 5 arguments. We compile it in 64bit mode and run it in WinDbg. We break at Line 19 and deassembles the _tmain function and take a look at its complete deassembled code. As we can see right before calling the Func function, the first 4 arguments a1, a2, a3 and a4 are moved to ecx, edx, r8d and r9d respectively. The 5th argument a5 is pushed onto the stack memory at [rsp+20h]. 
![Demo code](/images/x64_calling_convention/C001.png) 

Then we set a break point at Line 11 where is the beginning of the Func. When we hit the break point, we deassembe the Func. As we can see the values of ecx, edx, r8d, r9d are pushed onto the stack memory from [rsp+8h] to [rsp+20h], where is we called the Register Home Space. Note that these operations are done by the called function(the callee).
![Deassembled code of the Func](/images/x64_calling_convention/C002.png)

The memory layout should be like below.
![The stack memory layout when the calling function _tmain calls the called function Func](/images/x64_calling_convention/D002.png)
