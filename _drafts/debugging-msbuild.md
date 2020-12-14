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

TODO