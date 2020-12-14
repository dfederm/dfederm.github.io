---
layout: post
title: Debugging MSBuild
categories: [.NET, MSBuild]
tags: [.NET, MSBuild, build, debugging]
comments: true
---

At work I live primarily in the build space, specifically [MSBuild](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild){:target="_blank"}-based environments, and a common trend I've noticed is that many developers struggle with MSBuild. The reason for this isn't typically because the build space is "too hard", or at least not much harder than any other kind of programming, but instead because the MSBuild syntax is effectively its own language and debugging execution of that language is not something most developers know how to do. This article attempts to provide various techniques for debugging MSBuild execution.

First, it's important to understand the basics of MSBuild syntax. The official [MSBuild documentation](https://docs.microsoft.com/en-us/visualstudio/msbuild/msbuild-concepts){:target="_blank"} is quite detailed in this regard, so for the rest of this article I'll assume a basic understanding of MSBuild properties, items, and targets.

## The log viewer

MSBuild unfortunately does not have a full-blown debugging experience, in terms of breakpoints and stepping through the MSBuild syntax line-by-line, but instead one has to primarily rely on logging. However, MSBuild has quite verbose logging, as anyone who has enabled diagnostic logging can attest to. Diagnostic logging has much of the required information for understanding what's happening, but it can be near-impossible due to its incredible size and unstructured nature.

Enter the [MSBuild Structured Log Viewer](https://msbuildlog.com/){:target="_blank"}. Binary logging is built-into MSBuild itself, but as it's a binary file you need a special viewer to properly consume it.

The log viewer has a few options on the start page, but the only one with major functionality implications is a [recently added option](https://github.com/KirillOsenkov/MSBuildStructuredLog/commit/113e3e10805fb3ec917a5b41dee929135303c14f){:target="_blank"} to parent all targets directly under project instead of attempting (sometimes badly) to create a tree from the target graph. It now [defaults to being enabled](https://github.com/KirillOsenkov/MSBuildStructuredLog/commit/754a57e07f83fda9cb550cf005e5c5924303bb06){:target="_blank"}, so I also recommend this setting and will be using it throughout this article.

## Producing binary logs

As the binary logger is build-into MSBuild, enabling it is as simple as using command-line option `-binaryLogger`, or `-bl` for short, eg. As with other MSBuild command-line options, it works with the `dotnet` CLI.

When a specific file is not provided, it defaults to dropping an `msbuild.binlog` in the current directory.

Examples:
```bat
REM Produce msbuild.binlog
msbuild -bl
dotnet -bl

REM In a CI environment, you probably want to put the log somewhere specific
msbuild -BinaryLogger:path\to\logs\msbuild.binlog
```

## Basic debugging

To have an example to look at, I'll be using a trivial project structure which can be created with the following commands:

```bat
dotnet new console -o App
dotnet new classlib -o Lib1
dotnet new classlib -o Lib2
dotnet add App\App.csproj reference Lib1\Lib1.csproj Lib2\Lib2.csproj
```

After running `dotnet build App /bl` and opening the resulting `msbuild.binlog`, you should see something like this:

![collapsed structured log example](/assets/structured-log-viewer-collapsed.PNG){: .center }

There is quite a bit of top-level information, including:
* The full command-line. Note that `dotnet build` gets translated to running the .NET Core flavor of MSBuild with specific options.
* Environment variables. Recall that environment various get hoisted as MSBuild properties if the properties are not explicitly assigned, so this information can be very helpful.
* All project evaluations. Note that MSBuild will evaluate a project multiple times if the global properties differ. Also note that evaluation is basically the initial state of the project, before any targets have executed. So this can be helpful for debugging
* The root(s) of the target execution. In this case there is both the Restore target and the default targets. This is because `dotnet build` translates to `msbuild -restore` which does an implicit restore before building. You can disable this by providing `--no-restore` to `dotnet`.

There is a search feature which can help if you know the property, item, target, or file name you're interested in. In addition to just text searching, you can also filter by kind of thing, for example just searching properties.

For a given target, there is another target listed to the right which explains why the target executed. If you hover, you can see specificially whether it was because of `BeforeTargets`, `AfterTargets`, or `DependsOnTargets`. You can also tell whether the target actually executed based on its condition by whether it's dimmed.

A non-obvious trick is that if you double-click on a project or target, it will open the file it's contained in. This can help give you a glance into the logic of the target. You can take this a bit further and right-click on a project and select "Prepocess", which will give you the completely flattened xml for the entire project, exactly like the `-pp` MSBuild switch. The preprocess can be extremely helpful in undertanding the build logic.

As a general guide, you will mostly rely on the target execution log for determine what happened, while the preprocess will help answer why it happened. For example, the target execution log will show that a property was set to some specific value, while the preprocess will show the logic of why it was set to that value.

An interesting detail about the implicit restore you can observe is that there is a global property `MSBuildRestoreSessionId` set. This is because during the restore, packages may not have been downloaded yet and thus any build logic which should be imported from packages may be missing. Setting a global property to an effectively random value forces the restore to be evaluated and execute in a completely separate context than the the default targets. Then after restore when the default targets execute, it requires a new evaluation and imports from packages will actually be available. I'll go into more details about how restore and how `PackageReference` works in a future post.

## Debugging example: `@(Content)` item copying

In my opinion, the best way to understand how to debug MSBuild is to actually dive into the logs and see if we can use them to answer some questions. In this example, we'll dig into the logs to understand how content files in referenced projects propagate to a project's output folder.

First, create some dummy content files:

```bat
echo Foo > Lib1\Foo.txt
echo Bar > Lib2\Bar.txt
echo Baz > App\Baz.txt
```

Next, configure the content to be copied to the output directories.

```xml
  <!-- In Lib1\Lib1.csproj -->
  <ItemGroup>
    <Content Include="Foo.txt">
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    </Content>
  </ItemGroup>

  <!-- In Lib2\Lib2.csproj -->
  <ItemGroup>
    <Content Include="Bar.txt">
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    </Content>
  </ItemGroup>

  <!-- In App\App.csproj -->
  <ItemGroup>
    <Content Include="Baz.txt">
      <CopyToOutputDirectory>PreserveNewest</CopyToOutputDirectory>
    </Content>
  </ItemGroup>
```

Now if we build using `dotnet build App`, we'll see the files: `App\bin\Debug\net5.0\Foo.txt`, `App\bin\Debug\net5.0\Bar.txt`, and `App\bin\Debug\net5.0\Baz.txt`. So how did they get there?

First, search for "App\bin\Debug\net5.0\Foo.txt":

![Search results for "App\bin\Debug\net5.0\Foo.txt"](/assets/structured-log-viewer-search-1.PNG){: .center }

The results at the end look related to incremental clean, so the one we want to look as is under the `_CopyOutOfDateSourceItemsToOutputDirectory` target, especially since it says "Copying file" in the log message.

When navigating to that result, in fact all of the content the files we were interested in (and one we weren't) are here.

```
Copying file from "C:\Users\David\Code\tmp\msbuild-debugging\App\Baz.txt" to "C:\Users\David\Code\tmp\msbuild-debugging\App\bin\Debug\net5.0\Baz.txt".
Copying file from "C:\Users\David\Code\tmp\msbuild-debugging\Lib1\Foo.txt" to "C:\Users\David\Code\tmp\msbuild-debugging\App\bin\Debug\net5.0\Foo.txt".
Copying file from "C:\Users\David\Code\tmp\msbuild-debugging\App\obj\Debug\net5.0\apphost.exe" to "C:\Users\David\Code\tmp\msbuild-debugging\App\bin\Debug\net5.0\App.exe".
Copying file from "C:\Users\David\Code\tmp\msbuild-debugging\Lib2\Bar.txt" to "C:\Users\David\Code\tmp\msbuild-debugging\App\bin\Debug\net5.0\Bar.txt".
```

Upon double-clicking the target, we see the definition for `_CopyOutOfDateSourceItemsToOutputDirectory`:

```xml
  <Target
      Name="_CopyOutOfDateSourceItemsToOutputDirectory"
      Condition=" '@(_SourceItemsToCopyToOutputDirectory)' != '' "
      Inputs="@(_SourceItemsToCopyToOutputDirectory)"
      Outputs="@(_SourceItemsToCopyToOutputDirectory->'$(OutDir)%(TargetPath)')">

    <!--
        Not using SkipUnchangedFiles="true" because the application may want to change
        one of these files and not have an incremental build replace it.
        -->
    <Copy
        SourceFiles = "@(_SourceItemsToCopyToOutputDirectory)"
        DestinationFiles = "@(_SourceItemsToCopyToOutputDirectory->'$(OutDir)%(TargetPath)')"
        OverwriteReadOnlyFiles="$(OverwriteReadOnlyFiles)"
        Retries="$(CopyRetryCount)"
        RetryDelayMilliseconds="$(CopyRetryDelayMilliseconds)"
        UseHardlinksIfPossible="$(CreateHardLinksForAdditionalFilesIfPossible)"
        UseSymboliclinksIfPossible="$(CreateSymbolicLinksForAdditionalFilesIfPossible)"
            >

      <Output TaskParameter="DestinationFiles" ItemName="FileWrites"/>

    </Copy>

  </Target>
```

So the [`Copy` task](https://docs.microsoft.com/en-us/visualstudio/msbuild/copy-task){:target="_blank"} is called with `@(_SourceItemsToCopyToOutputDirectory)` items as the source, and copied to the `$(OutDir)` using their `%(TargetPath)` metadata.

Side note: by convention properties, items, and targets which are prefixed by an underscore should be considered "private". MSBuild doesn't have any true notion of scope or access modifiers, but it's an indication of an implementation detail in the build logic and may change behavior or even be removed in the future. Because of this, if you are writing your own custom build logic, you should not depend on "private" entities and instead look for the appropriate extension points.

We can look up the value of `$(OutDir)` in a pretty striaghtforward way by looking at the properties for the project. In this example, we see `OutDir = bin\Debug\net5.0\`. But how did that value come about? We can look ths up in the preprocess. After right-clicking the project, selecting preprocess, and doing a `ctrl+f` and looking for "`<OutDir`", we see this block of xml:

```xml
    <!-- Required for enabling Team Build for packaging app package-generating projects -->
    <OutDirWasSpecified Condition=" '$(OutDir)'!='' and '$(OutDirWasSpecified)'=='' ">true</OutDirWasSpecified>

    <OutDir Condition=" '$(OutDir)' == '' ">$(OutputPath)</OutDir>
    <!-- Example, bin\Debug\ -->
    <!-- Ensure OutDir has a trailing slash, so it can be concatenated -->
    <OutDir Condition="'$(OutDir)' != '' and !HasTrailingSlash('$(OutDir)')">$(OutDir)\</OutDir>
    <ProjectName Condition=" '$(ProjectName)' == '' ">$(MSBuildProjectName)</ProjectName>
    <!-- Example, MyProject -->

    <!-- For projects that generate app packages or ones that want a per-project output directory, update OutDir to include the project name -->
    <OutDir Condition="'$(OutDir)' != '' and '$(OutDirWasSpecified)' == 'true' and (('$(WindowsAppContainer)' == 'true' and '$(GenerateProjectSpecificOutputFolder)' != 'false') or '$(GenerateProjectSpecificOutputFolder)' == 'true')">$(OutDir)$(ProjectName)\</OutDir>
```

Because `$(OutDir)` wasn't specified before this, `$(OutDirWasSpecified)` remains unset and so effectively `$(OutDir)` is simply just `$(OutputPath)` with a possible trailing slash appended if needed.

If we then search for "`<OutputPath`", we'll find quite a few results.

```xml
    <BaseOutputPath Condition="'$(BaseOutputPath)' == ''">bin\</BaseOutputPath>
    <BaseOutputPath Condition="!HasTrailingSlash('$(BaseOutputPath)')">$(BaseOutputPath)\</BaseOutputPath>
    <OutputPath Condition="'$(OutputPath)' == '' and '$(PlatformName)' == 'AnyCPU'">$(BaseOutputPath)$(Configuration)\</OutputPath>
    <OutputPath Condition="'$(OutputPath)' == '' and '$(PlatformName)' != 'AnyCPU'">$(BaseOutputPath)$(PlatformName)\$(Configuration)\</OutputPath>
    <OutputPath Condition="!HasTrailingSlash('$(OutputPath)')">$(OutputPath)\</OutputPath>

<!-- ... -->

  <PropertyGroup Condition="'$(AppendTargetFrameworkToOutputPath)' == 'true' and '$(TargetFramework)' != '' and '$(_UnsupportedTargetFrameworkError)' != 'true'">
    <IntermediateOutputPath>$(IntermediateOutputPath)$(TargetFramework.ToLowerInvariant())\</IntermediateOutputPath>
    <OutputPath>$(OutputPath)$(TargetFramework.ToLowerInvariant())\</OutputPath>
  </PropertyGroup>

<!-- ... -->

  <PropertyGroup Condition="'$(AppendRuntimeIdentifierToOutputPath)' == 'true' and '$(RuntimeIdentifier)' != '' and '$(_UsingDefaultRuntimeIdentifier)' != 'true'">
    <IntermediateOutputPath>$(IntermediateOutputPath)$(RuntimeIdentifier)\</IntermediateOutputPath>
    <OutputPath>$(OutputPath)$(RuntimeIdentifier)\</OutputPath>
  </PropertyGroup>

<!-- ... -->

    <OutputPath Condition="'$(OutputPath)' != '' and !HasTrailingSlash('$(OutputPath)')">$(OutputPath)\</OutputPath>
    <OutputPath Condition=" '$(Platform)'=='' and '$(Configuration)'=='' and '$(OutputPath)'=='' ">bin\Debug\</OutputPath>
```

`$(OutputPath)` is set many times, but it's mostly just appended to in order to avoid collisions when building with various dimensions. It's basically just `bin\<platform-if-not-anycpu>\<configuration>\<target-framework>\<rid-if-set>`, with various properties to control its behavior if desired.

It's important here that all places where `$(OutDir)` is set are below all places where `$(OutputPath)` is set, so we don't have to worry about ordering issues for these two properties in this case.

Now we understand the `$(OutDir)` part of the copy destination, so next we should understand how the `@(_SourceItemsToCopyToOutputDirectory)` item is created. When searching we see:

![Search results for "_SourceItemsToCopyToOutputDirectory"](/assets/structured-log-viewer-search-2.PNG){: .center }

We have results from all three projects, which is expected since App depends on Lib1 and Lib2, so those other projects build first and would perform this logic themselves. How exactly App causes Lib1 and Lib2 to build first I will leave as an exercize to the reader.

To continue answering our original question, we want to look at the result for the App project, which leads us to the `GetCopyToOutputDirectoryItems` target, which is defined as:

```xml
  <Target
      Name="GetCopyToOutputDirectoryItems"
      Returns="@(AllItemsFullPathWithTargetPath)"
      KeepDuplicateOutputs=" '$(MSBuildDisableGetCopyToOutputDirectoryItemsOptimization)' == '' "
      DependsOnTargets="$(GetCopyToOutputDirectoryItemsDependsOn)">

    <!-- ... -->
    <CallTarget Targets="_GetCopyToOutputDirectoryItemsFromTransitiveProjectReferences">
      <Output TaskParameter="TargetOutputs" ItemName="_TransitiveItemsToCopyToOutputDirectory" />
    </CallTarget>

    <CallTarget Targets="_GetCopyToOutputDirectoryItemsFromThisProject">
      <Output TaskParameter="TargetOutputs" ItemName="_ThisProjectItemsToCopyToOutputDirectory" />
    </CallTarget>

    <ItemGroup Condition="'$(CopyConflictingTransitiveContent)' == 'false'">
      <_TransitiveItemsToCopyToOutputDirectory Remove="@(_ThisProjectItemsToCopyToOutputDirectory)" MatchOnMetadata="TargetPath" MatchOnMetadataOptions="PathLike" />
    </ItemGroup>

    <ItemGroup>
      <_TransitiveItemsToCopyToOutputDirectoryAlways               KeepDuplicates=" '$(_GCTODIKeepDuplicates)' != 'false' " KeepMetadata="$(_GCTODIKeepMetadata)" Include="@(_TransitiveItemsToCopyToOutputDirectory->'%(FullPath)')" Condition="'%(_TransitiveItemsToCopyToOutputDirectory.CopyToOutputDirectory)'=='Always'"/>
      <_TransitiveItemsToCopyToOutputDirectoryPreserveNewest       KeepDuplicates=" '$(_GCTODIKeepDuplicates)' != 'false' " KeepMetadata="$(_GCTODIKeepMetadata)" Include="@(_TransitiveItemsToCopyToOutputDirectory->'%(FullPath)')" Condition="'%(_TransitiveItemsToCopyToOutputDirectory.CopyToOutputDirectory)'=='PreserveNewest'"/>

      <_ThisProjectItemsToCopyToOutputDirectoryAlways              KeepDuplicates=" '$(_GCTODIKeepDuplicates)' != 'false' " KeepMetadata="$(_GCTODIKeepMetadata)" Include="@(_ThisProjectItemsToCopyToOutputDirectory->'%(FullPath)')" Condition="'%(_ThisProjectItemsToCopyToOutputDirectory.CopyToOutputDirectory)'=='Always'"/>
      <_ThisProjectItemsToCopyToOutputDirectoryPreserveNewest      KeepDuplicates=" '$(_GCTODIKeepDuplicates)' != 'false' " KeepMetadata="$(_GCTODIKeepMetadata)" Include="@(_ThisProjectItemsToCopyToOutputDirectory->'%(FullPath)')" Condition="'%(_ThisProjectItemsToCopyToOutputDirectory.CopyToOutputDirectory)'=='PreserveNewest'"/>

      <!-- Append the items from this project last so that they will be copied last. -->
      <_SourceItemsToCopyToOutputDirectoryAlways                   Include="@(_TransitiveItemsToCopyToOutputDirectoryAlways);@(_ThisProjectItemsToCopyToOutputDirectoryAlways)"/>
      <_SourceItemsToCopyToOutputDirectory                         Include="@(_TransitiveItemsToCopyToOutputDirectoryPreserveNewest);@(_ThisProjectItemsToCopyToOutputDirectoryPreserveNewest)"/>

      <!-- ... -->
    </ItemGroup>

  </Target>
```

And in the execution logs we see:

![The GetCopyToOutputDirectoryItems target](/assets/structured-log-viewer-gctodi.PNG){: .center }

Using the combination of the definition and the execution log, we see that this target ends up calling 2 other targets, `_GetCopyToOutputDirectoryItemsFromTransitiveProjectReferences` and `_GetCopyToOutputDirectoryItemsFromThisProject`, and aggregates and filters the resulting items into `@(_SourceItemsToCopyToOutputDirectoryAlways)` and `@(_SourceItemsToCopyToOutputDirectory)` items.

Based on the names we can guess what's going on already. One target gathers items from project references while the other gathers items from this project. Then they're separated into an "always" and a "preserve newest" item. We'll focus on `@(_SourceItemsToCopyToOutputDirectory)` since that's what we are tracing, but the "always" variant works very similarly except the file copies are unconditional instead of dependant on file timestamps.

Let's look at `_GetCopyToOutputDirectoryItemsFromThisProject` first since it's from this project and will likely be easier to follow. After searching abd finding the instance under the App project, we find that it's defined as:

```xml
  <Target
      Name="_GetCopyToOutputDirectoryItemsFromThisProject"
      DependsOnTargets="AssignTargetPaths;_PopulateCommonStateForGetCopyToOutputDirectoryItems"
      Returns="@(_ThisProjectItemsToCopyToOutputDirectory)">

    <ItemGroup>
      <_ThisProjectItemsToCopyToOutputDirectory       KeepMetadata="$(_GCTODIKeepMetadata)" Include="@(ContentWithTargetPath->'%(FullPath)')" Condition="'%(ContentWithTargetPath.CopyToOutputDirectory)'=='Always' AND '%(ContentWithTargetPath.MSBuildSourceProjectFile)'==''"/>
      <_ThisProjectItemsToCopyToOutputDirectory       KeepMetadata="$(_GCTODIKeepMetadata)" Include="@(ContentWithTargetPath->'%(FullPath)')" Condition="'%(ContentWithTargetPath.CopyToOutputDirectory)'=='PreserveNewest' AND '%(ContentWithTargetPath.MSBuildSourceProjectFile)'==''"/>
    </ItemGroup>

    <!-- ... -->

  </Target>
```

`_GetCopyToOutputDirectoryItemsFromThisProject` simply aggregates `@(ContentWithTargetPath)`, `@(_NoneWithTargetPath)`, `@(EmbeddedResource)`, and for some reason `@(Compile)` items which have either `%(CopyToOutputDirectory)` as either "Always" or "PreserveNewest".

Then if we look up `@(ContentWithTargetPath)` items, we'll find the `AssignTargetPaths` target:

```xml
  <Target
      Name="AssignTargetPaths"
      DependsOnTargets="$(AssignTargetPathsDependsOn)">

    <!-- ... -->

    <AssignTargetPath Files="@(Content)" RootFolder="$(MSBuildProjectDirectory)">
      <Output TaskParameter="AssignedFiles" ItemName="ContentWithTargetPath" />
    </AssignTargetPath>

    <!-- ... -->

  </Target>
```

The [AssignTargetPath task](https://github.com/dotnet/msbuild/blob/master/src/Tasks/AssignTargetPath.cs){:target="_blank"} adds the `%(TargetPath)` metadata for items, which is either based on the `%(Link)` metadata if provided, or the relative path of the file from the project directory if it's under the project directory, or simply the filename otherwise.

Finally, we now see how the `@(Content)` item for the current project (`Baz.txt` in our example) gets copied to the output folder.

But we still need to understand the content from the referenced projects, `Foo.txt` and `Bar.txt`. Upon searching for `_GetCopyToOutputDirectoryItemsFromTransitiveProjectReferences`, finding the result in the App project, and looking at the definition, we see:

```xml

  <PropertyGroup>
    <!-- ... -->
    <_RecursiveTargetForContentCopying>GetCopyToOutputDirectoryItems</_RecursiveTargetForContentCopying>
    <!-- ... -->
  </PropertyGroup>

  <!-- ... -->

  <Target
    Name="_GetCopyToOutputDirectoryItemsFromTransitiveProjectReferences"
    DependsOnTargets="_PopulateCommonStateForGetCopyToOutputDirectoryItems;_AddOutputPathToGlobalPropertiesToRemove"
    Returns="@(_TransitiveItemsToCopyToOutputDirectory)">

    <!-- Get items from child projects first. -->
    <MSBuild
        Projects="@(_MSBuildProjectReferenceExistent)"
        Targets="$(_RecursiveTargetForContentCopying)"
        BuildInParallel="$(BuildInParallel)"
        Properties="%(_MSBuildProjectReferenceExistent.SetConfiguration); %(_MSBuildProjectReferenceExistent.SetPlatform); %(_MSBuildProjectReferenceExistent.SetTargetFramework)"
        Condition="'@(_MSBuildProjectReferenceExistent)' != '' and '$(_GetChildProjectCopyToOutputDirectoryItems)' == 'true' and '%(_MSBuildProjectReferenceExistent.Private)' != 'false' and '$(UseCommonOutputDirectory)' != 'true'"
        ContinueOnError="$(ContinueOnError)"
        SkipNonexistentTargets="true"
        RemoveProperties="%(_MSBuildProjectReferenceExistent.GlobalPropertiesToRemove)$(_GlobalPropertiesToRemoveFromProjectReferences)">

      <Output TaskParameter="TargetOutputs" ItemName="_AllChildProjectItemsWithTargetPath"/>

    </MSBuild>

    <ItemGroup>
      <_TransitiveItemsToCopyToOutputDirectory   KeepDuplicates=" '$(_GCTODIKeepDuplicates)' != 'false' " KeepMetadata="$(_GCTODIKeepMetadata)" Include="@(_AllChildProjectItemsWithTargetPath->'%(FullPath)')" Condition="'%(_AllChildProjectItemsWithTargetPath.CopyToOutputDirectory)'=='Always'"/>
      <_TransitiveItemsToCopyToOutputDirectory   KeepDuplicates=" '$(_GCTODIKeepDuplicates)' != 'false' " KeepMetadata="$(_GCTODIKeepMetadata)" Include="@(_AllChildProjectItemsWithTargetPath->'%(FullPath)')" Condition="'%(_AllChildProjectItemsWithTargetPath.CopyToOutputDirectory)'=='PreserveNewest'"/>
    </ItemGroup>

    <!-- ... -->

  </Target>
```

So `_GetCopyToOutputDirectoryItemsFromTransitiveProjectReferences` simply calls the `GetCopyToOutputDirectoryItems` target on all project references.

As we've already seen, `GetCopyToOutputDirectoryItems` gathers the "copy items" (`@(Content)`, `@(None)`, etc. with specific `CopyToOutputDirectory` values) from a project and its dependencies recursively, so we finally fully understand how `Foo.txt` and `Bar.txt` were copied!

Better yet, we now know how to debug MSBuild!