---
layout: default
permalink: /posts/welcome
title:  "The Derby of Static Software Testing: Joern vs. CodeQl"
---

## Summary

Albeit I have to confess that my first temptation when looking for bugs in a source code file is to start to grep for `memcpy` or similar things, recently I had fun with two awesome tools for static software testing, namely, [Joern](https://joern.io/) and [CodeQl](https://codeql.github.com/). Both the tools share a similar phylosophy, that is, exporting an expressive domain-specific language to capture some code patterns and let the analyst interact with the code. In this quick blogpost, I will present some of the differences that I noticed when playing with the two tools, trying to give some indications and use-cases for each of the two.
I underline that this blogpost only focuses on C/C++ whereas the two static analysers function also on other languages (Java, Python, ..).



## Parsing the source code

Albeit both the tools need to deploy a parsing stage to build their internal representation of the source code, they make use of completely different paradigms.

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





## Queries

Lorem Ipsum is simply dummy text of the printing and typesetting industry. Lorem Ipsum has been the industry's standard dummy text ever since the 1500s, when an unknown printer took a galley of type and scrambled it to make a type specimen book. It has survived not only five centuries, but also the leap into electronic typesetting, remaining essentially unchanged. It was popularised in the 1960s with the release of Letraset sheets containing Lorem Ipsum passages, and more recently with desktop publishing software like Aldus PageMaker including versions of Lorem Ipsum.

## Conclusions

Lorem Ipsum is simply dummy text of the printing and typesetting industry. Lorem Ipsum has been the industry's standard dummy text ever since the 1500s, when an unknown printer took a galley of type and scrambled it to make a type specimen book. It has survived not only five centuries, but also the leap into electronic typesetting, remaining essentially unchanged. It was popularised in the 1960s with the release of Letraset sheets containing Lorem Ipsum passages, and more recently with desktop publishing software like Aldus PageMaker including versions of Lorem Ipsum.
