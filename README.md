# Optimization of the Harmony Model Checker and Compiler using Static Analysis

## Proposal

**What will you do?**

We'll be optimizing the model checker, and consequentially the compiler, for the [Harmony Concurrent Programming Language](https://github.com/harmonylang/harmony). In particular, Harmony does not do much in terms of compiler optimizations (although it does evaluate constant folding for simple expressions in the compiler rather than the model checker.) Due to so much heavy lifting from the model checker, state space explosion is unfortunately relatively common in Harmony. 

By using domain-specific static analysis information from concurrent programs (such as field accesses), we can create a _bridge_ between the compiler's initial output and the model checker's Kripke structure instance to provide beneficial static information which can reduce the state space, i.e. minimize the Kripke structure. This will allow for the verification of more sophisticated Harmony programs, without the need for better hardware. Particularly beneficial for Harmony, as its main use case is pedagogical concurrent programming, which can help students explore larger programs with their limited local resources.

**How will you do it?**

To begin with, there are a couple of resources we have to familiarize ourselves with. Thankfully, all resources regarding the [Harmony model checker and compiler](https://harmony.cs.cornell.edu/docs/textbook/howitworks/#model-checker) are open-source and well-documented. This will help us understand Harmony's architecture and locations in which we can carry static metadata from the front end to the back end (and eventually the model checker) without incurring additional compilation overhead. 

For understanding how the domain-specific model checker works, the [Handbook of Model Checking](https://link.springer.com/chapter/10.1007/978-3-319-10575-8_18) has a great chapter on Model Checking for concurrent programs (and it's very concise with just about ~20 relevant pages). An obvious question still lies, **what static information can we make use of in concurrent programs?** Thankfully, yet again, this research question has been answered in a 2014 paper on [Model checking of concurrent programs with static analysis of
field accesses](https://www.sciencedirect.com/science/article/pii/S016764231400478X?ref=cra_js_challenge&fr=RR-1). 

The main algorithm statically detects for each point in the code of each thread the set of fields that may be accessed after that point. A more sophisticated result is called _immutable field analysis_ (IFA), in which they eliminate additional unnecessary thread scheduling choices during the state space traversal. 

Implementing IFA in Harmony has two checkpoints:
- The original paper implements all of their analysis for Java Pathfinder. Thankfully, the way JPF makes use of atomic transitions in the state space is similar to how Harmony model checks a program with respect to an atomic representation:

> **JPF:** Sequence of instructions that consists of an instruction whose execution is globally-relevant in the given program state, followed by any
number of instructions whose execution is thread-local, and it ends with a non-deterministic thread scheduling choice. 

> **Harmony:** Even though Harmony does not really support input, there are three sources of non-determinism that make this exploration non-trivial: [...] _thread interleaving_: different threads run pseudo-concurrently with their instructions interleaved in arbitrary ways A thread can be in atomic mode or not. In atomic mode, the execution of the thread is not interleaved with other threads. A thread can also be in read-only mode or not. In read-only mode, the thread cannot write or deleted shared variables.

This means that we can scope a first pass of the analysis to just work on Harmony's thread interleaving and ignore _choose expressions_ and _interrupts_ (this is a closer analysis to that presented in the paper). Implementing the static analysis requires writing a module in the Harmony compiler's backend, which serves as a buffer between the compiler and the model checker. The static analysis algorithm works on the Kripke-like structure that's outputted by the compiler and uses a JSON of metadata with source-level information for modifying the structure. 

These modifications include annotating the objects for which there is no reference that an object exists in the heap yet, then only the thread that created the object can access it, and mapping them onto the threads in the final structure. 

- The second checkpoint requires implementing the actual field access analysis in the model checker. Section 5 of the 2014 paper describes in thorough detail how to naive forms of analysis work. To narrow the scope of this project, we will only be making use of the flow-insensitive intra-procedural analysis outlined in Section 6.1, which works on the previously annotated thread. 

**What are some possible challenges?**

This project requires a lot of reading and core implementation. We might not be able to provide any meaningful optimizations with regard to sound implementation of the immutable field analysis in Harmony. The core work will therefore focus on the compiler, and correctly annotating the JSON with the objects-threads mapping for which redundancies are found.

**How will you empirically measure success?**
Ideally, a full implementation would allow for running a set of [benchmarks in the Harmony repository](https://github.com/harmonylang/harmony/tree/master/test)
 while measuring their timeout rate using the following my local hardware (laptop) and UGC Linux. 
 
However, to make sure we have something of value, the first checkpoint will require testing for soundness (not optimization necessarily.) The annotated JSON files can be compared with atomic files as the model checker would normally work, and make use of the existing and working framework for smaller programs that succeed without this optimization (using smaller programs with a maximum of $2^3$ threads).

**Team members:**
@jpramos-me for now.