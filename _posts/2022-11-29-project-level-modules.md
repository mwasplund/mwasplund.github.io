---
layout: default
title: "Project Level Modules"
tags: build c++ modules soup
category: blog
---

## Project Level Modules
Awhile back I wrote a blog post outlining the three ways in which Modules can be integrated into C++ builds; through dynamic expansion during build evaluation, by utilizing a preprocessing phase to discover dependencies or by explicitly declaring the Interface dependencies. I believe that over time we will come to adopt all three variants to fit their own unique scenarios. I am personally invested in validating that a build system that contains sufficient knowledge of project dependencies can generate an implicit Interface Dependency graph by assigning each project a unique public Module Interface.

To that end, I have been working on a build system that I believe contains all the required components to build and share Modules effortlessly between Source Packages. Now that the project is to a point where I believe (hope) that the major components are in place, others can begin evaluating it for its viability as a solution for creating a unified C++ ecosystem. To help illustrate how this new system will work, I walk through the process of authoring and consuming a shared package that exposes only a single Module Interface using concrete examples that map to the simplest real-world scenarios.

### Library Author
As a Library Author it is entirely up to personal preference whether to continue to utilize header-based declarations or entirely migrate to a Module design that relies on Partition units to build up the core functionality. To keep this discussion as simple as possible, I will perform the minimal number of changes to convert an existing project over to export a single Module Interface.

Let us consider the popular, although recently archived project, json11. This project is a great test case since it is relatively simple, but is not a header only library. It takes very little to convert the single header and translation unit into a Public Module Interface.

1. Add a public named module declaration to the single translation unit.
```c++
export module json11;
```
1. Exclude external standard library includes from the public header and move them into the global module purview so they do not get included in the module and assigned module linkage.
1. Add an ‘export’ modifier to the json11 class.
```c++
export class Json final {
}; 
```
1. Create a Recipe to declare the Soup Build Package. Indicate that the single translation unit is a Public Module Interface.
```sml
Name: "json11"
Language: "C++|0.1"
Version: "1.1.0"
Interface: "json11.cpp"
```
For a full breakdown take a look at the comparison of my forked branch to the original master: https://github.com/dropbox/json11/compare/master...mwasplund:json11:master . Care was taken to ensure the code can compile as a module interface or using a legacy header include.

Once the project is built as a Soup Package with no local dependencies it can be published to the public feed. This will generate an archive of the source files and Recipe which is all that is required to build the project on any system that has Soup installed.
```
soup publish
```

### Library Consumer
We can now create an executable in a second Soup Package.
```sml
Name: "ParseJsonFile"
Language: "C++|0.4"
Version: "1.0.0"
Type: "Executable"
Source: [
  "Main.cpp"
]
```
The json11 package can now be consumed by including it in the set of dependencies from the Recipe. This can be accomplished by either manually editing the Recipe declaration to include the Runtime Dependency directly, or the Soup CLI includes a handy install command that will discover the latest version in the registry.
```
soup install json11
```
```sml
Dependencies: {
  Runtime: [
    "json11@1.1.0"
  ]
}
```
Once the dependency is referenced we can consume it automatically from within our code.
```c++
import json11;
```
A full working example can be found in the samples of the Soup Build Project: https://github.com/SoupBuild/Soup/tree/main/Samples/Cpp/ParseJsonFile

### Conclusion
This walkthrough is meant to show the ease of sharing a library with modules using a build system that understands the full dependency graph including an integrated package manager. For a more in depth view of how this can be extended to a more complex example feel free to explore the Soup Build source which heavily dogfoods itself as its own build system. 
