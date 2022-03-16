---
layout: default
title: "Modules In Our Builds"
tags: build c++ modules soup
category: blog
---

## Introduction
Historically, Translation Units have had the unique ability to be compiled in any order, irregardless of their dependencies. This was made possible by the preprocessor which included header files containing the shared symbol declarations of both the producer and consumer Translation Units. While this has simplified build definitions and allowed for easy parallelization, it also increased overall build times, incremental build scope, and has led to fragile builds with deep interconnected dependencies.
With the introduction of C++20, Modules aims to resolve a lot of the shortcomings of the C preprocessor when managing shared symbols, and also introduce an extra level of complexity when tracking Interface dependencies between Translation Units. Discussing how we wish to generate and consume Modules in our builds is critical to understanding requirements and influencing compiler vendors as they converge on a unified solution.

## Interface Dependencies
Although I provide some details about module dependencies through their interfaces, if you’d like a more thorough introduction of how modules work, check out [vector-of-bool’s blog](https://vector-of-bool.github.io/2019/03/10/modules-1.html). (There are a few discrepancies in this article on Module Implementation Units; still, it’s a great place to start learning about Modules.)

A C++ Translation unit that contains a `module` declaration with the optional `export` keyword is known as a Module Interface Unit. When compiled, these files will produce a new Compiled Module Interface (CMI) file alongside the object files. The object files contain the symbol implementations which can be safely ignored until they are linked into the final assembly. However, the CMI contains the symbol declarations that are consumed while compiling a Translation Unit that has an Interface Dependency on the exported module.

> Note: For simplification purposes, I am ignoring some complexity around inlining that may require extra contents in the CMI file.

A Translation Unit is said to have an Interface dependency on a module if it either implicitly or explicitly imports that module or a module that has an Interface dependency on that module. A direct Interface dependency will be created by a Translation Unit that imports a module or between a module implementation unit and its module interface unit.

### Examples
Let's look at a few quick examples to help familiarize ourselves with Module Interface Dependencies:

**A.cpp** - Has no Interface Dependencies.
```c++
export module ModuleA
/* Cool Code */
```

**B.cpp** - Has an interface dependency on module **ModuleA** from the direct import.
```c++
export module ModuleB
import ModuleA;
/* Declare Stuff */
```

**BImpl.cpp** - Has an implicit interface dependency on module **ModuleB** as an Implementation Unit and through that has an inherited Interface Dependency on **ModuleA**.
```c++
module ModuleB
/* Implement Stuff */
```

**Main.cpp** - Has an explicit interface dependency on module **ModuleB** from the import and through that has an inherited Interface Dependency on **ModuleA**.
```c++
import ModuleB;
/* More Great Code */
```

## Discover or Define?
Once Translation Units have an Interface Dependency graph there is a strict order to producing and consuming CMI files. Our builds must ensure that when evaluating an import declaration, the compiler is able to find the matching CMI and in turn the symbols exported by the module itself. To add extra complexity to the problem, a named Module has no requirement to match the containing file name. If we know the required Interface dependencies for a specific Translation Unit, we still must resolve which upstream Translation Unit produces those modules.

The question then becomes, do we let our builds discover the ordering for the module interdependencies during build evaluation, or do we explicitly tell the build system how to generate the dependency graph up front? There is no clear answer. Each option lends itself nicely to different aspects of the over all build. I believe our builds will evolve to incorporate both options.

## Discover
Auto discovery of internal interdependencies is an ideal solution for end developers. The dynamic nature of these builds allows developers to work entirely within their code and the build system responds to their needs. These benefits come with an extra level of complexity that can lead to undesirable build characteristics. Within this option there are two primary solutions. The build can either expansion dependencies as they are discovered or preprocess the input files to generate the dependency graph up front.

### Expansion
The first approach to handling module ordering within a build is to ignore the ordering up front and instead resolve and build upstream Interface Dependencies as they are discovered. This can be achieved by introducing a callback mechanism to allow the compiler ask the build system to resolve module import declarations. This allows for existing build systems to ignore the problem and continue to compile translation units in "any order". When an import declaration is seen, the compiler asks the build system for the CMI for the specific module. If the build system is unable to find the specified module interface, it will pause the current build and compile the requested module first. The build system would then be responsible to determine which translation unit is required to build a specific module interface file. To accomplish this, the build system must either enforce an arbitrary naming convention or pick a random file and monitor if the requested interface file was produced. This process is recursively followed until a Translation Unit is encountered that can be fully resolved and compiled. Once this top level file finishes, the previous files can resume compilation where they left off and finalize the import declarations.

> For further reading showing an example of how this might work, check out Nathan Sidwell's [A Module Mapper](http://www.open-std.org/jtc1/sc22/wg21/docs/papers/2020/p1184r2.pdf) post that outlines GCC's module mapper design.

Auto expansion allows build systems to continue to pretend that there is no order to building translation units. This may seem desirable at first glance, but is simply putting off the issue. As we adopt Modules into our projects, our builds will become increasingly interconnected through complex dependency graphs which will lead a large number of Translation Units compilations blocked on the output of others. This will lead to slowdowns on local builds as we chain to gether many concurrent compilations and could make distributed builds impossible. The dynamic nature of the required work for any given TU also makes it increasingly hard to track and evaluate incremental builds. The tight coupling between the compiler and build system increases overall complexity and maintenance costs. Not to mention the low probability that all vendors will adopt the same protocol. The key benefit this approach gives is a short term ability to more easily retrofit existing build systems to support modules, however I believe there are more streamlined approaches that will work better in the long run.

### Preprocessing
A very similar approach to dynamic expansion of Interface Dependencies during build evaluation is to rely on a preprocessor to generate the entire dependency graph upfront. It would then be a trivial problem to walk the dependency graph in order and build the Translation Units with the known set of dependency references.

This approach to building modules has the benefit that it allows for a build system to adapt dynamically to the underlying code, but requires no special considerations from the compiler, being entirely owned within the build system. There will be a performance overhead as the raw C++ files are processed multiple times, however this will be mitigated by the requirements that module declarations live at the start of the file. Depending on the complexity of the preprocessor implementation it may also have to take shortcuts by ignoring C preprocessor directives. By doing this the generated dependency graph may include a super set of possible import declarations if some are behind conditionals. However, in practice I do not believe this will be an issue as we migrate away from the C processor and rely on more modern coding practices.

The major downside to this approach is that a build system must still dynamically adapt to the dependency graph as it is discovered or natively integrate with the preprocessing phase. This can introduce extra complexity in the build system or require internal knowledge of C++ parsing. If the build system does not support these complex behaviors natively it may be forced to treat all of these Translation Units as a "batch" when performing the dependency graph generation and the individual Translation Unit compilations. This would mean that any change to a single file in the "batch" would invalidate the entire graph and require all files to be rebuilt.

The ability to dynamically adapt to Translation Unit internals without needing to repeat yourself within the build definition is desirable when designing a build system. However, because there will be extra processing overhead and increased incremental build complexity I believe this type of system will be useful only in locally scoped sections of a build. For instance; a preprocessor may be able to determine the entire interdependency graph for a single named module and all of its internal partition and implementation units.

## Define
The final way to determine the dependency graph within module interfaces is to simply tell the build system up front. Explicitly defining a dependency graph is straight forward and can support any combination of imports and exports. The main downside being it is hard to maintain and often requires repeating information for the build system that is already contained in the source files themselves.

Following the same example as before, one could imagine a build system that allows you to configure your `Source` list with a set of `Imports` to indicate the Translation Unit dependencies which are assumed to map to the Interface Dependencies.

**Recipe.toml**
```toml
Source = [
    { File = "Source/A.cpp", IsInterface = true },
    { File = "Source/B.cpp", IsInterface = true, Imports = [ "Source/A.cpp" ] },
    { File = "Source/BImpl.cpp", IsInterface = false, Imports = [ "Source/B.cpp" ] },
    { File = "Source/Main.cpp", IsInterface = false, Imports = [ "Source/B.cpp" ] },
]
```

This definition clearly tells the build system all the required information up from to generate the full Interface Dependency graph and guarantees that modules are compiled in the correct order with the correct CMI references. The only real downside is increased maintenance costs when writing build definitions. I believe this problem can be solved be leveraging existing knowledge in a build system.

### Existing Dependencies
If a build system is designed with the concept of projects and references then a dependency graph already exists. We can take advantage of the existing dependency graph to automatically map to our module Interface Dependencies. We can then mirror a library boundary to have a single module interface unit export the symbols that are public to the library. From here the build system can automatically reference the CMI files for the upstream dependencies when building and link against the static or dynamic library file as is done today.

> Aside: It is also advantageous at this stage for the build system to enforce a naming convention that requires a project name to match the internal module that will be exported. While we could conceivable allow for any module name, and rely on the individual project authors to coordinate their names, it will become impossible to manage as module usage proliferates. Modules will facilitate wide adoption of package managers which will inevitably lead to collisions between name ownership within named modules.

Let's walk through the above example broken apart into individual projects. Note: This example is a bit contrived as a project will generally contain more than a single source file. Most notable, there will be many implementation and partition units to fill in the details.

**ModuleA/Recipe.toml**
```toml
Name = "ModuleA"
Interface = "Source/A.cpp",
```

**ModuleB/Recipe.toml**
```toml
Name = "ModuleB"
Interface = "Source/B.cpp",
Source = [
    "Source/BImpl.cpp",
]
[Dependencies]
Runtime = [
    "../ModuleA/"
]
```

**Main/Recipe.toml**
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

## Summary
Modules aim to solve inherent limitations within the C Preprocessor and streamline sharing symbol definitions within C++. It also introduces a new ordering and dependency problem that will require extra work from our build systems. I believe that by leveraging existing dependency structure within our project references we can build a top level Interface Dependency graph for named modules. This can be augmented with a preprocessor or explicit definitions of Partition dependencies to build up the internal structure of a library/module combination.

For a working example of this design in practice check out my open source build system [Soup](https://github.com/SoupBuild/Soup).