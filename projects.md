---
layout: default
permalink: /projects
title: "Projects and Publications"
---

Here, I report a list of accepted papers and active projects that I worked on, so far. The list cannot be exhaustive as I cannot make all papers/projects public for several reasons (most typically, the paper is under submission). However, I will try to add the new entries as soon as possible.

# Publications

### ``Dissecting American Fuzzy Lop - A FuzzBench Evaluation'', A. Fioraldi, A. Mantovani, D. Maier, D. Balzarotti, FUZZING workshop 22

A careful evaluation of several aspects that are at the base of the success of AFL. Experiments include measurements about different hitcounting strategies, different mutations, timeouts, scores, etc.

**keywords and tools**: AFL, Google Fuzzbench, LLVM

### ``Fuzzing with Data Dependency Information'', A. Mantovani, A. Fioraldi, D. Balzarotti, EuroS&P 22

A novel approach where we investigate how stressing combinations of def-use pairs affects feedback-driven fuzzing on 38 real-world target programs. Our methodology relies on an LLVM-based instrumentation that augments the edge-coverage by introducing carefully selected data-dependency edges.

**keywords and tools**: AFL++, LLVM, crash triage, afl-cov, context-sensitivity, ngram coverage, data-dependency graph, Google Fuzzbench, 0-day bugs

### ``The Convergence of Source Code and Binary Vulnerability Discovery - A Case Study'', A. Mantovani, L. Compagna, Y. Shoshitaishvili, D. Balzarotti, Asia CCS 22

We experimented a new vulnerability approach where we lift a binary by means of a decompiler, and then we analyse the decompiled code with off-the-shelf static analysers.
This technique can be used when the source code is not available, for instance in case of firmware and allows to discover a huge set of vulnerabilities that would be difficult to capture just relying on assembly code.

**Keywords and tools**: Ida, Ghidra, Static Software Testing, Implementation of analysers with Joern and Code-QL, SAST techniques

--------

### ``RE-Mind: a First Look Inside the Mind of a Reverse Engineer'', A. Mantovani, S. Aonzo, Y. Fratantonio, D. Balzarotti, Usenix 22

For this work, I implemented a system to statically reverse engineer x64 binaries and I used it to collect information about how humans tipically perform reverse engineering with the goal of capturing the core aspects and translate these behaviors into automated reverse engineering techniques.


**Keywords and tools**: IDA, radare2, Linux reversing.


--------


### ``Does Every Second Count? Time-based Evolution of Malware Behavior in Sandboxes'',A. Kuchler, A. Mantovani, Y. Han, L. Bilge, D. Balzarotti, NDSS 21


My second malware analysis project instead performs a dynamic analysis on a large dataset of malware to understand their behavior inside the sandboxes, especially from the point of view of the timing.

**Keywords and tools**: IDA, Reversing, Windows, QEMU, dynamic analysis, binary analysis, PANDA framework, Machine Learning.



--------

### ``Prevalence and Impact of Low-Entropy Packing Schemes in the Malware Ecosystem'', A. Mantovani, S. Aonzo, A. Merlo, X. Ugarte-Pedrero, D. Balzarotti, NDSS 20


For this project, I developed some manual and then automated reverse engineering approaches to understand if a malware is encrypted and which decryption scheme it uses.
Our analysis pipeline includes a dynamic analyser, built on top of QEMU, a static analysis component, adopting common off-the-shelf tools such as PEid/DIEC and finally three Machine Learning classifiers, namely, SVM, RF and Neural Networks.
The paper shows modern packing schemes and how several malicious files use them to bypass common Antivirus detection mechanisms.

**Keywords and tools**:IDA, Reversing, Windows, QEMU, dynamic analysis, binary analysis, PANDA framework, Machine Learning.



--------


# Projects


### DARPA CHESS project 

A project that aims at improving automated vulnerability approaches thanks to human enhancement ([link](https://www.darpa.mil/program/computers-and-humans-exploring-software-security)).

Main tools and contributions:
+ The first tool consists of a system that helps a fuzzer to bypass complicated checks in the binary code (i.e., source code is not available). First, a gdb script recovers a set of points where the execution of the binary becomes stuck (crc checksums, magic bytes). Then the patching module performs some binary re-writing to modify the control flow of the program according to some policies that can be specified as an input by the analyst.

+ As a second contribution I wrote an LLVM pass that recovers the dynamic invariants of a program and maps them to the source code, thanks to the use of the DWARF format. This is useful for both making the code review easier and implementing directed fuzzing approaches (as I did).

**Keywords and tools**: gdb, reversing, elf patching, lief, llvm, program analysis, fuzzing



