---
layout: post
title: How does PackageReference work?
categories: [.NET, MSBuild, NuGet]
tags: [.NET, nuget, msbuild]
comments: true
---
[`PackageReference`](https://docs.microsoft.com/en-us/nuget/consume-packages/package-references-in-project-files){:target="_blank"} has replaced [`packages.config`](https://docs.microsoft.com/en-us/nuget/reference/packages-config){:target="_blank"} as the primary mechanism for consuming NuGet packages. For those looking to migrate, there is [documentation](https://docs.microsoft.com/en-us/nuget/consume-packages/migrate-packages-config-to-package-reference){:target="_blank"} available to help you. But how does it actually work under the hood?

Historically with `packages.config` files, NuGet's role was simply to download the exact packages at the exact versions you specified, and then copy the packages into a repository-relative path configured in your [`NuGet.Config`](https://docs.microsoft.com/en-us/nuget/reference/nuget-config-file){:target="_blank"} file, usually `/packages`. Actually consuming the package contents was ultimately up to the consuming projects, however the Visual Studio Package Manager UI would help update the relevant project with various `<Import>`, `<Reference>`, and `<Content>` elements based on convention.

With `PackageReference`, [these conventions](https://docs.microsoft.com/en-us/nuget/create-packages/creating-a-package#from-a-convention-based-working-directory){:target="_blank"} have been effectively codified. I becomes very cumbersome to consume packages these days which do not conform to the conventions. Additionally `PackageReference` adds much needed quality-of-life features, such as automatically pulling in dependencies and unifying package versions.

## Restore

As I hinted earlier, NuGet's job previously was to download packages only, so a `nuget restore` of a packages.config file did that and only that. Now with `PackageReference`, the restore process does not only that but also generates files per-project which describe the contents of each consumed package and is used during the build to dynamically add the equivalents of the previous `<Import>`, `<Reference>`, and `<Content>` elements which were present in projects.

One benefit of these generated files is that the project files are left much cleaner. The project file simply has a `PackageReference`, rather than consuming a bunch of stuff which happens to all be from a path inside that package with lots of duplication.

Another benefit is that the copy of all package contents from the [global package cache](https://docs.microsoft.com/en-us/nuget/Consume-Packages/managing-the-global-packages-and-cache-folders){:target="_blank"} to the repository-relative `/packages` directory is no longer necessary as the generted files can point directly into the global package cache. This can save a lot of disk space and a lot of restore time (at least in a clean repository). Note that the global package cache is `%UserProfile%\.nuget\packages` by default on Windows machines, but can be redirected as desired, for example to the same drive as your code which is ideally an SSD, by setting `%NUGET_PACKAGES%`.

These generated files are output to `$(RestoreOutputPath)`, which by default is `$(MSBuildProjectExtensionsPath)`, which by default is `$(BaseIntermediateOutputPath)`, which by default is `obj\` (Phew). The notable generated files are `project.assets.json`, `<project-file>.nuget.g.props`, and `<project-file>.nuget.g.targets`.

Let's start with the generated props and targets files as they're more straightforward.

## Generated props and targets

These generated props file is imported by this line in [`Microsoft.Common.props`](https://github.com/dotnet/msbuild/blob/main/src/Tasks/Microsoft.Common.props){:target="_blank"} (which is imported by [`Microsoft.NET.Sdk`](https://github.com/dotnet/sdk){:target="_blank"}):

```xml
<Import Project="$(MSBuildProjectExtensionsPath)$(MSBuildProjectFile).*.props" Condition="'$(ImportProjectExtensionProps)' == 'true' and exists('$(MSBuildProjectExtensionsPath)')" />
```

Similarly, the targets file is imported by a similar like in [`Microsoft.Common.targets`](https://github.com/dotnet/msbuild/blob/main/src/Tasks/Microsoft.Common.targets){:target="_blank"}.

The props file always defines a few properties which NuGet uses at build time like `$(ProjectAssetsFile)`, but the interesting part to a consumer is that the `<Import>` elements into packages which used to be directly in the projects are generated to these files, the packages' `build\<package-name>.props` in the `<project-file>.nuget.g.props` and the `build\<package-name>.targets` in the `<project-file>.nuget.g.targets`.

As an example, you'll see a section similar to this in the generated props file for a unit test project using [xUnit](https://xunit.net/){:target="_blank"}:

```xml
  <ImportGroup Condition="'$(ExcludeRestorePackageImports)' != 'true'">
    <Import Project="$(NuGetPackageRoot)xunit.runner.visualstudio\2.4.3\build\netcoreapp2.1\xunit.runner.visualstudio.props" Condition="Exists('$(NuGetPackageRoot)xunit.runner.visualstudio\2.4.3\build\netcoreapp2.1\xunit.runner.visualstudio.props')" />
    <Import Project="$(NuGetPackageRoot)xunit.core\2.4.1\build\xunit.core.props" Condition="Exists('$(NuGetPackageRoot)xunit.core\2.4.1\build\xunit.core.props')" />
    <Import Project="$(NuGetPackageRoot)microsoft.testplatform.testhost\17.1.0\build\netcoreapp2.1\Microsoft.TestPlatform.TestHost.props" Condition="Exists('$(NuGetPackageRoot)microsoft.testplatform.testhost\17.1.0\build\netcoreapp2.1\Microsoft.TestPlatform.TestHost.props')" />
    <Import Project="$(NuGetPackageRoot)microsoft.codecoverage\17.1.0\build\netstandard1.0\Microsoft.CodeCoverage.props" Condition="Exists('$(NuGetPackageRoot)microsoft.codecoverage\17.1.0\build\netstandard1.0\Microsoft.CodeCoverage.props')" />
    <Import Project="$(NuGetPackageRoot)microsoft.net.test.sdk\17.1.0\build\netcoreapp2.1\Microsoft.NET.Test.Sdk.props" Condition="Exists('$(NuGetPackageRoot)microsoft.net.test.sdk\17.1.0\build\netcoreapp2.1\Microsoft.NET.Test.Sdk.props')" />
  </ImportGroup>
```

Note that `$(NuGetPackageRoot)` is the global package cache directory as described earlier and is defined earlier in the same generated props file.

The generated props file also defines properties which point to package root directories for packages which have the [`GeneratePathProperty`](https://docs.microsoft.com/en-us/nuget/consume-packages/package-references-in-project-files#generatepathproperty){:target="_blank"} metadata defined. These properties look like `$(PkgNormalized_Package_Name)` and are mostly used as an escape valve for package which don't properly follow the conventions and using custom build logic in the project file to reach into the package is required.

## Project Assets File

TODO