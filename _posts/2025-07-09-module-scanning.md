---
layout: default
title: "Module Scanning"
tags: build c++ modules soup
category: blog
---

## Module Scanning

This is the third installment of my original [blog post](https://mwasplund.github.io/blog/2022/03/14/modules-in-our-builds) outlining the three ways in which Modules can be integrated into C++ builds: through dynamic expansion during build evaluation, by utilizing a preprocessing phase, or by explicitly declaring the interface dependencies. In the [second post](https://mwasplund.github.io/blog/2022/11/29/project-level-modules) I outlined how we can leverage the structure already available in the build system itself to build up dependnecies between modules at the library level. In this post I share the internal process of scanning translation units to dynamically discover the dependencies between named module and their internal partitions.

### Module Syntax

The new module declaration syntax was purposfully written to make it easy to parse the beginning of a translation unit to make a determiniation of if the translation is a module unit and what modules are imported. 

By having a scanner tool we can break our build into a preprocessing phase to build up the dependency graph within module and a second phase that uses these interdepdencies to ensure the correct dependencies are referenced and the required build order is enforced.

### Summary

With a proper build system setup to handle preprocessing translation units we can build up a dependency graph for our builds to ensure build order is correct and we can dynamically build complex interdependencies without requiring extra build structuring by engineers and will make module adoption very easy.

In this example I use partition units that build up a named module, however the same dependency graph can be built up between multiple named modules.