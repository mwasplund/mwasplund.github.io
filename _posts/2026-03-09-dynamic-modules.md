---
layout: default
title: "Dynamic Modules"
tags: build c++ modules soup
category: blog
---

## Editor Note: Bad Assumptions :)

I was wrong in my assumption that the global module fragment can only contain preprocessor directives implied that they cannot wrap around imports or export declarations. This invalidates my arguments for a simple scanner parser, and means we will need to rely on the compilers to do this for us. I will leave this post up as a "hypothetical" for if we could limit the preprocessor to not span `import` or `export` declarations on how simple we could have had it.

## Dynamic Modules

This is the third installment of my series [blog posts](https://mwasplund.github.io/blog/2022/03/14/modules-in-our-builds) outlining the three ways in which Modules can be integrated into C++ builds. For the first time in C++ history, Modules introduce a dependency hierarchy and implicit order to our builds. The order can be determined through dynamic expansion during build evaluation, by explicitly declaring the interface dependencies, or by running a preprocessing phase to dynamically detect dependencies. In the [second post](https://mwasplund.github.io/blog/2022/11/29/project-level-modules) I outlined how we can leverage the structure already present in the build system itself to generate a dependency graph between modules at the library level. In this post, I dive into the internal process of scanning translation units to dynamically discover the dependencies between named modules and their internal partitions.

### Module Declaration

There are many aspects of the C++ Module feature that make Modules difficult to integrate within our build systems. The most glaring challenge is the new requirement for order dependent builds which was not necessary in the past. However, the standards committee did provide us with a way to work around these issues. The new module declaration syntax was purposefully written to make it easy to parse the beginning of a translation unit to determine if a given unit is a module unit. If it is a module unit, we can then extract the required name and full set of imported Modules. This well known, and easy to parse grammar, allows us to write a simple scanner tool to process the least amount of tokens at the start of the file to determine the complete set of input dependencies and output module name from a given module unit. If we leverage a build system that is broken into a two phased approach, we can preprocess all translation units to dynamically build up the full dependency graph between named modules, their internal partition units, and implementation units. Once parsed it is a trivial problem to ensure all translation units are build in the correct order.

### Module Parser

The module parser is very simple and has a lexicon made up of only 7 token types (Identifier, Period, Colon, Semicolon, Module, Import, Export). Using this simple set of tokens we can write a formal grammar that consists of a subset of the full C++ language which can strictly process the `module` and `import` declarations at the beginning of the file.

#### Module Declaration
```
export? module module-name module-partition? attr? ;
```

We can easily check for the existence of the `module` token with the optional `export` prefix. If present, we attempt to parse the remaining module declaration. Otherwise we assume this is a standard translation unit. We must also handle the case that the first declaration we see is for the global module fragment. Once we detect a global module fragment we must skip ahead to the  module declaration. Luckily this is trivial since the global module fragment may only contain preprocessor directives which can be ignored by the lexer similar to comments.

#### Import Declaration
```
export? import module-name attr? ;
```

In all translation units, the import declarations must be grouped after the optional module declaration and before all other declarations. This allows us to look for `import` tokens until we encounter an unknown token. At which point we can assume we are in the main body of the translation unit, where no other module imports can exist. We can safely assume that a subsequent compiler invocation will validate the remainder of the translation unit.

### Two Phase Build

With the simple parser in hand, we now need to integrate it into our build process. We do this by breaking our builds into two phases: a preprocessing phase and a build generation phase. During the initial preprocessing phase, we feed the full set of translation units into the light weight module parser which will generate the extra metadata required to build up the complete dependency graph. It will identify if a translation unit is a module unit. It can then identify interface units along with the name and optional partition. Once the module declaration is fully parsed, the import declarations will be parsed to generate an full set of upstream dependencies.

With a now complete dependency graph available to the build system, we can easily transform these references into explicit build dependencies during the generate phase. It is now a simple task for the build system to ensure that all dependencies for a given translation unit have completed compilation before we begin processing the target unit.

### Summary

With a proper build system setup to handle preprocessing translation units, we can generate a dependency graph for our builds to ensure proper ordering. We can dynamically detect complex interdependencies without requiring extra effort by software engineers and help make module adoption more appealing. I have a created a working [sample](https://github.com/soup-build/soup/blob/main/docs/samples/cpp/module-partititons.md) showing how two phased dynamic processing makes writing module interfaces with many internal partitions effortless. Allowing engineers to focus more time on solving real problems and not fighting build systems.

### Next Steps

The primary downside to this approach is the increased build times from pre-processing every translation unit. I plan to create a followup blog post to help understand how this extra work impacts build times in real world scenarios. My instincts say that with the right amount of parallel processing and incremental build checks the overhead should be negligible. We will not know for sure until we test.