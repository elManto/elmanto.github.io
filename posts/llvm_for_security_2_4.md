---
layout: default
permalink: /posts/llvm_for_security_2_4
title:  "LLVM Passes for Security (Part 2/4)"
---


# LLVM Passes for Security: A Static Analysis Pass -- Anderson Pointer analysis  (Part 2/4)


## Abstract

Let's start with our first hands-on exercise! In this module, we begin with a super classical use case of LLVM for program analysis and we are going to implement a naive pointer analysis. Obviously the goal is not to implement a competitive analysis compared to state-of-the-art tools, but rather to move the first steps and develop your first static analysis pass. Thus, our pass will have all defects that a beginner-level analysis pass should have, i.e., flow-insensitive and context-insensitive. Note that we are not going to change the code but only to analyse it -- code transformations instead will be the topic of the next two lessons.

##### LLVM Concepts

- Iterate over the main elements of a program, i.e., Functions, Basic Blocks, Instructions
- Use of some of the typical LLVM APIs for these objects (Obtain the type info, other data, ..)
- Visit of some of the core instructions of the LLVM virtual instruction set

### What we want to implement

A classical program analysis problem that we may face is to understand the set of pointers that point to a certain set of variables. Being precise in this activity is of great interest in security and compilers but at the same time the complexity of the analysis is not trivial for low-level languages like C and C++. A huge amount of material is present w.r.t. this topic, and thus I don't go in details on this (if you're interested you can read 1,2,3,4). What you need to understand is that this problem is so complicated in C-like languages because when performing pointer arithmetic, we're allowed to perform arbitrary computations on the pointers.


A popular algorithm proposed in 1994 is the Andersen approach, a.k.a. inclusion based approach. In its essence, it consists of the following major steps:

- We go through the code looking for statements such as `x = y`
- We create a graph where the nodes are the variables and the edge expresses a constraint (for instance `addressof`, `copy`, ..)
- We visit each node and we update the points-to sets by solving the constraints. For example, two nodes `x` and `y` linked by a `copy` constraint will include the `y`'s points-to set in the `x` points-to set


As already stated, this just has educational purposes. Thus, we don't care about:

- field-sensitivity
- array-sensitivity
- context-sensitivity 
- flow-sensitivity

IMHO, these are all fundamental features that we would expect from a production tool, but I believe they would create too much noise for people moving the first steps in LLVM (moreover 1) I don't have infinite time and 2) state-of-the-art already comes with proper solutions).
 
### Our implementation

Before beginning, code is available at this [repo](https://github.com/elManto/LLVMPassesForSecurity) along with the Makefiles, scripts, ..

In the first part of our pass, we deal with the collection of the constraints. This happens by looking at several values present in the LLVM IR such as the instruction operands and the function arguments. The first thing that we want to do is the collect the return nodes for all functions that return a pointer type. Thus we need to iterate all functions with a classical LLVM construct:

```c
void AddFunctionReturnNodes(Module &M) {
    for (Function &F: M) {
        if (F.isIntrinsic() || F.isDeclaration())
            continue;
        if (F.getType()->isPointerTy()) {
            Value* Ret = static_cast<Value*>(&F);
            NF.createRetNode(Ret);
        }
    }
}

```
For each interesting node (i.e., when we return a pointer), the `NF` object of type `NodeFactory` let us take note of the type of node (`Ret`, `Alloc`, or `Pointer`), assign it a unique index and store its related info.

The following step of the constarints collection is to go through each single IR instruction:

```c
            for (BasicBlock &BB : F) {
                for (Instruction &I : BB) {
                    AddInstructionConstraints(I);
                }
```

At this point we need to choose the type of constraint depending on the instruction type. Thus, we check the type of each instruction, looking for memory-related operations or other instructions that may create a pointer variable (such as calls). When you need to check the type of a certain instruction, the best way is to use the `isa<>()` method. For instance, to check if instruction `I` is an `AllocaInst` you can write:

    if (isa<AllocaInst>(I)) {...}

More specifically for our case, we'll perform this check for the following operations (coupled with the correponsing constraint type): `AllocaInst` (AddressOf), `LoadInst` (Load), `StoreInst` (Store), `PHINode` (Copy), `CallInst` (Copy), `ReturnInst` (Copy) and `GetElementPtrInst` (Copy). 
Whenever we find an instruction that holds a constraint, we obtain the indexes representing for the source and destination nodes thanks to the `NodeFactory` class, and then we store the edge connecting the two (along with the type of edge, i.e., the type of constraint, in the `AllConstraints` object.

Thus, our code looks like (just for the first four instruction types):

```c
        if (isa<AllocaInst>(&I)) {
            if(!I.getType()->isPointerTy())
                return;
            int srcIdx = NF.createAllocSiteNode(&I);
            int destIdx = NF.getPointerNode(&I);
            AllConstraints.emplace_back(destIdx, srcIdx, AddressOf);
        }
        else if (isa<LoadInst>(&I)) {
            if (!I.getType()->isPointerTy())
                return;
            int srcIdx = NF.getPointerNode(I.getOperand(0));
            int destIdx = NF.getPointerNode(&I);
            AllConstraints.emplace_back(destIdx, srcIdx, Load);
        }
        else if (isa<StoreInst>(&I)) {
            if(I.getOperand(0)->getType()->isPointerTy()) {
                int srcIdx = NF.getPointerNode(I.getOperand(0));
                int destIdx = NF.getPointerNode(I.getOperand(1));
                AllConstraints.emplace_back(destIdx, srcIdx, Store);
            }
        }
        else if (isa<PHINode>(&I)) {
            if (!I.getType()->isPointerTy())
                return;
            PHINode* phi = static_cast<PHINode*>(&I);
            int destIdx = NF.getPointerNode(&I);
            for (unsigned i = 0; i < phi->getNumIncomingValues(); i++) {
                int srcIdx = NF.getPointerNode(phi->getIncomingValue(i));
                AllConstraints.emplace_back(destIdx, srcIdx, Copy);
            }
        }
        else if () { ... }
        ...
```


Additional code is needed to add constraints for the arguments. Very often LLVM classes provide iterator interfaces. For instance in this case, we can iterate arguments with an `arg_iterator`:

```c
    void AddArgConstraints(CallBase* CB, Function* F) {
        auto argumentIterator = CB->arg_begin();
        auto parameterIterator = F->arg_begin();

        while (argumentIterator != CB->arg_end() && parameterIterator != F->arg_end()) {
            Value* argument = *argumentIterator;
            Value* parameter = &*parameterIterator;
            if (argument->getType()->isPointerTy() && parameter->getType()->isPointerTy()) {
                int destIdx = NF.getPointerNode(parameter);
                int srcIdx = NF.getPointerNode(argument);
                AllConstraints.emplace_back(destIdx, srcIdx, Copy);
            }
            argumentIterator++;
            parameterIterator++;
        }
    }
```

This is the core of the LLVM code that we need to write for our pass. The rest of the code mainly implements the graph logic and its visit to solve the constraints and to extract the points-to set. Let's quickly see that, even though it does not add anything in terms of LLVM knowledge.

Essentially, at this point, we have all constraints grouped in the `AllConstraints` object. We need to use this vector to initialize a graph, that we represent as a vector whose item type is `PointsToNode`, a class defined in `Solver.h` that store info about successors (`Copy`), loads (`Load`), stores (`Store`) and augments the points-to set (our final result) in case of `AddressOf` constraint, that I recall to be related to the actual allocation site of a pointer (thus `AllocaInst` or `malloc`)..
Also, in case of `AddressOf` we initialize our worklist (i.e., we start visiting the graph from the actual allocation sites).

The following method performs the actual graph visit, updating the points-to sets. Essentially, whenever it reaches a new node it does:
1. propagate the points-to set for successor nodes (i.e., copies)
2. add new edges for store/loads to or from the pointee

```c
void AndersonGraph::solve() {
    while (!workList.empty()) {
        int idx = workList.front();
        workList.pop();
        PointsToNode node = graph[idx];
        for (int successor : node.getSuccessors())
            propagate(successor, node.getPtsSet());

        for (int pointee : node.getPtsSet()) {
            for (int load : node.getLoads())
                insertEdge(pointee, load);
            for (int store : node.getStores())
                insertEdge(store, pointee);
        }

    }
```

