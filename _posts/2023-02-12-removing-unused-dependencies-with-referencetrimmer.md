---
layout: post
title: Removing unused dependencies with ReferenceTrimmer
categories: [.NET, MSBuild]
tags: [.NET, dependencies, msbuild, references]
comments: true
---

It's [been a while]({% post_url 2018-02-19-trimming-unnecessary-dependencies-from-projects %}) since I first introduced [ReferenceTrimmer](https://github.com/dfederm/ReferenceTrimmer){:target="_blank"} and a lot has changed.

For background, ReferenceTrimmer is a [NuGet package](https://www.nuget.org/packages/ReferenceTrimmer){:target="_blank"} which helps identify unused dependencies which can be safely removed from your C# projects. Whether it's old style `<Reference>`, other projects in your repository referenced via `<ProjectReference>`, or NuGet's `<PackageReference>`, ReferenceTrimmer will help determine what isn't required and simplify your dependency graph. This can lead to faster builds, smaller outputs, and better maintainability for your repository.

Most notably among the changes are that it's now implemented as a combination of an [MSBuild task](https://learn.microsoft.com/en-us/visualstudio/msbuild/msbuild-tasks){:target="_blank"} and a [Roslyn analyzer](https://learn.microsoft.com/en-us/visualstudio/code-quality/roslyn-analyzers-overview){:target="_blank"} which seamlessly hook into your build process. A close second, and very related to the first, is that it uses the [`GetUsedAssemblyReferences`](https://learn.microsoft.com/en-us/dotnet/api/microsoft.codeanalysis.compilation.getusedassemblyreferences){:target="_blank"} Roslyn API to determine exactly which references the compiler used during compilation.

## Getting started

Because of the implementation being in an MSBuild task and Roslyn analyzer, the bulk of the work to use ReferenceTrimmer is to simply add a `PackageReference` to the [ReferenceTrimmer NuGet package](https://www.nuget.org/packages/ReferenceTrimmer){:target="_blank"}. That will automatically enable its logic as part of your build. It's recommended to add this to your `Directory.Build.props` or `Directory.Build.targets`, or if you're using NuGet's [Central Package Management](https://learn.microsoft.com/en-us/nuget/consume-packages/Central-Package-Management){:target="_blank"}, which I highly recommend, your `Directory.Packages.props` file at the root of your repo.

For better results, [IDE0005](https://learn.microsoft.com/en-us/dotnet/fundamentals/code-analysis/style-rules/ide0005){:target="_blank"} (Remove unnecessary using directives) should also be enabled, and unfortunately to enable this rule you need to enable [XML documentation comments](https://learn.microsoft.com/en-us/dotnet/csharp/language-reference/xmldoc/){:target="_blank"} (xmldoc) due to [dotnet/roslyn issue 41640](https://github.com/dotnet/roslyn/issues/41640){:target="_blank"}. This causes many new analyzers to kick in which you may have many violations for, so those would need to be fixed or suppressed. To enable xmldoc, set the `<GenerateDocumentationFile>` property to `true`.

And that's it, ReferenceTrimmer should now run as part of your build!

## How it works

ReferenceTrimmer consists of two parts: an MSBuild task and a Roslyn analyzer.

The task is named `CollectDeclaredReferencesTask` and as you can guess, its job is to gather the declared references. It gathers the list of references passed to the compiler and associates each of them with the `<Reference>`, `<ProjectReference>`, or `<PackageReference>` from which they originate. It also filters out references which are unavoidable such as implicitly defined references from the .NET SDK, as well as packages which contain build logic since that may be the true purpose of that packages as opposed to providing a referenced library.

This information from the task is dumped into a file `_ReferenceTrimmer_DeclaredReferences.json` under the project's intermediate output folder (usually `obj\Debug` or `obj\Release`) and this path is added as a `AdditionalFiles` item to [pass it to the analyzer](https://github.com/dotnet/roslyn/blob/main/docs/analyzers/Using%20Additional%20Files.md#in-a-project-file){:target="_blank"}.

Next, as part of compilation, the analyzer named `ReferenceTrimmerAnalyzer` will call the `GetUsedAssemblyReferences` API as previously mentioned to get the used references and compare them to the compared references provided by the task. Any declared references which are not used will cause a warning to be raised.

The warning code raised will depend on the originating reference type. It will be `RT0001` for `<Reference>` items, `RT0002` for `<ProjectReference>` items, or `RT0003` for `<PackageReference>` items. These are treated like any other compilation warning and so can be suppressed on a per-project basic with `<NoWarn>`. Additionally ReferenceTrimmer can be disabled entirely for a project by setting `$(EnableReferenceTrimmer)` to false.

Note that because the warnings are raised as part of compilation, projects with other language types like C++ or even [NoTargets projects](https://github.com/microsoft/MSBuildSdks/blob/main/src/NoTargets/README.md){:target="_blank"} will not cause warning to be raised nor need to be explicitly excluded from ReferenceTrimmer.

## Future work

Ideas for future improvement include:

* Better identifying `<ProjectReference>` which are required for runtime only. Or at least documenting explicit guidance around how to reference those properly such that the compiler doesn't "see" them but the outputs are still copied.
* Add the ability to exclude specific references from being warned on by adding some metadata to the reference. This would allow for more granular control rather than having to disable an entire rule for an entire project.
* Add support for C++ projects.
* Find and fix more edge-case bugs!

Contributions and bug reports are always welcome on [GitHub](https://github.com/dfederm/ReferenceTrimmer){:target="_blank"}, and I'm hopeful ReferenceTrimmer can be helpful in detangling your repos!
