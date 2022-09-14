---
layout: default
permalink: /posts/llvm_for_security_4_4
title:  "LLVM Passes for Security: (Part 4/4)"
---


# LLVM Passes for Security: A Pass for Fuzzing -- Coverage and Context-sensitivity  (Part 4/4)


## Abstract

This module doesn't add too many novel concepts, compared to the previous ones, but still, it shows a trending application of LLVM for security, i.e., fuzzing. Our goal is to create a pass that instrument a target to log the code coverage. As a further exercise, we'll then implement an additional instrumentation feature, i.e., context sensitivity. 
Our refernce tool will be AFL++, and thus our pass will work with this fuzzer, even though in principle many of the ideas we're going to exploare are equivalent with other fuzzers.


##### LLVM Concepts

- Add LLVM IR at compile time
- Use of external variables


### What we want to implement

I assume that you already have an idea about how fuzzers work. Anyway, the role of a compiler pass in fuzzing, is to instrument a target according to one of the many existing approaches, in such a way that the instrumentation will collect some information about the status of a certain execution. Such type of information, a.k.a. feedback, drives the fuzzer to discover different program points.
For instance, in the first part of this exercise, we develop an edge-coverage instrumentation AFL-style. This essentially logs when new edges are discovered and takes into account how many times these are executed.
In the second part instead, we'll implement a more sophisticated technique to take into account not just the edge-coverage but also the context, i.e., the routines/methods invoked before reaching a certain point.


### Our implementation

For this implementation, I'll refer to AFL++ commit 147654f8715d237fe45c1657c87b2fe36c4db22a. Let's start.

First, let's try to reproduce the edge-coverage with hitcounts, typical of AFL/AFL++. As usual, we'll need to iterate over all basic blocks of the CFG. As you probably know, the core idea is to assign a random id for each block and then encode an adge in the following way:

    shared_mem[cur_location ^ prev_location]++; 
    prev_location = cur_location >> 1;

Thus, this means that we have to access two variables in the runtime, namely, a variable for the shared memory between the fuzzer and the target (`__afl_are_ptr`) and a variable for the previous basic block (`__afl_prev_loc`). Thus, we'll start by creating two GlobalVariable objects:

```c
    GlobalVariable *AFLMapPtr = new GlobalVariable(M, Int8PTy, false, GlobalValue::ExternalLinkage, 0, "__afl_area_ptr");
    GlobalVariable *AFLPrevLoc = new GlobalVariable(M, Int32Ty, false, GlobalValue::ExternalLinkage, 0, "__afl_prev_loc", 0, GlobalVariable::GeneralDynamicTLSModel, 0, false);
```
Then for each basic block, we need to represent the pseudocode mentioned above. For the current location, we can just create a random number modulo the size of the map. Then we'll issue a Load instruction to read the value contained in `__afl_prev_loc`:

```c
            cur_loc = random() % map_size;
            Constant* CurLoc = ConstantInt::get(Int32Ty, cur_loc);

            LoadInst* LoadPrevLoc = IRB.CreateLoad(AFLPrevLoc);
            LoadPrevLoc->setMetadata(M.getMDKindID("nosanitize"), MDNode::get(*C, None));
            Value* PrevLoc = static_cast<Value*>(LoadPrevLoc);
```

At this point we can load the `__afl_are_ptr` at the index `cur_location ^ prev_location` and increase the counter by one:

```c
            LoadInst* MapPtr = IRB.CreateLoad(AFLMapPtr);
            MapPtr->setMetadata(M.getMDKindID("nosanitize"), MDNode::get(*C, None));

            Value* MapPtrIdx = IRB.CreateGEP(MapPtr, IRB.CreateXor(PrevLoc, CurLoc));

            LoadInst* NumberOfHits = IRB.CreateLoad(MapPtrIdx);
            NumberOfHits->setMetadata(M.getMDKindID("nosanitize"), MDNode::get(*C, None));

            Value* Add = IRB.CreateAdd(NumberOfHits, One);
```

Finally, we store the result of the addition at the same position inside the bitmap and we shift right the current location by 1:

```C
            StoreInst* UpdateHits = IRB.CreateStore(Add, MapPtrIdx);
            UpdateHits->setMetadata(M.getMDKindID("nosanitize"), MDNode::get(*C, None));

            StoreInst* UpdatePrevLoc = IRB.CreateStore(ConstantInt::get(Int32Ty, cur_loc >> 1), AFLPrevLoc);
            UpdatePrevLoc->setMetadata(M.getMDKindID("nosanitize"), MDNode::get(*C, None));

```

We're done! Now, we can think of many improvements. For instance, one thing that was introduced in AFL++ is to check that the hitcount never reaches 0 (otherwise this would result in a wrong coverage evaluation when the hitcount overflows). To achieve this, you can use the following code:

```C
            Constant* ZeroConst = ConstantInt::get(Int8Ty, 0);
            auto comparisonFlag = IRB.CreateICmpEQ(Add, ZeroConst);
            auto carry = IRB.CreateZExt(comparisonFlag, Int8Ty);
            Add = IRB.CreateAdd(Add, carry);
```

At this point there's no limit to the potential improvements that can introduce. You can use two shared maps, you can create different feedback mechanisms, introduce a more lightweight instrumentation policy, etc. Here, just to report an example, I implemented an additional feedback based on context-sensitivity, i.e., when we reach a program point, we don't just take into account the two basic blocks but also the context (function calls invoked so far essentially).
We assign a second random ID for each entry block and then this ID represents the context for that function. Thus, we update a variable to store the current value of the context (i.e., the xor of the previous function IDs).

```C
    LoadInst* PrevCtxLoad = IRB.CreateLoad(AFLContext);
    PrevCtx = static_cast<Value*>(PrevCtxLoad);
    PrevCtxLoad->setMetadata(M.getMDKindID("nosanitize"), MDNode::get(*C, None));

    unsigned cur_ctx = random() % MAP_SIZE;
    Value* CurCtx = ConstantInt::get(Int32Ty, cur_ctx);

    StoreInst* StoreCtx = IRB.CreateStore(IRB.CreateXor(PrevCtx, CurCtx), AFLContext);
    StoreCtx->setMetadata(M.getMDKindID("nosanitize"), MDNode::get(*C, None));
```

Now, if context-sensitivity is enabled, we need to include the context whenever we encode an edge in the shared memory. 
Thus, we can xor the `PrevCtx` variable with the `PrevLoc`:

```C
   PrevLoc = IRB.CreateZExt(IRB.CreateXor(PrevLoc, PrevCtx), Int32Ty);
```

Finally, we need some code to restore the context whenever we exit from the function, e.g., when we meet a `ReturnInst`:


```C
    if (isa<ReturnInst>(I)) {
        IRBuilder<> Restore_IRB(I);
        StoreInst* Restore = Restore_IRB.CreateStore(PrevCtx, AFLContext);

        Restore->setMetadata(M.getMDKindID("nosanitize"), MDNode::get(*C, None));
    }
```

Don't forget that at compile-time you'll have to link against the object file that contains the AFL++ runtime named `afl-compiler-rt.o` (see the `cc` compiler wrapper python script).
