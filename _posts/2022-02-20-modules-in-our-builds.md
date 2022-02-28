---
layout: default
title: "Modules In Our Builds"
tags: build c++ modules soup
category: blog
---

## Introduction
Until Modules, C++ Translation Units have had the interesting characteristic that they can be compiled in any order, irregardless of their dependencies. This is made possibly by the preprocessor including shared header files into both the producer and consumer Translation Units that contain the symbol declarations. While this simplifies build definitions and allows for easy parallelization, it also increases build times when re-parsing the same text file multiple times, increase incremental builds work when editing a file used by many Translation Units and often leaks preprocessor definitions creating unstable builds. Modules aims to resolve a lot of the shortcomings of the C preprocessor when managing shared symbols, but also introduces an extra level of complexity when tracking Interface dependencies between Translation Units. It is important to discuss how we wish to generate and consume Modules in our builds to help influence the compiler vendors as they converge on a unified solution.

## Interface Dependencies
I will go into some detail about module dependencies through their interfaces, but for a better introduction to the details of how modules work check out [vector-of-bool's blog](https://vector-of-bool.github.io/2019/03/10/modules-1.html)
```
Note: There seems to be some discrepancies on how Module Implementation Units work. My understanding is that all Module Implementation Units MUST have a matching Interface Unit due to the implicit import, however the example in the blog post shows importing Module Partition Implementation Units directly from the Prime Interface Unit. Besides this issue I find this blog to be the best place to start when learning about Modules.
```

A C++ Translation unit that contains a `module` declaration with the optional `export` keyword is known as a Module Interface Unit. When compiled, these files will produce a new Compiled Module Interface file alongside the object files. The object file contains the symbol implementations which will be linked into the final assembly, while the Compiled Module Interface will contain the symbol declarations to be used while compiling a Translation Unit that has an Interface Dependency on the exported module (Note: I am ignoring some complexity around inlining).

A Translation Unit is said to have an Interface dependency on a module if it either implicitly or explicitly imports that module or a module that has an Interface dependency on that module. A direct Interface dependency will be created by a Translation Unit that imports a module or between a module implementation unit its module interface unit.

### Examples

Lets look at a quick example to help familiarize ourselves with Module Interface Dependencies:

**ModuleA.cpp** - Has no Interface Dependencies.
```c++
export module A
/* Cool Code */
```

**ModuleB.cpp** - Has an interface dependency on module **A** from the direct import.
```c++
export module B
import A;
/* Declare Stuff */
```

**ModuleBImpl.cpp** - Has an implicit interface dependency on module **B** as an Implementation Unit and through that has an inherited Interface Dependency on **A**.
```c++
module B
/* Implement Stuff */
```

**Main.cpp** - Has an explicit interface dependency on module **B** as from the import and through that has an inherited Interface Dependency on **A**.
```c++
import B;
/* More Great Code */
```

```mermaid
  graph LR;
      A-->B;
      B-->Main;
```

## Discover or Define?
Now that our Translation Units have an Interface Dependency graph that requires generating and consuming Compile Module Interface files that follow this same graph we need to ensure that when evaluating an import declaration the compiler is able to find the matching CMI. To add extra complexity to the problem, a translation unit file name has no requirement to match the Module within. Even if we know the required modules for a specific Translation Unit, we still have to resolve which upstream Translation Unit produces those modules.

The question then becomes, do we let our builds discover the ordering for all of the module inter-dependencies or do we explicitly tell the build system the dependency graph?

### Auto Expansion
The first approach to handling ordering within a build with modules is to ignore it until it becomes a problem. introduce a callback mechanism whereby a compiler can request that the build system compile a Module before it can be imported. This allows for existing build systems to ignore the problem up front and continue to compile translation units in "any order". When an import declaration is seen, the compiler ask the build system for the CMI for the specific module. If the build system is unable to find the module interface, it will pause the current build and compile the requested module first. The build system would then be responsible to determine which translation unit is required to build a specific module interface file. At this point the build system must either enforce a requirement that a module name match the file name or pick a random file and monitor if the requested interface file was produced. This process is recursively followed until a unit is encountered that can be fully resolved and compiled. Once this top level file finishes, the previous translation units can continue compiling where they left off at the import declarations.

If we work through the above example we would tell the compiler to build Main.cpp, B, C and D in any order. Lets say that we start with A.cpp. When the compiler encounters the `import B` declaration it will attempt to load B.ifc and fail. It will pause compilation and ask the build system to please compile the B Module. The build system will attempt to compile B.cpp using the naming convention that a file must be the same name as the Module inside. From here the compiler will encounter the `import D` and attempt to load D.ifc. It will fail and again ask the build system to compile the module D. The build system will now successfully compile D.cpp since it has no other module dependencies. The compiler will resume building B.cpp and succeed. The compiler will attempt to finish A.cpp and fail to load C.ifc, which will call back to the build system to ask for the C module. The build system will compile C.cpp, which will succeed in loading the D.ifc file that already exists. The compiler will make a third attempt to finish compiling and will now finish.

Auto expansion allows builds to continue to pretend that there is no order to building translation units, but this quickly falls apart. It also introduces a tight coupling between the build system and the compiler implementation for the callback protocol and the naming convention used. This makes it a limited solution with one compiler and unmanageable with many.

### Preprocessing
A group of translation units could be compiled as a "batch" where a preprocessing phase parses the module unit export and import declarations to determine the dynamic dependency graph and builds the translation units in the required order with the required references or the compiler can implement a callback design where it asks the build system to imported modules as they are discovered.

This approach is aided by the fact that module unit MUST have the module declaration at the top of the file. This allows for a streamlined parser to only interpret the import and export declarations at the top of the file to build up the full dependency graph faster than processing the entire file.

The major downside to this approach is that a build system would need to maintain the output build graph to perform incremental build or risk building an entire "batch" of translation units when a single file has changed. 

### Declare
Explicitly defining a dependency graph is straight forward and can support any combination of imports and exports with the extra cost of maintenance.

If a build system is designed with the key dependency graph already in place between project references then it is a simple matter of mapping project structures to modules.

We can also take advantage of the existing dependency graph between projects to automatically incorporate module interface dependencies. This is where we can easily mirror a library boundary to have a single interface unit export the symbols that are public to the library. From here the build system can incorporate project references to automatically reference the module interface units for the upstream dependencies when building and link against the library symbols as is done today.

It is also advantageous at this stage to enforce a naming convention that requires a project name to match the internal module that will be exported. While we could conceivable allow for any module name, and rely on the individual project authors to coordinate their names, it will become impossible to manage as module usage proliferates. It is my goal that Modules can facilitate the adoption of a true package manager which will inevitable lead to collisions between name ownership within named modules.


<script src="https://cdnjs.cloudflare.com/ajax/libs/mermaid/8.14.0/mermaid.min.js"></script>
<script>
    const config = {
        startOnLoad:true,
        theme: 'default',
        flowchart: {
            useMaxWidth:false,
            htmlLabels:true
            }
    };
    mermaid.initialize(config);
    window.mermaid.init(undefined, document.querySelectorAll('.language-mermaid'));
</script>