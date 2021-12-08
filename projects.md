---
layout: default
permalink: /projects
title: "Projects and Publications"
---


# Publications

### ``The Convergence of Source Code and Binary Vulnerability Discovery - A Case Study'', A. Mantovani, L. Compagna, Y. Shoshitaishvili, D. Balzarotti, Asia CCS 22

We experimented a new vulnerability approach where we lift a binary by means of a decompiler, and then we analyse the decompiled code with off-the-shelf static analysers.
This technique can be used when the source code is not available, for instance in case of firmware and allows to discover a huge set of vulnerabilities that would be difficult to capture just relying on assembly code.

**Keywords and tools**: Ida, Ghidra, Static Software Testing, Implementation of analysers with Joern and Code-QL, SAST techniques


# Projects


### DARPA CHESS project 

A project that aims at improving automated vulnerability
approaches thanks to human enhancement (\url{https://www.darpa.mil/program/computers-and-humans-exploring-software-security}).

Main tools and contributions:
+ The first tool consists of a system that helps a fuzzer to bypass complicated checks in the binary code (i.e., source code is not available). First, a gdb script recovers a set of points where the execution of the binary becomes stuck (crc checksums, magic bytes). Then the patching module performs some binary re-writing to modify the control flow of the program according to some policies that can be specified as an input by the analyst.

+ As a second contribution I wrote an LLVM pass that recovers the dynamic invariants of a program and maps them to the source code, thanks to the use of the DWARF format. This is useful for both making the code review easier and implementing directed fuzzing approaches (as I did).

**Keywords and tools**: gdb, reversing, elf patching, lief, llvm, program analysis, fuzzing



