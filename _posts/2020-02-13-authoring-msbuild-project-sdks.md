---
layout: post
title: Authoring MSBuild Project SDKs
categories: [.NET, MSBuild, NuGet]
tags: [.NET, nuget, msbuild, sdk]
comments: true
---

You may have seen the term "SDK-style projects" referring to MSBuild projects which have an `Sdk` attribute on the root `<Project>` element, and generally are associated with .NET Core projects. This article explains how they work and when and how you should author your own.

## What is it?
First, what exact is an MSBuild project SDK? They're a new mechanism introduced in MSBuild 15 (Visual Studio 2017) which simplifies how MSBuild logic is injected into projects. Historically, projects would contain an `<Import>` to a common props file at the top and a common targets file specific to their project type. For example, C# projects imported `$(MSBuildExtensionsPath)\$(MSBuildToolsVersion)\Microsoft.Common.props` at the top and `$(MSBuildToolsPath)\Microsoft.CSharp.targets` at the bottom. This "props at the top, targets at the bottom" was such a common pattern for basically every project type, and so easy to get wrong for MSBuild novices, that the concept of `Sdk` was introduced.

## How does it work?
The documentation on [docs.microsoft.com](https://docs.microsoft.com/en-us/visualstudio/msbuild/how-to-use-project-sdk#reference-a-project-sdk){:target="_blank"} for MSBuild project SDKs is excellent, and I highly recommend using it as your primary reference for understanding how to use them.

The gist however is that when using either the `Sdk` attribute on the root [`<Project>` element](https://docs.microsoft.com/en-us/visualstudio/msbuild/project-element-msbuild){:target="_blank"} (eg: `<Project Sdk="...">`), or the [`<Sdk>` element](https://docs.microsoft.com/en-us/visualstudio/msbuild/sdk-element-msbuild){:target="_blank"} directly, the associated props and targets files will implicitly be imported at the top and bottom of the file they're used in. This is usually a project file but not necessarily.

So when you see syntax like `<Project Sdk="Microsoft.NET.Sdk">`, it's the same (even internally within MSBuild) as:

```xml
<Project>
  <Import Project="{Path to Microsoft.NET.Sdk}\Sdk\Sdk.props" />

  <!-- The rest of the project file -->

  <Import Project="{Path to Microsoft.NET.Sdk}\Sdk\Sdk.targets" />
</Project>
```

Similarly, when you see syntax like:

```xml
<Project>
  <!-- Content above the <Sdk> element -->
  <Sdk Name="MyCustomSdk" />
  <!-- Content below the <Sdk> element -->
</Project>
```

It's equivalent to:
```xml
<Project>
  <Import Project="{Path to MyCustomSdk}\Sdk\Sdk.props" />
  <!-- Content above the <Sdk> element -->
  <!-- Content below the <Sdk> element -->
  <Import Project="{Path to MyCustomSdk}\Sdk\Sdk.target" />
</Project>
```

Note how the location of the `<Sdk>` element does not matter; the props and targets are still at the top and bottom.

## Project SDK resolution
MSBuild uses a plugin model for resolving SDKs and two SDK resolvers ship with Visual Studio. Note that although it is technically a plugin model, it is not recommended to write your own SDK resolver as that would require anyone using it to install something custom into their Visual Studio installation which is no longer the recommended approach for extending your build. For referene though, the SDK resolvers can be found adjacent to MSBuild.exe under a folder called `SdkResolvers`

The two SDK resolvers that ship with Visual Studio and dotnet CLI are: `Microsoft.DotNet.MSBuildSdkResolver` and `Microsoft.Build.NuGetSdkResolver`.

`Microsoft.DotNet.MSBuildSdkResolver` is what resolves "built in" SDKs like `Microsoft.NET.Sdk` and `Microsoft.NET.Sdk.Web`. It first looks for your dotnet CLI (eg `C:\Program Files\dotnet\dotnet.exe`), then resolves the active dotnet SDK based on your [global.json](https://docs.microsoft.com/en-us/dotnet/core/tools/global-json#sdk){:target="_blank"} if one exists or the latest one installed (eg `C:\Program Files\dotnet\sdk\3.1.101`). Once it finds the dotnet SDK, it looks for the SDKs in the `Sdks` directory. So `Microsoft.NET.Sdk` may be located at a path similar to `C:\Program Files\dotnet\sdk\3.1.101\Sdks\Microsoft.NET.Sdk`.

`Microsoft.Build.NuGetSdkResolver`, or the "NuGet SDK Resolver", is the where MSBuild project SDKs become extensible. This SDK resolver will pull an SDK as a NuGet package from any configured NuGet feed. You can write your own MSBuild project SDK NuGet package for others to use.

Note that the NuGet SDK Resolver does require a version to be specified, either in the MSBuild XML for the SDK, or in `global.json` in the `msbuild-sdks` object.

It is important to realize that SDK resolution happens at MSBuild evaluation time. This means that the NuGet SDK Resolver will potentially download package while MSBuild is evaluating a project, which happens before a restore even. One way to think of it would even be a "restore before the restore". Because of this however, it's important that MSBuild project SDK NuGet packages are very lightweight in size, and that they are not overused.

## When you should create one
NuGet packages already have the ability to extend, augment, and customize your build. You can just drop a `/build/<package name>.props` and/or `/build/<package name>.targets` file in your NuGet package, and anyone with a `<PackageReference>` for your package get those automatically imported. For example the `Microsoft.Net.Compilers` package does this to completely override the default compiler targets and tasks with one from the package, taking over the compilation process.

And in general when you want to extend the build, you *should* use that mechanism for doing so. However, it is recommended to write an MSBuild project SDK when either a) you're defining your own completely new project type, or b) you're extending restore for projects.

### Defining your own project type
If the core behavior of the project doesn't fit into any of the existing SDKs like buildfing a C# project or building a web application, it may be a good scenario for writing your own MSBuild project SDK. In this scenario, it's generally consumed by declaring it in the project file's `<Project>` element.

Some examples of defining a custom project type are [Microsoft.Build.Traversal](https://github.com/microsoft/MSBuildSdks/tree/master/src/Traversal){:target="_blank"} and [Microsoft.Build.NoTargets](https://github.com/microsoft/MSBuildSdks/tree/master/src/NoTargets){:target="_blank"}. The former is intended for projects which simply allow MSBuild to discover other projects but have no build logic of their own, and the latter is for project which perform a simple task instead of compilation, like copying files around.

### Extending the restore
If an MSBuild project SDK needs to extend the behavior of restore, usually it's used as a standalone `<Sdk>` element, since the project as a whole will likely have the `<Project>` element's SDK attribute aligned with its project type. This isn't always the case though as the `<Project Sdk="...">` syntax does support a semicolon-delimeted list of SDKs, however this is atypical and for maintainability and understandability generally frowned upon. Additionally, you'll typically want to extend restore for all projects rather than just one, so it usually makes the most sense to add the SDK in your [Directory.Build.props or Directory.Build.targets](https://docs.microsoft.com/en-us/visualstudio/msbuild/customize-your-build#directorybuildprops-and-directorybuildtargets){:target="_blank"} files.

An example of extending the restore process is [Microsoft.Build.CentralPackageVersions](https://github.com/microsoft/MSBuildSdks/tree/master/src/CentralPackageVersions){:target="_blank"}.

To elaborate on why you need to use an MSBuild project SDK to extend the restore process, you need to understand how restore works a bit. Fully explaining restore will need to be a separate topic completely, but at a high level restore will gather all the `<PackageReference>` items, download and inspect the package, and generate a `*.nuget.g.props`, `*.nuget.g.targets`, and a `project.assets.json` file. The props and targets files import the props or targets files inside those packages by convention. `<PackageReference>` items are not used at all during the build at all, the generated files are. Thus, any `<PackageReference>` items inside of props or targets files from packages are ignored and do not affect the build. Furthermore, any updates to `<PackageReference>` items or changes in Restore behavior from a packages' props and targets would not behave as expected.

One caveat to that is if you restore multiple times in a row, as then the initial `<PackageReference>` items cause the generated props and targets files to be generated and contain `<PackageReference>` items which can then be discovered in a second restore, and this can go on recursively. However, Visual Studio and the dotnet CLI, as well as most CI systems, do not expect to need to restore multiple times for a build to work correctly. I only mention this though since anecdotally I've seen it cause many "works on my machine" moments since developers work in dirty enlistments which have previously been restored.

## Package structure
Finally, after all that context, I will explain how to actually author MSBuild project SDK NuGet packages. It's actually quite simple; all you need is an `Sdk\Sdk.props` and `Sdk\Sdk.targets` file in the package.

Unlike NuGet package with props and targets, both the props and targets files are required. If you only need one, you can simply have `<Project />` as the content of the other, but it must exist and must be a valid MSBuild project.

It's important to note that MSBuild project SDKs do not inherit the behavior of `<PackageReference>` items. For example, assemblies under `lib\{tfm}` aren't added as references. The only behavior is that `Sdk\Sdk.props` and `Sdk\Sdk.targets` are imported at the top and bottom. Because of this, I'd recommend that if you need to add references to create a separate package which your MSBuild project SDK adds a `<PackageReference>` item for.

The [MSBuildSdks repo on GitHub](https://github.com/microsoft/MSBuildSdks){:target="_blank"} has a bunch of great examples of well-written MSBuild project SDKs and define a few useful patterns which may be useful to other SDK authors. For example, I'd recommend adding a `<Using*Sdk>` property to help identify whether a particular SDK is in use. Microsoft.Build.NoTargets sets `UsingMicrosoftNoTargetsSdk` and the built-in Microsoft.Net.Sdk sets `UsingMicrosoftNETSdk`.