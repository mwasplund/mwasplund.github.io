---
layout: default
title: "Modules In Our Builds"
tags: build c++ modules soup
category: blog
---

## Introduction
Until the introduction of Modules in C++20, Translation Units have had the interesting characteristic that they can be compiled in any order, irregardless of their dependencies. This is made possibly by the preprocessor including header files containing the shared symbol declarations into both the producer and consumer Translation Units. While this simplifies build definitions and allows for easy parallelization, it also increases overall build times, incremental build scope and can lead to fragile builds. Modules aims to resolve a lot of the shortcomings of the C preprocessor when managing shared symbols, but also introduces an extra level of complexity when tracking Interface dependencies between Translation Units. It is important to discuss how we wish to generate and consume Modules in our builds to come to a consensus as a community and help influence the compiler vendors as they converge on a unified solution.

## Interface Dependencies
I will go into some detail about module dependencies through their interfaces, but for a better introduction to the details of how modules work check out [vector-of-bool's blog](https://vector-of-bool.github.io/2019/03/10/modules-1.html)

> Note: There seems to be some discrepancies on how Module Implementation Units work in the post. My understanding is that all Module Implementation Units MUST have a matching Interface Unit due to the implicit import, however the example in the blog post shows importing Module Partition Implementation Units directly from the Prime Interface Unit. Besides this issue I find this blog to be the best place to start when learning about Modules.

A C++ Translation unit that contains a `module` declaration with the optional `export` keyword is known as a Module Interface Unit. When compiled, these files will produce a new Compiled Module Interface (CMI) file alongside the object files. The object files contain the symbol implementations which can be safely ignored until they are linked into the final assembly. However, the Compiled Module Interface contains the symbol declarations that are consumed while compiling a Translation Unit that has an Interface Dependency on the exported module.

> Note: I am ignoring some complexity around inlining that may require other contents in the CMI file.

A Translation Unit is said to have an Interface dependency on a module if it either implicitly or explicitly imports that module or a module that has an Interface dependency on that module. A direct Interface dependency will be created by a Translation Unit that imports a module or between a module implementation unit its module interface unit.

### Examples

Lets look at a few quick examples to help familiarize ourselves with Module Interface Dependencies:

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
Now that our Translation Units have an Interface Dependency graph there is a strict order to producing and consuming Compile Module Interface files. Our builds must now ensure that when evaluating an import declaration the compiler is able to find the matching CMI. To add extra complexity to the problem, a Translation Unit file name has no requirement to match the named Module within. Even if we know the required modules for a specific Translation Unit, we still have to resolve which upstream Translation Unit produces those modules.

The question then becomes, do we let our builds discover the ordering for all of the module inter-dependencies during build evaluation, or do we explicitly define a set of rules to to tell the build system how to generate the dependency graph?

### Auto Expansion
The first approach to handling module ordering within a build is to ignore the ordering up front and instead resolve and build upstream Interface Dependencies as they are encountered. This can be achieved by introducing a callback mechanism to ask the build system to resolve module import declarations. This allows for existing build systems to ignore the problem and continue to compile translation units in "any order". When an import declaration is seen, the compiler ask the build system for the CMI for the specific module. If the build system is unable to find the specified module interface, it will pause the current build and compile the requested module first. The build system would then be responsible to determine which translation unit is required to build a specific module interface file. At this point the build system must either enforce an arbitrary naming convention or pick a random file and monitor if the requested interface file was produced. This process is recursively followed until a unit is encountered that can be fully resolved and compiled. Once this top level file finishes, the previous translation units can resume compilation where they left off and finalize the import declarations.

Auto expansion allows builds to continue to pretend that there is no order to building translation units; this may seem desirable at first, but is simply putting off the root issue with an overly complicated tight coupling of the compiler and the build system. As we adopt Modules into our projects, our builds will become increasing interconnected through complex dependency graphs which will lead a large number of Translation Units compilations blocked on the output of others. This will lead to slowdowns on local builds and could make distributed builds unmanageable. The dynamic nature of the required work for any given TU also makes it increasingly hard to track and evaluate incremental builds. Finally, the tight coupling between the compiler and build system makes it harder to maintain as they would both need to be updated in lockstep if there are ever any breaking changes and there is little chance that all vendors will adopt the same protocol.

### Preprocessing
A very similar approach to the dynamic expansion of Interface Dependencies during build evaluation is to run an preprocessor over a set of Translation Units to generate the entire dependency graph upfront. It would then be a trivial problem to build the translation units in the required order with the required references. 

This approach is aided by the fact that module unit must have the module declaration at the top of the file. This allows for a streamlined parser to only interpret the import and export declarations at the top of the file to build up the full dependency graph faster than processing the entire file. Depending on the complexity of the preprocessor implementation it may have to take shortcuts when evaluating the C++ code and ignore C preprocessor directives. This would mean that the graph may include a super set of possible import declarations. However, in practice I do not believe this will be an issue as we migrate away from the C processor and rely on more modern coding practices.

This approach to building Modules has the benefit that it allows for a build system to adopt dynamically to the underlying code, but requires no special considerations from the compiler and would be entirely owned within the build system. There will be an performance overhead as we process the raw C++ files multiple times, however this will be mitigated by the requirements that module declarations live at the start of the file. The major downside to this approach is that a build system would need to natively maintain the generated build graph to perform granular incremental build. If it does not it would be forced to build all of these units as a "batch" if any of the others contain changes.

Because of the ability to dynamically adapt to Translation Unit internals without needing to repeat yourself within the build definition, but because there will be extra processing overhead and increased incremental build complexity I believe this type of system will be very useful in locally scoped subsections of a build. For instance, a preprocessor may be able to determine the entire interdependency graph for a single named module and all of its internal partition and implementation units.

### Declare
Explicitly defining a dependency graph is straight forward and can support any combination of imports and exports with the extra cost of maintenance. The main downside being it is hard to maintain and often requires telling the build system the information that is already contained in the source files themselves.

Recipe.toml
```toml
Source = [
    { File = "Source/ModuleA.cpp", IsInterface = true },
    { File = "Source/ModuleB.cpp", IsInterface = true, Imports = [ "Source/ModuleA.cpp" ] },
    { File = "Source/ModuleBImpl.cpp", IsInterface = false, Imports = [ "Source/ModuleB.cpp" ] },
    { File = "Source/Main.cpp", IsInterface = false, Imports = [ "Source/ModuleB.cpp" ] },
]
```

However, if a build system is designed with the key dependency graph already in place between project references then it is a simple matter of mapping project structures to modules. We can take advantage of the existing dependency graph between projects to automatically incorporate module interface dependencies. This is where we can easily mirror a library boundary to have a single interface unit export the symbols that are public to the library. From here the build system can incorporate project references to automatically reference the module interface units for the upstream dependencies when building and link against the library symbols as is done today.

It is also advantageous at this stage to enforce a naming convention that requires a project name to match the internal module that will be exported. While we could conceivable allow for any module name, and rely on the individual project authors to coordinate their names, it will become impossible to manage as module usage proliferates. It is my goal that Modules can facilitate the adoption of a true package manager which will inevitable lead to collisions between name ownership within named modules.

ModuleA/Recipe.toml
```toml
Name = "A"
Interface = "Source/ModuleA.cpp",
```

ModuleB/Recipe.toml
```toml
Name = "B"
Interface = "Source/ModuleB.cpp",
Source = [
    "Source/ModuleBImpl.cpp",
]
[Dependencies]
Runtime = [
    "../ModuleA/"
]
```

Main/Recipe.toml
```toml
Name = "MyApplication"
Source = [
    "Main.cpp",
]
[Dependencies]
Runtime = [
    "../ModuleB/"
]
```

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