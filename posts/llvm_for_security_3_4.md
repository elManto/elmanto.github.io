---
layout: default
permalink: /posts/llvm_for_security_3_4
title:  "LLVM Passes for Security (Part 3/4)"
---


# LLVM Passes for Security: A First Transformation Pass -- AddressSanitizer  (Part 3/4)


## Abstract

It's time to implement our first transformation pass. Among the several applications that we could consider, I decided to pick up a very famous one: AddressSanitizer. Obviously I simplified a lot to make the things easier to understand, but you may want to add additional features to practice and improve your skills. Interestingly, for this exercise, we'll need to modify the behavior of our compiled application at runtime, meaning that we'll have to modify the LLVM IR.

##### LLVM Concepts

- Add LLVM IR at compile time
- Use of a runtime
- Instrumentation of program points (function calls, instructions, ..)

### What we want to implement

AddressSanitizer is a memory errors detector that works essentially in two steps. At compile-time instrumentation is injected and at runtime the actual checks are performed for each type of memory access (stack, heap, ..). For our implementation we'll focus on heap-related vulnerabilities. In particular, our tool will be able to detect heap-based buffer overflows (or underflows) and use-after-free vulns. Also, for laziness, I didn't implement an instrumentation logic to catch overflow/use-after-free issues originated inside libc APIs like memcpy, strcpy, etc. The point is that we'll see how to instrument several points in the IR and after learning that, instrumenting other APIs will be just a matter of writing more code.

In the follow up I assume that you already have a basic understanding of how ASAN works.

### Our implementation

First, the pass essentially iterates over all instructions in the module and seeks for "interesting" points to inject our instrumentation. We've already seen how to iterate over functions, blocks and instructions. 

Let's start by hooking the entry and exit points of each function. This will be useful to store the context (i.e., the callstack) so that when we find a bug we can report it. Overall, when you want to invoke an external function inside the module, you need to declare and implement it externally in an object file that you'll compile together with the target program. For instance in our case we'll have a file named `runtime.c` that implements the following two functions:

```c
extern void __hook_entry(char* name) {

    push_call(name);
}


extern void __hook_exit(char* name) {

    pop_call();
}
```
Our goal is to let our pass modify the IR in such a way that it invokes these two hooks whenever we invoke/return from a function. Obviously the `push_call` and `pop_call` methods simply push/pop the method name from an internal data structure ( a stack) totally managed inside the runtime.c file.


In our pass, therefore at compile time, we need first a way to recover those symbols. Luckily there's an LLVM APIs that does this job -- `getOrInsertFunction()`. For instance, in our case we'll create two `FunctionCallee` variables and initialize them in this way:

```c
    hook_entry = M.getOrInsertFunction("__hook_entry", VoidTy, Int8PTy);
    hook_exit = M.getOrInsertFunction("__hook_exit", VoidTy, Int8PTy);
```

Note that the `getOrInsertFunction` method, takes as an additional input the return type (`VoidTy` because the two hooks are both void) and the input parameters type (in this case just a pointer to char, thus `Int8PTy`).

Note that we cannot modify the IR while iterating it -- this would put the iterator in an unconsistent state and let the compiler crash. Thus, we need to save our program points (in this case the entry points of each function and the return instructions) in two vectors.
After this, we can go on with the actual IR generation:

```c
            Function* ParentFunc = I->getFunction();
            IRBuilder<> IRB(I);
            vector<Value*> args;

            Constant* fcn_name = ConstantDataArray::getString(*C, ParentFunc->getName(), true);
            AllocaInst* Alloca = IRB.CreateAlloca(Int8PTy);
            Value* FcnNameVal = static_cast<Value*>(Alloca);
            IRB.CreateStore(fcn_name, FcnNameVal);

            args.push_back(FcnNameVal);
            CallInst* call = IRB.CreateCall(HookRtn, args);
            call->setMetadata(M.getMDKindID("nosanitize"), MDNode::get(*C, None));
            DEBUG(errs() << "Installed a new hoook point\n");

```

What we need here is an `IRBuilder` that let us to inject IR starting from a certain instruction. After instanciating it, we first create a `Store` to save the function name in a variable and then we create a call to the hook function, passing the arguments that we want to pass.

Next, we want to instrument all memory accesses, thus the `Load` and the `Store`. The procedure here is similar -- the only difference is that we want to pass two parameter, the size of the memory access and its address:

```c
    void InstrumentMemoryAccesses(vector<Instruction*>& Accesses, Module& M, FunctionCallee SanitizerFunction) {

        for (Instruction* Access : Accesses) {
            vector<Value*> args;
            Value* memoryPointer = nullptr;
            if (StoreInst* ST = dyn_cast<StoreInst>(Access))
                memoryPointer = ST->getPointerOperand();
            else if (LoadInst* LO = dyn_cast<LoadInst>(Access))
                memoryPointer = LO->getPointerOperand();


            PointerType* pointerType = cast<PointerType>(memoryPointer->getType());
            uint64_t storeSize = Layout->getTypeStoreSize(pointerType->getPointerElementType());
            Constant* Size = ConstantInt::get(pointerType->getPointerElementType(), storeSize);

            IRBuilder<> IRB(Access);
            args.push_back(memoryPointer);
            args.push_back(Size);
            CallInst* hook = IRB.CreateCall(SanitizerFunction, args);
            hook->setMetadata(M.getMDKindID("nosanitize"), MDNode::get(*C, None));
        }
    }

```

Lastly, we replace all the `malloc` and `free` invocations with our hooks `__hook_malloc` and `__hook_free`. In this case, as the signature of the two functions is exactly the same, we just need to invoke the LLVM method `Call->setCalledFunction()`.


At this point we can start writing our runtime. Now, the LLVM pass actually is finished and the runtime actually doesn't introduce any new LLVM concepts, therefore my description here won't go in details but you will have the source code anyway.

Essentially, the malloc hook (`allocator.c`) allocates more memory than the amount requested by the target program. This is needed to allocate the left/right redzones in the shadow memory. The instrumented malloc is responsible for unpoisoning the user data (accesses are allowed) and for poisoning the right/left redzones (accesses abort). Moreover, there's some logic to manage the alignment in a proper way.

```c
void* __learnsan_malloc(size_t size) {
    struct chunk_begin* p = malloc(sizeof(struct chunk_struct) + size);
    if (!p)
        return NULL;

    learnsan_unpoison(p, sizeof(struct chunk_struct) + size);

    p->requested_size = size;
    p->aligned_orig = NULL;
    p->next = p->prev = NULL;



    learnsan_poison(p->redzone, REDZONE_SIZE, ASAN_HEAP_LEFT_RZ);
    if (size & (ALLOC_ALIGN_SIZE - 1))
        learnsan_poison((char*)&p[1] + size, (size & ~(ALLOC_ALIGN_SIZE - 1) + 8 - size + REDZONE_SIZE),
                        ASAN_HEAP_RIGHT_RZ);
    else
        learnsan_poison((char*)&p[1] + size, REDZONE_SIZE, ASAN_HEAP_RIGHT_RZ);

    memset(&p[1], 0xff, size);

    return &p[1];
}
```

At the other side of the spectrum, our custom free (`allocator.c`) simply releases the memory and poison it, to indicate that future accesses will produce a use-after-free error.

```c
void __learnsan_free(void* ptr) {
    if (!ptr)
        return;

    struct chunk_begin* p = ptr;
    p -= 1;

    size_t n = p->requested_size;
    if (p->aligned_orig)
        free(p->aligned_orig);
    else
        free(p);

    if (n & (ALLOC_ALIGN_SIZE - 1))
        n = (n & !(ALLOC_ALIGN_SIZE - 1)) + ALLOC_ALIGN_SIZE;

    learnsan_poison(ptr, n, ASAN_HEAP_FREED);
}
```

The store/load hooks map the accessed address to the shadow memory and check if the memory operation is valid or not (i.e., it touches a redzone). If yes, they print the stack trace and trigger the abort (and its naive message).
The memory is encoded at the same way of AddressSanitizer (8 bytes of application memory map to 1 byte of shadow memory). Thus, this reflects what you can find in the AddressSanitizer paper. For instance, to check a load of 8 bytes:

```c
int learnsan_load8(void* ptr) {
    uintptr_t mem = (uintptr_t)ptr;
    uint8_t* shadow_addr = (uint8_t*) MEM_TO_SHADOW(mem);
    return *shadow_addr != 0;
}
```
Where `MEM_TO_SHADOW` is a macro that extends to:
    #define MEM_TO_SHADOW(mem) (((mem) >> SHADOW_SCALE) + (SHADOW_OFFSET))



Finally, consider that when compiling your target application with our custom sanitizer, you'll have to link it with our object files for the runtime. In the Makefile I added the command `ld -r` to merge the object files into a single relocatable file. 


