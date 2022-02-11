---
layout: default
permalink: /posts/welcome
title:  "The Derby of Static Software Testing: Joern vs. CodeQl"
---

# The Derby of Static Software Testing: Joern vs CodeQl

Albeit I have to confess that my first temptation when looking for bugs in a source code file is to start to grep for `memcpy` or similar things, recently I had fun with two excellent tools for static software testing, namely, [Joern](https://joern.io/) and [CodeQl](https://codeql.github.com/). Both the tools share a similar philosophy, that is, exporting an expressive domain-specific language to capture some code patterns and let the analyst interact with the code. In this quick blog post, I will present some of the differences that I noticed when playing with the two tools, trying to give some indications and use-cases for each.


### Scope

Before starting with the actual post, I would like to underline three things.
1. Although both the tools can analyse several languages (Python, Java, ..), I focused my evaluation on C code
2. More importantly, this comparison wants to evaluate these two awesome tools from the usability point of view rather than the detection capabilities. This second purpose indeed is a bit more tricky to compare, as one can almost always think about a more generic/precise query to catch a specific bug with both the tools, and thus the detection is mostly dependent on the ``skill'' of the user
3. Also, this post is not a primer on Joern/CodeQl. I will try to explain all concepts I introduce, but a previous minimal experience on the topic would make the reading easier


### Parsing the source code

Both the tools need to deploy a parsing stage to build their internal representation of the source code. However, to accomplish this, they make use of completely different strategies.

On the one hand, CodeQl requires compiling the project under test. During the actual building, the extractor accomplishes several tasks. For instance, it constructs a representation of the relation between the source code files. Then, for each compiler invocation, it retrieves useful information such as syntactic data from the AST and semantic data (variables' name, types, ..).
These are stored in a CodeQl database for further processing and querying.

With a completely different perspective, Joern does not need to compile the codebase, but rather, it implements a fuzzy parsing approach that can fetch source-level information even for incomplete projects. Moreover, the tool's authors opted for a different code representation, the Code Property Graph (CPG), a mix of AST, CFG, and DDG, that under the hood consists of an in-memory graph database.

Now, these two different design choices have a direct impact on the performances and the usability of the tools.
Let's start with the performances. 
For small-to-medium projects, the two tools behave quite similarly.
For example, it took to me 22 seconds to generate the CodeQl database for the `file` project, whereas Joern operated the parsing in 19 seconds on a machine with 4 CPUs and 16 GB of RAM.
However, for large projects, things change a lot. For instance, I tried to run the analyzers against the entire `wireshark` codebase and I measured a total of roughly 43 minutes for CodeQl while Joern ran out-of-memory on the same machine.
Once I executed Joern on a different machine with much more RAM (128 GB), it could terminate, even though it took something around 6 hours.
That being said, honestly, I have to point out that the actual time to get CodeQl parsing the source code is probably more, as the compilation step failed several times before I could find a correct setup in terms of dependencies, compiler, toolchain, etc. and this is something you should care especially if compiling the code you are working on is painful (and in my experience, usually it is).

The fact that CodeQl requires the compilation of the project inherently affects its usability as in general, building a project is not fun and it will likely take more time to guess the exact configuration to build it.
This hints that we could prefer Joern to analyse small-to-medium codebases as well as portions of larger ones.
For example, one may want to analyse only the authentication module of a huge project statically.
In this scenario, Joern is the best, and the only thing we have to do is launch it and generate the CPG to query over it.
But if we need to consider the entire codebase of a program, then our first try should be more probably with CodeQl, at least in terms of parsing of the source code.

A particularly interesting use case instead, is to statically analyse decompiled code, i.e., the output of a decompiler, that can provide a way to statically test a binary in the absence of its source code.
I focused on this topic for my research paper entitled ``The Convergence of Source Code and Binary Vulnerability Discovery - A Case Study'' ( [link](https://www.s3.eurecom.fr/docs/asiaccs22_mantovani.pdf) ) and thus I will not go into details.
The only thing that I let you note is that in this specific case where the pseudocode typically cannot be recompiled, we had to manually fix the decompiler's output before compilation-based tools such as CodeQl could correctly execute.
Thus, in this scenario, my suggestion is that Joern can probably result in better doability of the approach.

To conclude with this first part, I would say there is definitely no winner, even though according to the situation and the goal of the analysis, one tool can do better than the other one.


### Queries

Let's now see some queries and compare how much the syntax is different depending on the two frameworks. I opted to include two case studies, a simple one and a more elaborated one, but for more examples of queries you can have a look at [my repo](https://github.com/elManto/StaticAnalysisQueries).

#### Integer overflow causing heap buffer overflow

The first example is a bug that was present in `libssh2` in 2012 (I learnt about this bug by reading the Joern paper). In the source code of the library we find the following statements:

```c
  uint32_t namelen = _libssh2_ntohu32(data + 9 + sizeof("exit-signal"));
  ..
  channelp->exit_signal = LIBSSH2_ALLOC(session, namelen + 1);
  ..
  memcpy(channelp->exit_signal, data + 13 + sizeof("exit_signal"), namelen);
```

Now, we have to keep two things in mind. First, `LIBSSH2_ALLOC` is a wrapper of the `malloc` function and therefore it responsible for the dynamic memory allocation. Second, the variable `namelen` is under control of the attacker and is defined as `uint32_t`. Thus, when a malicious user send data such as that `namelen + 1` results in an integer overflow, the final `memcpy` could produce a heap buffer overflow.

This super-classical vulnerable pattern can be easily detected by both the static analysers.
An example of query for CodeQl looks like the following snippet:

```c
  from MacroInvocation mi, Macro alloc, AddExpr add
  where alloc.hasName(".*ALLOC*.")
         and mi.getMacro() = alloc
         and mi.getExpr().(ExprCall).getArgument(0) = add
  select mi
```

In the *from* statement, we declare what elements we need. In our case we need a *Macro* variable, to infer some properties of the macro definition, a *MacroInvocation* that represents the set of instances of that macro when it is invoked along the code, and an *AddExpr* that collects all the addition expressions that we have in the source (sum of two integers, sum of two variables, ..).
The next step, is to specify the constraints of our subject in the *where* statement. For this example, we want all macros that contain the string ``ALLOC'' in the name and that, once invoked, use an addition as the 0-th argument (i.e., the first argument).
Finally, in the *select* statement, we just say what we want to display.

If we wanted to hunt the same bug with Joern, we should write something like:

```c
  cpg
    .call(".*ALLOC*.")
    .argument
    .isCallTo("<operator>.addition")
    .p
```

Here, we do not have variables as in the previous case, but rather, each field performs a walk on the edges of the CPG. Quickly, we have that the *.call* part retrieves all the calls to a certain method (in our case all methods containing ``ALLOC''), the *.argument* field retrieves the arguments of such calls and finally select those ones that represent an addition. The *.p* field just prints out the result.

The base idea behind the two rules is the same - catch the dynamic memory allocations whose argument is an addition. However, even if I kept the queries structure easy to reason about them, they present some differences.
The first difference is that in CodeQl I had to explicitly refer the exact argument that we are interested in (*.getArgument(0)*) while Joern allows to refer generically to all arguments of a call.
The second interesting difference, is that Joern can refer to all the invocations in the CPG with the simple *call* field (thus including both macro and function invocations) while for CodeQl I had to specify the fact that we are interested in a macro.
Thus, looking at this scenario we learn that from the syntax point of view, Joern is probably a bit more generic than CodeQl.

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

#### Off-by-one heap buffer overflow

For our second use case, we will focus on a slightly more complicated example. Here the bug is an off-by-one overflow that was present in `wireshark` in 2018.
In the function `de_sub_addr`, we can find a dynamic memory allocation and the result buffer is then passed to the function `IA5_7BIT_decode`:

```c
  *extracted_address = (gchar *)wmem_alloc(wmem_packet_scope(), ia5_string_len);
  ..
  IA5_7BIT_decode(*extracted_address, ia5_string, ia5_string_len);
  ..
```

The `IA5_7BIT_decode` looks like the following:

```c
  void IA5_7BIT_decode(unsigned char * dest, const unsigned char* src, int len)
  {
      int i, j;
      gunichar buf;
  
      for (i = 0, j = 0; j < len;  j++) {
          buf = char_def_ia5_alphabet_decode(src[j]);
          i += g_unichar_to_utf8(buf,&(dest[i]));
      }
      dest[i]=0;
      return;
  }
```

As you can see, after iterating for `len` times, the variable `i` evaluates exactly `len`, which in our case, as invoked in the previous snippet of code, is `ia5_string_len`. However, the addressable space in the buffer `*extracted_address` starts at 0 and ends at `ia5_string_len - 1`. Therefore, the assignment `dest[i] = 0` causes an overflow of one single byte at the index `*extracted_address[ia5_string_len]`.

There are two possible ways to write a rule for this bug. The first standard one, is to find all the data flows that have the dynamically allocated buffer as the source and an index access to that buffer as the sink. However, the issue with this strategy is that computing the dataflow in an interprocedural context can be very expensive, and indeed this strategy took roughly 4 hours with CodeQl and did not terminate with Joern, in addition to the high false positives that it produced.
While in many cases there could be only one way to model a bug, for this situation I thought about a different perspective to reduce the bug to an intraprocedural one, that on average, is way easier to detect.
Indeed, a possible idea here, is to consider that the overflowing memory access uses a variable that is initialized in the header of a for loop, but is used outside the loop.
Thus, I designed a rule that individuates the variables used as counters within the loop and then identifies all times such variables are used to access a buffer outside the loop.
Although I was expecting some false positives, this allowed me to do some tests in a more reasonable time, because in this way we can focus separately on each function instead of considering chains of multiple functions.

Here, I show the query written for CodeQl:

```c
  from AssignExpr e, ForStmt f, ArrayExpr ae, Expr src, Variable v
  where e.getEnclosingStmt() = f.getInitialization()
        and v.getAnAssignedValue() = e.getRValue()

        and TaintTracking::localTaint(
            DataFlow::exprNode(v.getAnAssignedValue()),
            DataFlow::exprNode(ae.getArrayOffset()))

        and not ae.getBasicBlock().inLoop()
  select ae, v
```

The idea here is that we compute the dataflow between the counter variable, that we isolate with the two statements *e.getEnclosingStmt() = f.getInitialization() && v.getAnAssignedValue() = e.getRValue()*, and the sink that in our case is the index used for the array access (in the snippet, *ae.getArrayOffset()*). To conclude, we declare that the array access must be outside the loop (*not ae.getBasicBlock().inLoop()*).


Here instead, the following snippet represents the script I used for Joern:

```c

  var loop_array_access = cpg.method.controlStructure.parserTypeName("ForStatement")
      .expressionDown.ast.isCall.filter(x => x.name.equals("<operator>.indirectIndexAccess"))
      .toSet
  val array_access = cpg.call("<operator>.indirectIndexAccess")
      .where(x => x.argument(2).isIdentifier).toSet
  var all_vars = cpg.method.controlStructure.parserTypeName("ForStatement")
      .expressionDown.ast.isIdentifier.toSet
  val buffer_iterator_vars = cpg.call("<operator>.indirectIndexAccess").argument(2)
      .isIdentifier.toSet
  val real_vars = all_vars.intersect(buffer_iterator_vars).toList
  array_access.diff(loop_array_access).asInstanceOf[Set[Call]]
      .foreach(elem => println(elem.code, elem.reachableBy(real_vars).code.p))

```

Because of the inner structure of the CPG, I had to express the same concept in a different way. This time I could not create a one-line and I relied on some intermediate variables to create the rule.
1. `loop_array_access` contains all accesses to a buffer within a for loop
2. `array_access` is the set of the buffer accesses by means of an index in the whole program (e.g., `buf` in `buf[i]`)
3. `all_vars` represents all variables defined in a for loop in the entire program
4. `buffer_iterator_vars` are all variables used to perform index a buffer (e.g., `i` in `buf[i]`)
5. `real_vars` are the actual variables that perform a buffer access within a loop and we obtain them with the intersection between `all_vars` and `buffer_iterator_vars`
6. finally we compute the difference between the set of all buffer accesses (`array_access`) and the ones that are performed inside a loop (`loop_array_access`) and for each access in the diff we check if it is reachable by one of the variables computed at the previous step

As you can see, in this case the two models differ a lot and require a different reasoning. 
From my personal point of view, I found the syntax of CodeQl easier, in addition to the fact that performance-wise Joern took more time to complete the processing.

### Conclusions

In this comparison, I wanted to share some of the aspects I faced during my recent work based on Joern/CodeQl. I believe there is no an actual winner of the game in the end, but depending on the situation we can prefer one or the other tool. For instance, Joern can be probably a bit more generic in its syntax, and faster when we want to look for vulns in medium-size codebases, whereas CodeQl is more suitable in contexts where the project size is huge and it exports an easier syntax also for non-trivial bugs.
Finally, both the tools are subject of continuous improvements and we can imagine that future updates will make them always more performant despite the fact that they are already two mature frameworks.
