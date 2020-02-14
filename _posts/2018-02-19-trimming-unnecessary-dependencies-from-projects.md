---
layout: post
title: Trimming Unnecessary Dependencies from Projects
categories: [.NET, MSBuild]
tags: [.NET, dependencies, msbuild, references]
comments: true
---
When you're working in large repositories with hundreds of [MsBuild](https://github.com/Microsoft/msbuild){:target="_blank"} projects, you're bound to have fairly complex build graphs. Over time, these can devolve and you may end up with lots of dependencies between projects which are no longer needed. This can cause builds to slow down as they are less parallelizable, and the developer experience can suffer as you unnecessarily rebuild libraries which have falsely depend on libraries you changed.

Luckily, the [Roslyn compiler](https://github.com/dotnet/roslyn){:target="_blank"} is smart enough to only include references which are actually used in an assembly's metadata. So you can actually compare the references which are passed to the compiler against the references that actually make it into the compiled assembly.

You can figure out which references make it into the assembly using a tool like [ILSpy](http://www.ilspy.net/){:target="_blank"} or [dotPeek](https://www.jetbrains.com/decompiler/){:target="_blank"}, or even use the reflection method [`Assembly.GetReferencedAssemblies`](https://docs.microsoft.com/en-us/dotnet/api/system.reflection.assembly.getreferencedassemblies?view=netstandard-2.0){:target="_blank"}.

There are three different ways that references can be included in your project: `Reference`, `ProjectReference`, and `PackageReference`.

## References and Project References
References and Project References are both fairly straightforward. Each reference gets resolved, usually by its `HintPath` if it isn't a framework assembly, and passed to the compiler. Similarly for project references, MsBuild does an inner build to determine each project reference's target assembly and passes that to the compiler for the outer build. For both of these, it's pretty straightforward to just compare each reference passed to the compiler and check whether it made it into the assembly's metadata.

## Transitivity
But wait, what if an indirect dependency is required at run time, but not at compile time? This can happen if you have a project with a dependency which itself has a dependency which isn't directly required by the original project. To build the project, the indirect dependency may not be needed, however if the project produces something runnable like an exe or a unit test assembly, certain code paths may require the indirect dependency to get loaded into the App Domain.

So for project which produce something runnable, we have to consider all transitive references and not just the references the main assembly has.

## Package References
Package references are where things get a little complicated. In the new [Sdk-style projects](https://docs.microsoft.com/en-us/visualstudio/msbuild/how-to-use-project-sdk){:target="_blank"}, MsBuild and NuGet work together to form the entire dependency graph for the package references you specified, collects all the assemblies for all of those packages, and passes every single one of them to the compiler. With the packaging of the framework assemblies themselves (ie. [`netstandard2.0`](https://www.nuget.org/packages/NETStandard.Library/){:target="_blank"} and [`netcoreapp2.0`](https://www.nuget.org/packages/Microsoft.NETCore.App/){:target="_blank"}), this can get fairly huge. As an example, for a fairly simple web application I have, a whopping 366 `/reference` parameters are passed to `csc.exe`. As if things weren't complicated enough, some packages like [`Microsoft.AspNetCore.All`](https://www.nuget.org/packages/Microsoft.AspNetCore.All/){:target="_blank"} are really just meta-packages which themselves don't have any assemblies but instead just have a number of dependencies which do contain assemblies to be referenced. And then packages like [`Microsoft.Net.Compilers`](https://www.nuget.org/packages/Microsoft.Net.Compilers/){:target="_blank"} don't add any references but instead provide additional build tooling by way of MsBuild props and targets.

So for package references, we have to account for any assembly in the package or any packages it depends on, and also for cases where they don't provide references at all.

## Introducing ReferenceTrimmer
If all of this seems overwhelming enough to make you not want to bother cleaning up your projects, fear not. I've create a little tool called [ReferenceTrimmer](https://github.com/dfederm/ReferenceTrimmer){:target="_blank"} which does all the work for you.

When you run it, it accounts for each of the things discussed above and prints out what it believes are unnecessary references, project references, and package references.

I'm sure it doesn't cover all cases yet, and likely reports false-positives and false-negatives, but it was able to find some issues in some of my smaller projects even. Contributions are always welcome if you find that it could do better though!
