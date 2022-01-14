---
layout: default
permalink: /posts/welcome
title:  "The Derby of Static Software Testing: Joern vs. CodeQl"
---

## The Derby of Static Software Testing: Joern vs CodeQl

Albeit I have to confess that my first temptation when looking for bugs in a source code file is to start to grep for `memcpy` or similar things, recently I had fun with two awesome tools for static software testing, namely, [Joern](https://joern.io/) and [CodeQl](https://codeql.github.com/). Both the tools share a similar phylosophy, that is, exporting an expressive domain-specific language to capture some code patterns and let the analyst interact with the code. In this quick blogpost, I will present some of the differences that I noticed when playing with the two tools, trying to give some indications and use-cases for each of the two.
I underline that this blogpost only focuses on C/C++ whereas the two static analysers function also on other languages (Java, Python, ..).



## Parsing the source code

Both the tools need to deploy a parsing stage to build their internal representation of the source code. However, to accomplish this, they make use of completely different strategies.

On the one hand, CodeQl requires to compile the project under test. During the actual building, the extractor accomplishes with several tasks. For instance, it constructs a representation of the relation between the source code files and then, for each compiler invocation, it retrieves the useful information such as syntactic data from the AST and semantic data (variables' name, types, ..).
These are stored in a CodeQl database for further processing and querying.

With a completely different perspective, Joern does not need to compile the codebase, but rather, it implements a fuzzy parsing approach that can extract the useful information even form incomplete projects. In this second case the authors of the tool opted for a different code representation, the Code Property Graph (CPG), a mix of AST, CFG and DDG, that under the hood consists of an in-memory graph database.

Now, these two different design choices have a direct impact on the performances and the usability of the tools.
Let's start with the performances. 
For small-to-medium projects, the two tools have similar performances.
For example, it took 22 seconds to generate the CodeQl database for `file`, whereas for Joern it took 19 seconds on a machine with 4 cpus and 16 GB of RAM.
However, for large projects the things change a lot. For instance, I tried to run the analyzers against the entire `wireshark` codebase and I measured a total of roughly 6 hours for CodeQl while Joern ran out-of-memory on the same machine.
Once I executed Joern on a different machine with much more RAM (128 GB) it could terminate, even though it took something around 24 hours.
That being said, honestly I have to point out that the actual time to get CodeQl parsing the source code is probably more, as the compilation step failed several times before I could find a correct setup in terms of dependencies, compiler, toolchain, etc.

The fact that CodeQl requires the compilation of the project inherently affects its usability as in general compile a project is not fun and it is very likely that it will take more time to guess the exact configuration to build it.
This hints that we could prefer Joern to analyse small-to-medium codebases as well as portions of larger ones.
For example, one may want to statically analyse only the authentication module of a huge project.
In this scenario Joern is the best and the only thing we have to do is launch it and generate the CPG to query over it.
But if our necessity is to consider the entire code of a program, then our first try should be more probably with CodeQl, at least in terms of parsing of the source code.

A particularly interesting use case instead, is to statically analyse decompiled code, i.e., the output of a decompiler.
I focused on this topic for my research paper entitled ``The Convergence of Source Code and Binary Vulnerability Discovery - A Case Study'' ( [link](https://www.s3.eurecom.fr/docs/asiaccs22_mantovani.pdf) ) and thus I will not go into details.
The only thing that I let you note is that in this specific case where the pseudocode typically cannot be recompiled, we had to manually fix the decompiler's output before compilation-based tools such as CodeQl could correctly execute.
Thus, in this scenario, my suggestion is that Joern can probably result in a better feasibility of the approach.

To conclude with this first part, I would say there is definitely no winner, even though according to the situation and the goal of the analysis one tool can do better than the other one.


## Queries

Let's now see some queries and compare how much the syntax is different depending on the two frameworks. I opted to include two case studies, a simple one and a more elaborated one, but for more examples of queries you can have a look at [my repo](https://github.com/elManto/StaticAnalysisQueries).

### Heap Buffer Overflow

The first example is a bug that was present in `libssh2` in 2012 (I learnt about this bug by reading the Joern paper). In the source code of the library we find the following statements:

```c
  uint32_t namelen = _libssh2_ntohu32(data + 9 + sizeof("exit-signal"));
  ..
  channelp->exit_signal = LIBSSH2_ALLOC(session, namelen + 1);
  ..
  memcpy(channelp->exit_signal, data + 13 + sizeof("exit_signal"), namelen);
```

Now, we have to keep two things in mind. First, `LIBSSH2_ALLOC` is a wrapper of the `malloc` function and therefore it responsible for the dynamic memory allocation. Second, the variable `namelen` is under control of the attacker and is defined as `uint32_t`. Thus, when a malicious user send data such as that `namelen + 1` results in an integer overflow, the final `memcpy` could produce a heap buffer overflow.

This super classical pattern, present in many CVE reports in different flavors, can be easily detected by both the static analysers.
An example of query for CodeQl looks like the following snippet:

```c
  from MacroInvocation mi, Macro alloc, AddExpr add
  where alloc.hasName(".*ALLOC*.")
         and mi.getMacro() = alloc
         and mi.getExpr().(ExprCall).getArgument(0) = add
  select mi
```

In the *from* statement, we declare what elements we need. In our case we need a *Macro* variable, to infer some properties of the macro definition, a *MacroInvocation* that represents the set of instancies of that macro when it is invoked in the code, and an *AddExpr* that collects all the addition expressions that we have in the source.
The next step, is to specify the constraints of our subject in the *where* statement. For this example, we want all macros that contain the string ``ALLOC'' in the name and that, once invoked, use an addition for the 0-th argument.
Finally, in the *select* statement, we just say what we want to display.

If we wanted to hunt the same bug with Joern, we should write something like:

```c
  cpg
    .call(".*ALLOC*.")
    .argument
    .isCallTo("<operator>.addition")
    .p
```

Here, we do not have variables as in the previous case, but rather, each field performs a walk on the edges of the CPG. Quickly, we have that the *.call* part retrieves all the calls to a certain method (in our case all methods containing ALLOC), the *.argument* field extracts the arguments of such calls and finally select those ones that represent an addition. The *.p* field just prints out the result.

The base idea behind the two rules is the same - catch the dynamic memory allocations whose argument is an addition. However, even if I kept the queries structure easy to reason about them, they present some differencies.
The first difference is that in CodeQl I had to explicitly refer the exact argument that we are interested in (*.getArgument(0)*) while Joern allows to refer generically to all arguments of a call.
The second interesting difference, is that Joern can refer to all the invocations in the CPG with the simple *call* field (thus including both macro and function invocations) while for CodeQl I had to specify the fact that we are interested in a macro.

Of course, both the queries can be improved to reduce the number of false positives and to extract more meaningful information. Even though the goal of the post is not a tutorial on CodeQl/Joern, I propose you a possible simple enhancement of the CodeQl rule, where we use the `DataFlow` library to identify the assignments whose variable is then used for the sum in the memory allocation.
This can give us more info about where the variable comes from, for instance, to understand if it is under the attacker's control.

```c
  from MacroInvocation mi, Macro alloc, AssignExpr src, AddExpr add
  where alloc.hasName("LIBSSH2_ALLOC")
         and mi.getMacro() = alloc
         and mi.getExpr().(ExprCall).getArgument(0) = add
         and  DataFlow::localFlow(DataFlow::exprNode(src),
              DataFlow::exprNode(mi.getExpr().(ExprCall).getArgument(0))
              )

  select src.getLocation(), mi
```

### Off-by-one heap buffer overflow

For our second use case, we will focus on a slightly more complicated example.

## Conclusions

Lorem Ipsum is simply dummy text of the printing and typesetting industry. Lorem Ipsum has been the industry's standard dummy text ever since the 1500s, when an unknown printer took a galley of type and scrambled it to make a type specimen book. It has survived not only five centuries, but also the leap into electronic typesetting, remaining essentially unchanged. It was popularised in the 1960s with the release of Letraset sheets containing Lorem Ipsum passages, and more recently with desktop publishing software like Aldus PageMaker including versions of Lorem Ipsum.
